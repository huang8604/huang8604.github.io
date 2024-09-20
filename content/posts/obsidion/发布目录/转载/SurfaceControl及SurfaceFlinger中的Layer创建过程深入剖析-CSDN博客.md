---
title: SurfaceControl及SurfaceFlinger中的Layer创建过程深入剖析-CSDN博客
tags:
  - clippings
  - blog
  - 转载
collections:
  - SurfaceControl
  - Framework
date: 2024-09-20T07:43:33.100Z
lastmod: 2024-09-20T07:53:40.620Z
---
### SurfaceComposerClient创建

SurfaceComposerClient对象是在哪里创建的呢?是在SurfaceSession构造时候创建\
frameworks/base/core/[jni](https://so.csdn.net/so/search?q=jni\&spm=1001.2101.3001.7020)/android\_view\_SurfaceSession.cpp

```cpp
static jlong nativeCreate(JNIEnv* env, jclass clazz) {
    SurfaceComposerClient* client = new SurfaceComposerClient();
    client->incStrong((void*)nativeCreate);
    return reinterpret_cast<jlong>(client);
}
```

这里的SurfaceSession是在哪里创建的呢？这里直接上堆栈

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/3411249efc552b899d80bae4cda25790.png#pic_center)\
实际是在ViewRootImpl构造期间就创建\
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/9e7b1e7eb9381fc78e89b8f4a6b36d9a.png#pic_center)

### SurfaceComposerClient的mClient初始化

这里的mClient是谁呢？类型是\
sp mClient;\
看得出是一个接口

看下SurfaceComposerClient构造时候会与sf进行跨进程createConnection创建链接，返回的对象就是Client对象。

```cpp
void SurfaceComposerClient::onFirstRef() {
    sp<ISurfaceComposer> sf(ComposerService::getComposerService());
    if (sf != nullptr && mStatus == NO_INIT) {
        sp<ISurfaceComposerClient> conn;
        conn = sf->createConnection();
        if (conn != nullptr) {
            mClient = conn;
            mStatus = NO_ERROR;
        }
    }
}
```

这里又出现了ComposerService::getComposerService()\
frameworks/native/libs/[gui](https://so.csdn.net/so/search?q=gui\&spm=1001.2101.3001.7020)/SurfaceComposerClient.cpp

```cpp
sp<ISurfaceComposer> ComposerService::getComposerService() {
    ComposerService& instance = ComposerService::getInstance();
    if (instance.mComposerService == nullptr) {
        if (ComposerService::getInstance().connectLocked()) {
            WindowInfosListenerReporter::getInstance()->reconnect(instance.mComposerService);
        }
    }
    return instance.mComposerService;
}
//connectLocked实现
bool ComposerService::connectLocked() {
    const String16 name("SurfaceFlinger");
    //就是获取sf的bp代理，赋值给mComposerService
    mComposerService = waitForService<ISurfaceComposer>(name);
    return true;
}
```

所以上面onFirstRef中sf->createConnection()调用到sf了，sf代码如下\
frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp

```cpp
sp<ISurfaceComposerClient> SurfaceFlinger::createConnection() {
    const sp<Client> client = new Client(this);
    return client->initCheck() == NO_ERROR ? client : nullptr;
}
```

sf端就是简单创建了一个Client对象，这里来看看\
其实Client本质是一个Binder对象的BpBinder即跨进程的代理，远端的BnBinder在SurfaceFlinger的Client.cpp\
思考为啥有了sf代理，还需要Client：\
哈哈，其实这里主要是为了帮助sf进行解耦一部分工作

### SurfaceControl，Layer创建

frameworks/base/core/jni/android\_view\_SurfaceControl.cpp

```cpp
static jlong nativeCreate(JNIEnv* env, jclass clazz, jobject sessionObj,
        jstring nameStr, jint w, jint h, jint format, jint flags, jlong parentObject,
        jobject metadataParcel) {
    sp<SurfaceComposerClient> client;
  
    status_t err = client->createSurfaceChecked(String8(name.c_str()), w, h, format, &surface,
                                                flags, parentHandle, std::move(metadata));
  
    surface->incStrong((void *)nativeCreate);
    return reinterpret_cast<jlong>(surface.get());
}
```

主要调用到了SurfaceComposerClient的createSurfaceChecked。

下面看看 SurfaceComposerClient::createSurfaceChecked方法

```cpp

status_t SurfaceComposerClient::createSurfaceChecked(const String8& name, uint32_t w, uint32_t h,
                                                     PixelFormat format,
                                                     sp<SurfaceControl>* outSurface, uint32_t flags,
                                                     const sp<IBinder>& parentHandle,
                                                     LayerMetadata metadata,
                                                     uint32_t* outTransformHint) {
 
        err = mClient->createSurface(name, w, h, format, flags, parentHandle, std::move(metadata),
                                     &handle, &gbp, &id, &transformHint);
    } 
        if (err == NO_ERROR) {
            *outSurface =
                    new SurfaceControl(this, handle, gbp, id, w, h, format, transformHint, flags);
        }
    return err;
}
```

最后会调用到Client的createSurface\
frameworks/native/services/surfaceflinger/Client.cpp

```cpp
status_t Client::createSurface(const String8& name, uint32_t /* w */, uint32_t /* h */,
                               PixelFormat /* format */, uint32_t flags,
                               const sp<IBinder>& parentHandle, LayerMetadata metadata,
                               sp<IBinder>* outHandle, sp<IGraphicBufferProducer>* /* gbp */,
                               int32_t* outLayerId, uint32_t* outTransformHint) {
    // We rely on createLayer to check permissions.
    LayerCreationArgs args(mFlinger.get(), this, name.c_str(), flags, std::move(metadata));
    return mFlinger->createLayer(args, outHandle, parentHandle, outLayerId, nullptr,
                                 outTransformHint);
}
```

接下来就是跨进程到了Sf端

```cpp

status_t SurfaceFlinger::createLayer(LayerCreationArgs& args, sp<IBinder>* outHandle,
                                     const sp<IBinder>& parentHandle, int32_t* outLayerId,
                                     const sp<Layer>& parentLayer, uint32_t* outTransformHint) {
  
    sp<Layer> layer;
	//根据不同的flags创建不同的Layer，常见情况：有buffer数据的一般是createBufferStateLayer，没有buffer的就是createContainerLayer
    switch (args.flags & ISurfaceComposerClient::eFXSurfaceMask) {
        case ISurfaceComposerClient::eFXSurfaceBufferQueue:
        case ISurfaceComposerClient::eFXSurfaceBufferState: {
            result = createBufferStateLayer(args, outHandle, &layer);
            std::atomic<int32_t>* pendingBufferCounter = layer->getPendingBufferCounter();
            if (pendingBufferCounter) {
                std::string counterName = layer->getPendingBufferCounterName();
                mBufferCountTracker.add((*outHandle)->localBinder(), counterName,
                                        pendingBufferCounter);
            }
        } break;
        case ISurfaceComposerClient::eFXSurfaceEffect:
            result = createEffectLayer(args, outHandle, &layer);
            break;
        case ISurfaceComposerClient::eFXSurfaceContainer:
            result = createContainerLayer(args, outHandle, &layer);
            break;
        default:
            result = BAD_VALUE;
            break;
    }
    int parentId = -1;
    sp<Layer> parentSp = parent.promote();
    if (parentSp != nullptr) {
        parentId = parentSp->getSequence();
    }
    //调用到addClientLayer方法
    result = addClientLayer(args.client, *outHandle, layer, parent, addToRoot, outTransformHint);
    if (result != NO_ERROR) {
        return result;
    }

    *outLayerId = layer->sequence;
    return result;
}

```

```cpp
status_t SurfaceFlinger::addClientLayer(const sp<Client>& client, const sp<IBinder>& handle,
                                        const sp<Layer>& layer, const wp<Layer>& parent,
                                        bool addToRoot, uint32_t* outTransformHint) {


    {
     //放入到mCreatedLayers集合中
        std::scoped_lock<std::mutex> lock(mCreatedLayersLock);
        mCreatedLayers.emplace_back(layer, parent, addToRoot);
    }

    if (client != nullptr) {
    /client与layer会进行绑定
        client->attachLayer(handle, layer);
    }
//设置事物flag
    setTransactionFlags(eTransactionNeeded);
    return NO_ERROR;
}

```

简单总结图：\
![1726818533519.png](https://picgo.myjojo.fun:666/i/2024/09/20/66ed28d90130d.png))

本文章对应视频手把手教你学framework：\
hal+perfetto+surfaceflinger\
<https://mp.weixin.qq.com/s/LbVLnu1udqExHVKxd74ILg>  s
