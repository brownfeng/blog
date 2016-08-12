---
title: RunLoop学习总结
date: 2016-07-26 21:17:16
tags:
 - runloop
 - iOS
---

通过前面几篇文章可以知道RunLoop实际上是一个事件处理的循环.只要一个线程启动了RunLoop,在它没有收到事件时候,它就会使得线程休眠,如果有事件,就调用相应的事件处理函数.

### RunLoop与线程的关系
RunLoop与线程是一一对应的.每个新建的线程(MovedainThread除外,主线程会默认启动)在默认状态下,它的RunLoop是没有开启的,必须手动调用`CurrentRunLoop`等方法才会开启RunLoop.通常iOS开发中,接触到的API分别是Cocoa的NSRunLoop和CoreFoundation的CFRunLoop.

### RunLoop的组成

RunLoop每次只能运行在一个mode中,它与mode是一对多的关系.RunLoopMode中包括不同类型的事件源,主要分成两大类:TimerSource, 各种异步输入InputSource.与此同时,RunLoop会在不同的声明周期给外界发送通知,告知观察者自己的状态.因此可以注册RunLoopObserver来观察RunLoop的状态.他们之间的关系如下:

* 一个runLoop包含多个Mode,每个Mode内有多个个Source/Timer/Observer(底层会在RunLoopMode中用Array存储)
* 每次启动RunLoop,需要指定一个Mode(`CFRunLoopRunSpecific`函数),如果需要切换Mode,只能退出RunLoop，再重新调用`CFRunLoopRunSpecific`,重新指定Mode
* Source/Timer/Observer统称为mode item,每个item可以同时加入多个mode,但一个item被重复加入同一个mode无效
* 如果一个mode中一个item都没有,RunLoop就会立即退出(AFNetworking2.x)

#### 1. RunLoopMode

RunLoop会运行在某个特定的mode下,称为RunLoopMode,具体mode可以使用系统的NSDefaultMode或者NSTrackingMode,也可以自己定义mode.如果想让事件在任何一个mode中处理,可以把事件源加入到CommonMode中.

#### 2. TimerSource

TimerSource在Cocoa框架中,一般是NSTimer.在使用它时,它会向RunLoop发送消息,RunLoop会根据Timer的类型去单次或者多次执行回调函数.在开始启动Timer时候,需要手动设定具体运行在哪一个RunLoopMode,默认情况下会运行在NSDefaultMode.

通过RunLoop的源码发现,TimerSource的触发实际是通过Source1(MachPort)唤醒RunLoop,然后调用回调方法的.

#### 3. 异步输入源InputSource

异步事件其实会包括的类型:
* source0: 非port事件,可以手动触发(内部只有一个callback函数指针)
* source1: 基于MachPort的Source事件,大多是内核发出的,需要调用`mach_msg`方法

#### 4. NSObject的selector源
由NSObject 的 performSelector:wait:...等产生,下文会细讲.

#### 5. RunLoopObserver
RunLoop的事件源中,Timer是同步事件,另外的是异步事件Source0,Source1,selector. 而RunLoopObserver可以让你在某个特定的时期处理一些事情,比如在RunLoop马上进入休眠状态时,或者在刚刚被唤醒时等等.

具体的RunLoop可以被观察的状态有:

* RunLoop进入时候(kCFRunLoopEntry)
* RunLoop将要处理Timer时(kCFRunLoopBeforeTimers)
* RunLoop将要处理Source0时(kCFRunLoopBeforeSources)
* RunLoop将要进入睡眠的时候(kCFRunLoopBeforeWaiting)
* RunLoop将要被唤醒的时(kCFRunLoopAfterWaiting)
* RunLoop停止的时候(kCFRunLoopExit)


你可以在RunLoop中观察一个或者多个状态.观察者也可以设置是否只观察一次或者重复多次观察,如果只观察一次,那么调用回调函数以后,观察者就自动被移除了.

### RunLoop具体循环逻辑
RunLoop源码中关键的函数有以下几个:

```
CFRunLoopRunSpecific(xxxx)//入口&出口
CFRunLoopRun(xxxx)//循环
```

具体的执行过程如下:

