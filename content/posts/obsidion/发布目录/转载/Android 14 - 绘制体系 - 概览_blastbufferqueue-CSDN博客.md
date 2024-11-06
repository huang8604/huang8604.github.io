---
title: Android 14 - 绘制体系 - 概览_blastbufferqueue-CSDN博客
author: 
created: 2024-10-11
tags:
  - clippings
  - 转载
  - blog
collections: 图形显示
source: https://blog.csdn.net/temp7695/article/details/139205476
date: 2024-11-06T06:57:53.165Z
lastmod: 2024-11-05T01:33:59.458Z
---
从Android 12开始，Android的绘制系统有结构性变化， 在绘制的生产消费者模式中，新增BLASTBufferQueue，客户端进程自行进行queue的生产和消费，随后通过Transation提交到SurfaceFlinger，如此可以使得各进程将缓存提交到SufrfaceFlinger后合并到同一事务后同步提交，在同一帧生效。实际上，从Android12到Android14整个绘制系统在各个环节也都有了或大或小的调整，比如Android13发布了1.3版本的Vulkan, Android14新增了TextureView，等等。本文基于Android14。1

## **Android 绘制系统整体架构：**

![cf0cdc25ecef8aafa7ec629537646867\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/6708ff6c6d970.png)

从上到下可以理解为“生产者（Producer）”到“消费者（Consumer）”的处理过程。

首先，从WindowManagerService的角度，每个窗口称为Window，一个Window一般是一个APP的页面，或者Status Bar，或者Navigation Bar，或者WallPaper，这些都是一个个Window。WindowManagerService（WMS）作为服务端，对所有客户端窗口的添加、层级、布局等进行统一管理。在WMS端，每个Window对应一个Surface。Surface可以理解为图像数据缓存的持有者，以及Canvas的持有者。Canvas是画布，提供了绘制各种图形的能力供开发者使用。一个客户端窗口在建立之初，会先向WMS去申请一个Surface，WMS在创建了Surface之后，通过binder返回给客户端。客户端拿到Surface后，会去创建一个BLASTBufferQueue来管理图像内存的申请。每次要使用Surface的Canvas进行绘制前，需要先向BLASTBufferQueue申请一块内存（dequeue），我们这里称为Buffer，然后再将生成的图像数据写入Buffer。这个向BLASTBufferQueue申请Buffer并写入图像数据的过程，可以认为是“生产”阶段。随后，enqueue这个buffer，将其提交给SurfaceFlinger去合成。这个阶段，可以理解为图像Buffer的“消费”阶段。

SurfaceFlinger（SF）是负责与Hardware层沟通，维护着设备挂载、VSync信号收发、Layer合成等工作。WMS的每个Surface在SurfaceFlinger中都对应生成一个Layer对象。客户端将某个Surface上的Buffer提交给SurfaceFlinger，实际上就是更新了对应Layer的Buffer数据。SurfaceFlinger调用HWComposer将这些Layer进行合成并显示在屏幕。

Android在HAL层提供称为一个Hardware Composer的组件，用于隔离与具体硬件的交互。Hardware Composer简称HWComposer或HWC2（之所以是2，是早期已有一个HWC版本，只支持软件合成）。SurfaceFlinger把Layer数据交给HWComposer，各厂商来负责HWComposer合成接口的具体实现。在合成完毕后，将数据提交到屏幕设备的缓存（一般称为Frame Buffer），屏幕就显示出画面来了。

上面的过程，可以拆解为几部分：

1. Surface的创建与管理。
2. 客户端（EndPoint）绘制（Draw）和渲染（Render）图像。
3. 第三部分，是硬件Composition（合成）工作
4. Vsync：由硬件产生的信号，用于同步framebuffer的生产和消费。SurfaceFlinger对Vsync进行了使用和管理，并向上分发给APP。Vsync是不断绘制的驱动力，也是图像缓存有序投送到屏幕的重要机制。

现在，分别讨论下四部分：

1. **Surface的****创建****与管理**

![41a9699ee80b12c7024b0eb587446d57\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/6708ff664a402.png)

在Surface的创建过程中，有几个角色贯穿其中：

