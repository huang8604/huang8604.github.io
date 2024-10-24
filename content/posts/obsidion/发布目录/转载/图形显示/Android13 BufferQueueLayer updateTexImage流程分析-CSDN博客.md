---
title: Android13 BufferQueueLayer updateTexImage流程分析-CSDN博客
author: 
created: 2024-09-27
tags:
  - clippings
  - 转载
  - blog
collections: 图形显示
date: 2024-09-27T02:50:31.553Z
lastmod: 2024-09-27T03:15:39.000Z
---
BufferQueueLayer的updateTexImage方法用于将当前图形缓冲区的内容更新到纹理中，代码如下：

```cpp
//frameworks/native/services/surfaceflinger/BufferQueueLayer.cpp
status_t BufferQueueLayer::updateTexImage(bool& recomputeVisibleRegions, nsecs_t latchTime,
                                          nsecs_t expectedPresentTime) {
    // This boolean is used to make sure that SurfaceFlinger's shadow copy
    // of the buffer queue isn't modified when the buffer queue is returning
    // BufferItem's that weren't actually queued. This can happen in shared
    // buffer mode.
    bool queuedBuffer = false;
    const int32_t layerId = getSequence();
    LayerRejecter r(mDrawingState, getDrawingState(), recomputeVisibleRegions,
                    getProducerStickyTransform() != 0, mName,
                    getTransformToDisplayInverse());
 
 
    if (isRemovedFromCurrentState()) {
        expectedPresentTime = 0;
    }
 
 
    // updateTexImage() below might drop the some buffers at the head of the queue if there is a
    // buffer behind them which is timely to be presented. However this buffer may not be signaled
    // yet. The code below makes sure that this wouldn't happen by setting maxFrameNumber to the
    // last buffer that was signaled.
    uint64_t lastSignaledFrameNumber = mLastFrameNumberReceived;
    {
        Mutex::Autolock lock(mQueueItemLock);
        for (size_t i = 0; i < mQueueItems.size(); i++) {
            bool fenceSignaled =
                    mQueueItems[i].item.mFenceTime->getSignalTime() != Fence::SIGNAL_TIME_PENDING;
            if (!fenceSignaled) {
                break;
            }
            lastSignaledFrameNumber = mQueueItems[i].item.mFrameNumber;
        }
    }
    const uint64_t maxFrameNumberToAcquire =
            std::min(mLastFrameNumberReceived.load(), lastSignaledFrameNumber);
 
 
    bool autoRefresh;
    status_t updateResult = mConsumer->updateTexImage(&r, expectedPresentTime, &autoRefresh,
                                                      &queuedBuffer, maxFrameNumberToAcquire);
    mDrawingState.autoRefresh = autoRefresh;
    if (updateResult == BufferQueue::PRESENT_LATER) {
        // Producer doesn't want buffer to be displayed yet.  Signal a
        // layer update so we check again at the next opportunity.
 // Producer 不希望显示缓冲区。 发出图层更新的信号，以便我们在下次有机会时再次检查。
        mFlinger->onLayerUpdate(); // (692) SurfaceFlinger onLayerUpdate流程分析 
        return BAD_VALUE;
    } else if (updateResult == BufferLayerConsumer::BUFFER_REJECTED) {
        // If the buffer has been rejected, remove it from the shadow queue
        // and return early
        if (queuedBuffer) {
            Mutex::Autolock lock(mQueueItemLock);
            if (mQueuedFrames > 0) {
                mConsumer->mergeSurfaceDamage(mQueueItems[0].item.mSurfaceDamage);
                mFlinger->mTimeStats->removeTimeRecord(layerId, mQueueItems[0].item.mFrameNumber);
                if (mQueueItems[0].surfaceFrame) {
                    addSurfaceFrameDroppedForBuffer(mQueueItems[0].surfaceFrame);
                }
                mQueueItems.erase(mQueueItems.begin());
                mQueuedFrames--;
            }
        }
        return BAD_VALUE;
    } else if (updateResult != NO_ERROR || mUpdateTexImageFailed) {
        // This can occur if something goes wrong when trying to create the
        // EGLImage for this buffer. If this happens, the buffer has already
        // been released, so we need to clean up the queue and bug out
        // early.
        if (queuedBuffer) {
            Mutex::Autolock lock(mQueueItemLock);
            for (auto& [item, surfaceFrame] : mQueueItems) {
                if (surfaceFrame) {
                    addSurfaceFrameDroppedForBuffer(surfaceFrame);
                }
            }
            mQueueItems.clear();
            mQueuedFrames = 0;
            mFlinger->mTimeStats->onDestroy(layerId);
            mFlinger->mFrameTracer->onDestroy(layerId);
        }
 
 
        // Once we have hit this state, the shadow queue may no longer
        // correctly reflect the incoming BufferQueue's contents, so even if
        // updateTexImage starts working, the only safe course of action is
        // to continue to ignore updates.
        mUpdateTexImageFailed = true;
 
 
        return BAD_VALUE;
    }
 
 
    bool more_frames_pending = false;
    if (queuedBuffer) {
        // Autolock scope
        auto currentFrameNumber = mConsumer->getFrameNumber();
 
 
        Mutex::Autolock lock(mQueueItemLock);
 
 
        // Remove any stale buffers that have been dropped during
        // updateTexImage
        while (mQueuedFrames > 0 && mQueueItems[0].item.mFrameNumber != currentFrameNumber) {
            mConsumer->mergeSurfaceDamage(mQueueItems[0].item.mSurfaceDamage);
            mFlinger->mTimeStats->removeTimeRecord(layerId, mQueueItems[0].item.mFrameNumber);
            if (mQueueItems[0].surfaceFrame) {
                addSurfaceFrameDroppedForBuffer(mQueueItems[0].surfaceFrame);
            }
            mQueueItems.erase(mQueueItems.begin());
            mQueuedFrames--;
        }
 
 
        uint64_t bufferID = mQueueItems[0].item.mGraphicBuffer->getId();
        mFlinger->mTimeStats->setLatchTime(layerId, currentFrameNumber, latchTime);
        mFlinger->mFrameTracer->traceTimestamp(layerId, bufferID, currentFrameNumber, latchTime,
                                               FrameTracer::FrameEvent::LATCH);
 
 
        if (mQueueItems[0].surfaceFrame) {
            addSurfaceFramePresentedForBuffer(mQueueItems[0].surfaceFrame,
                                              mQueueItems[0].item.mFenceTime->getSignalTime(),
                                              latchTime);
        }
        mQueueItems.erase(mQueueItems.begin());
        more_frames_pending = (mQueuedFrames.fetch_sub(1) > 1);
    }
 
 
    // Decrement the queued-frames count.  Signal another event if we
    // have more frames pending.
    //减少排队的帧计数。 如果我们有更多待处理的帧，则发出另一个事件的信号。
    if ((queuedBuffer && more_frames_pending) || mDrawingState.autoRefresh) {
        mFlinger->onLayerUpdate(); // (692) SurfaceFlinger onLayerUpdate流程分析 | 知识管理 - PingCode 
    }
 
 
    return NO_ERROR;
}
```

