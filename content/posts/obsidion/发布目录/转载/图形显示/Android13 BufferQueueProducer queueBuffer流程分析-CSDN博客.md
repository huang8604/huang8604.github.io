---
title: Android13 BufferQueueProducer queueBuffer流程分析-CSDN博客
created: 2024-09-26
tags:
  - clippings
  - 转载
  - blog
collections:
  - 图形显示
date: 2024-09-26T08:38:34.333Z
lastmod: 2024-09-26T08:56:35.781Z
---
BufferQueueProducer的queueBuffer方法用于将图形缓冲区添加到队列中。当应用程序完成对图形缓冲区的绘制后，可以调用queueBuffer方法将其提交给SurfaceFlinger进行显示。

```cpp
//frameworks/native/libs/gui/BufferQueueProducer.cpp
class BufferQueueProducer : public BnGraphicBufferProducer {
status_t BufferQueueProducer::queueBuffer(int slot,
        const QueueBufferInput &input, QueueBufferOutput *output) {
    ATRACE_CALL();
    ATRACE_BUFFER_INDEX(slot);
 
 
    int64_t requestedPresentTimestamp;
    bool isAutoTimestamp;
    android_dataspace dataSpace;
    Rect crop(Rect::EMPTY_RECT);
    int scalingMode;
    uint32_t transform;
    uint32_t stickyTransform;
    sp<Fence> acquireFence;
    bool getFrameTimestamps = false;
    input.deflate(&requestedPresentTimestamp, &isAutoTimestamp, &dataSpace,
            &crop, &scalingMode, &transform, &acquireFence, &stickyTransform,
            &getFrameTimestamps);
    const Region& surfaceDamage = input.getSurfaceDamage();
    const HdrMetadata& hdrMetadata = input.getHdrMetadata();
 
 
    if (acquireFence == nullptr) {
        BQ_LOGE("queueBuffer: fence is NULL");
        return BAD_VALUE;
    }
 
 
    auto acquireFenceTime = std::make_shared<FenceTime>(acquireFence);
 
 
    switch (scalingMode) {
        case NATIVE_WINDOW_SCALING_MODE_FREEZE:
        case NATIVE_WINDOW_SCALING_MODE_SCALE_TO_WINDOW:
        case NATIVE_WINDOW_SCALING_MODE_SCALE_CROP:
        case NATIVE_WINDOW_SCALING_MODE_NO_SCALE_CROP:
            break;
        default:
            BQ_LOGE("queueBuffer: unknown scaling mode %d", scalingMode);
            return BAD_VALUE;
    }
 
 
    sp<IConsumerListener> frameAvailableListener;
    sp<IConsumerListener> frameReplacedListener;
    int callbackTicket = 0;
    uint64_t currentFrameNumber = 0;
    BufferItem item;
    { // Autolock scope
        std::lock_guard<std::mutex> lock(mCore->mMutex);
 
 
        if (mCore->mIsAbandoned) {
            BQ_LOGE("queueBuffer: BufferQueue has been abandoned");
            return NO_INIT;
        }
 
 
        if (mCore->mConnectedApi == BufferQueueCore::NO_CONNECTED_API) {
            BQ_LOGE("queueBuffer: BufferQueue has no connected producer");
            return NO_INIT;
        }
 
 
        if (slot < 0 || slot >= BufferQueueDefs::NUM_BUFFER_SLOTS) {
            BQ_LOGE("queueBuffer: slot index %d out of range [0, %d)",
                    slot, BufferQueueDefs::NUM_BUFFER_SLOTS);
            return BAD_VALUE;
        } else if (!mSlots[slot].mBufferState.isDequeued()) {
            BQ_LOGE("queueBuffer: slot %d is not owned by the producer "
                    "(state = %s)", slot, mSlots[slot].mBufferState.string());
            return BAD_VALUE;
        } else if (!mSlots[slot].mRequestBufferCalled) {
            BQ_LOGE("queueBuffer: slot %d was queued without requesting "
                    "a buffer", slot);
            return BAD_VALUE;
        }
 
 
        // If shared buffer mode has just been enabled, cache the slot of the
        // first buffer that is queued and mark it as the shared buffer.
        if (mCore->mSharedBufferMode && mCore->mSharedBufferSlot ==
                BufferQueueCore::INVALID_BUFFER_SLOT) {
            mCore->mSharedBufferSlot = slot;
            mSlots[slot].mBufferState.mShared = true;
        }
 
 
        BQ_LOGV("queueBuffer: slot=%d/%" PRIu64 " time=%" PRIu64 " dataSpace=%d"
                " validHdrMetadataTypes=0x%x crop=[%d,%d,%d,%d] transform=%#x scale=%s",
                slot, mCore->mFrameCounter + 1, requestedPresentTimestamp, dataSpace,
                hdrMetadata.validTypes, crop.left, crop.top, crop.right, crop.bottom,
                transform,
                BufferItem::scalingModeName(static_cast<uint32_t>(scalingMode)));
 
 
        const sp<GraphicBuffer>& graphicBuffer(mSlots[slot].mGraphicBuffer);
        Rect bufferRect(graphicBuffer->getWidth(), graphicBuffer->getHeight());
        Rect croppedRect(Rect::EMPTY_RECT);
        crop.intersect(bufferRect, &croppedRect);
        if (croppedRect != crop) {
            BQ_LOGE("queueBuffer: crop rect is not contained within the "
                    "buffer in slot %d", slot);
            return BAD_VALUE;
        }
 
 
        // Override UNKNOWN dataspace with consumer default
        if (dataSpace == HAL_DATASPACE_UNKNOWN) {
            dataSpace = mCore->mDefaultBufferDataSpace;
        }
 
 
        mSlots[slot].mFence = acquireFence;
        mSlots[slot].mBufferState.queue();
 
 
        // Increment the frame counter and store a local version of it
        // for use outside the lock on mCore->mMutex.
        ++mCore->mFrameCounter;
        currentFrameNumber = mCore->mFrameCounter;
        mSlots[slot].mFrameNumber = currentFrameNumber;
 
 
        item.mAcquireCalled = mSlots[slot].mAcquireCalled;
        item.mGraphicBuffer = mSlots[slot].mGraphicBuffer;
        item.mCrop = crop;
        item.mTransform = transform &
                ~static_cast<uint32_t>(NATIVE_WINDOW_TRANSFORM_INVERSE_DISPLAY);
        item.mTransformToDisplayInverse =
                (transform & NATIVE_WINDOW_TRANSFORM_INVERSE_DISPLAY) != 0;
        item.mScalingMode = static_cast<uint32_t>(scalingMode);
        item.mTimestamp = requestedPresentTimestamp;
        item.mIsAutoTimestamp = isAutoTimestamp;
        item.mDataSpace = dataSpace;
        item.mHdrMetadata = hdrMetadata;
        item.mFrameNumber = currentFrameNumber;
        item.mSlot = slot;
        item.mFence = acquireFence;
        item.mFenceTime = acquireFenceTime;
        item.mIsDroppable = mCore->mAsyncMode ||
                (mConsumerIsSurfaceFlinger && mCore->mQueueBufferCanDrop) ||
                (mCore->mLegacyBufferDrop && mCore->mQueueBufferCanDrop) ||
                (mCore->mSharedBufferMode && mCore->mSharedBufferSlot == slot);
        item.mSurfaceDamage = surfaceDamage;
        item.mQueuedBuffer = true;
        item.mAutoRefresh = mCore->mSharedBufferMode && mCore->mAutoRefresh;
        item.mApi = mCore->mConnectedApi;
 
 
        mStickyTransform = stickyTransform;
 
 
        // Cache the shared buffer data so that the BufferItem can be recreated.
        if (mCore->mSharedBufferMode) {
            mCore->mSharedBufferCache.crop = crop;
            mCore->mSharedBufferCache.transform = transform;
            mCore->mSharedBufferCache.scalingMode = static_cast<uint32_t>(
                    scalingMode);
            mCore->mSharedBufferCache.dataspace = dataSpace;
        }
 
 
        output->bufferReplaced = false;
        if (mCore->mQueue.empty()) {
            // When the queue is empty, we can ignore mDequeueBufferCannotBlock
            // and simply queue this buffer
            mCore->mQueue.push_back(item);
            frameAvailableListener = mCore->mConsumerListener;
        } else {
            // When the queue is not empty, we need to look at the last buffer
            // in the queue to see if we need to replace it
            const BufferItem& last = mCore->mQueue.itemAt(
                    mCore->mQueue.size() - 1);
            if (last.mIsDroppable) {
 
 
                if (!last.mIsStale) {
                    mSlots[last.mSlot].mBufferState.freeQueued();
 
 
                    // After leaving shared buffer mode, the shared buffer will
                    // still be around. Mark it as no longer shared if this
                    // operation causes it to be free.
                    if (!mCore->mSharedBufferMode &&
                            mSlots[last.mSlot].mBufferState.isFree()) {
                        mSlots[last.mSlot].mBufferState.mShared = false;
                    }
                    // Don't put the shared buffer on the free list.
                    if (!mSlots[last.mSlot].mBufferState.isShared()) {
                        mCore->mActiveBuffers.erase(last.mSlot);
                        mCore->mFreeBuffers.push_back(last.mSlot);
                        output->bufferReplaced = true;
                    }
                }
 
 
                // Make sure to merge the damage rect from the frame we're about
                // to drop into the new frame's damage rect.
                if (last.mSurfaceDamage.bounds() == Rect::INVALID_RECT ||
                    item.mSurfaceDamage.bounds() == Rect::INVALID_RECT) {
                    item.mSurfaceDamage = Region::INVALID_REGION;
                } else {
                    item.mSurfaceDamage |= last.mSurfaceDamage;
                }
 
 
                // Overwrite the droppable buffer with the incoming one
                mCore->mQueue.editItemAt(mCore->mQueue.size() - 1) = item;
                frameReplacedListener = mCore->mConsumerListener;
            } else {
                mCore->mQueue.push_back(item);
                frameAvailableListener = mCore->mConsumerListener;
            }
        }
 
 
        mCore->mBufferHasBeenQueued = true;
        mCore->mDequeueCondition.notify_all();
        mCore->mLastQueuedSlot = slot;
 
 
        output->width = mCore->mDefaultWidth;
        output->height = mCore->mDefaultHeight;
        output->transformHint = mCore->mTransformHintInUse = mCore->mTransformHint;
        output->numPendingBuffers = static_cast<uint32_t>(mCore->mQueue.size());
        output->nextFrameNumber = mCore->mFrameCounter + 1;
 
 
        ATRACE_INT(mCore->mConsumerName.string(),
                static_cast<int32_t>(mCore->mQueue.size()));
#ifndef NO_BINDER
        mCore->mOccupancyTracker.registerOccupancyChange(mCore->mQueue.size());
#endif
        // Take a ticket for the callback functions
        callbackTicket = mNextCallbackTicket++;
 
 
        VALIDATE_CONSISTENCY();
    } // Autolock scope
 
 
    // It is okay not to clear the GraphicBuffer when the consumer is SurfaceFlinger because
    // it is guaranteed that the BufferQueue is inside SurfaceFlinger's process and
    // there will be no Binder call
    if (!mConsumerIsSurfaceFlinger) {
        item.mGraphicBuffer.clear();
    }
 
 
    // Update and get FrameEventHistory.
    nsecs_t postedTime = systemTime(SYSTEM_TIME_MONOTONIC);
    NewFrameEventsEntry newFrameEventsEntry = {
        currentFrameNumber,
        postedTime,
        requestedPresentTimestamp,
        std::move(acquireFenceTime)
    };
    addAndGetFrameTimestamps(&newFrameEventsEntry,
            getFrameTimestamps ? &output->frameTimestamps : nullptr);
 
 
    // Call back without the main BufferQueue lock held, but with the callback
    // lock held so we can ensure that callbacks occur in order
 
 
    int connectedApi;
    sp<Fence> lastQueuedFence;
 
 
    { // scope for the lock
        std::unique_lock<std::mutex> lock(mCallbackMutex);
        while (callbackTicket != mCurrentCallbackTicket) {
            mCallbackCondition.wait(lock);
        }
 
 
        if (frameAvailableListener != nullptr) {
            frameAvailableListener->onFrameAvailable(item); //通知消费者去消费
        } else if (frameReplacedListener != nullptr) {
            frameReplacedListener->onFrameReplaced(item);
        }
 
 
        connectedApi = mCore->mConnectedApi;
        lastQueuedFence = std::move(mLastQueueBufferFence);
 
 
        mLastQueueBufferFence = std::move(acquireFence);
        mLastQueuedCrop = item.mCrop;
        mLastQueuedTransform = item.mTransform;
 
 
        ++mCurrentCallbackTicket;
        mCallbackCondition.notify_all();
    }
 
 
    // Wait without lock held
    if (connectedApi == NATIVE_WINDOW_API_EGL) {
        // Waiting here allows for two full buffers to be queued but not a
        // third. In the event that frames take varying time, this makes a
        // small trade-off in favor of latency rather than throughput.
        lastQueuedFence->waitForever("Throttling EGL Production");
    }
 
 
    return NO_ERROR;
}
}
```