PhoneWindow：一个Activity对应一个PhoneWindow，代表一个应用窗口。在AMS创建Activity之初，PhoneWindow在服务端的对应的window对象（ActivityRecord）已经添加到WMS。

ViewRootImpl：其主要作用是与服务端通信，承接外部触发的绘制调用，从而从上往下对整个View树进行绘制。可以把ViewRootImp理解为View的调度者。ViewRootImp在逻辑上是View Hierarchy的最顶层，但其并不是一个真正的View。他持有一个字View--DecorView，DecorView才是真正的View，是View树的最上层，包含着Activity的画面内容。在Activity的resume阶段，ViewRootImpl的relayout方法会将DecorView添加到WMS中，这样Activity的内容就显示了出来。逻辑上，我们可以把DecorView也理解为一个Window。Activity对应一个PhoneWindow，再通过ViewRootImpl将DecorView在WMS端添加为PhoneWindow的子Window。

WMS的Session：客户端一个进程对应WMS里的一个Session，客户端持有Session的binder客户端，在窗口添加等事务上，客户端都是通过这个Session来与WMS通信的。

WindowContainer：WMS端管理系统整体的Window体系，包括其位置、层级关系。它是通过WindowContainer这个类来表达一个Window的。DisplayContent代表一个屏幕级别的Window，DisplayArea代表一块屏幕上的一块区域，比如平板等大屏幕设备上，可能一块屏幕上同时显示多个应用区域，此时就用DisplayArea表达。WindowToken简单理解为对应一个客户端Window，比如一个应用的Activity，这里需要注意的是，Activity的WindowToken是作为ActivityRecord存在的，也就是说ActivityRecord是WindowToken的子类。而Activity的具体内容的承载者，DecorView，对应WindowState。上面所有的DisplayContent、DisplayArea、WindowToken、WindowState等，都是WindowContainer的子类，这些Window在WMS内是以window树的形式组织起来的。事实上，在DisplayContent下面，还有一个层级，称为Feature，具体的层级结构见[Android12 - WMS之WindowContainer树（DisplayArea）\_android windowcontainer-CSDN博客](https://blog.csdn.net/temp7695/article/details/136630565 "Android12 - WMS之WindowContainer树（DisplayArea）_android windowcontainer-CSDN博客")。当客户端通过Session接口调用添加DecorView时，WMS端会生成一个对应的WindowState对象，并将其作为Activity对应的ActivityRecord（也就是WindowToken）的子window。

SurfaceControl：在WMS端，每个WindowContainer对应一个SurfaceControl。SurfaceControl是WMS端管理Surface的具体对象，在WMS端，可以理解一个SurfaceControl就代表一个Surface。SurfaceControl在SurfaceFlinge端对应一个Layer，持有一个layer的句柄handle。所有的绘制动作，最后都会提交到SurfaceFinger作为Layer去合成。SurfaceControl的作用，或者Surface的作用，主要是将客户端的窗口与SurfaceFlinger的Layer关联起来。在客户端Add一个DecorView时， 在WMS端对应创建的WindowState会同时创建一个SurfaceControl、Layer，随后将SurfaceControl返回给客户端。客户端拿到SurfaceControl之后转换成Surface。后续的绘制就在这个Surface上进行。

SurfaceComposerClient：是一个Binder，主要作用是SurfaceControl调用SurfaceFlinger过程中，作为一个通道的角色。由于SurfaceControl在WMS、客户端都持有，所以客户端、WMS都可以通过这个通道调用SF。比如Layer的创建、Graphihc Buffer的提交等。

## **1. 客户端绘制和渲染**

客户端通过Surface中提供的Canvas进行绘制，Canvas是基于Skia的SKCanvas。Skia（[https://skia.org/](https://skia.org/ "https://skia.org/")）是由Google管理的开源2D（也可以支持3D）图像库，目前Android、Google Chrome、ChromeOS、Mozilla FireFox、FireFoxOS上都使用Skia作为绘制引擎。Skia可以集成OPEN GL和Vulkan进行3D绘制。Android Q以后，Skia作用被加强，即使硬件加速场景中，绘制也会先封装成Skia的GrOpList再提交给GPU。在Android 14中， Skia包的目录为external/skia。

渲染的过程是将画好的图像，进行栅格化（Rasterizer），变成一个个像素，这是一个非常耗时的过程。Android 3以前，只支持软件渲染，即Software Render。过程如下：![68e16cc08a8c89424329ac7c071be04c\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/6708ff66a7384.png)

APP在View的onDraw阶段使用Canvas绘制后，通过Skia进行软件的栅格化，即通过CPU计算，将绘制内容转化成一个个像素信息，随后投送给屏幕进行显示。由于软件渲染效率低，当下软件渲染只是作为兼容方案得以保留，默认使用硬件加速。

硬件加速的流程简单表述如下：

![1949a8de23b1f248e4cbe33b7f0564d1\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/6708ff6682e64.png)

Android将硬件加速相关能力封装在hwui组件中，hwui地址：platform/frameworks/base/libs/hwui

在硬件加速模式下，APP在onDraw中通过Canvas绘制的内容将最终被封装成DisplayList的一个个GrOp绘制命令，然后通过OpenGL或者Vulkan交由GPU进行渲染，随后将结果投送给屏幕显示。而具体是使用OpenGL还是Vulkan是可选择的。早期Android只使用OpenGL，由于Vulkan支持多线程渲染等性能方面的优势，Android逐渐倾向使用Vulkan进行渲染。另外，在哪些维度上进行硬件加速也是可选的：

![9f1bdfab2d0ff4fa5885eae27b92bc62\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/6708ff66c3c78.png)

即在整体使用硬件加速的情况下，如果某个View的绘制暂时不支持硬件加速，或者在某些位移动画上为了减少渲染成本，可以动过设置View的layerType = LAYER\_TYPE\_SOFTWARE来单纯在某个特定View上使用Software Render。

硬件加速除了利用GPU来加速渲染效率外， 本身在计算渲染范围时相较软件渲染也更加高效，即软件渲染每次更新一个View局部，将使得整个View hierarchy都重新渲染。而硬件加速如只标注有变化的部分，所谓damage area，将绘制指令保存在DisplayList中，如此大大提高渲染速度。

#### **OpenGL** **E\*\*\*\*S** **VS** **Vulkan**

以下为OpenGL ES和Vulkan在Android上发布的版本历史。

![68c2993c0de6ba0211cdda7f25340ccf\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/6708ff6a9ad3a.png)

Vulkan作为一个面向更低级别规范、跨平台的API，可以提供更细粒度的内存管理和资源管理

![186573e3338933bda50cd6a18b0511d5\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/6708ff6b9be41.png)

以下为Vulkan与OpenGL ES的使用率（from GDC 2023 [https://www.youtube.com/watch?v=C7OjI7CpjLw\&t=1188s](https://www.youtube.com/watch?v=C7OjI7CpjLw\&t=1188s "https://www.youtube.com/watch?v=C7OjI7CpjLw\&t=1188s")）：

![ad747698269a2f53cff161306b5fe4ad\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/6708ff6a83179.png)

对于未来计划，OpenGL ES将不会再有功能更新，新的功能将只会在Vulkan上支持。因此，Vulkan是未来Android主推的渲染引擎。

无论是OpenGL还是Vulkan，都需要GPU的支持。例如常见的车载高端芯片高通8155，明确标明了支持: OpenGL ES 3.x、Vulkan[https://www.qualcomm.com/content/dam/qcomm-martech/dm-assets/documents/qul7413\_sa8155\_productbrief\_r4.pdf](https://www.qualcomm.com/content/dam/qcomm-martech/dm-assets/documents/qul7413_sa8155_productbrief_r4.pdf "https://www.qualcomm.com/content/dam/qcomm-martech/dm-assets/documents/qul7413_sa8155_productbrief_r4.pdf")

## **2. HW** **Composer**\*\*（HWC2）\*\***图像合成**

前面提到过，每个window对应一个SF中的Layer，合成（Composition）工作就是将这些Layer进程合并成一个完整的屏幕内容，提交给硬件屏幕显示出来。大概过程如下：

![f4b02a02af8af825b8014f375ba3d035\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/6708ff6acc505.png)

这个页面有三个Layer：StatusBar、NavigationBar和中间的APP内容页面，其中可能会有重叠的部分，称为Overlay。Composition的工作就是将这三个Layer合并成一个画面，计算重叠部分的颜色，提交给屏幕显示出来。

合成的工作发生在渲染后的内容提交给SurfaceFlinger之后。大致流程如下：

![3732c69b0236014961750502068f9fa3\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/6708ff6bdee11.png)

合成有硬件合成的部分和软件合成的部分。硬件合成除了更高效的同时，可以将合成工作从GPU解放出来，提高GPU效率，节省能耗。嵌入式设备的SOC中，硬件的合成一般由独立的DPU（Display Processing）完成。

比如高通SA8155这款SOC的布局如下：

![c78362a611392a99389a11a5b4cd0f18\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/6708ff6dce145.png)

其中GPU的部分负责渲染，“Dispay Processing”的部分用来处理合成工作。

由于硬件对合成Layer数量是有限制的，例如高通QCS2290支持4个Layer、AMD有的芯片支持7个等）以及Layer的PixelFormat（比如支持PIXEL\_FORMAT\_RGBA\_8888，不支持YUV）是有限制的，因此在硬件合成之前，如果合成Layer过多或者Format不满足要求，会需要使用GPU先进行一轮软件合成，合并或转换一些Layer的格式。

![95c82c9537ef99c6eb5ff34b2851a230\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/6708ff683be92.png)

![50e4bd399b933bc8f5d57898657d7c51\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/6708ff68e17e0.png)

软件合成过程。（Google I/O '18）

## **4. V\*\*\*\*SYNC**

#### **VSync简介：**

首先关注两个重要概念：

**refresh** **rate** - 60Hz ：代表每秒钟屏幕可以更新多少次，这一值早期是固定的，依赖于硬件。现代旗舰设备的屏幕都支持多个刷新率，从60Hz~165Hz不等，而且是可以由App层定制刷新率。

![cf6858514b95f0902f03bbe8db613e86\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/6708ff6b9b007.png)

**frame rate**：每秒钟GPU可以绘制多少帧，值越大越好

![a4f7b8ed69c217f89e235fc5a76ec2e4\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/6708ff695377f.png)

VSync是一个通用概念，在Linux、PC、移动设备上都有所实现。

想象一下绘制过程是这样的：GPU绘制数据，将绘制结果投掷给屏幕显示出来。

![a95fdfd2c1c22d3a60530a6af8d83519\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/6708ff6e1b3a7.png)

问题是，refresh rate和Frame Rate并不保证是一致的频率，也就是是说GPU渲染的时间并不能保证就正好是16ms（60Hz）内完成的。如果只有一块内存（Frame Buffer）用来交换数据，假如Refresh Rate大于Frame Rate，由于GPU是从上到下写这块内存的，在当屏幕来取数据的时候，GPU刚刚在旧帧基础上写了一半的新帧，此时就会出现图片撕裂问题，如：

![ac576080581ef956894d564fa665a6eb\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/6708ff6e9958d.png)

解决方法是双缓存方案：

提供Back Buffer和Frame Buffer两个缓存，屏幕始终从Frame Buffer取数据显示，GPU往Back Buffer里写，当GPU完全将数据写好后，再将Back Buffer整个拷贝到Frame Buffer。这样就能保证屏幕每次都取到完整的帧。

![81f699547147eb8ff9bbdc9daa09fd66\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/6708ff6eab452.png)

此时仍有一个问题，如果GPU的Frame Rate大于屏幕的Refresh Rate，那么屏幕再取到下一帧前，可能GPU都写完好几帧了，就会出现丢帧现象。此时就需要VSync：

![f004223d535f85cf477b256ad26ac5a4\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/6708ff6e6afc0.png)

屏幕根据自己的刷新频率，去给上层发送一个VSync信号，GPU在拿到这个VSync信号后，才去绘制。这样就能同步屏幕与上层绘制的节奏了。

![6d6f4f3f715eda915493456e5e71dc38\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/6708ff6e34301.png)

如果屏幕的Refresh Rate大于GPU的Frame Rate怎么办？

![2fc3a238855a303d088fa40636620f34\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/6708ff6dcb246.png)

屏幕将会仍然显示旧帧。比如中间方框的两次刷新，屏幕仍然显示前一次的帧内容。

#### **Android的V\*\*\*\*SYNC**

实际上Android的VSync要复杂得多，主要由SurfaceFlinger负责实现。通过之前的介绍我们知道一帧的绘制过程有APP绘制渲染、SurfaceFlinger合成、Display硬件读取帧缓存显示图片三个阶段，如果每一个阶段都依赖VSync信号来执行，那可能会出现这种情况：

![f660323b4324fd57fa1a92a198f83148\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/6708ff6981a1f.png)

也就是说，VSync1的时候APP正在绘制渲染，SF还没有可以合成的东西，所以什么都不做；等到VSync2的时候，Render1的工作已经完成，可以做合成了；VSync3的时候，合成做完了，才可以显示到屏幕上。从绘制渲染到显示经历了3个VSync。面对这种情况，Android对VSync的设计如下：

![f09c930c8ea399518652623c1a9e5539\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/6708ff6a50d83.png)

即有三种信号

HW\_VSYNC\_\[ID]：由底层硬件按Refresh Rate的频率发出，一般为60Hz、90Hz、120H等等，随后会通过HWC通知给SurfaceFlinger。

VSYNC-app：SurfaceFlinger通知给上层应用的VSYNC，用于控制和驱动应用的绘制渲染。

VSYNC-sf:通知给SurfaceFlinger自身的，用于合成Layer的信号。

VSYNC-app和VSYNC-sf相对于HW\_VSYNC\_\[ID]，并不是同步发送的，而是有一定的延迟，称为相位差。从HW\_VSYNC\_\[ID]到VSYNC-app发出的时间差称为app phase，HW\_VSYNC\_\[ID]到VSYNC-sf发出的时间差称为sf phase。这种设计的好处是，如果在同一个VSync周期内，经sf phase后在执行合成时恰好前一步的Render完成了，就可在一个周期完成两步，而不用非得等下一个VSync。

另外，Android并非直接把硬件的HW\_VSYNC\_\[ID]信号直接分发给应用和SurfaceFlinger，而是通过先收集HW\_VSYNC\_\[ID]样本，再根据屏幕Refresh Rate、预先配置的相位差等信息，经过计算后模拟出来的VSYNC-app和VSYNC-sf。

![e3eb9556840a382cceecdea3d73af1f8\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/6708ff69aa46e.png)

由于只需要一定的硬件VSync样本后便可以模拟出预期的VSYNC-app和VSYNC-sf，因此并不需要一直接收HW\_VSYNC\_\[ID]信号，在收到足够的样本数后（在Android 14中为6个）就可以关闭硬件VSync的接收。在每次将合成数据提交给屏幕后，会返回一个硬件VSync时间戳（PresentFence），此时SF会对比当前模拟VSync与硬件VSync是否误差过大，如果过大，会重新打开硬件VSync收集样本重新计算。另外，每次终端应用主动请求VSync时，也会判断前后两次模拟的VSync时间差是否超过750ms，如果是则重新请求打开硬件VSync。在systrace上硬件VSync打开的TAG是HW\_VSYNC\_ON\_\[ID]。

参考资料：[https://source.android.com/docs/core/graphics/implement-vsync](https://source.android.com/docs/core/graphics/implement-vsync "https://source.android.com/docs/core/graphics/implement-vsync")

#### **可变刷新率**

现代旗舰机屏幕的刷新率是可变的，比如Pixel 5：

![b99c51dcd12ba54836fa82d9a8d7eeaf\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/6708ff6c1761b.png)

可以看到，该屏幕是支持60Hz、90Hz两种刷新率的。

而且应用层也可以在应用级别、窗口级别指定具体的刷新率。在经过应用层指定后，最终的刷新率并不一定是指定的值，而是经过SurfaceFlinger综合计算后得出。具体见[https://developer.android.com/media/optimize/performance/frame-rate](https://developer.android.com/media/optimize/performance/frame-rate "https://developer.android.com/media/optimize/performance/frame-rate")
