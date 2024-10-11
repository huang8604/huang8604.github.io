---
title: Android13 Surface创建流程分析-CSDN博客
author: 
created: 2024-09-25
tags:
  - clippings
  - blog
  - Framework
  - 转载
collections:
  - 图形显示
date: 2024-09-25T09:39:12.990Z
lastmod: 2024-10-09T08:50:35.835Z
---
Surface是Android中用于表示一个图像缓冲区的类，Surface是在SurfaceControl创建时创建的，代码如下：

```cpp
//frameworks/native/services/surfaceflinger/Client.cpp
class Client : public BnSurfaceComposerClient
status_t Client::createSurface(const String8& name, uint32_t w, uint32_t h, PixelFormat format,
                               uint32_t flags, const sp<IBinder>& parentHandle,
                               LayerMetadata metadata, sp<IBinder>* handle,
                               sp<IGraphicBufferProducer>* gbp) {
    // We rely on createLayer to check permissions.
    return mFlinger->createLayer(name, this, w, h, format, flags, std::move(metadata), handle, gbp, parentHandle);
}
}
```

## SurfaceFlinger createLayer

mFlinger是SurfaceFlinger，所以Client的具体实现还是依靠的SurfaceFlinger，并且注意，在应用层的Surface在SurfaceFlinger进程名叫Layer，它和应用层的Surface是一 一对应的关系，我们来看createLayer函数：

```cpp
//frameworks/native/service/SurfaceFlinger/SurfaeFlinger.cpp
class SurfaceFlinger : public BnSurfaceComposer,
                       public PriorityDumper,
                       private IBinder::DeathRecipient,
                       private HWC2::ComposerCallback,
                       private ICompositor,
                       private scheduler::ISchedulerCallback {
status_t SurfaceFlinger::createLayer(const String8& name, const sp<Client>& client, uint32_t w,
                                     uint32_t h, PixelFormat format, uint32_t flags,
                                     LayerMetadata metadata, sp<IBinder>* handle,
                                     sp<IGraphicBufferProducer>* gbp,
                                     const sp<IBinder>& parentHandle,
                                     const sp<Layer>& parentLayer) {
    //宽高合法检查
    if (int32_t(w|h) < 0) {
        ALOGE("createLayer() failed, w or h is negative (w=%d, h=%d)",
                int(w), int(h));
        return BAD_VALUE;
    }
 
 
    status_t result = NO_ERROR;
    //layer
    sp<Layer> layer;
 
 
 	......
    //根据不同flag创建不同layer
    switch (flags & ISurfaceComposerClient::eFXSurfaceMask) {
        case ISurfaceComposerClient::eFXSurfaceBufferQueue:
            result = createBufferQueueLayer(client, uniqueName, w, h, flags, std::move(metadata),
                                            format, handle, gbp, &layer);
 
 
            break;
        case ISurfaceComposerClient::eFXSurfaceBufferState:
            result = createBufferStateLayer(client, uniqueName, w, h, flags, std::move(metadata),
                                            handle, &layer);
            break;
        case ISurfaceComposerClient::eFXSurfaceColor:
            // check if buffer size is set for color layer.
            if (w > 0 || h > 0) {
                ALOGE("createLayer() failed, w or h cannot be set for color layer (w=%d, h=%d)",
                      int(w), int(h));
                return BAD_VALUE;
            }
 
 
            result = createColorLayer(client, uniqueName, w, h, flags, std::move(metadata), handle,
                                      &layer);
            break;
        case ISurfaceComposerClient::eFXSurfaceContainer:
            // check if buffer size is set for container layer.
            if (w > 0 || h > 0) {
                ALOGE("createLayer() failed, w or h cannot be set for container layer (w=%d, h=%d)",
                      int(w), int(h));
                return BAD_VALUE;
            }
            result = createContainerLayer(client, uniqueName, w, h, flags, std::move(metadata),
                                          handle, &layer);
            break;
        default:
            result = BAD_VALUE;
            break;
    }
 
 
    .......
    return result;
}
}
```

### SurfaceFlinger createBufferQueueLayer

上面函数的核心代码就是根据应用请求不同的flag创建不同的显示Layer，从上面代码看创建的Layer有四种类型，我们看看系统中大多数界面的Layer ，flage 为eFXSurfaceBufferQueueyer，我们就大致看看createBufferQueueLayer函数。

```cpp
//frameworks/native/service/surfaceflinger/SurfaeFlinger.cpp
status_t SurfaceFlinger::createBufferQueueLayer(const sp<Client>& client, const String8& name,
                                                uint32_t w, uint32_t h, uint32_t flags,
                                                LayerMetadata metadata, PixelFormat& format,
                                                sp<IBinder>* handle,
                                                sp<IGraphicBufferProducer>* gbp,
                                                sp<Layer>* outLayer) {
    // initialize the surfaces
    switch (format) {
    case PIXEL_FORMAT_TRANSPARENT:
    case PIXEL_FORMAT_TRANSLUCENT:
        format = PIXEL_FORMAT_RGBA_8888;
        break;
    case PIXEL_FORMAT_OPAQUE:
        format = PIXEL_FORMAT_RGBX_8888;
        break;
    }
 
 
    sp<BufferQueueLayer> layer = getFactory().createBufferQueueLayer(
            LayerCreationArgs(this, client, name, w, h, flags, std::move(metadata)));
    status_t err = layer->setDefaultBufferProperties(w, h, format);
    if (err == NO_ERROR) {
        *handle = layer->getHandle();
        *gbp = layer->getProducer();
        *outLayer = layer;
    }
 
 
    return err;
}
```

可以看到上面函数的核心是调用getFactory().createBufferQueueLayer，最终创建的是Layer的子类BufferQueueLayer：

```cpp
//frameworks/native/service/surfaceflinger/SurfaeFlinger.cpp
sp<BufferQueueLayer> createBufferQueueLayer(const LayerCreationArgs& args) override {
    return new BufferQueueLayer(args);
}
```

SurfaceFlingerFactory的createBufferQueueLayer函数只是new了一个BufferQueueLayer，继续看BufferQueueLayer\[构造函数]：

```cpp
//frameworks/native/services/surfaceflinger/BufferQueueLayer.cpp
BufferQueueLayer::BufferQueueLayer(const LayerCreationArgs& args) 
: BufferLayer(args) {}
```

#### \[\[Android13 BufferQueueLayer onFirstRef流程分析-CSDN博客]]

它的构造函数中其实没做什么事情，但是别忘了这个类终极父类是RefBase，BufferQueueLayer被new出来的时候会走进它的onFirstRef()方法里面：