将Surface进程数据传过来，然后封装成BufferItem，放入BufferQueueCore的mQueue中，再通过frameAvailableListener通知消费者去消费。

#### BpConsumerListener onFrameAvailable

调用frameAvailableListener(IConsumerListener)的onFrameAvailable方法，IConsumerListener是一个接口由BpConsumerListener实现，调用BpConsumerListener的onFrameAvailable方法：

```cpp
//frameworks/native/libs/gui/IConsumerListener.cpp
class BpConsumerListener : public SafeBpInterface<IConsumerListener> {
    void onFrameAvailable(const BufferItem& item) override {
        callRemoteAsync<decltype(&IConsumerListener::onFrameAvailable)>(Tag::ON_FRAME_AVAILABLE,
                                                                        item);
    }
}
```

这里的frameAvailableListener是BufferQueueCore的mConsumerListener。

BufferQueueCore的mConsumerListener是ConsumerBase构造函数传过来的BufferQueue::ProxyConsumerListener。

```cpp
//frameworks/native/libs/gui/BufferLayer.cpp
class ProxyConsumerListener : public BnConsumerListener {
    void BufferQueue::ProxyConsumerListener::onFrameAvailable(
            const BufferItem& item) {
        sp<ConsumerListener> listener(mConsumerListener.promote());
        if (listener != nullptr) {
            listener->onFrameAvailable(item);
        }
    }
}
```