```
1 通知观察者runLoop已经启动(DoObserver)
2 记录RunLoop启动事件,并设定RunLoop运行的超时条件

**** RunLoop正式进入循环 *****

3 通知观察者任何即将要开始的定时器(DoObserver)
4 通知观察着任何即将启动的非基于端口的源(DoObserver)
5 调用RunLoopMode的blocks链中的block方法(DoBlocks)
6 启动任何准备好的非基于端口的source0源(DoSource0)
7 调用RunLoopMode的blocks链中的block方法(DoBlocks)
8 检查MachPort端口是否有消息要处理，如果有立即进入步骤12
9 如果没有消息处理,通知观察者线程进入休眠(DoObserver)

**** RunLoop休眠: zzz...****

10 将线程至于休眠直到任意下面的事件发生(调用`mach_msg`使RunLoop休眠)
	A.某一时间到达基于端口的源
	B.定时器启动
	C.runLoop设置的时间已经超过
	D.runLoop被显示唤醒
11 通知观察者线程将被唤醒(DoObserver)
12 处理未处理的事件(有dispatchPort显示是哪个port的事件需要处理)
	A.如果livePort为NULL,啥都不做(会跳出循环)
	B.如果livePort是wakeUpPort,说明RunLoop运行超时(会跳出循环)
	C.如果livePort是timerPort,说明timerSource启动(DoTimers),进入步骤3
	D.如果livePort是dispatchPort,说明系统的libDispatch向主线程送消息,
	  会调dispatch_async(dispatch_get_main_queue(),block)
	E.如果livePort是其他port,通过mach_msg取出回调,处理事件(DoSource1),
	  进入步骤3
	
**** RunLoop退出循环 ****

13 通知观察者runLoop结束
```

从RunLoop的底层源码可以看出,RunLoop中的各种DoXXXX函数与最终的callout系统调用关系如下:

* DoObserver
`__CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__()`
* DoBlocks 
`__CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__()`
* DoSource0 `__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__()`
* DoTimers `__CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__()`
* dispatchPort `__CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__()`
* DoSource1 `__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__()`

>注意,以上callout调用,对于传入RunLoopMode中的Timers,Source0s,Blocks都是数组,因此对数组内的每个满足条件的成员都会进行callout调用,而source1和dispatchPort由于唤醒时每次只能执行一个.

>repeat Timers执行的时间不一定是完全准确的,如果错过某次执行,只能等到下次RunLoop循环时候再执行了


#### 何时使用RunLoop

自己配置并启动它，你不需要再任何情况下都去启动一个线程的runLoop。runLoop在你
要和线程有更多的交互时才需要，比如以下情况：

* 使用端口或者自定义输入源来和其他线程通信；
* 使用线程的NSTimer (可以用dispatch_timer代替)
* Cocoa中使用任何performSelector
* 让线程周期性干活(AFNetworking2.x)


### CFRunLoop相关内容
#### CoreFoundation中RunLoop的接口

一般使用的是CoreFoundation中的CFRunLoop,因此这里主要总结CFRunLoop的对外接口.RunLoop中对外主要有以下几个接口类:

* CFRunLoopRef
* CFRunLoopModeRef (并未对外暴露)
* CFRunLoopSourceRef: 异步的source0和source1事件源
	* source0: 结构体中只有perform回调函数,它并不能主动触发事件,使用时,需要手动触发(`CFRunLoopSourceSignal()`).CFRunLoopSourceSignal将这个source标记为待处理，后调用CFRunLoopWakeUp来唤醒runLoop，让其处理这个事件(也就是source0无法唤醒RunLoop)
    * source1: 结构体中有mach_port和invoke回调函数,通常是内核和其他进程给该RunLoop发送的消息,从源码可以看出source1,是可以自动唤醒RunLoop的,因为它是通过port方式发送事件的.
* CFRunLoopTimerRef: 同步的TimerSource, 在Cocoa中是NSTimer.它的底层结构体中包含触发时间和回调，当其加入到runLoop中是，runLoop会注册对应的时间点，当时
间点到时,RunLoop会收到wakeUpPort(RunLoop的一个属性)source1类型的消息,将唤醒以执行Timer的回调；
* CFRunLoopObserverRef: 注册观察者时候可以添加一个回调函数,当RunLoop的状态变化时会通知注册的观察者,注册的回调函数在此时调用

> 具体的demo可以参考:
> https://github.com/brownfeng/RunLoopDemo
#### RunLoop底层结构体

CFRunLoop的结构如下

```
struct __CFRunLoop {
    CFMutableSetRef _commonModes;     
    CFMutableSetRef _commonModeItems; 
    CFRunLoopModeRef _currentMode;    
    CFMutableSetRef _modes;           
    ...
};
```

CFRunLoopMode的结构如下：

```
struct __CFRunLoopMode {
    CFStringRef _name;            
    CFMutableSetRef _sources0;    
    CFMutableSetRef _sources1;    
    CFMutableArrayRef _observers; 
    CFMutableArrayRef _timers;    
    ...
};
```

其中，CFRunLoop对外暴露的管理Mode的接口有两个:

```
CFRunLoopAddCommonMode
CFRunLoopRunInMode
```

Mode暴露的管理mode item的接口有下面几个：

