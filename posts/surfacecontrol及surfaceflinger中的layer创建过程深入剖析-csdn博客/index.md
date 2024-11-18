# SurfaceControl及SurfaceFlinger中的Layer创建过程深入剖析-CSDN博客

### SurfaceComposerClient创建

SurfaceComposerClient对象是在哪里创建的呢?是在SurfaceSession构造时候创建\
frameworks/base/core/[jni](https://so.csdn.net/so/search?q=jni\&amp;spm=1001.2101.3001.7020)/android\_view\_SurfaceSession.cpp

```cpp
static jlong nativeCreate(JNIEnv* env, jclass clazz) {
    SurfaceComposerClient* client = new SurfaceComposerClient();
    client-&gt;incStrong((void*)nativeCreate);
    return reinterpret_cast&lt;jlong&gt;(client);
}
```

这里的SurfaceSession是在哪里创建的呢？这里直接上堆栈

![1726819413770.png](https://picgo.myjojo.fun:666/i/2024/09/20/66ed2c488d9a5.png)\
实际是在ViewRootImpl构造期间就创建\
![在这里插入图片描述](https://picgo.myjojo.fun:666/i/2024/09/20/66ed2c6f12c37.png#pic_center)

### SurfaceComposerClient的mClient初始化

这里的mClient是谁呢？类型是\
sp mClient;\
看得出是一个接口

看下SurfaceComposerClient构造时候会与sf进行跨进程createConnection创建链接，返回的对象就是Client对象。

```cpp
void SurfaceComposerClient::onFirstRef() {
    sp&lt;ISurfaceComposer&gt; sf(ComposerService::getComposerService());
    if (sf != nullptr &amp;&amp; mStatus == NO_INIT) {
        sp&lt;ISurfaceComposerClient&gt; conn;
        conn = sf-&gt;createConnection();
        if (conn != nullptr) {
            mClient = conn;
            mStatus = NO_ERROR;
        }
    }
}
```

这里又出现了ComposerService::getComposerService()\
frameworks/native/libs/[gui](https://so.csdn.net/so/search?q=gui\&amp;spm=1001.2101.3001.7020)/SurfaceComposerClient.cpp

```cpp
sp&lt;ISurfaceComposer&gt; ComposerService::getComposerService() {
    ComposerService&amp; instance = ComposerService::getInstance();
    if (instance.mComposerService == nullptr) {
        if (ComposerService::getInstance().connectLocked()) {
            WindowInfosListenerReporter::getInstance()-&gt;reconnect(instance.mComposerService);
        }
    }
    return instance.mComposerService;
}
//connectLocked实现
bool ComposerService::connectLocked() {
    const String16 name(&#34;SurfaceFlinger&#34;);
    //就是获取sf的bp代理，赋值给mComposerService
    mComposerService = waitForService&lt;ISurfaceComposer&gt;(name);
    return true;
}
```

所以上面onFirstRef中sf-&gt;createConnection()调用到sf了，sf代码如下\
frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp

```cpp
sp&lt;ISurfaceComposerClient&gt; SurfaceFlinger::createConnection() {
    const sp&lt;Client&gt; client = new Client(this);
    return client-&gt;initCheck() == NO_ERROR ? client : nullptr;
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
    sp&lt;SurfaceComposerClient&gt; client;
  
    status_t err = client-&gt;createSurfaceChecked(String8(name.c_str()), w, h, format, &amp;surface,
                                                flags, parentHandle, std::move(metadata));
  
    surface-&gt;incStrong((void *)nativeCreate);
    return reinterpret_cast&lt;jlong&gt;(surface.get());
}
```

主要调用到了SurfaceComposerClient的createSurfaceChecked。

下面看看 SurfaceComposerClient::createSurfaceChecked方法

```cpp

status_t SurfaceComposerClient::createSurfaceChecked(const String8&amp; name, uint32_t w, uint32_t h,
                                                     PixelFormat format,
                                                     sp&lt;SurfaceControl&gt;* outSurface, uint32_t flags,
                                                     const sp&lt;IBinder&gt;&amp; parentHandle,
                                                     LayerMetadata metadata,
                                                     uint32_t* outTransformHint) {
 
        err = mClient-&gt;createSurface(name, w, h, format, flags, parentHandle, std::move(metadata),
                                     &amp;handle, &amp;gbp, &amp;id, &amp;transformHint);
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
status_t Client::createSurface(const String8&amp; name, uint32_t /* w */, uint32_t /* h */,
                               PixelFormat /* format */, uint32_t flags,
                               const sp&lt;IBinder&gt;&amp; parentHandle, LayerMetadata metadata,
                               sp&lt;IBinder&gt;* outHandle, sp&lt;IGraphicBufferProducer&gt;* /* gbp */,
                               int32_t* outLayerId, uint32_t* outTransformHint) {
    // We rely on createLayer to check permissions.
    LayerCreationArgs args(mFlinger.get(), this, name.c_str(), flags, std::move(metadata));
    return mFlinger-&gt;createLayer(args, outHandle, parentHandle, outLayerId, nullptr,
                                 outTransformHint);
}
```

接下来就是跨进程到了Sf端

```cpp

status_t SurfaceFlinger::createLayer(LayerCreationArgs&amp; args, sp&lt;IBinder&gt;* outHandle,
                                     const sp&lt;IBinder&gt;&amp; parentHandle, int32_t* outLayerId,
                                     const sp&lt;Layer&gt;&amp; parentLayer, uint32_t* outTransformHint) {
  
    sp&lt;Layer&gt; layer;
	//根据不同的flags创建不同的Layer，常见情况：有buffer数据的一般是createBufferStateLayer，没有buffer的就是createContainerLayer
    switch (args.flags &amp; ISurfaceComposerClient::eFXSurfaceMask) {
        case ISurfaceComposerClient::eFXSurfaceBufferQueue:
        case ISurfaceComposerClient::eFXSurfaceBufferState: {
            result = createBufferStateLayer(args, outHandle, &amp;layer);
            std::atomic&lt;int32_t&gt;* pendingBufferCounter = layer-&gt;getPendingBufferCounter();
            if (pendingBufferCounter) {
                std::string counterName = layer-&gt;getPendingBufferCounterName();
                mBufferCountTracker.add((*outHandle)-&gt;localBinder(), counterName,
                                        pendingBufferCounter);
            }
        } break;
        case ISurfaceComposerClient::eFXSurfaceEffect:
            result = createEffectLayer(args, outHandle, &amp;layer);
            break;
        case ISurfaceComposerClient::eFXSurfaceContainer:
            result = createContainerLayer(args, outHandle, &amp;layer);
            break;
        default:
            result = BAD_VALUE;
            break;
    }
    int parentId = -1;
    sp&lt;Layer&gt; parentSp = parent.promote();
    if (parentSp != nullptr) {
        parentId = parentSp-&gt;getSequence();
    }
    //调用到addClientLayer方法
    result = addClientLayer(args.client, *outHandle, layer, parent, addToRoot, outTransformHint);
    if (result != NO_ERROR) {
        return result;
    }

    *outLayerId = layer-&gt;sequence;
    return result;
}

```

```cpp
status_t SurfaceFlinger::addClientLayer(const sp&lt;Client&gt;&amp; client, const sp&lt;IBinder&gt;&amp; handle,
                                        const sp&lt;Layer&gt;&amp; layer, const wp&lt;Layer&gt;&amp; parent,
                                        bool addToRoot, uint32_t* outTransformHint) {


    {
     //放入到mCreatedLayers集合中
        std::scoped_lock&lt;std::mutex&gt; lock(mCreatedLayersLock);
        mCreatedLayers.emplace_back(layer, parent, addToRoot);
    }

    if (client != nullptr) {
    /client与layer会进行绑定
        client-&gt;attachLayer(handle, layer);
    }
//设置事物flag
    setTransactionFlags(eTransactionNeeded);
    return NO_ERROR;
}

```

简单总结图：\
![1726818533519.png](https://picgo.myjojo.fun:666/i/2024/09/20/66ed28d90130d.png))


---

> 作者: [wentao](https://github.com/huang8604)  
> URL: https://huang8604.github.io/posts/surfacecontrol%E5%8F%8Asurfaceflinger%E4%B8%AD%E7%9A%84layer%E5%88%9B%E5%BB%BA%E8%BF%87%E7%A8%8B%E6%B7%B1%E5%85%A5%E5%89%96%E6%9E%90-csdn%E5%8D%9A%E5%AE%A2/  

