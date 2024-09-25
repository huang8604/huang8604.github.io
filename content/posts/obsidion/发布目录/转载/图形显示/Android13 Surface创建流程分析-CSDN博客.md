---
title: Android13 Surface创建流程分析-CSDN博客
source: https://blog.csdn.net/liuning1985622/article/details/138463081
author: 
published: 
created: 2024-09-25
tags:
  - clippings
  - blog
  - Framework
  - 转载
collections:
  - 图形显示
date: 2024-09-25T09:39:12.990Z
lastmod: 2024-09-25T09:42:40.225Z
---
Surface是[Android](https://so.csdn.net/so/search?q=Android\&spm=1001.2101.3001.7020)中用于表示一个图像缓冲区的类，Surface是在SurfaceControl创建时创建的，代码如下：

```cpp
class Client : public BnSurfaceComposerClientstatus_t Client::createSurface(const String8& name, uint32_t w, uint32_t h, PixelFormat format,                               uint32_t flags, const sp<IBinder>& parentHandle,                               LayerMetadata metadata, sp<IBinder>* handle,                               sp<IGraphicBufferProducer>* gbp) {return mFlinger->createLayer(name, this, w, h, format, flags, std::move(metadata), handle, gbp, parentHandle);}}
```

## SurfaceFlinger createLayer

mFlinger是SurfaceFlinger，所以Client的具体实现还是依靠的SurfaceFlinger，并且注意，在应用层的Surface在SurfaceFlinger进程名叫Layer，它和应用层的Surface是一 一对应的关系，我们来看createLayer函数：

```cpp
class SurfaceFlinger : public BnSurfaceComposer,public PriorityDumper,private IBinder::DeathRecipient,private HWC2::ComposerCallback,private ICompositor,private scheduler::ISchedulerCallback {status_t SurfaceFlinger::createLayer(const String8& name, const sp<Client>& client, uint32_t w,                                     uint32_t h, PixelFormat format, uint32_t flags,                                     LayerMetadata metadata, sp<IBinder>* handle,                                     sp<IGraphicBufferProducer>* gbp,                                     const sp<IBinder>& parentHandle,                                     const sp<Layer>& parentLayer) {if (int32_t(w|h) < 0) {ALOGE("createLayer() failed, w or h is negative (w=%d, h=%d)",int(w), int(h));return BAD_VALUE;    }status_t result = NO_ERROR;    sp<Layer> layer; 	......switch (flags & ISurfaceComposerClient::eFXSurfaceMask) {case ISurfaceComposerClient::eFXSurfaceBufferQueue:            result = createBufferQueueLayer(client, uniqueName, w, h, flags, std::move(metadata),                                            format, handle, gbp, &layer);break;case ISurfaceComposerClient::eFXSurfaceBufferState:            result = createBufferStateLayer(client, uniqueName, w, h, flags, std::move(metadata),                                            handle, &layer);break;case ISurfaceComposerClient::eFXSurfaceColor:if (w > 0 || h > 0) {ALOGE("createLayer() failed, w or h cannot be set for color layer (w=%d, h=%d)",int(w), int(h));return BAD_VALUE;            }            result = createColorLayer(client, uniqueName, w, h, flags, std::move(metadata), handle,                                      &layer);break;case ISurfaceComposerClient::eFXSurfaceContainer:if (w > 0 || h > 0) {ALOGE("createLayer() failed, w or h cannot be set for container layer (w=%d, h=%d)",int(w), int(h));return BAD_VALUE;            }            result = createContainerLayer(client, uniqueName, w, h, flags, std::move(metadata),                                          handle, &layer);break;default:            result = BAD_VALUE;break;    }    .......return result;}}
```

### SurfaceFlinger createBufferQueueLayer

上面函数的核心代码就是根据应用请求[不同的](https://so.csdn.net/so/search?q=%E4%B8%8D%E5%90%8C%E7%9A%84\&spm=1001.2101.3001.7020)flag创建不同的显示Layer，从上面代码看创建的Layer有四种类型，我们看看系统中大多数界面的Layer ，flage 为eFXSurfaceBufferQueueyer，我们就大致看看createBufferQueueLayer函数。

```cpp
status_t SurfaceFlinger::createBufferQueueLayer(const sp<Client>& client, const String8& name,                                                uint32_t w, uint32_t h, uint32_t flags,                                                LayerMetadata metadata, PixelFormat& format,                                                sp<IBinder>* handle,                                                sp<IGraphicBufferProducer>* gbp,                                                sp<Layer>* outLayer) {switch (format) {case PIXEL_FORMAT_TRANSPARENT:case PIXEL_FORMAT_TRANSLUCENT:        format = PIXEL_FORMAT_RGBA_8888;break;case PIXEL_FORMAT_OPAQUE:        format = PIXEL_FORMAT_RGBX_8888;break;    }    sp<BufferQueueLayer> layer = getFactory().createBufferQueueLayer(LayerCreationArgs(this, client, name, w, h, flags, std::move(metadata)));status_t err = layer->setDefaultBufferProperties(w, h, format);if (err == NO_ERROR) {        *handle = layer->getHandle();        *gbp = layer->getProducer();        *outLayer = layer;    }return err;}
```

可以看到上面函数的核心是调用getFactory().createBufferQueueLayer，最终创建的是Layer的子类BufferQueueLayer：

```cpp
sp<BufferQueueLayer> createBufferQueueLayer(const LayerCreationArgs& args) override {return new BufferQueueLayer(args);}
```

SurfaceFlingerFactory的createBufferQueueLayer函数只是new了一个BufferQueueLayer，继续看BufferQueueLayer[构造函数](https://so.csdn.net/so/search?q=%E6%9E%84%E9%80%A0%E5%87%BD%E6%95%B0\&spm=1001.2101.3001.7020)：

```cpp
BufferQueueLayer::BufferQueueLayer(const LayerCreationArgs& args) : BufferLayer(args) {}
```

#### BufferQueueLayer onFirstRef

它的构造函数中其实没做什么事情，但是别忘了这个类终极父类是RefBase，BufferQueueLayer被new出来的时候会走进它的onFirstRef()方法里面：

[Android13 BufferQueueLayer onFirstRef流程分析-CSDN博客](https://blog.csdn.net/liuning1985622/article/details/138467228?spm=1001.2014.3001.5502 "Android13 BufferQueueLayer onFirstRef流程分析-CSDN博客")
