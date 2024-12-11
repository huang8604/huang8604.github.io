# Android13 BufferQueueProducer DequeueBuffer流程分析-CSDN博客

BufferQueueProducer的dequeueBuffer方法是Android系统中用于从BufferQueue中获取可用的缓冲区的方法。BufferQueue是一个用于在不同线程之间传递图像数据的队列，它通常用于图像渲染和视频编解码等场景。

dequeueBuffer方法的作用是从BufferQueue中获取一个可用的缓冲区，并返回该缓冲区的索引。如果没有可用的缓冲区，则该方法会阻塞，直到有可用的缓冲区为止。

在调用dequeueBuffer方法之前，需要先通过BufferQueue的getBufferCount方法获取可用缓冲区的数量。然后，通过dequeueBuffer方法获取一个可用的缓冲区，并将其索引作为参数传递给其他相关的方法，如图像渲染或视频编解码等。

代码如下：

```cpp
//frameworks/native/libs/gui/BufferQueueProducer.cpp
class BufferQueueProducer : public BnGraphicBufferProducer {
status_t BufferQueueProducer::dequeueBuffer(int* outSlot, sp&lt;android::Fence&gt;* outFence,
                                            uint32_t width, uint32_t height, PixelFormat format,
                                            uint64_t usage, uint64_t* outBufferAge,
                                            FrameEventHistoryDelta* outTimestamps) {
    ATRACE_CALL();
    { // Autolock scope
        std::lock_guard&lt;std::mutex&gt; lock(mCore-&gt;mMutex);
        mConsumerName = mCore-&gt;mConsumerName;
 
 
        if (mCore-&gt;mIsAbandoned) {
            BQ_LOGE(&#34;dequeueBuffer: BufferQueue has been abandoned&#34;);
            return NO_INIT;
        }
 
 
        if (mCore-&gt;mConnectedApi == BufferQueueCore::NO_CONNECTED_API) {
            BQ_LOGE(&#34;dequeueBuffer: BufferQueue has no connected producer&#34;);
            return NO_INIT;
        }
    } // Autolock scope
 
 
    BQ_LOGV(&#34;dequeueBuffer: w=%u h=%u format=%#x, usage=%#&#34; PRIx64, width, height, format, usage);
 
 
    if ((width &amp;&amp; !height) || (!width &amp;&amp; height)) {
        BQ_LOGE(&#34;dequeueBuffer: invalid size: w=%u h=%u&#34;, width, height);
        return BAD_VALUE;
    }
 
 
    status_t returnFlags = NO_ERROR;
    EGLDisplay eglDisplay = EGL_NO_DISPLAY;
    EGLSyncKHR eglFence = EGL_NO_SYNC_KHR;
    bool attachedByConsumer = false;
 
 
    { // Autolock scope
        std::unique_lock&lt;std::mutex&gt; lock(mCore-&gt;mMutex);
 
 
        // If we don&#39;t have a free buffer, but we are currently allocating, we wait until allocation
        // is finished such that we don&#39;t allocate in parallel.
        if (mCore-&gt;mFreeBuffers.empty() &amp;&amp; mCore-&gt;mIsAllocating) {
            mDequeueWaitingForAllocation = true;
            mCore-&gt;waitWhileAllocatingLocked(lock);
            mDequeueWaitingForAllocation = false;
            mDequeueWaitingForAllocationCondition.notify_all();
        }
 
 
        if (format == 0) {
            format = mCore-&gt;mDefaultBufferFormat;
        }
 
 
        // Enable the usage bits the consumer requested
        usage |= mCore-&gt;mConsumerUsageBits;
 
 
        const bool useDefaultSize = !width &amp;&amp; !height;
        if (useDefaultSize) {
            width = mCore-&gt;mDefaultWidth;
            height = mCore-&gt;mDefaultHeight;
            if (mCore-&gt;mAutoPrerotation &amp;&amp;
                (mCore-&gt;mTransformHintInUse &amp; NATIVE_WINDOW_TRANSFORM_ROT_90)) {
                std::swap(width, height);
            }
        }
 
 
        int found = BufferItem::INVALID_BUFFER_SLOT;
        while (found == BufferItem::INVALID_BUFFER_SLOT) {
            status_t status = waitForFreeSlotThenRelock(FreeSlotCaller::Dequeue, lock, &amp;found);
            if (status != NO_ERROR) {
                return status;
            }
 
 
            // This should not happen
            if (found == BufferQueueCore::INVALID_BUFFER_SLOT) {
                BQ_LOGE(&#34;dequeueBuffer: no available buffer slots&#34;);
                return -EBUSY;
            }
 
 
            const sp&lt;GraphicBuffer&gt;&amp; buffer(mSlots[found].mGraphicBuffer);
 
 
            // If we are not allowed to allocate new buffers,
            // waitForFreeSlotThenRelock must have returned a slot containing a
            // buffer. If this buffer would require reallocation to meet the
            // requested attributes, we free it and attempt to get another one.
            if (!mCore-&gt;mAllowAllocation) {
                if (buffer-&gt;needsReallocation(width, height, format, BQ_LAYER_COUNT, usage)) { //检查是否已分配了GraphicBuffer
                    if (mCore-&gt;mSharedBufferSlot == found) {
                        BQ_LOGE(&#34;dequeueBuffer: cannot re-allocate a sharedbuffer&#34;);
                        return BAD_VALUE;
                    }
                    mCore-&gt;mFreeSlots.insert(found);
                    mCore-&gt;clearBufferSlotLocked(found);
                    found = BufferItem::INVALID_BUFFER_SLOT;
                    continue;
                }
            }
        }
 
 
        const sp&lt;GraphicBuffer&gt;&amp; buffer(mSlots[found].mGraphicBuffer);
        if (mCore-&gt;mSharedBufferSlot == found &amp;&amp;
                buffer-&gt;needsReallocation(width, height, format, BQ_LAYER_COUNT, usage)) {
            BQ_LOGE(&#34;dequeueBuffer: cannot re-allocate a shared&#34;
                    &#34;buffer&#34;);
 
 
            return BAD_VALUE;
        }
 
 
        if (mCore-&gt;mSharedBufferSlot != found) {
            mCore-&gt;mActiveBuffers.insert(found);
        }
        *outSlot = found;
        ATRACE_BUFFER_INDEX(found);
 
 
        attachedByConsumer = mSlots[found].mNeedsReallocation;
        mSlots[found].mNeedsReallocation = false;
 
 
        mSlots[found].mBufferState.dequeue();
 
 
        if ((buffer == nullptr) ||
                buffer-&gt;needsReallocation(width, height, format, BQ_LAYER_COUNT, usage))
        {
            mSlots[found].mAcquireCalled = false;
            mSlots[found].mGraphicBuffer = nullptr;
            mSlots[found].mRequestBufferCalled = false;
            mSlots[found].mEglDisplay = EGL_NO_DISPLAY;
            mSlots[found].mEglFence = EGL_NO_SYNC_KHR;
            mSlots[found].mFence = Fence::NO_FENCE;
            mCore-&gt;mBufferAge = 0;
            mCore-&gt;mIsAllocating = true;
 
 
            returnFlags |= BUFFER_NEEDS_REALLOCATION; //发现需要分配buffer,置个标记
        } else {
            // We add 1 because that will be the frame number when this buffer
            // is queued
            mCore-&gt;mBufferAge = mCore-&gt;mFrameCounter &#43; 1 - mSlots[found].mFrameNumber;
        }
 
 
        BQ_LOGV(&#34;dequeueBuffer: setting buffer age to %&#34; PRIu64,
                mCore-&gt;mBufferAge);
 
 
        if (CC_UNLIKELY(mSlots[found].mFence == nullptr)) {
            BQ_LOGE(&#34;dequeueBuffer: about to return a NULL fence - &#34;
                    &#34;slot=%d w=%d h=%d format=%u&#34;,
                    found, buffer-&gt;width, buffer-&gt;height, buffer-&gt;format);
        }
 
 
        eglDisplay = mSlots[found].mEglDisplay;
        eglFence = mSlots[found].mEglFence;
        // Don&#39;t return a fence in shared buffer mode, except for the first
        // frame.
        *outFence = (mCore-&gt;mSharedBufferMode &amp;&amp;
                mCore-&gt;mSharedBufferSlot == found) ?
                Fence::NO_FENCE : mSlots[found].mFence;
        mSlots[found].mEglFence = EGL_NO_SYNC_KHR;
        mSlots[found].mFence = Fence::NO_FENCE;
 
 
        // If shared buffer mode has just been enabled, cache the slot of the
        // first buffer that is dequeued and mark it as the shared buffer.
        if (mCore-&gt;mSharedBufferMode &amp;&amp; mCore-&gt;mSharedBufferSlot ==
                BufferQueueCore::INVALID_BUFFER_SLOT) {
            mCore-&gt;mSharedBufferSlot = found;
            mSlots[found].mBufferState.mShared = true;
        }
 
 
        if (!(returnFlags &amp; BUFFER_NEEDS_REALLOCATION)) {
            if (mCore-&gt;mConsumerListener != nullptr) {
                mCore-&gt;mConsumerListener-&gt;onFrameDequeued(mSlots[*outSlot].mGraphicBuffer-&gt;getId());
            }
        }
    } // Autolock scope
 
 
    if (returnFlags &amp; BUFFER_NEEDS_REALLOCATION) {
        BQ_LOGV(&#34;dequeueBuffer: allocating a new buffer for slot %d&#34;, *outSlot);
 //新创建一个新的GraphicBuffer给到对应的slot
 //new GraphicBuffer 见新的说明[[Android13 GraphicBuffer 创建流程-CSDN博客]]
        sp&lt;GraphicBuffer&gt; graphicBuffer = new GraphicBuffer(
                width, height, format, BQ_LAYER_COUNT, usage,
                {mConsumerName.string(), mConsumerName.size()});
 
 
        status_t error = graphicBuffer-&gt;initCheck();
 
 
        { // Autolock scope
            std::lock_guard&lt;std::mutex&gt; lock(mCore-&gt;mMutex);
 
 
            if (error == NO_ERROR &amp;&amp; !mCore-&gt;mIsAbandoned) {
                graphicBuffer-&gt;setGenerationNumber(mCore-&gt;mGenerationNumber);
                mSlots[*outSlot].mGraphicBuffer = graphicBuffer; //把GraphicBuffer给到对应的slot
                if (mCore-&gt;mConsumerListener != nullptr) {
                    mCore-&gt;mConsumerListener-&gt;onFrameDequeued(
                            mSlots[*outSlot].mGraphicBuffer-&gt;getId());
                }
            }
 
 
            mCore-&gt;mIsAllocating = false;
            mCore-&gt;mIsAllocatingCondition.notify_all();
 
 
            if (error != NO_ERROR) {
                mCore-&gt;mFreeSlots.insert(*outSlot);
                mCore-&gt;clearBufferSlotLocked(*outSlot);
                BQ_LOGE(&#34;dequeueBuffer: createGraphicBuffer failed&#34;);
                return error;
            }
 
 
            if (mCore-&gt;mIsAbandoned) {
                mCore-&gt;mFreeSlots.insert(*outSlot);
                mCore-&gt;clearBufferSlotLocked(*outSlot);
                BQ_LOGE(&#34;dequeueBuffer: BufferQueue has been abandoned&#34;);
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
        // synchronizing access to it. It&#39;s too late at this point to abort the
        // dequeue operation.
        if (result == EGL_FALSE) {
            BQ_LOGE(&#34;dequeueBuffer: error %#x waiting for fence&#34;,
                    eglGetError());
        } else if (result == EGL_TIMEOUT_EXPIRED_KHR) {
            BQ_LOGE(&#34;dequeueBuffer: timeout waiting for fence&#34;);
        }
        eglDestroySyncKHR(eglDisplay, eglFence);
    }
 
 
    BQ_LOGV(&#34;dequeueBuffer: returning slot=%d/%&#34; PRIu64 &#34; buf=%p flags=%#x&#34;,
            *outSlot,
            mSlots[*outSlot].mFrameNumber,
            mSlots[*outSlot].mGraphicBuffer-&gt;handle, returnFlags);
 
 
    if (outBufferAge) {
        *outBufferAge = mCore-&gt;mBufferAge;
    }
    addAndGetFrameTimestamps(nullptr, outTimestamps);
 
 
    return returnFlags; //注意在应用第一次请求buffer, dequeueBuffer返回时对应的GraphicBuffer已经创建完成并给到了对应的slot上，但返回给应用的flags里还是带有BUFFER_NEEDS_REALLOCATION标记的
}
}
```

## new GraphicBuffer

通过new的方式创建GraphicBuffer对象：

[Android13 GraphicBuffer 创建流程-CSDN博客](/Android13%20GraphicBuffer%20%E5%88%9B%E5%BB%BA%E6%B5%81%E7%A8%8B-CSDN%E5%8D%9A%E5%AE%A2)


---

> 作者: [wentao](https://github.com/huang8604)  
> URL: https://huang8604.github.io/posts/android13-bufferqueueproducer-dequeuebuffer%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-csdn%E5%8D%9A%E5%AE%A2/  

