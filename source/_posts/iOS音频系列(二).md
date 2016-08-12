---
title: iOS音频系列(二)--CoreAudio
tags:
- iOS
- CoreAudio
---

这一篇主要是CoreAudio官方文档的重点内容的笔记。

### 通过回调函数与CoreAudio交互

iOS的CoreAudio是通过callback函数与App交互的。其中需要设置回调函数有以下几种情况：

* CoreAudio会向回调函数给App传入PCM音频数据，然后App需要在回调函数中将音频数据写入文件文件系统。（录音时候）
* CoreAudio会需要向App请求一些音频数据，App通过从文件系统中读取音频数据，然后通过callback函数传递给CoreAudio。（播放时候）
* 通过注册属性观察者，监听CoreAudio的属性，注册回调函数

下面是一个使用Audio Queue Services的属性监听器的callback函数的调用模板。

```
typedef void (*AudioQueuePropertyListenerProc) (
                void *                  inUserData,
                AudioQueueRef           inAQ,
                AudioQueuePropertyID    inID
            );

```

在实现和使用这个callback函数时候，你需要完成两件事：

* 实现这个函数。例如，你可以实现property listener callback，根据audio是否在running或者stop状态，去改变更新UI。
* 注册callback函数时候带上userData数据，在callback函数触发时候使用。

下面是一个property listener callback函数的实现：

```
static void propertyListenerCallback (
    void                    *inUserData,
    AudioQueueRef           queueObject,
    AudioQueuePropertyID    propertyID
) {
    AudioPlayer *player = (AudioPlayer *) inUserData;
        // gets a reference to the playback object
    [player.notificationDelegate updateUserInterfaceOnAudioQueueStateChange: player];
        // your notificationDelegate class implements the UI update method
}
```

下面是注册一个callback函数的例子：

```
AudioQueueAddPropertyListener (
    self.queueObject,                // the object that will invoke your callback
    kAudioQueueProperty_IsRunning,   // the ID of the property you want to listen for
    propertyListenerCallback,        // a reference to your callback function
    self
);
```

********

### Audio Data Formats

这部分内容是iOS中支持的音频格式，理解以后对后面的音频相关的编程能够理解更加深刻。

#### iOS中通用的音频数据类型

在CoreAudio中，使用`AudioStreamBasicDescription`和`AudioStreamPacketDescription`这两个类型描述了通用的音频数据类型,包括压缩音频数据，非压缩的音频数据。他们的数据结构如下：

``` objc
struct AudioStreamBasicDescription {
    Float64 mSampleRate;
    UInt32  mFormatID;
    UInt32  mFormatFlags;
    UInt32  mBytesPerPacket;
    UInt32  mFramesPerPacket;
    UInt32  mBytesPerFrame;
    UInt32  mChannelsPerFrame;
    UInt32  mBitsPerChannel;
    UInt32  mReserved;				//0
};
typedef struct AudioStreamBasicDescription  AudioStreamBasicDescription;

struct  AudioStreamPacketDescription {
    SInt64  mStartOffset;
    UInt32  mVariableFramesInPacket;
    UInt32  mDataByteSize;
};
typedef struct AudioStreamPacketDescription AudioStreamPacketDescription;
```

>compressed audio formats use a varying number of bits per sample. For these formats, the value of the mBitsPerChannel member is 0.

#### Audio Data Packets

前面定义过，将一个或者多个frames称为一个packet。或者说packet是最有意义的一组frames，它在audio file中代表一个有意义的时间单元。使用Core Audio中一般是对packets进行处理的。

每个audio data格式在packets被装配完成以后，它的fromat就被确定了。ASBD数据结构通过`mBytesPerPacket`和`mFramesPerPacket`描述音频格式的packet信息，其中页包含其他的信息。

在整个Core Audio中可能会用到三种不同的packets：

* CBR (constant bit rate) formats：例如 linear PCM and IMA/ADPCM，所有的packet使用相同的大小。
* VBR (variable bit rate) formats：例如 AAC，Apple Lossless，MP3，所有的packets拥有相同的frames，但是每个sample中的bits数目不同。
* VFR (variable frame rate) formats：packets拥有数目不同的的frames。