```
CFRunLoopAddSource
CFRunLoopAddObserver
CFRunLoopAddTimer
CFRunLoopRemoveSource
CFRunLoopRemoveObserver
CFRunLoopRemoveTimer
```

### Cocoa框架中RunLoop相关的内容

#### 1 AutoReleasePool

AutoReleasePool是Apple中清理临时变量,释放内容的机制,在app的main函数中,所有的内容都是包裹在一个AutoReleasePool中的.这个过程中,Apple在RunLoop中注册三个RunLoopObserver:

* 第一个RunLoopObserver关注kCFRunLoopEntry状态,callback函数中会自动创建autoReleasePool,并且observer优先级最高(RunLoop第一个调用它的回调),保证应用启动后所有的操作都在autoreleasePool中运行.
* 第二个RunLoopObserver关注kCFRunLoopBeforeWaiting状态,在此时autoreleasePool会释放旧的池,并创建一个新的autoreleasePool.
* 第三个RunLoopObserver关注kCFRunLoopExit状态,此时释放autoreleasePool,并且保证observer的优先级最低,即在所有其他的observers都执行完以后才执行autorelease相关的observer的callback函数.

> 深入学习可以参考:
> http://blog.sunnyxx.com/2014/10/15/behind-autorelease/

#### 2 iOS事件响应

RunLoop解释了为何iOS应用能够接受到屏幕触摸等事件.Apple在iOS app启动时候在main RunLoop中注册一个Source1事件,是基于Mach Port(进程间通信)的系统层面的进程.

当一个触摸事件发生以后,IOKit会生成一个IOHIDEvent事件,并由系统层面的SpringBoard接受按键(锁屏/静音),触摸,加速,接近传感器等几种Event,然后通过进程间通信的mach port发送给App的main RunLoop.然后在DoSource1中,source1的回调会触发,内部会调用系统`_UIApplicationHandleEventQueue()`进行事件分发.

`_UIApplicationHandleEventQueue()`方法会把IOHIDEvent处理,并包装成常见的UIEvent事件进行处理或分发,其中包括识别UIGesture/处理屏幕旋转/发送给UIWindow等.通常事件比如UIButton点击,touchesBegin/Move/End/Cancel等事件都是在这个Source1回调中完成.

#### 3 手势识别GestureRecognizer
在上面的事件响应中,回调方法`_UIApplicationHandleEventQueue()`识别了一个手势以后,首先会调用Cancel将当前的touchesBegin/Move/End系列的回调打断.随后系统会将UIGestureRecognizer标记为等待处理.

然后苹果会在RunLoop中注册一个Observer观察kCFRunLoopBeforeWaiting(睡眠前).这个观察者的回调函数是`_UIGestureRecognizerUpdateObserver()`,它会获取刚才所有被标记为等待处理等的GestureRecognizer,并且执行GestureRecognizer的回调.

#### 4 界面更新(UI update)
在App改变UI时,例如修改view的Frame,更新UIView/CALayer的层次,或者手动调用UIView/CALayer的setNeedsLayout/setNeedsDisplay方法以后,这个UIView/CALayer就被标记为等待处理,并且提交到一个全局容器.

然后苹果在RunLoop中注册一个Observer观察kCFRunLoopBeforeWaiting和kCFRunLoopExit状态.在状态触发DoObserver的callback中会调用函数`_ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv()`.这个函数中会遍历所有的等待处理的UIView/CALayer,执行绘制和调整,更新UI界面.

这个函数的调用栈如下:

```
_ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv()
    QuartzCore:CA::Transaction::observer_callback://DoObserver
        CA::Transaction::commit();//提交到全局容器
            CA::Context::commit_transaction();
                CA::Layer::layout_and_display_if_needed();
                    CA::Layer::layout_if_needed();
                        [CALayer layoutSublayers];
                            [UIView layoutSubviews];
                    CA::Layer::display_if_needed();
                        [CALayer display];
                            [UIView drawRect];

```

#### 5 NSTimer定时器

NSTimer的CF层是CFRunLoopTimerRef,它们可以toll-free bridged.当使用`CFRunLoopAddTimer()`将NSTimer&CFRunLoopTimerRef注册到RunLoop中以后,RunLoop会为它重复的时间点注册好事件(有一定的时间容忍度,触发的时间误差).如果中间某次触发被错过,那么这次触发时间点的回调也会被跳过去. 实际中NSTimer的触发是通过source1触发的,如果timer被触发会将RunLoop的有一个Mach Port-`timePort`-发消息,然后会唤醒RunLoop,调用DoTimers.可以参考上面的RunLoop运行流程.