## BufferLayerConsumer updateTexImage

调用BufferLayerConsumer的updateTexImage方法：

```cpp
//frameworks/native/surfaces/surfaceflienger/BufferLayerConsumer.cpp
status_t BufferLayerConsumer::updateTexImage(BufferRejecter* rejecter, nsecs_t expectedPresentTime,
                                             bool* autoRefresh, bool* queuedBuffer,
                                             uint64_t maxFrameNumber) {
    ATRACE_CALL();
    BLC_LOGV("updateTexImage");
    Mutex::Autolock lock(mMutex);
 
 
    if (mAbandoned) {
        BLC_LOGE("updateTexImage: BufferLayerConsumer is abandoned!");
        return NO_INIT;
    }
 
 
    BufferItem item;
 
 
    // Acquire the next buffer.
    // In asynchronous mode the list is guaranteed to be one buffer
    // deep, while in synchronous mode we use the oldest buffer.
    status_t err = acquireBufferLocked(&item, expectedPresentTime, maxFrameNumber);
    if (err != NO_ERROR) {
        if (err == BufferQueue::NO_BUFFER_AVAILABLE) {
            err = NO_ERROR;
        } else if (err == BufferQueue::PRESENT_LATER) {
            // return the error, without logging
        } else {
            BLC_LOGE("updateTexImage: acquire failed: %s (%d)", strerror(-err), err);
        }
        return err;
    }
 
 
    if (autoRefresh) {
        *autoRefresh = item.mAutoRefresh;
    }
 
 
    if (queuedBuffer) {
        *queuedBuffer = item.mQueuedBuffer;
    }
 
 
    // We call the rejecter here, in case the caller has a reason to
    // not accept this buffer.  This is used by SurfaceFlinger to
    // reject buffers which have the wrong size
    //我们在这里调用拒绝器，以防调用者有理由不接受此缓冲区， SurfaceFlinger 使用它来拒绝大小错误的缓冲区
    int slot = item.mSlot;
    if (rejecter && rejecter->reject(mSlots[slot].mGraphicBuffer, item)) {
        releaseBufferLocked(slot, mSlots[slot].mGraphicBuffer);
        return BUFFER_REJECTED;
    }
 
 
    // Release the previous buffer.
    err = updateAndReleaseLocked(item, &mPendingRelease);
    if (err != NO_ERROR) {
        return err;
    }
    return err;
}
```