在Core Audio中使用VBR或者VFR格式，使用ASPD结构体只能用来描述单个packet。如果是record或者play VBR或者VFR音频文件，需要涉及到多个ASPD结构。

在AudioFileService等接口中，都是通过pakcets工作的。例如`AudioFileReadPackets`会得到一系列的packets，同时会得到一个数组的`AudioStreamPacketDescription`。

下面是通过packets计算audio data buffer的大小：

			
``` objc
- (void) calculateSizesFor: (Float64) seconds {
 
    UInt32 maxPacketSize;
    UInt32 propertySize = sizeof (maxPacketSize);
 
    AudioFileGetProperty (
        audioFileID,
        kAudioFilePropertyPacketSizeUpperBound,
        &propertySize,
        &maxPacketSize
    );
 
    static const int maxBufferSize = 0x10000;   // limit maximum size to 64K
    static const int minBufferSize = 0x4000;    // limit minimum size to 16K
 
    if (audioFormat.mFramesPerPacket) {
        Float64 numPacketsForTime =
            audioFormat.mSampleRate / audioFormat.mFramesPerPacket * seconds;
        [self setBufferByteSize: numPacketsForTime * maxPacketSize];
    } else {
        // if frames per packet is zero, then the codec doesn't know the
        // relationship between packets and time. Return a default buffer size
        [self setBufferByteSize:
            maxBufferSize > maxPacketSize ? maxBufferSize : maxPacketSize];
    }
 
    // clamp buffer size to our specified range
    if (bufferByteSize > maxBufferSize && bufferByteSize > maxPacketSize) {
        [self setBufferByteSize: maxBufferSize];
    } else {
        if (bufferByteSize < minBufferSize) {
            [self setBufferByteSize: minBufferSize];
        }
    }
 
    [self setNumPacketsToRead: self.bufferByteSize / maxPacketSize];
}
```

#### Data Format Conversion

将音频数据从一种audio data转换成另外一种audio data。常见的有三中音频格式转换：

* Decoding an audio format (such as AAC (Advanced Audio Coding)) to linear PCM format.
* Converting linear PCM data into a different audio format.
* Converting between different variants of linear PCM (for example, converting 16-bit signed integer linear PCM to 8.24 fixed-point linear PCM).

*******

### Sound Files

如果要使用声音文件，你需要使用Audio File Services的接口。一般而且，在iOS中需要与音频文件的创建，操作都离不开Audio File Services。

使用`AudioFileGetGlobalInfoSize`和`AudioFileGetGlobalInfo`分别分配info的内存和获取info的内容。你可以获取以下的内容：

* Readable file types
* Writable file types
* For each writable type, the audio data formats you can put into the file

#### Creating a New Sound File

为了创建一个能够存储音频数据的audio file，你需要进行以下三步：

* 使用CFURL或者NSURL表示的系统文件的路径
* 你需要创建的文件的类型的标识identifier，这些identifier定义在`Audio File Types`枚举中。例如，为了创建一个CAF文件，你需要使用`kAudioFileCAFType`的identifier。
* 创建过程中你需要提供音频数据的ASBD结构体。为了获取ASBD，你可以先提供ASBD结构体的部分成员的值，然后通过函数让`Audio File Services`将剩余的信息填满。

下面是创建一个AudioFile的方法：

``` objc
AudioFileCreateWithURL (
    audioFileURL,
    kAudioFileCAFType,
    &audioFormat,
    kAudioFileFlags_EraseFile,
    &audioFileID   // the function provides the new file object here
);
```

#### Opening a Sound File

为了打开sound file，需要使用`AudioFileOpenURL`函数，该函数会返回一个唯一ID，供后面使用。

为了获取sound file的一些属性，通常使用`AudioFileGetPropertyInfo`和`AudioFileGetProperty`，日常使用的属性以下：

* kAudioFilePropertyFileFormat
* kAudioFilePropertyDataFormat
* kAudioFilePropertyMagicCookieData
* kAudioFilePropertyChannelLayout

#### Reading From and Writing To a Sound File

