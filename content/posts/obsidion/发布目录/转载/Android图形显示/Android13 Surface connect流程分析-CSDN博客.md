---
title: Android13 Surface connect流程分析-CSDN博客
created: 2024-09-26
tags:
  - clippings
  - 转载
  - blog
collections:
  - 图形显示
date: 2024-09-26T06:49:46.173Z
lastmod: 2024-10-09T09:24:09.000Z
---
Surface的connect方法用于建立与BufferQueueCore[Android13 BufferQueueLayer onFirstRef流程分析-CSDN博客#new BufferQueueCore](Android13%20BufferQueueLayer%20onFirstRef%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-CSDN%E5%8D%9A%E5%AE%A2#new%20BufferQueueCore)连接，代码如下：

```java
//frameworks/base/core/java/android/view/Surface.java
class Surface : public ANativeObjectBase<ANativeWindow, Surface, RefBase>
{
	int Surface::connect(int api) {
	    static sp<IProducerListener> listener = new StubProducerListener();
	    return connect(api, listener);
	}
}
```

调用重载方法：

```java
//frameworks/base/core/java/android/view/Surface.java
class Surface : public ANativeObjectBase<ANativeWindow, Surface, RefBase>
{
int Surface::connect(int api, const sp<IProducerListener>& listener) {
    return connect(api, listener, false);
}
}
```

调用重载方法：

```java
//frameworks/base/core/java/android/view/Surface.java
sp<IGraphicBufferProducer> mGraphicBufferProducer;
class Surface : public ANativeObjectBase<ANativeWindow, Surface, RefBase>
{
int Surface::connect(
        int api, const sp<IProducerListener>& listener, bool reportBufferRemoval) {
    ATRACE_CALL();
    ALOGV("Surface::connect");
    Mutex::Autolock lock(mMutex);
    IGraphicBufferProducer::QueueBufferOutput output;
    mReportRemovedBuffers = reportBufferRemoval;
    int err = mGraphicBufferProducer->connect(listener, api, mProducerControlledByApp, &output);
    if (err == NO_ERROR) {
        mDefaultWidth = output.width;
        mDefaultHeight = output.height;
        mNextFrameNumber = output.nextFrameNumber;
        mMaxBufferCount = output.maxBufferCount;
 
 
        // Ignore transform hint if sticky transform is set or transform to display inverse flag is
        // set. Transform hint should be ignored if the client is expected to always submit buffers
        // in the same orientation.
        if (mStickyTransform == 0 && !transformToDisplayInverse()) {
            mTransformHint = output.transformHint;
        }
 
 
        mConsumerRunningBehind = (output.numPendingBuffers >= 2);
    }
    if (!err && api == NATIVE_WINDOW_API_CPU) {
        mConnectedToCpu = true;
        // Clear the dirty region in case we're switching from a non-CPU API
        mDirtyRegion.clear();
    } else if (!err) {
        // Initialize the dirty region for tracking surface damage
        mDirtyRegion = Region::INVALID_REGION;
    }
 
 
    return err;
}
}
```

调用mGraphicBufferProducer(IGraphicBufferProducer)的connect方法，IGraphicBufferProducer是一个接口，由BpGraphicBufferProducer 实现：

```cpp
//frameworks/native/libs/gui/IGraphicBufferProducer.cpp
class BpGraphicBufferProducer : public BpInterface<IGraphicBufferProducer>
{
    virtual status_t connect(const sp<IProducerListener>& listener,
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
        status_t result = remote()->transact(CONNECT, data, &reply);
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
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    switch(code) {
        case CONNECT: {
            CHECK_INTERFACE(IGraphicBufferProducer, data, reply);
            sp<IProducerListener> listener;
            if (data.readInt32() == 1) {
                listener = IProducerListener::asInterface(data.readStrongBinder());
            }
            int api = data.readInt32();
            bool producerControlledByApp = data.readInt32();
            QueueBufferOutput output;
            status_t res = connect(listener, api, producerControlledByApp, &output);
            reply->write(output);
            reply->writeInt32(res);
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
sp<BufferQueueCore> mCore;
status_t BufferQueueProducer::connect(const sp<IProducerListener>& listener,
        int api, bool producerControlledByApp, QueueBufferOutput *output) {
    ATRACE_CALL();
    std::lock_guard<std::mutex> lock(mCore->mMutex);
    mConsumerName = mCore->mConsumerName;
    BQ_LOGV("connect: api=%d producerControlledByApp=%s", api,
            producerControlledByApp ? "true" : "false");
 
 
    if (mCore->mIsAbandoned) {
        BQ_LOGE("connect: BufferQueue has been abandoned");
        return NO_INIT;
    }
 
 
    if (mCore->mConsumerListener == nullptr) {
        BQ_LOGE("connect: BufferQueue has no consumer");
        return NO_INIT;
    }
 
 
    if (output == nullptr) {
        BQ_LOGE("connect: output was NULL");
        return BAD_VALUE;
    }
 
 
    //如果已经连接过，就不允许再链接。也就是BufferQueueCore只允许由一个connected producer
    if (mCore->mConnectedApi != BufferQueueCore::NO_CONNECTED_API) {
        BQ_LOGE("connect: already connected (cur=%d req=%d)",
                mCore->mConnectedApi, api);
        return BAD_VALUE;
    }
 
 
    int delta = mCore->getMaxBufferCountLocked(mCore->mAsyncMode,
            mDequeueTimeout < 0 ?
            mCore->mConsumerControlledByApp && producerControlledByApp : false,
            mCore->mMaxBufferCount) -
            mCore->getMaxBufferCountLocked();
    if (!mCore->adjustAvailableSlotsLocked(delta)) {
        BQ_LOGE("connect: BufferQueue failed to adjust the number of available "
                "slots. Delta = %d", delta);
        return BAD_VALUE;
    }
 
 
    int status = NO_ERROR;
    switch (api) {
        case NATIVE_WINDOW_API_EGL:
        case NATIVE_WINDOW_API_CPU:
        case NATIVE_WINDOW_API_MEDIA:
        case NATIVE_WINDOW_API_CAMERA:
            mCore->mConnectedApi = api;
 
 
            output->width = mCore->mDefaultWidth;
            output->height = mCore->mDefaultHeight;
            output->transformHint = mCore->mTransformHintInUse = mCore->mTransformHint;
            output->numPendingBuffers =
                    static_cast<uint32_t>(mCore->mQueue.size());
            output->nextFrameNumber = mCore->mFrameCounter + 1;
            output->bufferReplaced = false;
            output->maxBufferCount = mCore->mMaxBufferCount;
 
 
            if (listener != nullptr) {
                // Set up a death notification so that we can disconnect
                // automatically if the remote producer dies
#ifndef NO_BINDER
                if (IInterface::asBinder(listener)->remoteBinder() != nullptr) {
                    status = IInterface::asBinder(listener)->linkToDeath(
                            static_cast<IBinder::DeathRecipient*>(this));
                    if (status != NO_ERROR) {
                        BQ_LOGE("connect: linkToDeath failed: %s (%d)",
                                strerror(-status), status);
                    }
                    mCore->mLinkedToDeath = listener;
                }
#endif
                mCore->mConnectedProducerListener = listener;
                mCore->mBufferReleasedCbEnabled = listener->needsReleaseNotify();
            }
            break;
        default:
            BQ_LOGE("connect: unknown API %d", api);
            status = BAD_VALUE;
            break;
    }
    mCore->mConnectedPid = BufferQueueThreadState::getCallingPid();
    mCore->mBufferHasBeenQueued = false;
    mCore->mDequeueBufferCannotBlock = false;
    mCore->mQueueBufferCanDrop = false;
    mCore->mLegacyBufferDrop = true;
    if (mCore->mConsumerControlledByApp && producerControlledByApp) {
        mCore->mDequeueBufferCannotBlock = mDequeueTimeout < 0;
        mCore->mQueueBufferCanDrop = mDequeueTimeout <= 0;
    }
 
 
    mCore->mAllowAllocation = true;
    VALIDATE_CONSISTENCY();
    return status;
}
}
```
