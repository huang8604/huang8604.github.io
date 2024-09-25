---
title: Android Surface-CSDN博客
source: https://blog.csdn.net/liuning1985622/article/details/138462996
author: 
created: 2024-09-25
tags:
  - clippings
  - blog
  - 转载
collections:
  - 图形显示
date: 2024-09-25T09:26:31.080Z
lastmod: 2024-09-25T09:42:30.039Z
---
## 一、Surface介绍

在Android系统中，Surface是一种用于图形和视频渲染的抽象概念，它可以用来将应用程序绘制的图形或视频显示在屏幕上。一个Surface代表一个屏幕表面，可以是整个屏幕或者应用程序UI的一个独立窗口。

Surface通过一个SurfaceHolder对象来提供访问接口。SurfaceHolder管理了Surface的[生命周期](https://so.csdn.net/so/search?q=%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F\&spm=1001.2101.3001.7020)和绘画信息，并提供了锁定(SurfaceHolder.lockCanvas())和解锁(SurfaceHolder.lockCanvas())和解锁(SurfaceHolder.unlockCanvasAndPost())Canvas对象接口，使得应用程序可以直接在Surface上进行绘制操作。

一个Surface可以包含多个Buffer，每个Buffer都包含了一个图像或视频的副本，应用程序可以在一个Buffer中渲染图形或视频数据，而使用另一个Buffer时，只需要将绘图信息提交后，即可直接显示另一个Buffer的图像或视频内容，从而提高了渲染效率和性能。

在Android系统中，Surface还被广泛用于多媒体、游戏和[图形渲染](https://so.csdn.net/so/search?q=%E5%9B%BE%E5%BD%A2%E6%B8%B2%E6%9F%93\&spm=1001.2101.3001.7020)等应用程序场景。例如，MediaCodec和MediaPlayer类使用Surface作为视频输出的目标，OpenGL ES库也使用Surface作为渲染目标，而游戏引擎中也常常会使用Surface作为游戏画面的输出目标。

### surface和surfaceflinger之间的关系

一个surface与surfaceflinger中一个layer一一对应。layer中有bufferqueue用于接收surface通过producer发过来的图形数据。

![f0498d5488fdf3cfad66b1086af3b95c\_MD5](https://picgo.myjojo.fun:666/i/2024/09/25/66f3d7a09ed5a.png)

### surface的绘制流程

surface的绘制流程分别LockCanvas->drawBitmap->UnlockCanvasAndPost几个阶段，最后通知SurfaceFlinger进行合成，整体流程如下：

![522b95594154bda4fa8914d1d8551f03\_MD5](https://picgo.myjojo.fun:666/i/2024/09/25/66f3d7a0abd02.png)

## 三、Surface相关类和接口

### JAVA类

#### Surface

Surface是Android中用于表示一个图像缓冲区的类，Surface包括JAVA部分，JNI和C++部分。

Surface代码位于：

frameworks/base/core/java/android/view/Surface.java

Surface的定义：

```java
public class Surface implements Parcelable {}
```

#### SurfaceControl

SurfaceControl是一个用于创建和管理Surface的类，其中的Surface对象就是一个用于图像展示的承载器。而SurfaceControl对象，则是用于对Surface对象进行操作的控制器。使用SurfaceControl，应用程序可以根据需要创建新的Surface，或将现有的Surface与当前会话进行关联，通过SurfaceControl可以对Surface属性进行设置，例如对Surface进行裁剪、设置Surface位置和大小、设置Surface的透明度等。

SurfaceControl通过SurfaceSession来和SurfaceFlinger进行交互、完成对Surface的创建、设置和控制等操作，SurfaceControl包括JAVA部分，JNI和C++部分。

SurfaceControl代码位于：

frameworks/base/core/java/android/view/SurfaceControl.java

SurfaceControl的定义：

```java
public final class SurfaceControl implements Parcelable {public static class Builder {}}
```

#### SurfaceSession

SurfaceSession是Android系统中与图形表面相关的一个关键类，它提供了与SurfaceFlinger服务通信以创建和管理图形表面连接的API，SurfaceSession的主要作用包括：

* 创建和管理图形表面连接：SurfaceSession充当了应用程序和系统级SurfaceFlinger服务之间的中介，通过IPC机制与SurfaceFlinger通信，创建和管理图形表面连接。
* 分配和管理表面标识符：图形表面在Android系统中是由唯一的标识符来进行识别的和管理的，SurfaceSession类会分配和管理这些标识符，避免表面之间出现冲突和重复问题 。同时，SurfaceSession还会将每个表面与其所属的进程关联起来，以确保安全和可靠的表面交互。
* 提供与表面相关的API接口：SurfaceSession类提供了一系列与表面相关的API接口，包括创建和设置表面属性，通过BufferQueue和IGraphicBufferProducer进行表面缓冲区管理和交互等。这些API接口为开发人员提供了方便的方法来创建和管理表面，同时实现了内部和外部之间的隔离，确保了系统的安全和稳定性。

SurfaceSession代码位于：

frameworks/base/core/java/android/view/SurfaceSession.java

SurfaceSession的定义：

```java
public final class SurfaceSession {}
```

### C++类

#### Surface

Surface是Android中用于表示一个图像缓冲区的类，Surface包括JAVA部分，JNI和C++部分。

Surface代码位于：

frameworks/base/core/jni/android\_view\_Surface.cpp

framework/native/libs/gui/Surface.cpp

frameworks/native/include/gui/Surface.h

Surface的定义：

```cpp
class Surface : public ANativeObjectBase<ANativeWindow, Surface, RefBase> {}
```

Surface方法：

```cpp
int connect(int api, const sp<IProducerListener>& listener)：连接int Surface::disconnect(int api, IGraphicBufferProducer::DisconnectMode mode) ：断开int queueBuffers(const std::vector<BatchQueuedBuffer>& buffers)：生产者填充缓存区并返回给队列int dequeueBuffers(std::vector<BatchBuffer>* buffers)：生产者请求一块空闲的缓存区int cancelBuffers(const std::vector<BatchBuffer>& buffers)：关闭缓冲区
```

#### SurfaceControl

SurfaceControl是一个用于创建和管理Surface的类，其中的Surface对象就是一个用于图像展示的承载器。而SurfaceControl对象，则是用于对Surface对象进行操作的控制器。使用SurfaceControl，应用程序可以根据需要创建新的Surface，或将现有的Surface与当前会话进行关联，通过SurfaceControl可以对Surface属性进行设置，例如对Surface进行裁剪、设置Surface位置和大小、设置Surface的透明度等。

SurfaceControl通过SurfaceSession来和SurfaceFlinger进行交互、完成对Surface的创建、设置和控制等操作，SurfaceControl包括JAVA部分，JNI和C++部分。

SurfaceControl代码位于：

frameworks/base/core/jni/android\_view\_SurfaceControl.cpp

frameworks/native/libs/gui/SurfaceControl.cpp

frameworks/native/include/gui/SurfaceControl.h

SurfaceControl的定义：

```cpp
public final class SurfaceControl implements Parcelable {}class SurfaceControl : public RefBase{}
```

#### ComposerService

This holds our connection to the composer service (i.e. SurfaceFlinger).

ComposerService代码位于：

frameworks/native/libs/gui/include/private/gui/ComposerService.h

ComposerService的定义：

```cpp
class ComposerService : public Singleton<ComposerService> {}
```

#### ComposerServiceAIDL

This holds our connection to the composer service (i.e. SurfaceFlinger).

ComposerServiceAIDL 代码位于：

frameworks/native/libs/gui/include/private/gui/ComposerServiceAIDL .h

ComposerServiceAIDL 的定义：

```cpp
class ComposerServiceAIDL : public Singleton<ComposerServiceAIDL> {}
```

## 三、Surface相关流程分析

### \[\[Android13 Surface创建流程分析-CSDN博客]]

### SurfaceControl创建流程分析

### SurfaceSession创建流程分析

### Surface connect流程分析

### Surface lockCanvas流程分析

### Surface unlockCanvasAndPost流程分析

### Surface dequeueBuffer流程分析

### Surface queueBuffer流程分析