BufferQueue::ProxyConsumerListener中的listener又是ConsumerBase。

```cpp
//frameworks/native/libs/gui/ConsumerBase.cpp
class ConsumerBase : public virtual RefBase, protected ConsumerListener {
    void ConsumerBase::onFrameAvailable(const BufferItem& item) {
        CB_LOGV("onFrameAvailable");
 
 
        sp<FrameAvailableListener> listener;
        { // scope for the lock
            Mutex::Autolock lock(mFrameAvailableMutex);
            listener = mFrameAvailableListener.promote();
        }
 
 
        if (listener != nullptr) {
            CB_LOGV("actually calling onFrameAvailable");
            listener->onFrameAvailable(item);
        }
    }
}
```

ConsumerBase中又有一个mFrameAvailableListener，这又是通过外部调用ConsumerBase的setFrameAvailableListener函数传递过来的BufferQueueLayer。

当BufferQueueLayer有新的buffer到来时，会调用之前在SurfaceFlinger中Layer的创建时注册的ContentsChangedListener的onFrameAvailable方法：

```cpp
//frameworks/native/services/surfaceflinger/BufferQueueLayer.cpp
void BufferQueueLayer::ContentsChangedListener::onFrameAvailable(const BufferItem& item) {
    Mutex::Autolock lock(mMutex);
    if (mBufferQueueLayer != nullptr) {
        mBufferQueueLayer->onFrameAvailable(item);
    }
}
```

##### BufferQueueLayer onFrameAvailable

调用BufferQueueLayer的onFrameAvailable方法，通知图像或视频帧已经可用并准备好显示：

[Android13 BufferQueueLayer onFrameAvailable流程分析-CSDN博客](/Android13%20BufferQueueLayer%20onFrameAvailable%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-CSDN%E5%8D%9A%E5%AE%A2)]
