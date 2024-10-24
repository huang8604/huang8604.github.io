---
title: Android SurfaceFlinger-CSDN博客
author: 
created: 2024-09-27
tags:
  - clippings
  - 转载
  - blog
collections:
  - 图形显示
  - 总结文章
source: https://blog.csdn.net/liuning1985622/article/details/138464728
date: 2024-09-27T03:29:57.493Z
lastmod: 2024-09-29T06:49:56.000Z
---
## 一、SurfaceFlinger介绍

SurfaceFlinger是Android系统中的一个重要组件，它主要负责窗口管理与界面显示。具体来说，SurfaceFlinger作为系统的显示引擎，负责接收各应用程序发来的图像数据，并组合成一张完整的画面，并输出到显示屏上。SurfaceFlinger通过重绘整个屏幕或部分屏幕来更新UI界面。它还提供了多种硬件加速技术，如OpenGL ES、Vulkan等，使应用程序能够更快地渲染UI界面。

SurfaceFlinger还支持窗口叠加、透明度、混合模式等特性，以支持复杂的多层UI界面。此外，它还能管理所有Surface对象（如视频、图片等），并为每个Surface对象分配一个BufferQueue（缓冲区队列），确保每个Surface都能按时完成显示。

SurfaceFlinger启动流程：

![7c3cdce4fc9cf51f45117ffe66f4b801\_MD5](https://picgo.myjojo.fun:666/i/2024/09/27/66f62bcc4a82e.png)

SurfaceFlinger送显的流程：

![67c08f8816f82eb2d4aa474e0b8f0afc\_MD5](https://picgo.myjojo.fun:666/i/2024/09/27/66f62bcb63ed4.png)

## 二、Surface的渲染

Android系统的UI从绘制到显示在屏幕上可分为两个步骤：

1、Android App进程：将UI绘制到一个图形缓冲区GraphicBuffer中，然后通知SurfaceFlinger进行合成。

2、SurfaceFlinger进程：将GraphicBuffer数据合成并交给屏幕缓冲区去显示，这一步本身就是通过软件（Skia）和硬件（Open GL和 HardWare Composer）去完成的。

![4323522b253b39d24b7785bf122431d9\_MD5](https://picgo.myjojo.fun:666/i/2024/09/27/66f62bcceef78.png)

#### 软件渲染介绍

当App更新部分UI时，CPU会遍历ViewTree计算处需要重绘的赃区，接着在View层次结构中绘制所有跟脏区相交的区域，因此软件绘制会绘制到不需要重绘的视图。

软件绘制的绘制过程是在主线程进行的，可能会造成卡顿等情况。

软件绘制把要绘制的内容写进一个Bitmap位图，在之后的渲染过程中，这个Bitmap的像素内容会填充到Surface的缓冲区里。

软件绘制使用Skia库，Skia是Google开发的一个跨平台的2D图形库，它提供了一组强大的API，可以实现高质量的2D图形渲染。

需要注意的是，软件渲染的性能相对较低，而且可能会因为大量的图像计算而占用大量CPU资源，导致应用程序的运行速度变慢。

如下为软件渲染流程：

1、构建视图树：Android应用程序中的UI由视图树来组织的，首先需要构建视图树。

2、计算布局：在获得视图树之后，系统需要计算每个视图的大小和位置，并将它们放置在正确的位置上，以完成布局。

3、绘制视图：接下来，系统将开始使用软件渲染引擎(Skia)，对每一个视图进行主意绘制，即使用CPU执行图像计算并将图像渲染到GraphicBuffer上。

4、合成图像：当所有的视图都绘制完成后，SurfaceFlinger会将它们合成在一起，生成最终屏幕图像，并将其放到主显示缓冲区中。

#### 硬件渲染介绍

当App更新部分UI时，CPU会计算出脏区，但是不会立即执行绘制命令，而是将darwXXX函数作为绘制指令(DrawOp)记录在一个列表(DisplayList中)，然后交给单独的Render线程使用GPU进行硬件加速渲染。

只需要针对需要更新的View对象的脏区进行记录或更新，无需更新的View对象则能重用先前DisplayList中记录的指令。

硬件加速是在单独的Render线程中绘制的，分担了主线程的压力，提高了响应速度。

硬件绘制使用OpenGL在GPU上完成，OpenGL是跨平台的图形API，为2D/3D图形处理硬件制定了标志的软件接口。

如下为硬件渲染的流程：

1、构建视图树：与软件渲染一样，硬件渲染也需要构建应用程序的视图树。

2、计算布局：与软件渲染一样，硬件渲染也需要对每个视图计算布局。

3、创建OpenGL的渲染上下文和纹理：Android使用OpenGL ES作为硬件加速的渲染引擎。因此，当硬件加速被开启时，系统会创建OpenGL的渲染上下文，同时为每个视图分配一个渲染纹理。

4、上传纹理数据：在计算好视图的布局之后，系统将视图的图像数据上传到渲染纹理中。这通常由GPU完成。

5、绘制纹理：最后，系统将视图的渲染纹理绑定到OpenGL的渲染上下文中，并执行GPU计算，即使用GPU绘制图像。

6、合成图像：当所有的视图都绘制完成后，SurfaceFlinger会将它们合成在一起，生成最终屏幕图像，并将其放到主显示缓冲区中。

#### Android在什么时候使用软件渲染、什么时候使用硬件渲染？

在Android系统中，软件渲染和硬件渲染是根据应用程序的需求和设备的性能决定的的。通常情况，系统会优先使用硬件渲染来提高图形的速度和效率。如果系统检测到设备的GPU不支持某种特定的图像特效或功能，或在硬件渲染的过程中出现问题，那么系统将退回到软件渲染。下列情况可能触发系统使用软件渲染：

1、设备的GPU不支持当前应用程序的某些特性或图像格式。

2、当前应用程序需要在屏幕上叠加多个图像层，而GPU能力不足以处理所有的层级叠加。

3、在某些情况下，使用硬件加速可能会导致应用程序出现性能问题，此时系统可能会切换到使用软件渲染。

#### SurfaceFlinger的双缓冲与VSYNC机制

![aeed01eef4364229d6c76a2bd2375b45\_MD5](https://picgo.myjojo.fun:666/i/2024/09/27/66f62bccd609c.png)

#### 缓冲机制介绍

显示缓冲机制是指定在计算机系统中，将显示数据缓存到到内存中以提高处理速度的一种计算，在图形和图像处理中，显示缓冲机制可以提升图像、图形的渲染效率和质量。

在早期的Android版本中，SurfaceFlinger使用的是三重缓冲区机制，即前台、后台和显示三个缓冲区。从Android8.0开始，SurfaceFlinger该为采用双缓冲区机制，只有前台和后台两个缓冲区。采用三重缓冲区机制的优点在于可以减少屏幕撕裂现象的产生，提高显示画面的连贯性。但同时增加了内存占用，因此在Android8.0及以上版本中，为了提高性能和减少内存占用，SurfaceFlinger改为采用双缓冲区机制。

双缓冲区机制下，前台缓冲区用于显示当前帧的内容，后台缓冲区用于绘制下一帧的内容。当绘制下一帧完成后，SurfaceFlinger会将后台缓冲区的内容交换到前台缓冲区，从而实现帧与帧之间的切换。这种机制可以减少一次缓冲区的创建及销毁操作，以提高性能和解约内存。

三重缓存机制是Android中的一种优化技术，用于提高绘制性能。它包括前缓冲区（Front Buffer）、后缓冲区（Back Buffer）和显示缓冲区（Display Buffer）。前缓冲区用于显示当前帧，后缓冲区用于绘制下一帧，而显示缓冲区则用于将前缓冲区的内容显示在屏幕上。当VSYNC信号到达时，前缓冲区和后缓冲区会进行交换，从而实现流畅的显示效果。

#### VSYNC机制介绍

在图形和图像显示中，VSYNC是一个非常重要的概念，它是指垂直同步信号，用于控制显示器刷新率和显示时序。在Android系统中，VSYNC机制被广泛应用于控制屏幕显示和动画渲染，以提高显示效果和性能。

在Android中，每台设备都有一个硬件VSYNC信号发生器，用于生成恒定的刷新率信号。通常情况下这个信号的刷新率为60Hz或90Hz，Android显示框架可以根据这个信号来调整自己的显示和渲染速度。在SurfaceFlinger的生成者消费者模型中，当硬件VSYNC信号发生时，SurfaceFlinger会开始读取缓冲区中的数据，并将其显示在屏幕上。因此，如果Android显示框架在VSYNC前完成了数据处理和绘制，则可以实现零延迟的图像或动画渲染。

VSYNC机制的主要优点是可以避免在图像渲染过程中出现屏幕撕裂和卡顿现象，同时，使用硬件信号，可以使图形和图像处理与显示保持同步，提高显示效果和渲染性能。

由于图像绘制和屏幕读取 使用的是同个buffer，所以屏幕刷新时可能读取到的是不完整的一帧画面。双缓存，让绘制和显示器拥有各自的buffer：GPU 始终将完成的一帧图像数据写入到 Back Buffer，而显示器使用 Frame Buffer，当屏幕刷新时，Frame Buffer 并不会发生变化，当Back buffer准备就绪后，它们才进行交换。

什么时候进行两个buffer的交换呢？

当扫描完一帧画面后，设备需要重新回到第一行以进入下一次的循环，此时有一段时间空隙，称为VerticalBlanking Interval(VBI)。那，这个时间点就是我们进行缓冲区交换的最佳时间。因为此时屏幕没有在刷新，也就避免了交换过程中出现 screen tearing的状况。

VSYNC 信号是由屏幕（显示设备）产生的，利用VBI时期出现的vertical sync pulse（垂直同步脉冲）来保证双缓冲在最佳时间点才进行交换。并且以 60fps 的固定频率发送给 Android 系统，Android 系统中的 SurfaceFlinger 接收发送的 VSYNC 信号。VSYNC 信号表明可对屏幕进行刷新而不会产生撕裂。另外，交换是指各自的内存地址，可以认为该操作是瞬间完成。

Android 4.4上又引入了Vsync虚拟化，通过DispSyncThread把Vsync虚拟化成Vsync-app和Vsync-sf，Vsync-app和Vsync-sf直接有固定的时机偏移，各自分别掌控着App和SurfaceFlinger的工作节奏，他们一前一后保持着绘制任务的流水节奏。

CPU/GPU根据VSYNC信号同步处理数据，可以让CPU/GPU有完整的16ms时间来处理数据，减少了jank（丢帧）。

一句话总结，VSync同步使得CPU/GPU充分利用了16.6ms时间，减少jank。

![b51202e8eb94635309ad5cd671d044cb\_MD5](https://picgo.myjojo.fun:666/i/2024/09/27/66f62bcc4ad5d.png)

应用在每个Vsync信号到来后都会通过dequeueBuffer/queueBuffer来申请buffer和提交绘图数据，Surfaceflinger都会在下一个vsync信号到来时取走buffer去做合成和显示， 并在下一下个vsync时将buffer还回来，再次循环。

![99132b2ee5d388a22a3bab7b7ade4839\_MD5](https://picgo.myjojo.fun:666/i/2024/09/27/66f62bccd8ab4.png)

## 三、SurfaceFlinger相关类和接口

如下为SurfaceFlinger相关类和接口：

![b1924e4d3b3881fc9ba91d6042dedeb0\_MD5](https://picgo.myjojo.fun:666/i/2024/09/27/66f62bccdc3c0.png)

横向角度考查Surface相关类的关系：

![7ebe399467b523e4a5fe3a89865fd682\_MD5](https://picgo.myjojo.fun:666/i/2024/09/27/66f62bcb3d7da.png)

### Region

Android UI Region是一个用于定义和操作矩形区域的类。它可以用于在屏幕上绘制和处理图形、文字和触摸事件等。Region类提供了一系列方法来创建、合并、裁剪和判断区域之间的关系。

Region代码位于：

frameworks/native/include/ui/Region.h

frameworks/native/libs/ui/Region.cpp

Region的定义：

```cpp
class Region : public LightFlattenable<Region>}
```

### ISurfaceComposer

ISurfaceComposer是Android系统中的一个接口，它是SurfaceFlinger服务的客户端与服务器端通信的主要接口，具体方法如下：

createDisplay：创建一个新的Display。

destroyDisplay：销毁指定Display。

setLayer：设置Surface的层级关系。

setAlpha：设置Surface的透明度。

setPosition：设置Surface的位置。

setSize：设置Surface的大小。

setCrop：设置Surface的裁剪区域。

setTransform：设置Surface的变换矩阵，用于实现缩放、选择效果。

ISurfaceComposer代码位于：

frameworks/native/include/gui/ISurfaceComposer.h

frameworks/native/libs/gui/ISurfaceComposer.cpp

ISurfaceComposer的定义：

```cpp
class ISurfaceComposer: public IInterface {}
```

### ISurfaceComposerClient

ISurfaceComposer是Android系统中的一个接口，它是SurfaceFlinger服务的客户端与服务器端通信的主要接口，常见方法如下：

createSurface：创建一个新的Surface。

clearLayerFrameStats：清除Layer的帧间统计信息。

getLayerFrameStats：获取指定Layer的帧间通信信息。

ISurfaceComposerClient代码位于：

frameworks/native/include/gui/ISurfaceComposerClient.h

frameworks/native/libs/gui/ISurfaceComposerClient.cpp

ISurfaceComposerClient的定义：

```cpp
class ISurfaceComposerClient : public IInterface {}
```

### SurfaceComposerClient

SurfaceComposerClient是Android系统中用于创建和管理图形表面的一个关键类，它提供了一个用于控件的图形界面API，使开发人员可以创建和操作图形表面，并将其渲染到屏幕上显示，SurfaceComposerClient的主要作用包括：

* 创建和管理Surface对象：SurfaceComposerClient类提供了创建和管理Surface对象的方法，包括创建新的表面，设置表面的大小和位置，调整表面的缩放比例和透明度等操作，开发人员可以使用SurfaceComposerClient创建和管理Surface对象，并将其添加到应用程序的UI界面中。
* 创建和管理图层：开发人员可以使用SurfaceComposerClient类创建和管理图形图层，以实现在屏幕上显示复杂的图形效果。SurfaceComposerClient为开发人员提供了一组API函数来创建和管理图层，包括设置图层大小和位置、设置图层缩放和透明度、设置混合模式和动画等。
* 与SurfaceFlinger服务的交互：SurfaceComposerClient通过IPC机制与系统级SurfaceFlinger服务通信，从而实现了应用程序和SurfaceFlinger之间的交互。开发人员可以使用SurfaceComposerClient提供的函数与SurfaceFlinger进行通信，如请求刷新屏幕、销毁表面、释放资源等。

SurfaceComposerClient代码位于：

frameworks/native/libs/gui/SurfaceComposerClient.cpp

frameworks/native/include/gui/SurfaceComposerClient.h

SurfaceComposerClient的定义：

```cpp
struct SurfaceControlStats {}
class SurfaceComposerClient : public RefBase {}
class ScreenshotClient {}
class JankDataListener : public VirtualLightRefBase {}
class TransactionCompletedListener : public BnTransactionCompletedListener {}
```

### DisplayEventReceiver

DisplayEventReceiver（显示事件接收器），用于接收来自底层硬件的显示事件。它是一个系统级别的组件，用于监听和处理与显示相关的事件，例如屏幕刷新、触摸事件等。开发者可以通过继承DisplayEventReceiver类，并实现onHotplug和onVsync方法来处理相应的事件。

DisplayEventReceiver代码位于：

frameworks/native/libs/gui/include/gui/DisplayEventReceiver.h

frameworks/native/libs/gui/DisplayEventReceiver.cpp

DisplayEventReceiver的定义：

```cpp
class DisplayEventReceiver {}
```

### SurfaceFlingerFactory

SurfaceFlinger的构造工厂。

SurfaceFlinger代码位于：

frameworks/native/service/surfaceflinger/SurfaeFlinger.h

frameworks/native/service/surfaceflinger/SurfaeFlinger.cpp

SurfaceFlinger的定义：

```cpp
class Factory {}
```

### SurfaceFlinger

SurfaceFlinger类是Android系统中的一个重要组件，它负责管理和渲染所有的图形界面。

SurfaceFlinger代码位于：

frameworks/native/service/surfaceflinger/SurfaeFlinger.h

frameworks/native/service/surfaceflinger/SurfaeFlinger.cpp

SurfaceFlinger的定义：

```cpp
class SurfaceFlinger : public BnSurfaceComposer,
                       public PriorityDumper,
                       public ClientCache::ErasedRecipient,
                       private IBinder::DeathRecipient,
                       private HWC2::ComposerCallback {}
```

SurfaceFlinger方法：

void init() ANDROID\_API：初始化

void run() ANDROID\_API：运行

void initScheduler(const sp\<DisplayDevice>& display) REQUIRES(mStateLock)：初始化调度器。

void processDisplayChangesLocked() REQUIRES(mStateLock)：处理显示变化。

void processDisplayRemoved(const wp\<IBinder>& displayToken) REQUIRES(mStateLock)：处理显示删除。

void processDisplayChanged(const wp\<IBinder>& displayToken, const DisplayDeviceState& currentState, const DisplayDeviceState& drawingState) REQUIRES(mStateLock)：处理显示变化

bool latchBuffers()：负责将应用程序的图形缓冲区（buffer）与显示器进行关联，实现图像的显示。具体来说，它的主要功能是将应用程序的图形缓冲区复制到显示器的帧缓冲区（framebuffer）中。

void onLayerFirstRef(Layer\*)：在创建一个新的图层时被调用。

void onLayerDestroyed(Layer\*)：在图层消耗时被调用。

void onLayerUpdate()：在图层更新时被调用。

//HWC2::ComposerCallback:

void onComposerHalVsync(hal::HWDisplayId, int64\_t timestamp, std::optional<hal::VsyncPeriodNanos>)：

void onComposerHalHotplug(hal::HWDisplayId, hal::Connection)：

void onComposerHalRefresh(hal::HWDisplayId)：

void onComposerHalVsyncPeriodTimingChanged(hal::HWDisplayId, const hal::VsyncPeriodChangeTimeline&)：

void onComposerHalSeamlessPossible(hal::HWDisplayId) ：

void onComposerHalVsyncIdle(hal::HWDisplayId)：

//ICompositor overrides:

bool commit(nsecs\_t frameTime, int64\_t vsyncId, nsecs\_t expectedVsyncTime) override：提交图层和显示的事务。

void composite(nsecs\_t frameTime, int64\_t vsyncId) override：用于将多个窗口的图像进行合成，主要负责对相关要进行上帧的layer进行，识别排序好，然后合成。

### Client

在Surface中，Client类是客户端进程与服务端进程之间的一个桥梁，用于完成客户端进程与服务端进程之间的通信交互，Client实现了ISurfaceComposerClient接口，Client常见方法如下：

attachLayer：将一个Layer与Surface进行关联。

detachLayer：将一个Layer与Surface分离。

getLayerUser：获取指定Layer的用户。

createSurface：创建一个新的Surface。

clearLayerFrameStats：清除Layer的帧间统计信息。

getLayerFrameStats：获取指定Layer的帧间通信信息。

Client代码位于：

frameworks/native/services/surfaceflinger/Client.h

frameworks/native/services/surfaceflinger/Client.cpp

Client的定义：

```c++
class Client : public BnSurfaceComposerClient {}
```

### Layer

Layer是SurfaceFlinger进行合成的基本操作单元。Layer在应用请求创建Surface的时候在SurfaceFlinger内部创建，因此一个Surface对应一个Layer。Layer其实是一个FrameBuffer,每个FrameBuffer中有两个GraphicBuffer记作FrontBuffer和BackBuffer。SurfaceFlinger能够创建四种类型的Layer，BufferQueueLayer，BufferStateLayer，ColorLayer，ContainerLayer，最常用的就是BufferQueueLayer。

Layer代码位于：

frameworks/native/services/surfaceflinger/Layer.h

frameworks/native/services/surfaceflinger/Layer.cpp

Layer的定义：

```cpp
class Layer : public virtual compositionengine::LayerFE {}
```

### BufferLayer

BufferLayer类是SurfaceFlinger中用于管理和处理图形缓冲区的类。

BufferLayer代码位于：

frameworks/native/services/surfaceflinger/BufferLayer.h

frameworks/native/services/surfaceflinger/BufferLayer.cpp

BufferLayer的定义：

```cpp
class BufferLayer : public Layer {}
```

### BufferQueueLayer

BufferQueueLayer是一种基于缓冲队列的图层，它使用一个缓冲队列来存储多个图像数据。当应用程序更新图像时，BufferQueueLayer会将新的图像数据添加到缓冲队列的末尾，并在每一帧显示时使用队列中的下一个缓冲区。BufferQueueLayer适用于需要处理多个图像数据的场景，例如多个应用程序同时显示图像或需要实现双缓冲机制的应用程序。

BufferQueueLayer代码位于：

frameworks/native/services/surfaceflinger/BufferQueueLayer.h

frameworks/native/services/surfaceflinger/BufferQueueLayer.cpp

BufferQueueLayer的定义：

```cpp
class BufferQueueLayer : public BufferLayer, public BufferLayerConsumer::ContentsChangedListener {}
```

### BufferStateLayer

BufferStateLayer是一种基于缓冲状态的图层，它使用一个缓冲区来存储图像数据。当应用程序更新图像时，BufferStateLayer会将新的图像数据写入缓冲区，并在下一帧显示时使用该缓冲区的内容。BufferStateLayer适用于频繁更新图像的场景，例如视频播放器或动画应用程序。

BufferStateLayer代码位于：

frameworks/native/services/surfaceflinger/BufferStateLayer.h

frameworks/native/services/surfaceflinger/BufferStateLayer.cpp

BufferStateLayer的定义：

```cpp
class BufferStateLayer : public BufferLayer {}
```

### LayerHistory

Android的LayerHistory是一个用于跟踪和管理应用程序窗口层级的类。它主要用于记录和管理应用程序中各个窗口的显示顺序和状态。

LayerHistory维护了一个窗口层级的列表，每个窗口都有一个唯一的标识符和一个状态。当应用程序中的窗口发生变化时，LayerHistory会相应地更新窗口的状态，并记录下窗口的显示顺序。

LayerHistory提供了一些方法来管理窗口层级，包括添加窗口、移除窗口、更新窗口状态等。通过这些方法，开发者可以方便地控制应用程序中各个窗口的显示和隐藏。

此外，LayerHistory还提供了一些回调方法，用于通知开发者窗口层级的变化。开发者可以通过这些回调方法来处理窗口的显示和隐藏事件，以及其他与窗口相关的操作。

LayerHistory代码位于：

frameworks/native/services/surfaceflinger/Scheduler/LayerHistory.h

frameworks/native/services/surfaceflinger/Scheduler/LayerHistory.cpp

LayerHistory的定义

```cpp
class LayerHistory {}
```

### SurfaceInterceptor

Android的SurfaceInterceptor是一个用于拦截和处理Surface相关事件的类。它是Android系统中的一个重要组件，用于在Surface的创建、销毁、绘制等过程中进行拦截和处理。

SurfaceInterceptor可以用于实现一些特定的功能，比如在Surface创建之前进行一些额外的处理，或者在Surface销毁之后进行一些清理工作。它可以拦截Surface的各种事件，包括Surface的创建、大小变化、绘制等事件，并且可以对这些事件进行处理和修改。

通过使用SurfaceInterceptor，开发者可以在Surface的生命周期中进行一些自定义的操作，例如在Surface创建之前添加一些特效，或者在Surface销毁之后进行一些资源释放。这样可以增强应用程序的功能和用户体验。

SurfaceInterceptor代码位于：

frameworks/native/services/surfaceflinger/SurfaceInterceptor.h

frameworks/native/services/surfaceflinger/SurfaceInterceptor.cpp

SurfaceInterceptor的定义：

```cpp
class SurfaceInterceptor final : public android::SurfaceInterceptor {}
```

### MonitoredProducer

MonitoredProducer类是生产者的封装类，在BufferQueueLayer::onFirstRef()中创建。

MonitoredProducer代码位于：

frameworks/native/services/surfaceflinger/MonitoredProducer.h

frameworks/native/services/surfaceflinger/MonitoredProducer.cpp

MonitoredProducer的定义：

```cpp
class MonitoredProducer : public BnGraphicBufferProducer {}
```

### Scheduler

SurfaceFlinger的调度器

Scheduler代码位于：

frameworks/native/services/surfaceflinger/Scheduler/Scheduler.h

frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp

Scheduler的定义：

```cpp
struct ISchedulerCallback {}class Scheduler : impl::MessageQueue {}
```

### FrameTimeline

Android的FrameTimeline类是一个用于跟踪应用程序帧率的工具类。它提供了一种简单的方式来测量和监视应用程序在屏幕上每一帧的渲染时间。通过使用FrameTimeline类，开发人员可以更好地了解应用程序的性能表现，并进行优化。

FrameTimeline代码位于：

frameworks/native/services/surfaceflinger/FrameTimeline/FrameTimeline.h

frameworks/native/services/surfaceflinger/FrameTimeline/FrameTimeline.cpp

FrameTimeline的定义：

```cpp
class TokenManager {}
class TraceCookieCounter {}
class SurfaceFrame {}
class FrameTimeline {}
class TokenManager : public android::frametimeline::TokenManager {
class FrameTimeline : public android::frametimeline::FrameTimeline {}
```

### EventThread

SurfaceFlinger是Android系统中负责显示管理的一个重要组件，而EventThread是SurfaceFlinger中的一个方法。EventThread方法主要用于处理输入事件。

在SurfaceFlinger中，EventThread方法是一个循环，它不断地从输入队列中获取输入事件，并将这些事件分发给合适的窗口或者Surface进行处理。具体来说，EventThread方法会执行以下几个步骤：

1. 从输入队列中获取输入事件：EventThread会从输入队列中获取最新的输入事件，包括触摸事件、按键事件等。
2. 分发事件给窗口或Surface：根据事件的类型和目标，EventThread会将事件分发给对应的窗口或Surface进行处理。例如，如果是触摸事件，EventThread会将事件发送给当前位于触摸位置下的窗口或Surface。
3. 处理事件：接收到事件的窗口或Surface会根据事件的类型进行相应的处理。例如，对于触摸事件，窗口或Surface可以根据触摸位置进行界面的滚动、缩放等操作。
4. 循环处理：EventThread会不断地循环执行上述步骤，以确保输入事件能够及时地被处理。

总结来说，SurfaceFlinger的EventThread方法是负责处理输入事件的一个循环，它将输入事件分发给对应的窗口或Surface进行处理，以实现用户与Android系统的交互。

EventThread代码位于：

frameworks/native/services/surfaceflinger/Scheduler/EventThread.h

frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp

EventThread的定义：

```cpp
class VSyncSource {}
class EventThreadConnection : public gui::BnDisplayEventConnection {}
class EventThread {}
class EventThread : public android::EventThread, private VSyncSource::Callback {}
```

EventThread方法：

```cpp
void EventThread::onVSyncEvent(nsecs_t timestamp, VSyncSource::VSyncData vsyncData)：
```

### DisplayEventDispatcher

DisplayEventDispatcher（显示事件分发器），用于分发显示事件的类。它负责将底层硬件产生的显示事件发送给相应的接收器（如DisplayEventReceiver）。DisplayEventDispatcher会根据事件的类型和目标接收器的要求，将事件分发到正确的接收器中。

DisplayEventDispatcher代码位于：

frameworks/native/libs/gui/include/gui/DisplayEventDispatcher.h

frameworks/native/libs/gui/DisplayEventDispatcher.cpp

DisplayEventDispatcher的定义：

```cpp
class DisplayEventDispatcher : public LooperCallback {}
```

### DispSyncSource

DispSyncSource是一个Android系统中的类，用于同步显示刷新。它是Android系统中用于处理显示刷新的一个重要组件。

DispSyncSource的主要作用是将应用程序的渲染帧与显示设备的刷新进行同步，以确保图像在屏幕上正确显示。它通过提供一个时间基准来协调应用程序的渲染和显示设备的刷新操作。

DispSyncSource使用了垂直同步（VSync）信号来进行同步。VSync信号是显示设备在每次刷新时发送的一个信号，用于告知系统开始下一帧的显示操作。DispSyncSource会监听VSync信号，并在接收到信号时通知应用程序进行渲染。

除了同步显示刷新外，DispSyncSource还可以提供一些其他功能，例如计算帧率、延迟等信息，以及控制应用程序的渲染速率。

总之，DispSyncSource是Android系统中用于同步应用程序渲染和显示设备刷新的重要组件，它能够确保图像在屏幕上正确显示，并提供一些其他功能来优化显示效果。

DispSyncSource代码位于：

frameworks/native/services/surfaceflinger/Scheduler/DispSyncSource.h

frameworks/native/services/surfaceflinger/Scheduler/DispSyncSource.cpp

DispSyncSource的定义：

```cpp
class CallbackRepeater {}
class DispSyncSource final : public VSyncSource {}
```

### VsyncModulator

VsyncModulator是一个用于调整垂直同步信号（Vsync）的模块。垂直同步是指在图形渲染过程中，显示器按照固定的刷新率来更新屏幕上的图像。VsyncModulator的作用是根据特定的需求，对Vsync信号进行调整，以实现更好的图像显示效果或者优化性能。

VsyncModulator可以用于以下几个方面：

1. 防止撕裂：当图像的渲染速度与显示器的刷新速率不匹配时，会出现图像撕裂现象。VsyncModulator可以通过调整Vsync信号，使得图像在显示器刷新的时候进行更新，从而避免撕裂现象的发生。
2. 优化性能：在某些情况下，图形渲染速度可能超过了显示器的刷新速率，导致性能浪费。VsyncModulator可以通过调整Vsync信号，限制图形渲染速度，以提高性能。
3. 减少延迟：VsyncModulator可以通过调整Vsync信号的时机，减少图像显示的延迟，提升用户体验。

总之，VsyncModulator是一个用于调整垂直同步信号的模块，可以用于优化图像显示效果、提高性能和减少延迟。

VsyncModulator代码位于：

frameworks/native/services/surfaceflinger/Scheduler/VsyncModulator.h

frameworks/native/services/surfaceflinger/Scheduler/VsyncModulator.cpp

VsyncModulator的定义：

```cpp
class VsyncModulator : public IBinder::DeathRecipient {}
```

### VSyncCallbackRegistration

VSyncCallbackRegistration是一个用于注册VSync回调函数的类。在Android系统中，VSync是指显示设备以固定的频率刷新屏幕的过程。通过注册VSync回调函数，应用程序可以在每次VSync事件发生时得到通知，并执行相应的操作。VSyncCallbackRegistration提供了一种机制，允许应用程序注册自己的VSync回调函数，并在每次VSync事件发生时被调用。

VSyncCallbackRegistration代码位于：

frameworks/native/services/surfaceflinger/Scheduler/VSyncCallbackRegistration.h

frameworks/native/services/surfaceflinger/Scheduler/VSyncCallbackRegistration.cpp

VSyncCallbackRegistration的定义：

```cpp
class VSyncDispatch {}
class VSyncCallbackRegistration {}
```

### VSyncDispatchTimerQueue

VSyncDispatchTimerQueue是一个用于调度VSync回调函数的队列。当应用程序注册了VSync回调函数后，这些回调函数会被添加到VSyncDispatchTimerQueue中进行调度。VSyncDispatchTimerQueue会根据VSync事件的发生时间，按照一定的顺序执行已注册的回调函数。通过使用VSyncDispatchTimerQueue，可以确保VSync回调函数在正确的时间点被执行，从而实现与屏幕刷新同步的效果。

VSyncDispatchTimerQueue 代码位于：

frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatchTimerQueue.h

frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatchTimerQueue.cpp

VSyncDispatchTimerQueue 的定义：

```cpp
class VSyncDispatchTimerQueueEntry {}
class VSyncDispatchTimerQueue : public VSyncDispatch {}
```

### MessageQueue

surfaceflinger的消息队列

MessageQueue代码位于：

frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.h

frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp

MessageQueue的定义：

```cpp
struct ICompositor {}
class Task : public MessageHandler {}
class MessageQueue {}
class MessageQueue : public android::MessageQueue {}
```

### FrameTracer

FrameTracer是SurfaceFlinger的一个工具，用于分析和调试界面渲染的性能问题。FrameTracer可以帮助开发者追踪每一帧的渲染过程，包括每个Surface的绘制时间、合成时间、刷新时间等信息。通过这些信息，开发者可以了解到每一帧的渲染耗时情况，从而找出性能瓶颈并进行优化。

FrameTracer代码位于：

frameworks/native/services/surfaceflinger/FrameTracer/FrameTracer.cpp

frameworks/native/services/surfaceflinger/FrameTracer/FrameTracer.h

FrameTracer的定义：

```cpp
class FrameTracer {}
```

## 四、SurfaceFlinger相关流程分析

### SurfaceFlinger启动流程分析

[发布目录/转载/图形显示/Android13 SurfaceFlinger启动流程分析-CSDN博客|Android13 SurfaceFlinger启动流程分析-CSDN博客](%E5%8F%91%E5%B8%83%E7%9B%AE%E5%BD%95/%E8%BD%AC%E8%BD%BD/%E5%9B%BE%E5%BD%A2%E6%98%BE%E7%A4%BA/Android13%20SurfaceFlinger%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-CSDN%E5%8D%9A%E5%AE%A2%7CAndroid13%20SurfaceFlinger%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-CSDN%E5%8D%9A%E5%AE%A2)

### SurfaceFlinger commit(提交)流程分析

[Android13 SurfaceFlinger commit(提交)流程分析-CSDN博客](/Android13%20SurfaceFlinger%20commit\(%E6%8F%90%E4%BA%A4\)%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-CSDN%E5%8D%9A%E5%AE%A2)

### SurfaceFlinger composite(合成)流程分析

[Android13 SurfaceFlinger composite(合成)流程分析-CSDN博客](/Android13%20SurfaceFlinger%20composite\(%E5%90%88%E6%88%90\)%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-CSDN%E5%8D%9A%E5%AE%A2)

### SurfaceFlinger onLayerUpdate流程分析

[Android13 SurfaceFlinger onLayerUpdate流程分析-CSDN博客](/Android13%20SurfaceFlinger%20onLayerUpdate%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-CSDN%E5%8D%9A%E5%AE%A2)

### SurfaceFlinger onComposerHalVsync流程分析

[发布目录/转载/图形显示/Android13 SurfaceFlinger onComposerHalVsync流程分析-CSDN博客|Android13 SurfaceFlinger onComposerHalVsync流程分析-CSDN博客](%E5%8F%91%E5%B8%83%E7%9B%AE%E5%BD%95/%E8%BD%AC%E8%BD%BD/%E5%9B%BE%E5%BD%A2%E6%98%BE%E7%A4%BA/Android13%20SurfaceFlinger%20onComposerHalVsync%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-CSDN%E5%8D%9A%E5%AE%A2%7CAndroid13%20SurfaceFlinger%20onComposerHalVsync%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-CSDN%E5%8D%9A%E5%AE%A2)

### SurfaceFlinger onComposerHalHotplug流程分析

[Android13 SurfaceFlinger onComposerHalHotplug流程分析-CSDN博客](/Android13%20SurfaceFlinger%20onComposerHalHotplug%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-CSDN%E5%8D%9A%E5%AE%A2)

### SurfaceFlinger onComposerHalRefresh流程分析

[Android13 SurfaceFlinger onComposerHalRefresh流程分析-CSDN博客](/Android13%20SurfaceFlinger%20onComposerHalRefresh%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-CSDN%E5%8D%9A%E5%AE%A2)

### EventThread threadMain

[Android13 EventThread threadMain流程分析-CSDN博客](/Android13%20EventThread%20threadMain%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-CSDN%E5%8D%9A%E5%AE%A2)