上面方法的主要处理如下：

1、调用BufferLayerConsumer的acquireBufferLocked方法，从BufferQueue中获取一个可用的图像缓冲区。

2、调用BufferLayerConsumer的updateAndReleaseLocked方法，释放图像缓冲区。

下面分别进行分析：

### BufferLayerConsumer acquireBufferLocked

调用BufferLayerConsumer的acquireBufferLocked方法，从BufferQueue中获取一个可用的图像缓冲区：

```cpp
//frameworks/native/services/surfaceflinger/BufferLayerConsumer.cpp
status_t BufferLayerConsumer::acquireBufferLocked(BufferItem* item, nsecs_t presentWhen,
                                                  uint64_t maxFrameNumber) {
    status_t err = ConsumerBase::acquireBufferLocked(item, presentWhen, maxFrameNumber);
    if (err != NO_ERROR) {
        return err;
    }
 
 
    // If item->mGraphicBuffer is not null, this buffer has not been acquired
    // before, so we need to clean up old references.
    if (item->mGraphicBuffer != nullptr) {
        std::lock_guard<std::mutex> lock(mImagesMutex);
        if (mImages[item->mSlot] == nullptr || mImages[item->mSlot]->getBuffer() == nullptr ||
            mImages[item->mSlot]->getBuffer()->getId() != item->mGraphicBuffer->getId()) {
            mImages[item->mSlot] = std::make_shared<
                    renderengine::impl::ExternalTexture>(item->mGraphicBuffer, mRE,
                                                         renderengine::impl::ExternalTexture::
                                                                 Usage::READABLE);
        }
    }
 
 
    return NO_ERROR;
}
```

#### ConsumerBase acquireBufferLocked

BufferLayerConsumer继承于ConsumerBase，调用ConsumerBase的acquireBufferLocked方法：

```cpp
//frameworks/native/libs/gui/ConsumerBase.cpp
sp<IGraphicBufferConsumer> mConsumer;
status_t ConsumerBase::acquireBufferLocked(BufferItem *item,
        nsecs_t presentWhen, uint64_t maxFrameNumber) {
    if (mAbandoned) {
        CB_LOGE("acquireBufferLocked: ConsumerBase is abandoned!");
        return NO_INIT;
    }
 
 
    status_t err = mConsumer->acquireBuffer(item, presentWhen, maxFrameNumber);
    if (err != NO_ERROR) {
        return err;
    }
 
 
    if (item->mGraphicBuffer != nullptr) {
        if (mSlots[item->mSlot].mGraphicBuffer != nullptr) {
            freeBufferLocked(item->mSlot);
        }
        mSlots[item->mSlot].mGraphicBuffer = item->mGraphicBuffer;
    }
 
 
    mSlots[item->mSlot].mFrameNumber = item->mFrameNumber;
    mSlots[item->mSlot].mFence = item->mFence;
 
 
    CB_LOGV("acquireBufferLocked: -> slot=%d/%" PRIu64,
            item->mSlot, item->mFrameNumber);
 
 
    return OK;
}
```