iOS中，我们经常需要使用`Audio File Services`去读写audio data到sound file中。读和写是一对相反的内容，操作的对象都可以是bytes或者packets，但是一般而言都是直接使用的packets。

>* 读写VBR数据，只能使用packet
* 直接使用packet，更加容易计算时间

******

### iPhone Audio File Formats

iOS支持的sound file格式如下：

|Format name|Format filename extensions|
|:---------:|:------------------------:|
|AIFF|.aif, .aiff|
|CAF|.caf|
|MPEG-1, layer 3|.mp3|
|MPEG-2 or MPEG-4 ADTS|.aac|
|MPEG-4|.m4a, .mp4|
|WAV|.wav|

iOS中的native format是CAF file format。

****

### Audio Sessions: Cooperating with Core Audio

在iOS中，app在运行过程中有可能接到电话，如果此时正在播放sound，系统会做一定的处理。

AudioSession就是在这种情况的中间人，每个app都会有一个audio session。在播放或者录音时候需要session在做正确的事情，需要我们自己弄清楚以下的情况：

* app收到系统的中断时候应该如何响应，比如收到phone call？
* 你是否需要app的sound和其他后台运行的app的sounds混合播放，或者需要独占播放？
* 你需要app如何响应远音频路径的响应，比如拔插耳机时候

AudioSession提供了三种类型的接口：

* **Categories**： A category is a key that identifies a set of audio behaviors for your application. By setting a category, you indicate your audio intentions to iOS, such as whether your audio should continue when the screen locks.
* **Interruptions and route changes** ：Your audio session posts notifications when your audio is interrupted, when an interruption ends, and when the hardware audio route changes. These notifications let you respond to changes in the larger audio environment—such as an interruption due to in an incoming phone call—gracefully.
* **Hardware characteristics**：You can query the audio session to discover characteristics of the device your application is running on, such as hardware sample rate, number of hardware channels, and whether audio input is available.

#### Audio Session Default Behavior
Audio Session拥有一些默认的行为策略：

* 当用户将静音开关静音时，audio就会静音。
* 当用户锁屏（手动，自动）时候，audio就会静音。
* 当你app的audio启动时，其他app正在使用的audio就会静音。

audio session的这个特定的默认的行为策略被称为`kAudioSessionCategory_SoloAmbientSound`。同时，它还包括其他的多种策略选择。

#### Interruptions: Deactivation and Activation

默认的audio session的一个典型的特征是，audio会在中断以后自动恢复活动。Audio session有两个重要的状态：`active`和`inactive`。只有当Audio session处于`active`状态时候，app才能使用audio。

在app启动以后，你的默认的audio session就会是`active`状态。然而，如果一个电话被打进来，你的session就会立刻被置为`inactive`，然后app中的audio就会停止。这个电话就被称为一个中断，如果用户选择忽略电话，app就会继续运行。但是此时你的audio session依然会是`inactive`状态，audio也就不会工作。

如果你使用 Audio Queue Services操作audio，我们就需要给中断注册listener回调函数，手动去重启audio session。具体内容可以见`Audio Session Programming Guide`。

#### Determining if Audio Input is Available

一个录音的app只有在设备的音频硬件可用的时候才能录音。为了检查这个属性，需要使用audio session的`kAudioSessionProperty_AudioInputAvailable`属性。

``` objc
UInt32 audioInputIsAvailable;
UInt32 propertySize = sizeof (audioInputIsAvailable);
 
AudioSessionGetProperty (
    kAudioSessionProperty_AudioInputAvailable,
    &propertySize,
    &audioInputIsAvailable // A nonzero value on output means that
                           // audio input is available
);
```
#### Using Your Audio Session

