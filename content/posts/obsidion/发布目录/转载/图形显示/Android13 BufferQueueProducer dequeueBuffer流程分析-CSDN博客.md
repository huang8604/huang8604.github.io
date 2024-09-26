---
title: Android13 BufferQueueProducer dequeueBuffer流程分析-CSDN博客
created: 2024-09-26
tags:
  - clippings
  - 转载
  - blog
collections:
  - 图形显示
date: 2024-09-26T07:18:32.230Z
lastmod: 2024-09-26T07:41:14.498Z
---
BufferQueueProducer的dequeueBuffer方法是Android系统中用于从BufferQueue中获取可用的缓冲区的方法。BufferQueue是一个用于在不同线程之间传递图像数据的队列，它通常用于图像渲染和视频编解码等场景。

dequeueBuffer方法的作用是从BufferQueue中获取一个可用的缓冲区，并返回该缓冲区的索引。如果没有可用的缓冲区，则该方法会阻塞，直到有可用的缓冲区为止。

在调用dequeueBuffer方法之前，需要先通过BufferQueue的getBufferCount方法获取可用缓冲区的数量。然后，通过dequeueBuffer方法获取一个可用的缓冲区，并将其索引作为参数传递给其他相关的方法，如图像渲染或视频编解码等。

代码如下：