调用mConsumer(IGraphicBufferConsumer)的acquireBuffer方法，IGraphicBufferConsumer是一个接口，由BpGraphicBufferConsumer实现：

```cpp
//frameworks/native/libs/gui/IGraphicBufferConsumer.cpp
class BpGraphicBufferConsumer : public SafeBpInterface<IGraphicBufferConsumer> {
    status_t acquireBuffer(BufferItem* buffer, nsecs_t presentWhen,
                           uint64_t maxFrameNumber) override {
        using Signature = decltype(&IGraphicBufferConsumer::acquireBuffer);
        return callRemote<Signature>(Tag::ACQUIRE_BUFFER, buffer, presentWhen, maxFrameNumber);
    }
}
```

调用callRemote方法进行远程调用，之后会运行BnGraphicBufferConsumer的onTransact方法：

```cpp
//frameworks/native/libs/gui/IGraphicBufferConsumer.cpp
status_t BnGraphicBufferConsumer::onTransact(uint32_t code, const Parcel& data, Parcel* reply,
                                             uint32_t flags) {
    if (code < IBinder::FIRST_CALL_TRANSACTION || code > static_cast<uint32_t>(Tag::LAST)) {
        return BBinder::onTransact(code, data, reply, flags);
    }
    auto tag = static_cast<Tag>(code);
    switch (tag) {
        case Tag::ACQUIRE_BUFFER:
            return callLocal(data, reply, &IGraphicBufferConsumer::acquireBuffer);
        }
    }
}
```

##### BuffQueueConsumer acquireBuffer

BuffQueueConsumer继承于BnGraphicBufferConsumer，调用BnGraphicBufferConsumer的acquireBuffer方法：

[Android13 BufferQueueConsumer acquireBuffer流程分析-CSDN博客](/Android13%20BufferQueueConsumer%20acquireBuffer%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-CSDN%E5%8D%9A%E5%AE%A2)

### BufferLayerConsumer updateAndReleaseLocked

调用BufferLayerConsumer的updateAndReleaseLocked方法，释放图像缓冲区：

```cpp
//frameworks/native/services/surfaceflinger/BufferLayerConsumer.cpp
status_t BufferLayerConsumer::updateAndReleaseLocked(const BufferItem& item,
                                                     PendingRelease* pendingRelease) {
    status_t err = NO_ERROR;
 
 
    int slot = item.mSlot;
 
 
    BLC_LOGV("updateAndRelease: (slot=%d buf=%p) -> (slot=%d buf=%p)", mCurrentTexture,
             (mCurrentTextureBuffer != nullptr && mCurrentTextureBuffer->getBuffer() != nullptr)
                     ? mCurrentTextureBuffer->getBuffer()->handle
                     : 0,
             slot, mSlots[slot].mGraphicBuffer->handle);
 
 
    // Hang onto the pointer so that it isn't freed in the call to
    // releaseBufferLocked() if we're in shared buffer mode and both buffers are
    // the same.
 
 
    std::shared_ptr<renderengine::ExternalTexture> nextTextureBuffer;
    {
        std::lock_guard<std::mutex> lock(mImagesMutex);
        nextTextureBuffer = mImages[slot];
    }
 
 
    // release old buffer
    if (mCurrentTexture != BufferQueue::INVALID_BUFFER_SLOT) {
        if (pendingRelease == nullptr) {
            status_t status =
                    releaseBufferLocked(mCurrentTexture, mCurrentTextureBuffer->getBuffer());
            if (status < NO_ERROR) {
                BLC_LOGE("updateAndRelease: failed to release buffer: %s (%d)", strerror(-status),
                         status);
                err = status;
                // keep going, with error raised [?]
            }
        } else {
            pendingRelease->currentTexture = mCurrentTexture;
            pendingRelease->graphicBuffer = mCurrentTextureBuffer->getBuffer();
            pendingRelease->isPending = true;
        }
    }
 
 
    // Update the BufferLayerConsumer state.
    mCurrentTexture = slot;
    mCurrentTextureBuffer = nextTextureBuffer;
    mCurrentCrop = item.mCrop;
    mCurrentTransform = item.mTransform;
    mCurrentScalingMode = item.mScalingMode;
    mCurrentTimestamp = item.mTimestamp;
    mCurrentDataSpace = static_cast<ui::Dataspace>(item.mDataSpace);
    mCurrentHdrMetadata = item.mHdrMetadata;
    mCurrentFence = item.mFence;
    mCurrentFenceTime = item.mFenceTime;
    mCurrentFrameNumber = item.mFrameNumber;
    mCurrentTransformToDisplayInverse = item.mTransformToDisplayInverse;
    mCurrentSurfaceDamage = item.mSurfaceDamage;
    mCurrentApi = item.mApi;
 
 
    computeCurrentTransformMatrixLocked();
 
 
    return err;
}
```

