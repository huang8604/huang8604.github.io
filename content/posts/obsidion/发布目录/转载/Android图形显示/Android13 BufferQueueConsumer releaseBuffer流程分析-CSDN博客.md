---
title: Android13 BufferQueueConsumer releaseBuffer流程分析-CSDN博客
author: 
created: 2024-09-27
tags:
  - clippings
  - 转载
  - blog
collections: 图形显示
source: https://blog.csdn.net/liuning1985622/article/details/138468880?spm=1001.2014.3001.5502
date: 2024-11-06T06:57:52.386Z
lastmod: 2024-11-05T01:33:58.615Z
---
BufferQueueConsumer的releaseBuffer方法用于释放图形缓冲区。

当应用程序使用图形缓冲区进行绘制或渲染操作时，需要从BufferQueueConsumer中获取可用的缓冲区。使用完毕后，可以通过调用releaseBuffer方法将缓冲区释放回给BufferQueueConsumer。

releaseBuffer方法的作用是将指定的缓冲区添加到可用缓冲区队列中，以便其他应用程序或系统可以继续使用该缓冲区进行绘制或渲染操作。释放缓冲区后，应用程序不再拥有该缓冲区的所有权。

BufferQueueConsumer的releaseBuffer方法代码如下：

```cpp
//frameworks/native/lib/gui/BufferQueueConsumer.cpp
status_t BufferQueueConsumer::releaseBuffer(int slot, uint64_t frameNumber,
        const sp<Fence>& releaseFence, EGLDisplay eglDisplay,
        EGLSyncKHR eglFence) {
    ATRACE_CALL();
    ATRACE_BUFFER_INDEX(slot);
 
 
    if (slot < 0 || slot >= BufferQueueDefs::NUM_BUFFER_SLOTS ||
            releaseFence == nullptr) {
        BQ_LOGE("releaseBuffer: slot %d out of range or fence %p NULL", slot,
                releaseFence.get());
        return BAD_VALUE;
    }
 
 
    sp<IProducerListener> listener;
    { // Autolock scope
        std::lock_guard<std::mutex> lock(mCore->mMutex);
 
 
        // If the frame number has changed because the buffer has been reallocated,
        // we can ignore this releaseBuffer for the old buffer.
        // Ignore this for the shared buffer where the frame number can easily
        // get out of sync due to the buffer being queued and acquired at the
        // same time.
        if (frameNumber != mSlots[slot].mFrameNumber &&
                !mSlots[slot].mBufferState.isShared()) {
            return STALE_BUFFER_SLOT;
        }
 
 
        if (!mSlots[slot].mBufferState.isAcquired()) {
            BQ_LOGE("releaseBuffer: attempted to release buffer slot %d "
                    "but its state was %s", slot,
                    mSlots[slot].mBufferState.string());
            return BAD_VALUE;
        }
 
 
        mSlots[slot].mEglDisplay = eglDisplay;
        mSlots[slot].mEglFence = eglFence;
        mSlots[slot].mFence = releaseFence;
        mSlots[slot].mBufferState.release();
 
 
        // After leaving shared buffer mode, the shared buffer will
        // still be around. Mark it as no longer shared if this
        // operation causes it to be free.
        if (!mCore->mSharedBufferMode && mSlots[slot].mBufferState.isFree()) {
            mSlots[slot].mBufferState.mShared = false;
        }
        // Don't put the shared buffer on the free list.
        if (!mSlots[slot].mBufferState.isShared()) {
            mCore->mActiveBuffers.erase(slot);
            mCore->mFreeBuffers.push_back(slot);
        }
 
 
        if (mCore->mBufferReleasedCbEnabled) {
            listener = mCore->mConnectedProducerListener;
        }
        BQ_LOGV("releaseBuffer: releasing slot %d", slot);
 
 
        mCore->mDequeueCondition.notify_all();
        VALIDATE_CONSISTENCY();
    } // Autolock scope
 
 
    // Call back without lock held
    if (listener != nullptr) {
        listener->onBufferReleased();
    }
 
 
    return NO_ERROR;
}
```