```cpp
//frameworks/native/libs/gui/BufferQueueProducer.cpp
class BufferQueueProducer : public BnGraphicBufferProducer {
status_t BufferQueueProducer::dequeueBuffer(int* outSlot, sp<android::Fence>* outFence,
                                            uint32_t width, uint32_t height, PixelFormat format,
                                            uint64_t usage, uint64_t* outBufferAge,
                                            FrameEventHistoryDelta* outTimestamps) {
    ATRACE_CALL();
    { // Autolock scope
        std::lock_guard<std::mutex> lock(mCore->mMutex);
        mConsumerName = mCore->mConsumerName;
 
 
        if (mCore->mIsAbandoned) {
            BQ_LOGE("dequeueBuffer: BufferQueue has been abandoned");
            return NO_INIT;
        }
 
 
        if (mCore->mConnectedApi == BufferQueueCore::NO_CONNECTED_API) {
            BQ_LOGE("dequeueBuffer: BufferQueue has no connected producer");
            return NO_INIT;
        }
    } // Autolock scope
 
 
    BQ_LOGV("dequeueBuffer: w=%u h=%u format=%#x, usage=%#" PRIx64, width, height, format, usage);
 
 
    if ((width && !height) || (!width && height)) {
        BQ_LOGE("dequeueBuffer: invalid size: w=%u h=%u", width, height);
        return BAD_VALUE;
    }
 
 
    status_t returnFlags = NO_ERROR;
    EGLDisplay eglDisplay = EGL_NO_DISPLAY;
    EGLSyncKHR eglFence = EGL_NO_SYNC_KHR;
    bool attachedByConsumer = false;
 
 
    { // Autolock scope
        std::unique_lock<std::mutex> lock(mCore->mMutex);
 
 
        // If we don't have a free buffer, but we are currently allocating, we wait until allocation
        // is finished such that we don't allocate in parallel.
        if (mCore->mFreeBuffers.empty() && mCore->mIsAllocating) {
            mDequeueWaitingForAllocation = true;
            mCore->waitWhileAllocatingLocked(lock);
            mDequeueWaitingForAllocation = false;
            mDequeueWaitingForAllocationCondition.notify_all();
        }
 
 
        if (format == 0) {
            format = mCore->mDefaultBufferFormat;
        }
 
 
        // Enable the usage bits the consumer requested
        usage |= mCore->mConsumerUsageBits;
 
 
        const bool useDefaultSize = !width && !height;
        if (useDefaultSize) {
            width = mCore->mDefaultWidth;
            height = mCore->mDefaultHeight;
            if (mCore->mAutoPrerotation &&
                (mCore->mTransformHintInUse & NATIVE_WINDOW_TRANSFORM_ROT_90)) {
                std::swap(width, height);
            }
        }
 
 
        int found = BufferItem::INVALID_BUFFER_SLOT;
        while (found == BufferItem::INVALID_BUFFER_SLOT) {
            status_t status = waitForFreeSlotThenRelock(FreeSlotCaller::Dequeue, lock, &found);
            if (status != NO_ERROR) {
                return status;
            }
 
 
            // This should not happen
            if (found == BufferQueueCore::INVALID_BUFFER_SLOT) {
                BQ_LOGE("dequeueBuffer: no available buffer slots");
                return -EBUSY;
            }
 
 
            const sp<GraphicBuffer>& buffer(mSlots[found].mGraphicBuffer);
 
 
            // If we are not allowed to allocate new buffers,
            // waitForFreeSlotThenRelock must have returned a slot containing a
            // buffer. If this buffer would require reallocation to meet the
            // requested attributes, we free it and attempt to get another one.
            if (!mCore->mAllowAllocation) {
                if (buffer->needsReallocation(width, height, format, BQ_LAYER_COUNT, usage)) { //检查是否已分配了GraphicBuffer
                    if (mCore->mSharedBufferSlot == found) {
                        BQ_LOGE("dequeueBuffer: cannot re-allocate a sharedbuffer");
                        return BAD_VALUE;
                    }
                    mCore->mFreeSlots.insert(found);
                    mCore->clearBufferSlotLocked(found);
                    found = BufferItem::INVALID_BUFFER_SLOT;
                    continue;
                }
            }
        }
 
 
        const sp<GraphicBuffer>& buffer(mSlots[found].mGraphicBuffer);
        if (mCore->mSharedBufferSlot == found &&
                buffer->needsReallocation(width, height, format, BQ_LAYER_COUNT, usage)) {
            BQ_LOGE("dequeueBuffer: cannot re-allocate a shared"
                    "buffer");
 
 
            return BAD_VALUE;
        }
 
 
        if (mCore->mSharedBufferSlot != found) {
            mCore->mActiveBuffers.insert(found);
        }
        *outSlot = found;
        ATRACE_BUFFER_INDEX(found);
 
 
        attachedByConsumer = mSlots[found].mNeedsReallocation;
        mSlots[found].mNeedsReallocation = false;
 
 
        mSlots[found].mBufferState.dequeue();
 
 
        if ((buffer == nullptr) ||
                buffer->needsReallocation(width, height, format, BQ_LAYER_COUNT, usage))
        {
            mSlots[found].mAcquireCalled = false;
            mSlots[found].mGraphicBuffer = nullptr;
            mSlots[found].mRequestBufferCalled = false;
            mSlots[found].mEglDisplay = EGL_NO_DISPLAY;
            mSlots[found].mEglFence = EGL_NO_SYNC_KHR;
            mSlots[found].mFence = Fence::NO_FENCE;
            mCore->mBufferAge = 0;
            mCore->mIsAllocating = true;
 
 
            returnFlags |= BUFFER_NEEDS_REALLOCATION; //发现需要分配buffer,置个标记
        } else {
            // We add 1 because that will be the frame number when this buffer
            // is queued
            mCore->mBufferAge = mCore->mFrameCounter + 1 - mSlots[found].mFrameNumber;
        }
 
 
        BQ_LOGV("dequeueBuffer: setting buffer age to %" PRIu64,
                mCore->mBufferAge);
 
 
        if (CC_UNLIKELY(mSlots[found].mFence == nullptr)) {
            BQ_LOGE("dequeueBuffer: about to return a NULL fence - "
                    "slot=%d w=%d h=%d format=%u",
                    found, buffer->width, buffer->height, buffer->format);
        }
 
 
        eglDisplay = mSlots[found].mEglDisplay;
        eglFence = mSlots[found].mEglFence;
        // Don't return a fence in shared buffer mode, except for the first
        // frame.
        *outFence = (mCore->mSharedBufferMode &&
                mCore->mSharedBufferSlot == found) ?
                Fence::NO_FENCE : mSlots[found].mFence;
        mSlots[found].mEglFence = EGL_NO_SYNC_KHR;
        mSlots[found].mFence = Fence::NO_FENCE;
 
 
        // If shared buffer mode has just been enabled, cache the slot of the
        // first buffer that is dequeued and mark it as the shared buffer.
        if (mCore->mSharedBufferMode && mCore->mSharedBufferSlot ==
                BufferQueueCore::INVALID_BUFFER_SLOT) {
            mCore->mSharedBufferSlot = found;
            mSlots[found].mBufferState.mShared = true;
        }
 
 
        if (!(returnFlags & BUFFER_NEEDS_REALLOCATION)) {
            if (mCore->mConsumerListener != nullptr) {
                mCore->mConsumerListener->onFrameDequeued(mSlots[*outSlot].mGraphicBuffer->getId());
            }
        }
    } // Autolock scope
 
 
    if (returnFlags & BUFFER_NEEDS_REALLOCATION) {
        BQ_LOGV("dequeueBuffer: allocating a new buffer for slot %d", *outSlot);
 //新创建一个新的GraphicBuffer给到对应的slot
 //new GraphicBuffer 见新的说明[[Android13 GraphicBuffer 创建流程-CSDN博客]]
        sp<GraphicBuffer> graphicBuffer = new GraphicBuffer(
                width, height, format, BQ_LAYER_COUNT, usage,
                {mConsumerName.string(), mConsumerName.size()});
 
 
        status_t error = graphicBuffer->initCheck();
 
 
        { // Autolock scope
            std::lock_guard<std::mutex> lock(mCore->mMutex);
 
 
            if (error == NO_ERROR && !mCore->mIsAbandoned) {
                graphicBuffer->setGenerationNumber(mCore->mGenerationNumber);
                mSlots[*outSlot].mGraphicBuffer = graphicBuffer; //把GraphicBuffer给到对应的slot
                if (mCore->mConsumerListener != nullptr) {
                    mCore->mConsumerListener->onFrameDequeued(
                            mSlots[*outSlot].mGraphicBuffer->getId());
                }
            }
 
 
            mCore->mIsAllocating = false;
            mCore->mIsAllocatingCondition.notify_all();
 
 
            if (error != NO_ERROR) {
                mCore->mFreeSlots.insert(*outSlot);
                mCore->clearBufferSlotLocked(*outSlot);
                BQ_LOGE("dequeueBuffer: createGraphicBuffer failed");
                return error;
            }
 
 
            if (mCore->mIsAbandoned) {
                mCore->mFreeSlots.insert(*outSlot);
                mCore->clearBufferSlotLocked(*outSlot);
                BQ_LOGE("dequeueBuffer: BufferQueue has been abandoned");
                return NO_INIT;
            }
 
 
            VALIDATE_CONSISTENCY();
        } // Autolock scope
    }
 
 
    if (attachedByConsumer) {
        returnFlags |= BUFFER_NEEDS_REALLOCATION;
    }
 
 
    if (eglFence != EGL_NO_SYNC_KHR) {
        EGLint result = eglClientWaitSyncKHR(eglDisplay, eglFence, 0,
                1000000000);
        // If something goes wrong, log the error, but return the buffer without
        // synchronizing access to it. It's too late at this point to abort the
        // dequeue operation.
        if (result == EGL_FALSE) {
            BQ_LOGE("dequeueBuffer: error %#x waiting for fence",
                    eglGetError());
        } else if (result == EGL_TIMEOUT_EXPIRED_KHR) {
            BQ_LOGE("dequeueBuffer: timeout waiting for fence");
        }
        eglDestroySyncKHR(eglDisplay, eglFence);
    }
 
 
    BQ_LOGV("dequeueBuffer: returning slot=%d/%" PRIu64 " buf=%p flags=%#x",
            *outSlot,
            mSlots[*outSlot].mFrameNumber,
            mSlots[*outSlot].mGraphicBuffer->handle, returnFlags);
 
 
    if (outBufferAge) {
        *outBufferAge = mCore->mBufferAge;
    }
    addAndGetFrameTimestamps(nullptr, outTimestamps);
 
 
    return returnFlags; //注意在应用第一次请求buffer, dequeueBuffer返回时对应的GraphicBuffer已经创建完成并给到了对应的slot上，但返回给应用的flags里还是带有BUFFER_NEEDS_REALLOCATION标记的
}
}
```

## new GraphicBuffer

通过new的方式创建GraphicBuffer对象：

[Android13 GraphicBuffer 创建流程-CSDN博客](/Android13%20GraphicBuffer%20%E5%88%9B%E5%BB%BA%E6%B5%81%E7%A8%8B-CSDN%E5%8D%9A%E5%AE%A2)
