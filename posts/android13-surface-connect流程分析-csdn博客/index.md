# Android13 Surface Connect流程分析-CSDN博客

Surface的connect方法用于建立与BufferQueueCore[Android13 BufferQueueLayer onFirstRef流程分析-CSDN博客#new BufferQueueCore](Android13%20BufferQueueLayer%20onFirstRef%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-CSDN%E5%8D%9A%E5%AE%A2#new%20BufferQueueCore)连接，代码如下：

```java
//frameworks/base/core/java/android/view/Surface.java
class Surface : public ANativeObjectBase&lt;ANativeWindow, Surface, RefBase&gt;
{
	int Surface::connect(int api) {
	    static sp&lt;IProducerListener&gt; listener = new StubProducerListener();
	    return connect(api, listener);
	}
}
```

调用重载方法：

```java
//frameworks/base/core/java/android/view/Surface.java
class Surface : public ANativeObjectBase&lt;ANativeWindow, Surface, RefBase&gt;
{
int Surface::connect(int api, const sp&lt;IProducerListener&gt;&amp; listener) {
    return connect(api, listener, false);
}
}
```

调用重载方法：

```java
//frameworks/base/core/java/android/view/Surface.java
sp&lt;IGraphicBufferProducer&gt; mGraphicBufferProducer;
class Surface : public ANativeObjectBase&lt;ANativeWindow, Surface, RefBase&gt;
{
int Surface::connect(
        int api, const sp&lt;IProducerListener&gt;&amp; listener, bool reportBufferRemoval) {
    ATRACE_CALL();
    ALOGV(&#34;Surface::connect&#34;);
    Mutex::Autolock lock(mMutex);
    IGraphicBufferProducer::QueueBufferOutput output;
    mReportRemovedBuffers = reportBufferRemoval;
    int err = mGraphicBufferProducer-&gt;connect(listener, api, mProducerControlledByApp, &amp;output);
    if (err == NO_ERROR) {
        mDefaultWidth = output.width;
        mDefaultHeight = output.height;
        mNextFrameNumber = output.nextFrameNumber;
        mMaxBufferCount = output.maxBufferCount;
 
 
        // Ignore transform hint if sticky transform is set or transform to display inverse flag is
        // set. Transform hint should be ignored if the client is expected to always submit buffers
        // in the same orientation.
        if (mStickyTransform == 0 &amp;&amp; !transformToDisplayInverse()) {
            mTransformHint = output.transformHint;
        }
 
 
        mConsumerRunningBehind = (output.numPendingBuffers &gt;= 2);
    }
    if (!err &amp;&amp; api == NATIVE_WINDOW_API_CPU) {
        mConnectedToCpu = true;
        // Clear the dirty region in case we&#39;re switching from a non-CPU API
        mDirtyRegion.clear();
    } else if (!err) {
        // Initialize the dirty region for tracking surface damage
        mDirtyRegion = Region::INVALID_REGION;
    }
 
 
    return err;
}
}
```

调用mGraphicBufferProducer(IGraphicBufferProducer)的[connect](https://so.csdn.net/so/search?q=connect\&amp;spm=1001.2101.3001.7020)方法，IGraphicBufferProducer是一个接口，由BpGraphicBufferProducer 实现：

```cpp
//frameworks/native/libs/gui/IGraphicBufferProducer.cpp
class BpGraphicBufferProducer : public BpInterface&lt;IGraphicBufferProducer&gt;
{
    virtual status_t connect(const sp&lt;IProducerListener&gt;&amp; listener,
            int api, bool producerControlledByApp, QueueBufferOutput* output) {
        Parcel data, reply;
        data.writeInterfaceToken(IGraphicBufferProducer::getInterfaceDescriptor());
        if (listener != nullptr) {
            data.writeInt32(1);
            data.writeStrongBinder(IInterface::asBinder(listener));
        } else {
            data.writeInt32(0);
        }
        data.writeInt32(api);
        data.writeInt32(producerControlledByApp);
        status_t result = remote()-&gt;transact(CONNECT, data, &amp;reply);
        if (result != NO_ERROR) {
            return result;
        }
        reply.read(*output);
        result = reply.readInt32();
        return result;
    }
}
```

发送CONNECT消息，发送的消息在onTransact中处理：

```cpp
//frameworks/native/libs/gui/IGraphicBufferProducer.cpp
status_t BnGraphicBufferProducer::onTransact(
    uint32_t code, const Parcel&amp; data, Parcel* reply, uint32_t flags)
{
    switch(code) {
        case CONNECT: {
            CHECK_INTERFACE(IGraphicBufferProducer, data, reply);
            sp&lt;IProducerListener&gt; listener;
            if (data.readInt32() == 1) {
                listener = IProducerListener::asInterface(data.readStrongBinder());
            }
            int api = data.readInt32();
            bool producerControlledByApp = data.readInt32();
            QueueBufferOutput output;
            status_t res = connect(listener, api, producerControlledByApp, &amp;output);
            reply-&gt;write(output);
            reply-&gt;writeInt32(res);
            return NO_ERROR;
        }
    }
}
```

## BufferQueueProducer connect

调用BnGraphicBufferProducer的connect方法，BufferQueueProducer继承于BnGraphicBufferProducer，调用BufferQueueProducer的connect方法：

```cpp
//frameworks/native/libs/gui/BufferQueueProducer.cpp
class BufferQueueProducer : public BnGraphicBufferProducer,
                            private IBinder::DeathRecipient {
sp&lt;BufferQueueCore&gt; mCore;
status_t BufferQueueProducer::connect(const sp&lt;IProducerListener&gt;&amp; listener,
        int api, bool producerControlledByApp, QueueBufferOutput *output) {
    ATRACE_CALL();
    std::lock_guard&lt;std::mutex&gt; lock(mCore-&gt;mMutex);
    mConsumerName = mCore-&gt;mConsumerName;
    BQ_LOGV(&#34;connect: api=%d producerControlledByApp=%s&#34;, api,
            producerControlledByApp ? &#34;true&#34; : &#34;false&#34;);
 
 
    if (mCore-&gt;mIsAbandoned) {
        BQ_LOGE(&#34;connect: BufferQueue has been abandoned&#34;);
        return NO_INIT;
    }
 
 
    if (mCore-&gt;mConsumerListener == nullptr) {
        BQ_LOGE(&#34;connect: BufferQueue has no consumer&#34;);
        return NO_INIT;
    }
 
 
    if (output == nullptr) {
        BQ_LOGE(&#34;connect: output was NULL&#34;);
        return BAD_VALUE;
    }
 
 
    //如果已经连接过，就不允许再链接。也就是BufferQueueCore只允许由一个connected producer
    if (mCore-&gt;mConnectedApi != BufferQueueCore::NO_CONNECTED_API) {
        BQ_LOGE(&#34;connect: already connected (cur=%d req=%d)&#34;,
                mCore-&gt;mConnectedApi, api);
        return BAD_VALUE;
    }
 
 
    int delta = mCore-&gt;getMaxBufferCountLocked(mCore-&gt;mAsyncMode,
            mDequeueTimeout &lt; 0 ?
            mCore-&gt;mConsumerControlledByApp &amp;&amp; producerControlledByApp : false,
            mCore-&gt;mMaxBufferCount) -
            mCore-&gt;getMaxBufferCountLocked();
    if (!mCore-&gt;adjustAvailableSlotsLocked(delta)) {
        BQ_LOGE(&#34;connect: BufferQueue failed to adjust the number of available &#34;
                &#34;slots. Delta = %d&#34;, delta);
        return BAD_VALUE;
    }
 
 
    int status = NO_ERROR;
    switch (api) {
        case NATIVE_WINDOW_API_EGL:
        case NATIVE_WINDOW_API_CPU:
        case NATIVE_WINDOW_API_MEDIA:
        case NATIVE_WINDOW_API_CAMERA:
            mCore-&gt;mConnectedApi = api;
 
 
            output-&gt;width = mCore-&gt;mDefaultWidth;
            output-&gt;height = mCore-&gt;mDefaultHeight;
            output-&gt;transformHint = mCore-&gt;mTransformHintInUse = mCore-&gt;mTransformHint;
            output-&gt;numPendingBuffers =
                    static_cast&lt;uint32_t&gt;(mCore-&gt;mQueue.size());
            output-&gt;nextFrameNumber = mCore-&gt;mFrameCounter &#43; 1;
            output-&gt;bufferReplaced = false;
            output-&gt;maxBufferCount = mCore-&gt;mMaxBufferCount;
 
 
            if (listener != nullptr) {
                // Set up a death notification so that we can disconnect
                // automatically if the remote producer dies
#ifndef NO_BINDER
                if (IInterface::asBinder(listener)-&gt;remoteBinder() != nullptr) {
                    status = IInterface::asBinder(listener)-&gt;linkToDeath(
                            static_cast&lt;IBinder::DeathRecipient*&gt;(this));
                    if (status != NO_ERROR) {
                        BQ_LOGE(&#34;connect: linkToDeath failed: %s (%d)&#34;,
                                strerror(-status), status);
                    }
                    mCore-&gt;mLinkedToDeath = listener;
                }
#endif
                mCore-&gt;mConnectedProducerListener = listener;
                mCore-&gt;mBufferReleasedCbEnabled = listener-&gt;needsReleaseNotify();
            }
            break;
        default:
            BQ_LOGE(&#34;connect: unknown API %d&#34;, api);
            status = BAD_VALUE;
            break;
    }
    mCore-&gt;mConnectedPid = BufferQueueThreadState::getCallingPid();
    mCore-&gt;mBufferHasBeenQueued = false;
    mCore-&gt;mDequeueBufferCannotBlock = false;
    mCore-&gt;mQueueBufferCanDrop = false;
    mCore-&gt;mLegacyBufferDrop = true;
    if (mCore-&gt;mConsumerControlledByApp &amp;&amp; producerControlledByApp) {
        mCore-&gt;mDequeueBufferCannotBlock = mDequeueTimeout &lt; 0;
        mCore-&gt;mQueueBufferCanDrop = mDequeueTimeout &lt;= 0;
    }
 
 
    mCore-&gt;mAllowAllocation = true;
    VALIDATE_CONSISTENCY();
    return status;
}
}
```


---

> 作者: [wentao](https://github.com/huang8604)  
> URL: https://huang8604.github.io/posts/android13-surface-connect%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-csdn%E5%8D%9A%E5%AE%A2/  

