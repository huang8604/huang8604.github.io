---
title: Android13 BufferQueueConsumer acquireBuffer流程分析-CSDN博客
author: 
created: 2024-09-27
tags:
  - clippings
  - 转载
  - blog
collections: 图形显示
source: https://blog.csdn.net/liuning1985622/article/details/138468857?spm=1001.2014.3001.5502
date: 2024-11-06T06:57:52.203Z
lastmod: 2024-11-05T01:33:58.694Z
---
BufferQueueConsumer的acquireBuffer方法的主要作用是从BufferQueue中获取一个可用的图像缓冲区，并返回一个GraphicBuffer对象。它可以用于在应用程序中进行图像处理、渲染或显示等操作。

在调用acquireBuffer方法时，它会首先检查是否有可用的图像缓冲区。如果有可用的缓冲区，则会将其标记为“已使用”，并返回该缓冲区的GraphicBuffer对象。如果没有可用的缓冲区，则会等待直到有可用的缓冲区为止。

BufferQueueConsumer的acquireBuffer方法代码如下：

```cpp
//frameworks/native/libs/gui/BufferQueueConsumer.cpp
status_t BufferQueueConsumer::acquireBuffer(BufferItem* outBuffer,
        nsecs_t expectedPresent, uint64_t maxFrameNumber) {
    ATRACE_CALL();
 
 
    int numDroppedBuffers = 0;
    sp<IProducerListener> listener;
    {
        std::unique_lock<std::mutex> lock(mCore->mMutex);
 
 
        // Check that the consumer doesn't currently have the maximum number of
        // buffers acquired. We allow the max buffer count to be exceeded by one
        // buffer so that the consumer can successfully set up the newly acquired
        // buffer before releasing the old one.
        int numAcquiredBuffers = 0;
        for (int s : mCore->mActiveBuffers) {
            if (mSlots[s].mBufferState.isAcquired()) {
                ++numAcquiredBuffers;
            }
        }
        const bool acquireNonDroppableBuffer = mCore->mAllowExtraAcquire &&
                numAcquiredBuffers == mCore->mMaxAcquiredBufferCount + 1;
        if (numAcquiredBuffers >= mCore->mMaxAcquiredBufferCount + 1 &&
            !acquireNonDroppableBuffer) {
            BQ_LOGE("acquireBuffer: max acquired buffer count reached: %d (max %d)",
                    numAcquiredBuffers, mCore->mMaxAcquiredBufferCount);
            return INVALID_OPERATION;
        }
 
 
        bool sharedBufferAvailable = mCore->mSharedBufferMode &&
                mCore->mAutoRefresh && mCore->mSharedBufferSlot !=
                BufferQueueCore::INVALID_BUFFER_SLOT;
 
 
        // In asynchronous mode the list is guaranteed to be one buffer deep,
        // while in synchronous mode we use the oldest buffer.
        if (mCore->mQueue.empty() && !sharedBufferAvailable) {
            return NO_BUFFER_AVAILABLE;
        }
 
 
        BufferQueueCore::Fifo::iterator front(mCore->mQueue.begin());
 
 
        // If expectedPresent is specified, we may not want to return a buffer yet.
        // If it's specified and there's more than one buffer queued, we may want
        // to drop a buffer.
        // Skip this if we're in shared buffer mode and the queue is empty,
        // since in that case we'll just return the shared buffer.
        if (expectedPresent != 0 && !mCore->mQueue.empty()) {
            // The 'expectedPresent' argument indicates when the buffer is expected
            // to be presented on-screen. If the buffer's desired present time is
            // earlier (less) than expectedPresent -- meaning it will be displayed
            // on time or possibly late if we show it as soon as possible -- we
            // acquire and return it. If we don't want to display it until after the
            // expectedPresent time, we return PRESENT_LATER without acquiring it.
            //
            // To be safe, we don't defer acquisition if expectedPresent is more
            // than one second in the future beyond the desired present time
            // (i.e., we'd be holding the buffer for a long time).
            //
            // NOTE: Code assumes monotonic time values from the system clock
            // are positive.
 
 
            // Start by checking to see if we can drop frames. We skip this check if
            // the timestamps are being auto-generated by Surface. If the app isn't
            // generating timestamps explicitly, it probably doesn't want frames to
            // be discarded based on them.
            while (mCore->mQueue.size() > 1 && !mCore->mQueue[0].mIsAutoTimestamp) {
                const BufferItem& bufferItem(mCore->mQueue[1]);
 
 
                // If dropping entry[0] would leave us with a buffer that the
                // consumer is not yet ready for, don't drop it.
                if (maxFrameNumber && bufferItem.mFrameNumber > maxFrameNumber) {
                    break;
                }
 
 
                // If entry[1] is timely, drop entry[0] (and repeat). We apply an
                // additional criterion here: we only drop the earlier buffer if our
                // desiredPresent falls within +/- 1 second of the expected present.
                // Otherwise, bogus desiredPresent times (e.g., 0 or a small
                // relative timestamp), which normally mean "ignore the timestamp
                // and acquire immediately", would cause us to drop frames.
                //
                // We may want to add an additional criterion: don't drop the
                // earlier buffer if entry[1]'s fence hasn't signaled yet.
                nsecs_t desiredPresent = bufferItem.mTimestamp;
                if (desiredPresent < expectedPresent - MAX_REASONABLE_NSEC ||
                        desiredPresent > expectedPresent) {
                    // This buffer is set to display in the near future, or
                    // desiredPresent is garbage. Either way we don't want to drop
                    // the previous buffer just to get this on the screen sooner.
                    BQ_LOGV("acquireBuffer: nodrop desire=%" PRId64 " expect=%"
                            PRId64 " (%" PRId64 ") now=%" PRId64,
                            desiredPresent, expectedPresent,
                            desiredPresent - expectedPresent,
                            systemTime(CLOCK_MONOTONIC));
                    break;
                }
 
 
                BQ_LOGV("acquireBuffer: drop desire=%" PRId64 " expect=%" PRId64
                        " size=%zu",
                        desiredPresent, expectedPresent, mCore->mQueue.size());
 
 
                if (!front->mIsStale) {
                    // Front buffer is still in mSlots, so mark the slot as free
                    mSlots[front->mSlot].mBufferState.freeQueued();
 
 
                    // After leaving shared buffer mode, the shared buffer will
                    // still be around. Mark it as no longer shared if this
                    // operation causes it to be free.
                    if (!mCore->mSharedBufferMode &&
                            mSlots[front->mSlot].mBufferState.isFree()) {
                        mSlots[front->mSlot].mBufferState.mShared = false;
                    }
 
 
                    // Don't put the shared buffer on the free list
                    if (!mSlots[front->mSlot].mBufferState.isShared()) {
                        mCore->mActiveBuffers.erase(front->mSlot);
                        mCore->mFreeBuffers.push_back(front->mSlot);
                    }
 
 
                    if (mCore->mBufferReleasedCbEnabled) {
                        listener = mCore->mConnectedProducerListener;
                    }
                    ++numDroppedBuffers;
                }
 
 
                mCore->mQueue.erase(front);
                front = mCore->mQueue.begin();
            }
 
 
            // See if the front buffer is ready to be acquired
            nsecs_t desiredPresent = front->mTimestamp;
            bool bufferIsDue = desiredPresent <= expectedPresent ||
                    desiredPresent > expectedPresent + MAX_REASONABLE_NSEC;
            bool consumerIsReady = maxFrameNumber > 0 ?
                    front->mFrameNumber <= maxFrameNumber : true;
            if (!bufferIsDue || !consumerIsReady) {
                BQ_LOGV("acquireBuffer: defer desire=%" PRId64 " expect=%" PRId64
                        " (%" PRId64 ") now=%" PRId64 " frame=%" PRIu64
                        " consumer=%" PRIu64,
                        desiredPresent, expectedPresent,
                        desiredPresent - expectedPresent,
                        systemTime(CLOCK_MONOTONIC),
                        front->mFrameNumber, maxFrameNumber);
                ATRACE_NAME("PRESENT_LATER");
                return PRESENT_LATER;
            }
 
 
            BQ_LOGV("acquireBuffer: accept desire=%" PRId64 " expect=%" PRId64 " "
                    "(%" PRId64 ") now=%" PRId64, desiredPresent, expectedPresent,
                    desiredPresent - expectedPresent,
                    systemTime(CLOCK_MONOTONIC));
        }
 
 
        int slot = BufferQueueCore::INVALID_BUFFER_SLOT;
 
 
        if (sharedBufferAvailable && mCore->mQueue.empty()) {
            // make sure the buffer has finished allocating before acquiring it
            mCore->waitWhileAllocatingLocked(lock);
 
 
            slot = mCore->mSharedBufferSlot;
 
 
            // Recreate the BufferItem for the shared buffer from the data that
            // was cached when it was last queued.
            outBuffer->mGraphicBuffer = mSlots[slot].mGraphicBuffer;
            outBuffer->mFence = Fence::NO_FENCE;
            outBuffer->mFenceTime = FenceTime::NO_FENCE;
            outBuffer->mCrop = mCore->mSharedBufferCache.crop;
            outBuffer->mTransform = mCore->mSharedBufferCache.transform &
                    ~static_cast<uint32_t>(
                    NATIVE_WINDOW_TRANSFORM_INVERSE_DISPLAY);
            outBuffer->mScalingMode = mCore->mSharedBufferCache.scalingMode;
            outBuffer->mDataSpace = mCore->mSharedBufferCache.dataspace;
            outBuffer->mFrameNumber = mCore->mFrameCounter;
            outBuffer->mSlot = slot;
            outBuffer->mAcquireCalled = mSlots[slot].mAcquireCalled;
            outBuffer->mTransformToDisplayInverse =
                    (mCore->mSharedBufferCache.transform &
                    NATIVE_WINDOW_TRANSFORM_INVERSE_DISPLAY) != 0;
            outBuffer->mSurfaceDamage = Region::INVALID_REGION;
            outBuffer->mQueuedBuffer = false;
            outBuffer->mIsStale = false;
            outBuffer->mAutoRefresh = mCore->mSharedBufferMode &&
                    mCore->mAutoRefresh;
        } else if (acquireNonDroppableBuffer && front->mIsDroppable) {
            BQ_LOGV("acquireBuffer: front buffer is not droppable");
            return NO_BUFFER_AVAILABLE;
        } else {
            slot = front->mSlot;
            *outBuffer = *front;
        }
 
 
        ATRACE_BUFFER_INDEX(slot);
 
 
        BQ_LOGV("acquireBuffer: acquiring { slot=%d/%" PRIu64 " buffer=%p }",
                slot, outBuffer->mFrameNumber, outBuffer->mGraphicBuffer->handle);
 
 
        if (!outBuffer->mIsStale) {
            mSlots[slot].mAcquireCalled = true;
            // Don't decrease the queue count if the BufferItem wasn't
            // previously in the queue. This happens in shared buffer mode when
            // the queue is empty and the BufferItem is created above.
            if (mCore->mQueue.empty()) {
                mSlots[slot].mBufferState.acquireNotInQueue();
            } else {
                mSlots[slot].mBufferState.acquire();
            }
            mSlots[slot].mFence = Fence::NO_FENCE;
        }
 
 
        // If the buffer has previously been acquired by the consumer, set
        // mGraphicBuffer to NULL to avoid unnecessarily remapping this buffer
        // on the consumer side
        if (outBuffer->mAcquireCalled) {
            outBuffer->mGraphicBuffer = nullptr;
        }
 
 
        mCore->mQueue.erase(front);
 
 
        // We might have freed a slot while dropping old buffers, or the producer
        // may be blocked waiting for the number of buffers in the queue to
        // decrease.
        mCore->mDequeueCondition.notify_all();
 
 
        ATRACE_INT(mCore->mConsumerName.string(),
                static_cast<int32_t>(mCore->mQueue.size()));
#ifndef NO_BINDER
        mCore->mOccupancyTracker.registerOccupancyChange(mCore->mQueue.size());
#endif
        VALIDATE_CONSISTENCY();
    }
 
 
    if (listener != nullptr) {
        for (int i = 0; i < numDroppedBuffers; ++i) {
            listener->onBufferReleased();
        }
    }
 
 
    return NO_ERROR;
}
```
