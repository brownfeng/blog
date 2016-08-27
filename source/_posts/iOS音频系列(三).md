---
title: iOS音频系列(三)--AudioQueue
date: 2016-07-27
tags:
- iOS
- CoreAudio
- AudioQueue
---

本篇是AudioQueue的官方文档的笔记。Audio Queue Services可以play和record以下三类任何audio data：

* Linear PCM.
* Any compressed format supported natively on the Apple platform you are developing for.
* Any other format for which a user has an installed codec.

对于最后一种类型，我们可以在使用AudioQueue同时自己将自己需要的format转化成LPCM。AudioQueue是对mic和speaker的高度抽象，同时可以非常简单的时间音频codecs。与此同时，它也有一些高级功能，例如多个音频的同步播放，回放等等。

****

## About Audio Queues


这章会了解到audio queue的功能，结构体，以及内部运行的机理。具体的内容包括audio queues，audio queue buffers，audio queue会使用到的callback等。还有就是audio queue的状态以及参数。

### What Is an Audio Queue?

An audio queue 是iOS中play和record audio的对象.底层是`AudioQueueRef`。Audio queue可以完成以下工作：

* Connecting to audio hardware
* Managing memory
* Employing codecs, as needed, for compressed audio formats
* Mediating recording or playback

#### Audio Queue Architecture

Audio queue的具体结构有以下几个部分构成：

* A set of **audio queue buffers**, each of which is a temporary repository for some audio data
* A **buffer queue**, an ordered list for the audio queue buffers
* An **audio queue callback** function, that you write

根据我们使用audio queue的用途（record or play），具体的结构略有不同，仅仅只是callback函数函数的内容不同。

#### Audio Queues for Recording

一个用于record 的audio queue，需要使用`AudioQueueNewInput`方法创建，它的具体结构如图：