#### ConsumerBase releaseBufferLocked

BufferLayerConsumer继承于ConsumerBase，调用ConsumerBase的releaseBufferLocked方法：

```cpp
//frameworks/native/libs/gui/ConsumerBase.cpp
status_t ConsumerBase::releaseBufferLocked(
        int slot, const sp<GraphicBuffer> graphicBuffer,
        EGLDisplay display, EGLSyncKHR eglFence) {
    if (mAbandoned) {
        CB_LOGE("releaseBufferLocked: ConsumerBase is abandoned!");
        return NO_INIT;
    }
    // If consumer no longer tracks this graphicBuffer (we received a new
    // buffer on the same slot), the buffer producer is definitely no longer
    // tracking it.
    if (!stillTracking(slot, graphicBuffer)) {
        return OK;
    }
 
 
    CB_LOGV("releaseBufferLocked: slot=%d/%" PRIu64,
            slot, mSlots[slot].mFrameNumber);
    status_t err = mConsumer->releaseBuffer(slot, mSlots[slot].mFrameNumber,
            display, eglFence, mSlots[slot].mFence);
    if (err == IGraphicBufferConsumer::STALE_BUFFER_SLOT) {
        freeBufferLocked(slot);
    }
 
 
    mPrevFinalReleaseFence = mSlots[slot].mFence;
    mSlots[slot].mFence = Fence::NO_FENCE;
 
 
    return err;
}
```

调用mConsumer(IGraphicBufferConsumer)的releaseBuffer

```cpp
//frameworks/native/include/gui/IGraphicBufferConsumer.h
using ReleaseBuffer = decltype(&IGraphicBufferConsumer::releaseHelper);
//frameworks/native/libs/gui/IGraphicBufferConsumer.cpp
status_t BnGraphicBufferConsumer::onTransact(uint32_t code, const Parcel& data, Parcel* reply,
                                             uint32_t flags) {
    if (code < IBinder::FIRST_CALL_TRANSACTION || code > static_cast<uint32_t>(Tag::LAST)) {
        return BBinder::onTransact(code, data, reply, flags);
    }
    auto tag = static_cast<Tag>(code);
    switch (tag) {
        case Tag::RELEASE_BUFFER:
            return callLocal(data, reply, &IGraphicBufferConsumer::releaseHelper);
        }
    }
}
```

seBuffer方法，IGraphicBufferConsumer是一个接口，由BpGraphicBufferConsumer实现：

```cpp
//frameworks/native/libs/gui/IGraphicBufferConsumer.cpp
class BpGraphicBufferConsumer : public SafeBpInterface<IGraphicBufferConsumer> {
    status_t releaseBuffer(int buf, uint64_t frameNumber,
                           EGLDisplay display __attribute__((unused)),
                           EGLSyncKHR fence __attribute__((unused)),
                           const sp<Fence>& releaseFence) override {
        return callRemote<ReleaseBuffer>(Tag::RELEASE_BUFFER, buf, frameNumber, releaseFence);
    }
}
```

调用callRemote方法进行远程调用，之后会运行BnGraphicBufferConsumer的onTransact方法：

##### BuffQueueConsumer releaseBuffer

BuffQueueConsumer继承于BnGraphicBufferConsumer，调用BnGraphicBufferConsumer的ReleaseBuffer 方法：

[Android13 BufferQueueConsumer releaseBuffer流程分析-CSDN博客](/Android13%20BufferQueueConsumer%20releaseBuffer%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-CSDN%E5%8D%9A%E5%AE%A2)