你的app同时只能有一个audio session策略，你的所有的audio都需要遵循这个`active`策略的特点。如何响应中断，在Audio Session Programming Guide`中有更加详细的内容。

> 如果要测试Audio session，需要使用真机

****

### Playback using the AVAudioPlayer Class

`AVAudioPlayer`提供了简单的OC接口用于audio播放。如果非网络stream，或者需要精确控制，apple推荐使用这个类，它可以用于播放iOS支持的任何audio format，同时这个类并不需要去设置audio session，因为它会在中断发生以后自动恢复播放，除非你需要指定特地的行为。

它可以完成以下工作：

* Play sounds of any duration
* Play sounds from files or memory buffers
* Loop sounds
* Play multiple sounds simultaneously
* Control relative playback level for each sound you are playing
* Seek to a particular point in a sound file, which supports such application features as fast forward and rewind
* Obtain data that you can use for audio level metering

下面就是使用`AVAudioPlayer`的具体流程：

1. Configuring an AVAudioPlayer object

``` objc
NSString *soundFilePath =
                [[NSBundle mainBundle] pathForResource: @"sound"
                                                ofType: @"wav"];
 
NSURL *fileURL = [[NSURL alloc] initFileURLWithPath: soundFilePath];
 
AVAudioPlayer *newPlayer =
                [[AVAudioPlayer alloc] initWithContentsOfURL: fileURL
                                                       error: nil];
[fileURL release];
 
self.player = newPlayer;
[newPlayer release];
 
[self.player prepareToPlay];
[self.player setDelegate: self];
```

你的delegate对象用于处理`interruptions`或者音频播放停止以后的操作。

2. Implementing an AVAudioPlayer delegate method 

``` objc
- (void) audioPlayerDidFinishPlaying: (AVAudioPlayer *) player
                        successfully: (BOOL) flag {
    if (flag == YES) {
        [self.button setTitle: @"Play" forState: UIControlStateNormal];
    }
}
```

3. Controlling an AVAudioPlayer object

``` objc
- (IBAction) playOrPause: (id) sender {
 
    // if already playing, then pause
    if (self.player.playing) {
        [self.button setTitle: @"Play" forState: UIControlStateHighlighted];
        [self.button setTitle: @"Play" forState: UIControlStateNormal];
        [self.player pause];
 
    // if stopped or paused, start playing
    } else {
        [self.button setTitle: @"Pause" forState: UIControlStateHighlighted];
        [self.button setTitle: @"Pause" forState: UIControlStateNormal];
        [self.player play];
    }
}
```

****

### Recording and Playback using Audio Queue Services

Audio Queue Services是一个更加直观的record和play audio的方式。同时它还有更多个高级功能，可以使用这个服务完成更多的工作，比如对LPCM数据进行压缩等等。它和`AVAudioPlayer`是iOS中唯二可以播放压缩后音频格式的接口。使用Audio Queue Service播放和录音都是通过回调方法完成的。

#### Creating an Audio Queue Object

为了创建Audio Queue对象，它分成两类：

* AudioQueueNewInput用于录音
* AudioQueueNewOutput用于播放

使用audio queue object播放audio file，需要一些几个步骤：

1. 创建数据结构用于管理audio queue需要的信息，例如audio format，audio fileID等。
2. 定义callback函数，用于管理audio queue buffers。callback会使用Audio File Service去读取audio file用来播放
3. 使用AudioQueueNewOutput用来播放audio file。

Creating an audio queue object具体的代码如下：

``` objc
static const int kNumberBuffers = 3;
// Create a data structure to manage information needed by the audio queue
struct myAQStruct {
    AudioFileID                     mAudioFile;
    CAStreamBasicDescription        mDataFormat;
    AudioQueueRef                   mQueue;
    AudioQueueBufferRef             mBuffers[kNumberBuffers];
    SInt64                          mCurrentPacket;
    UInt32                          mNumPacketsToRead;
    AudioStreamPacketDescription    *mPacketDescs;
    bool                            mDone;
};
// Define a playback audio queue callback function
static void AQTestBufferCallback(
    void                   *inUserData,
    AudioQueueRef          inAQ,
    AudioQueueBufferRef    inCompleteAQBuffer
) {
    myAQStruct *myInfo = (myAQStruct *)inUserData;
    if (myInfo->mDone) return;
    UInt32 numBytes;
    UInt32 nPackets = myInfo->mNumPacketsToRead;
 
    AudioFileReadPackets (
        myInfo->mAudioFile,
        false,
        &numBytes,
        myInfo->mPacketDescs,
        myInfo->mCurrentPacket,
        &nPackets,
        inCompleteAQBuffer->mAudioData
    );
    if (nPackets > 0) {
        inCompleteAQBuffer->mAudioDataByteSize = numBytes;
        AudioQueueEnqueueBuffer (
            inAQ,
            inCompleteAQBuffer,
            (myInfo->mPacketDescs ? nPackets : 0),
            myInfo->mPacketDescs
        );
        myInfo->mCurrentPacket += nPackets;
    } else {
        AudioQueueStop (
            myInfo->mQueue,
            false
        );
        myInfo->mDone = true;
    }
}
// Instantiate an audio queue object
AudioQueueNewOutput (
    &myInfo.mDataFormat,
    AQTestBufferCallback,
    &myInfo,
    CFRunLoopGetCurrent(),
    kCFRunLoopCommonModes,
    0,
    &myInfo.mQueue
);
```

#### Controlling Audio Queue Playback Level

Audio Queue对象提供了两个方式控制音频level。第一种是直接使用`AudioQueueSetParameter`以及`kAudioQueueParam_Volume`参数，就可以设置，设置完成以后会立即生效。

``` objc
Float32 volume = 1;
AudioQueueSetParameter (
    myAQstruct.audioQueueObject,
    kAudioQueueParam_Volume,
    volume
);