![A recording audio queue](https://developer.apple.com/library/prerelease/content/documentation/MusicAudio/Conceptual/AudioQueueProgrammingGuide/Art/recording_architecture_2x.png)

#### Audio Queues for Playback

一个用于play的audio queue，需要使用`AudioQueueNewOutput`函数创建，

![A playback audio queue](https://developer.apple.com/library/prerelease/content/documentation/MusicAudio/Conceptual/AudioQueueProgrammingGuide/Art/playback_architecture_2x.png)

#### Audio Queue Buffers

**audio queue buffer**的数据结构如下：

```objc
typedef struct AudioQueueBuffer {
    const UInt32   mAudioDataBytesCapacity;
    void *const    mAudioData;
    UInt32         mAudioDataByteSize;
    void           *mUserData;
} AudioQueueBuffer;
typedef AudioQueueBuffer *AudioQueueBufferRef;
```

其中**mAudioData**字段表示这个buffer中的有用数据的地址，其他的字段用来辅助audio queue来管理使用这个buffer。一个audio queue可以使用任何数目的buffers。但是我们一般选择3个，比较好管理。

Audio queue通过下面的方式管理它们内部的buffers：

* An audio queue allocates a buffer when you call the AudioQueueAllocateBuffer function.
* When you release an audio queue by calling the AudioQueueDispose function, the queue releases its buffers.

### The Buffer Queue and Enqueuing

buffer queue是由audio buffers组成的，是audio queue中的buffers。我们前面介绍了audio queue是如何使用callback管理内部的buffers。不论当前是用于record或者是pleyback，将buffer放到audio queue都是需要我们在callback函数中去手动调用的。

#### The Recording Process

![The recording process](https://developer.apple.com/library/prerelease/content/documentation/MusicAudio/Conceptual/AudioQueueProgrammingGuide/Art/recording_callback_function_2x.png)

1. In step 1 , recording begins. The audio queue fills a buffer with acquired data.
2. In step 2, the first buffer has been filled. The audio queue invokes the callback, handing it the full buffer (buffer 1). The callback (step 3) writes the contents of the buffer to an audio file. At the same time, the audio queue fills another buffer (buffer 2) with freshly acquired data.
3. In step 4, the callback enqueues the buffer (buffer 1) that it has just written to disk, putting it in line to be filled again. The audio queue again invokes the callback (step 5), handing it the next full buffer (buffer 2). The callback (step 6) writes the contents of this buffer to the audio file. This looping steady state continues until the user stops the recording.

#### The Playback Process

![The playback process](https://developer.apple.com/library/prerelease/content/documentation/MusicAudio/Conceptual/AudioQueueProgrammingGuide/Art/playback_callback_function_2x.png)

#### Controlling the Playback Process

Audio queue buffers在queue是顺序播放的，我们可以通`theAudioQueueEnqueueBufferWithParameters`方法来进行控制

### The Audio Queue Callback Function

Audio queue在运行过程中会不断的调用callback函数，通常间隔时间和audio queue buffer的大小相关，一般是几秒一次。

audio queue callback主要任务是将audio queue buffer归还给audio queue。callback中通过`AudioQueueEnqueueBuffer`方法将buffer加载到audio queue的最后。在playback中，可以使用`AudioQueueEnqueueBufferWithParameters`在enqueue的过程中进行更多的控制。

#### The Recording Audio Queue Callback Function

如果你仅仅使用audio queue去将record的audio data写入file system，callback的方法实现的原型如下：

```objc
AudioQueueInputCallback (
    void                               *inUserData,
    AudioQueueRef                      inAQ,
    AudioQueueBufferRef                inBuffer,
    const AudioTimeStamp               *inStartTime,
    UInt32                             inNumberPacketDescriptions,
    const AudioStreamPacketDescription *inPacketDescs
);
```

一个recording audio queue会触发我们注册的callback，会在callback的参数中传入所有需要的关于audio data的相关信息：

* **inUserData** 是一个自定义的结构体，用来存储audio queueu以及audio queue buffer的状态信息，也包括AudioFileID，audio data format等。
* **inAQ** 表示哪个audio queue触发这个callback。
* **inBuffer** 是一个audio queue buffer，它的内容是由audio queue填充的，内部包括最新的audio data。并且这些audio data已经根据初始化时候传递的格式参数格式化好的数据。
* **inStartTime** 表示这个buffer中的第一个采样的采样时间点，一般app中不太需要这个参数。
* **inNumberPacketDescriptions** 表示**inPacketDescs**参数中的packet descriptions的个数。如果你是录入VBR format，audio queue就会在callback中提供这个参数，如果是CBR，audio queue就不会使用packet descriptions参数，这个参数会是NULL。
* **inPacketDescs** 表示buffer中samples相关的一系列的packet descriptions。是否设置同上一个参数。

#### The Playback Audio Queue Callback Function

这个片段会介绍如果使用playing audio queue，那么callback应该的信息：

```objc
AudioQueueOutputCallback (
    void                  *inUserData,
    AudioQueueRef         inAQ,
    AudioQueueBufferRef   inBuffer
);
```

一个playback audio queue会触发这个callback，提供一些关于audio data的有用信息：

* **inUserData** 见上
* **inAQ** 表示哪个audio queue触发这个callback。
* **inBuffer** 表示被audio queue设置为空的audio queue buffer，你需要在callback中将其内部信息填满，填充内容是你从AudioFile中读取的audio data。

****

### Using Codecs and Audio Data Formats

我们日常使用Audio Queue Services时，都会使用codecs（audio data coding/decoding componets）用来在不同audio format之间进行转化。

每个audio queue都有一个audio data format，可以在`AudioStreamBasicDescription`结构体中得到。当我们在ASBD中指定了`mFormatID`以后，audio queue在向buffer中填充数据时候就会使用相应的codec。同样如果指定sample rate和channel count，audio queue也会同样。具体的过程见下图：

![Audio format conversion during recording](https://developer.apple.com/library/ios/documentation/MusicAudio/Conceptual/AudioQueueProgrammingGuide/Art/recording_codec_2x.png)

* 第一步中，app会告知audio queue开始record，同时告诉它使用的的data format。
* 第二步中，audio queue将获取到的new data使用codec转化成目标format。然后audio queue会调用callback函数，传入格式化以后的audio data。
* 第三步中，callback函数会将格式化以后的audio data写入file中。

整个过程中，callback函数压根就不需要知道data fromat是什么。

![Audio format conversion during playback](https://developer.apple.com/library/ios/documentation/MusicAudio/Conceptual/AudioQueueProgrammingGuide/Art/playback_codec_2x.png)

在播放过程中，正好和录音过程相反，只需要在创建audio queue时候将data format告知即可。

*****

### Audio Queue Control and State

audio queue在创建和销毁的过程有一个声明周期。app需要管理它的声明周期，控制它的状态，具体控制状态的方法如下：

* Start (`AudioQueueStart`).初始化audio queue用来record或者playback。
* Prime (`AudioQueuePrime`).对于playback，在调用`AudioQueueStart`挚爱去哪调用确保数据可用，这个方法和record没有关系。
* Stop (`AudioQueueStop`). 调用以后会重置audio queue，然后会停止record或者playback。在playback应用中，一般在没有audio data可以播放时候调用。
* Pause (`AudioQueuePause`). 在record或者playback中调用这个方法不会影响到buffers。如果需要恢复，调用`AudioQueueStart`。
* Flush (`AudioQueueFlush`). 在enqueue最后一个audio queue buffer以后调用这个方法，确保所有的数据被record或者play（主要是在midst processing的数据）。
* Reset (`AudioQueueReset`). 调用以后立即停止audio queue，然后将所有的buffers移除，重置所有的DSP状态等到。

在调用`AudioQueueStop`方法时候有两种模式：同步和异步。

* Synchronous stopping happens immediately, without regard for previously buffered audio data.
* Asynchronous stopping happens after all queued buffers have been played or recorded.

****

## Recording Audio

当我们的record使用Audio Queue Services，存储的路径可以是磁盘上的任何地方，或者网络，或者内存中。这部分内容记录大多数的使用场景，存储在磁盘中。

具体的步骤如下：

1. 定义一个结构体去存储状态，format，文件路径等信息。
2. 完成audio queue callback函数，其中将record以后的数据进行存储。
3. 为audio queue buffers计算出合适的大小，并且在file中写入magic cookies。
4. 初始化自定义的结构体
5. 创建recording audio queue，然后给它创建3个audio queue buffers，然后创建一个file用来存储record以后的audio data。
6. 启动audio queue
7. 当audio queue停止以后，dispose它以及buffers

具体的实现内容可以参考Apple官方文档：[Recording Audio](https://developer.apple.com/library/prerelease/content/documentation/MusicAudio/Conceptual/AudioQueueProgrammingGuide/AQRecord/RecordingAudio.html)。


## Playing Audio

当我们使用Audio Queue Service去play audio时，音频源文件可以是任何在disk file或者memory中，这部分内容是如何用Audio Queue Service播放存储在disk上的audio file。

具体的步骤如下：

1. 定义一个结构体管理Audio queue的状态，format，file path等
2. 完成audio queue callback函数去进行实际的播放
3. 创建一个函数用来计算最适合的audio queue buffer的大小
4. 打开audio file，确定它的audio data format
5. 创建audio queue，对它进行配置
6. 为audio queue创建buffers，然后启动audio queue，当播放结束，callback让audio queue停止播放
7. 销毁audio queue

具体的实现内容可以参考Apple官方文档：[Playing Audio](https://developer.apple.com/library/prerelease/content/documentation/MusicAudio/Conceptual/AudioQueueProgrammingGuide/AQPlayback/PlayingAudio.html)。



## 可运行的Demo

请参考我的github: [https://github.com/brownfeng/AudioQueueServiceDemo](https://github.com/brownfeng/AudioQueueServiceDemo)