> CADisplayLink 是一个和屏幕刷新一致的定时器,如果两次屏幕刷新时候在执行一个长时间任务,那其中就会有一帧被跳过,造成界面卡顿.Facebook的AsyncDisplayLink使用RunLoop来解决丢帧的问题.
> 
> 还有一个GCD的定时器`dispatch_timer`,RunLoop的超时机制也是使用`dispatch_timer`.

#### 6 部分PerformSelector事件源

NSObject的部分与时间相关的PerformSelector方法与RunLoop密切相关.当调用NSObject的performSelector:afterDelay以后,实际内部会创建一个Timer并添加RunLoopMode中,等待Timer触发,然后调用DoTimers,其中包含的回调方法就是selector.因此如果当前线程没有开启RunLoop,这个方法无效.类似的NSObject的performSelector:onThread:也是类似,需要线程开启RunLoop.具体涉及到的perform方法包括:

```
- performSelector:afterDelay:
– performSelector:withObject:afterDelay:
– performSelectorOnMainThread:withObject:waitUntilDone:
– performSelectorOnMainThread:withObject:waitUntilDone:modes:
– performSelector:onThread:withObject:waitUntilDone:modes:
- performSelector:onThread:withObject:waitUntilDone:
```

#### 7 GCD与RunLoop

GCD的中部分接口使用了RunLoop,提交到mainQueue的blocks.当调用`dispatch_async(dispatch_get_main_queue(), block)`时,libDispatch(系统的某个进程)会向app的进程的main runloop的dispatchPort(是一个mach port)发送消息.此时RunLoop如果在休眠状态,会被唤醒,并从dispatch port中取出消息,并在会在回调方法`__CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__()`中执行这个block.

> libDispatch分发到main queue的block才会与RunLoop有关,分发到其他的子线程的内容,还是又libDispatch处理.
> 
> RunLoop的运行超时是由GCD的dispatch_timer控制的.

#### 8 关于网络请求

关于网络请求的接口，主要有以下几层:

* CFSocket:是最底层的接口，只负责socket的通信
* CFNetwork:是基于CFSocket等接口的上层封装,ASIHttpRequest工作在这层
* NSURLConnection:是基于CFNetwork的更高层的封装，提供面向对象的接口，
AFNetworking2.x工作于这一层
* NSURLSession:是ios7中新增的接口,表面上和NSURLConnection并列,但底层
仍然用到NSURLConnection的部分功能,AFNetworking3.x和Alamofire在这层

因此,现在使用的大多网络库都与CFSocket,CFNetwork层相关.

通常使用NSURLConnection时,你会传入一个Delegate,当调用[connection start]后，这个Delegate就会不停收到事件回调。实际上,start 这个函数的内部会
会获取 CurrentRunLoop，然后在其中的 DefaultMode 添加了4个 Source0 (即需要手动
触发的Source)。CFMultiplexerSource 是负责各种 Delegate 回调的，CFHTTPCookie
Storage 是处理各种 Cookie 的。

>NSURLConnection的工作过程可以参考:
http://blog.ibireme.com/2015/05/18/runloop/#base

### RunLoop的应用举例
#### 1 AFNetworking2.x

使用NSURLConnection时候,需要在后台自定义线程中接受Delegate回调.AFNetworking创建一个自定义线程,然后在其中添加一个Mach Port,然后启动RunLoop(前面提到过,如果RunLoop中没有source/timer/observer会立即退出).在以后线程以后通过NSObject的`performSelector:onThread:withObject:waitUnitlDone:modes:`方法,将`[self.connection scheduleInRunLoop:runLoop forMode:runLoopMode]`.

>可以参考:
>https://github.com/brownfeng/SourceSet/tree/master/AFNetworking2.x

#### 2 AsyncDisplayKit
AsyncDisplayKit是Facebook推出的用于保持界面流畅性的框架,其原理大致如下：

UI线程中一旦出现繁重的任务就会导致界面卡顿,这类任务通常分为3类:排版,绘制,UI对象操作。通过各种方法将前两种任务丢到后台运行,最后一类操作只能在主线程中执行,因此,ASDK仿照QuartzCore/UIKit框架的模式,实现了一套类似的界面更新的机制:即在主线程的RunLoop中添加一个 Observer,监听kCFRunLoopBeforeWaiting和 kCFRunLoopExit事件,在收到回调时,遍历所有之前放入队列的待处理的任务,然后一一执行。

#### 3 NSTimer在TrackingMode运行
#### 4 TableView延迟加载图片
#### 5 RunLoop解决大图加载的问题
#### 6 RunLoop监控App卡顿

>可以参考文章:
>
>http://www.jianshu.com/p/929d855c5a5a
>http://www.jianshu.com/p/924cb2b218f5

### 参考内容
http://blog.ibireme.com/2015/05/18/runloop/

http://yun.baidu.com/share/link?shareid=2268593032&uk=2885973690