```

也可以通过`AudioQueueEnqueueBufferWithParameters`给audio queue buffer设置。这种方式只有在audio queue buffer 开始播放时候才起作用。

#### Indicating Audio Queue Playback Level

你也可以直接查询audio queue的`kAudioQueueProperty_CurrentLevelMeterDB`属性，得到的值是一组`AudioQueueLevelMeterState`结构体（一个channel一个数组），具体的结构体是`AudioQueueLevelMeterState`，显示如下：

```
typedef struct AudioQueueLevelMeterState {
    Float32     mAveragePower;
    Float32     mPeakPower;
};  AudioQueueLevelMeterState;
```

*****

### System Sounds: Alerts and Sound Effects

如果你需要播放的音频时间少于30s，那么可以使用System Sound Services。调用`AudioServicesPlaySystemSound`函数可以立即播放一个sound file。你也可以调用`AudioServicesPlayAlertSound`播放alert声音。这两个方法都会在手机静音的情况下振动。

当然，你也在调用`AudioServicesPlaySystemSound`方法使用`kSystemSoundID_Vibrate`属性，显示的触发振动。

为了使用`AudioServicesPlaySystemSound`方法播放sound，首先需要将sound file注册到系统中，得到一个sound ID，然后才能播放。

下面一段代码显示了使用System Sound Services去play sound：

``` objc
#include <AudioToolbox/AudioToolbox.h>
#include <CoreFoundation/CoreFoundation.h>
 
// Define a callback to be called when the sound is finished
// playing. Useful when you need to free memory after playing.
static void MyCompletionCallback (
    SystemSoundID  mySSID,
    void * myURLRef
) {
        AudioServicesDisposeSystemSoundID (mySSID);
        CFRelease (myURLRef);
        CFRunLoopStop (CFRunLoopGetCurrent());
}
 
int main (int argc, const char * argv[]) {
    // Set up the pieces needed to play a sound.
    SystemSoundID    mySSID;
    CFURLRef        myURLRef;
    myURLRef = CFURLCreateWithFileSystemPath (
        kCFAllocatorDefault,
        CFSTR ("../../ComedyHorns.aif"),
        kCFURLPOSIXPathStyle,
        FALSE
    );
 
    // create a system sound ID to represent the sound file
    OSStatus error = AudioServicesCreateSystemSoundID (myURLRef, &mySSID);
 
    // Register the sound completion callback.
    // Again, useful when you need to free memory after playing.
    AudioServicesAddSystemSoundCompletion (
        mySSID,
        NULL,
        NULL,
        MyCompletionCallback,
        (void *) myURLRef
    );
 
    // Play the sound file.
    AudioServicesPlaySystemSound (mySSID);
 
    // Invoke a run loop on the current thread to keep the application
    // running long enough for the sound to play; the sound completion
    // callback later stops this run loop.
    CFRunLoopRun ();
    return 0;
}
```