# Android13 BufferQueueProducer QueueBuffer流程分析-CSDN博客

BufferQueueProducer的queueBuffer方法用于将图形缓冲区添加到队列中。当应用程序完成对图形缓冲区的绘制后，可以调用queueBuffer方法将其提交给SurfaceFlinger进行显示。

```cpp
//frameworks/native/libs/gui/BufferQueueProducer.cpp
class BufferQueueProducer : public BnGraphicBufferProducer {
status_t BufferQueueProducer::queueBuffer(int slot,
        const QueueBufferInput &amp;input, QueueBufferOutput *output) {
    ATRACE_CALL();
    ATRACE_BUFFER_INDEX(slot);
 
 
    int64_t requestedPresentTimestamp;
    bool isAutoTimestamp;
    android_dataspace dataSpace;
    Rect crop(Rect::EMPTY_RECT);
    int scalingMode;
    uint32_t transform;
    uint32_t stickyTransform;
    sp&lt;Fence&gt; acquireFence;
    bool getFrameTimestamps = false;
    input.deflate(&amp;requestedPresentTimestamp, &amp;isAutoTimestamp, &amp;dataSpace,
            &amp;crop, &amp;scalingMode, &amp;transform, &amp;acquireFence, &amp;stickyTransform,
            &amp;getFrameTimestamps);
    const Region&amp; surfaceDamage = input.getSurfaceDamage();
    const HdrMetadata&amp; hdrMetadata = input.getHdrMetadata();
 
 
    if (acquireFence == nullptr) {
        BQ_LOGE(&#34;queueBuffer: fence is NULL&#34;);
        return BAD_VALUE;
    }
 
 
    auto acquireFenceTime = std::make_shared&lt;FenceTime&gt;(acquireFence);
 
 
    switch (scalingMode) {
        case NATIVE_WINDOW_SCALING_MODE_FREEZE:
        case NATIVE_WINDOW_SCALING_MODE_SCALE_TO_WINDOW:
        case NATIVE_WINDOW_SCALING_MODE_SCALE_CROP:
        case NATIVE_WINDOW_SCALING_MODE_NO_SCALE_CROP:
            break;
        default:
            BQ_LOGE(&#34;queueBuffer: unknown scaling mode %d&#34;, scalingMode);
            return BAD_VALUE;
    }
 
 
    sp&lt;IConsumerListener&gt; frameAvailableListener;
    sp&lt;IConsumerListener&gt; frameReplacedListener;
    int callbackTicket = 0;
    uint64_t currentFrameNumber = 0;
    BufferItem item;
    { // Autolock scope
        std::lock_guard&lt;std::mutex&gt; lock(mCore-&gt;mMutex);
 
 
        if (mCore-&gt;mIsAbandoned) {
            BQ_LOGE(&#34;queueBuffer: BufferQueue has been abandoned&#34;);
            return NO_INIT;
        }
 
 
        if (mCore-&gt;mConnectedApi == BufferQueueCore::NO_CONNECTED_API) {
            BQ_LOGE(&#34;queueBuffer: BufferQueue has no connected producer&#34;);
            return NO_INIT;
        }
 
 
        if (slot &lt; 0 || slot &gt;= BufferQueueDefs::NUM_BUFFER_SLOTS) {
            BQ_LOGE(&#34;queueBuffer: slot index %d out of range [0, %d)&#34;,
                    slot, BufferQueueDefs::NUM_BUFFER_SLOTS);
            return BAD_VALUE;
        } else if (!mSlots[slot].mBufferState.isDequeued()) {
            BQ_LOGE(&#34;queueBuffer: slot %d is not owned by the producer &#34;
                    &#34;(state = %s)&#34;, slot, mSlots[slot].mBufferState.string());
            return BAD_VALUE;
        } else if (!mSlots[slot].mRequestBufferCalled) {
            BQ_LOGE(&#34;queueBuffer: slot %d was queued without requesting &#34;
                    &#34;a buffer&#34;, slot);
            return BAD_VALUE;
        }
 
 
        // If shared buffer mode has just been enabled, cache the slot of the
        // first buffer that is queued and mark it as the shared buffer.
        if (mCore-&gt;mSharedBufferMode &amp;&amp; mCore-&gt;mSharedBufferSlot ==
                BufferQueueCore::INVALID_BUFFER_SLOT) {
            mCore-&gt;mSharedBufferSlot = slot;
            mSlots[slot].mBufferState.mShared = true;
        }
 
 
        BQ_LOGV(&#34;queueBuffer: slot=%d/%&#34; PRIu64 &#34; time=%&#34; PRIu64 &#34; dataSpace=%d&#34;
                &#34; validHdrMetadataTypes=0x%x crop=[%d,%d,%d,%d] transform=%#x scale=%s&#34;,
                slot, mCore-&gt;mFrameCounter &#43; 1, requestedPresentTimestamp, dataSpace,
                hdrMetadata.validTypes, crop.left, crop.top, crop.right, crop.bottom,
                transform,
                BufferItem::scalingModeName(static_cast&lt;uint32_t&gt;(scalingMode)));
 
 
        const sp&lt;GraphicBuffer&gt;&amp; graphicBuffer(mSlots[slot].mGraphicBuffer);
        Rect bufferRect(graphicBuffer-&gt;getWidth(), graphicBuffer-&gt;getHeight());
        Rect croppedRect(Rect::EMPTY_RECT);
        crop.intersect(bufferRect, &amp;croppedRect);
        if (croppedRect != crop) {
            BQ_LOGE(&#34;queueBuffer: crop rect is not contained within the &#34;
                    &#34;buffer in slot %d&#34;, slot);
            return BAD_VALUE;
        }
 
 
        // Override UNKNOWN dataspace with consumer default
        if (dataSpace == HAL_DATASPACE_UNKNOWN) {
            dataSpace = mCore-&gt;mDefaultBufferDataSpace;
        }
 
 
        mSlots[slot].mFence = acquireFence;
        mSlots[slot].mBufferState.queue();
 
 
        // Increment the frame counter and store a local version of it
        // for use outside the lock on mCore-&gt;mMutex.
        &#43;&#43;mCore-&gt;mFrameCounter;
        currentFrameNumber = mCore-&gt;mFrameCounter;
        mSlots[slot].mFrameNumber = currentFrameNumber;
 
 
        item.mAcquireCalled = mSlots[slot].mAcquireCalled;
        item.mGraphicBuffer = mSlots[slot].mGraphicBuffer;
        item.mCrop = crop;
        item.mTransform = transform &amp;
                ~static_cast&lt;uint32_t&gt;(NATIVE_WINDOW_TRANSFORM_INVERSE_DISPLAY);
        item.mTransformToDisplayInverse =
                (transform &amp; NATIVE_WINDOW_TRANSFORM_INVERSE_DISPLAY) != 0;
        item.mScalingMode = static_cast&lt;uint32_t&gt;(scalingMode);
        item.mTimestamp = requestedPresentTimestamp;
        item.mIsAutoTimestamp = isAutoTimestamp;
        item.mDataSpace = dataSpace;
        item.mHdrMetadata = hdrMetadata;
        item.mFrameNumber = currentFrameNumber;
        item.mSlot = slot;
        item.mFence = acquireFence;
        item.mFenceTime = acquireFenceTime;
        item.mIsDroppable = mCore-&gt;mAsyncMode ||
                (mConsumerIsSurfaceFlinger &amp;&amp; mCore-&gt;mQueueBufferCanDrop) ||
                (mCore-&gt;mLegacyBufferDrop &amp;&amp; mCore-&gt;mQueueBufferCanDrop) ||
                (mCore-&gt;mSharedBufferMode &amp;&amp; mCore-&gt;mSharedBufferSlot == slot);
        item.mSurfaceDamage = surfaceDamage;
        item.mQueuedBuffer = true;
        item.mAutoRefresh = mCore-&gt;mSharedBufferMode &amp;&amp; mCore-&gt;mAutoRefresh;
        item.mApi = mCore-&gt;mConnectedApi;
 
 
        mStickyTransform = stickyTransform;
 
 
        // Cache the shared buffer data so that the BufferItem can be recreated.
        if (mCore-&gt;mSharedBufferMode) {
            mCore-&gt;mSharedBufferCache.crop = crop;
            mCore-&gt;mSharedBufferCache.transform = transform;
            mCore-&gt;mSharedBufferCache.scalingMode = static_cast&lt;uint32_t&gt;(
                    scalingMode);
            mCore-&gt;mSharedBufferCache.dataspace = dataSpace;
        }
 
 
        output-&gt;bufferReplaced = false;
        if (mCore-&gt;mQueue.empty()) {
            // When the queue is empty, we can ignore mDequeueBufferCannotBlock
            // and simply queue this buffer
            mCore-&gt;mQueue.push_back(item);
            frameAvailableListener = mCore-&gt;mConsumerListener;
        } else {
            // When the queue is not empty, we need to look at the last buffer
            // in the queue to see if we need to replace it
            const BufferItem&amp; last = mCore-&gt;mQueue.itemAt(
                    mCore-&gt;mQueue.size() - 1);
            if (last.mIsDroppable) {
 
 
                if (!last.mIsStale) {
                    mSlots[last.mSlot].mBufferState.freeQueued();
 
 
                    // After leaving shared buffer mode, the shared buffer will
                    // still be around. Mark it as no longer shared if this
                    // operation causes it to be free.
                    if (!mCore-&gt;mSharedBufferMode &amp;&amp;
                            mSlots[last.mSlot].mBufferState.isFree()) {
                        mSlots[last.mSlot].mBufferState.mShared = false;
                    }
                    // Don&#39;t put the shared buffer on the free list.
                    if (!mSlots[last.mSlot].mBufferState.isShared()) {
                        mCore-&gt;mActiveBuffers.erase(last.mSlot);
                        mCore-&gt;mFreeBuffers.push_back(last.mSlot);
                        output-&gt;bufferReplaced = true;
                    }
                }
 
 
                // Make sure to merge the damage rect from the frame we&#39;re about
                // to drop into the new frame&#39;s damage rect.
                if (last.mSurfaceDamage.bounds() == Rect::INVALID_RECT ||
                    item.mSurfaceDamage.bounds() == Rect::INVALID_RECT) {
                    item.mSurfaceDamage = Region::INVALID_REGION;
                } else {
                    item.mSurfaceDamage |= last.mSurfaceDamage;
                }
 
 
                // Overwrite the droppable buffer with the incoming one
                mCore-&gt;mQueue.editItemAt(mCore-&gt;mQueue.size() - 1) = item;
                frameReplacedListener = mCore-&gt;mConsumerListener;
            } else {
                mCore-&gt;mQueue.push_back(item);
                frameAvailableListener = mCore-&gt;mConsumerListener;
            }
        }
 
 
        mCore-&gt;mBufferHasBeenQueued = true;
        mCore-&gt;mDequeueCondition.notify_all();
        mCore-&gt;mLastQueuedSlot = slot;
 
 
        output-&gt;width = mCore-&gt;mDefaultWidth;
        output-&gt;height = mCore-&gt;mDefaultHeight;
        output-&gt;transformHint = mCore-&gt;mTransformHintInUse = mCore-&gt;mTransformHint;
        output-&gt;numPendingBuffers = static_cast&lt;uint32_t&gt;(mCore-&gt;mQueue.size());
        output-&gt;nextFrameNumber = mCore-&gt;mFrameCounter &#43; 1;
 
 
        ATRACE_INT(mCore-&gt;mConsumerName.string(),
                static_cast&lt;int32_t&gt;(mCore-&gt;mQueue.size()));
#ifndef NO_BINDER
        mCore-&gt;mOccupancyTracker.registerOccupancyChange(mCore-&gt;mQueue.size());
#endif
        // Take a ticket for the callback functions
        callbackTicket = mNextCallbackTicket&#43;&#43;;
 
 
        VALIDATE_CONSISTENCY();
    } // Autolock scope
 
 
    // It is okay not to clear the GraphicBuffer when the consumer is SurfaceFlinger because
    // it is guaranteed that the BufferQueue is inside SurfaceFlinger&#39;s process and
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
    addAndGetFrameTimestamps(&amp;newFrameEventsEntry,
            getFrameTimestamps ? &amp;output-&gt;frameTimestamps : nullptr);
 
 
    // Call back without the main BufferQueue lock held, but with the callback
    // lock held so we can ensure that callbacks occur in order
 
 
    int connectedApi;
    sp&lt;Fence&gt; lastQueuedFence;
 
 
    { // scope for the lock
        std::unique_lock&lt;std::mutex&gt; lock(mCallbackMutex);
        while (callbackTicket != mCurrentCallbackTicket) {
            mCallbackCondition.wait(lock);
        }
 
 
        if (frameAvailableListener != nullptr) {
            frameAvailableListener-&gt;onFrameAvailable(item); //通知消费者去消费
        } else if (frameReplacedListener != nullptr) {
            frameReplacedListener-&gt;onFrameReplaced(item);
        }
 
 
        connectedApi = mCore-&gt;mConnectedApi;
        lastQueuedFence = std::move(mLastQueueBufferFence);
 
 
        mLastQueueBufferFence = std::move(acquireFence);
        mLastQueuedCrop = item.mCrop;
        mLastQueuedTransform = item.mTransform;
 
 
        &#43;&#43;mCurrentCallbackTicket;
        mCallbackCondition.notify_all();
    }
 
 
    // Wait without lock held
    if (connectedApi == NATIVE_WINDOW_API_EGL) {
        // Waiting here allows for two full buffers to be queued but not a
        // third. In the event that frames take varying time, this makes a
        // small trade-off in favor of latency rather than throughput.
        lastQueuedFence-&gt;waitForever(&#34;Throttling EGL Production&#34;);
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
class BpConsumerListener : public SafeBpInterface&lt;IConsumerListener&gt; {
    void onFrameAvailable(const BufferItem&amp; item) override {
        callRemoteAsync&lt;decltype(&amp;IConsumerListener::onFrameAvailable)&gt;(Tag::ON_FRAME_AVAILABLE,
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
            const BufferItem&amp; item) {
        sp&lt;ConsumerListener&gt; listener(mConsumerListener.promote());
        if (listener != nullptr) {
            listener-&gt;onFrameAvailable(item);
        }
    }
}
```

BufferQueue::ProxyConsumerListener中的listener又是ConsumerBase。

```cpp
//frameworks/native/libs/gui/ConsumerBase.cpp
class ConsumerBase : public virtual RefBase, protected ConsumerListener {
    void ConsumerBase::onFrameAvailable(const BufferItem&amp; item) {
        CB_LOGV(&#34;onFrameAvailable&#34;);
 
 
        sp&lt;FrameAvailableListener&gt; listener;
        { // scope for the lock
            Mutex::Autolock lock(mFrameAvailableMutex);
            listener = mFrameAvailableListener.promote();
        }
 
 
        if (listener != nullptr) {
            CB_LOGV(&#34;actually calling onFrameAvailable&#34;);
            listener-&gt;onFrameAvailable(item);
        }
    }
}
```

ConsumerBase中又有一个mFrameAvailableListener，这又是通过外部调用ConsumerBase的setFrameAvailableListener函数传递过来的BufferQueueLayer。

当BufferQueueLayer有新的buffer到来时，会调用之前在SurfaceFlinger中Layer的创建时注册的ContentsChangedListener的onFrameAvailable方法：

```cpp
//frameworks/native/services/surfaceflinger/BufferQueueLayer.cpp
void BufferQueueLayer::ContentsChangedListener::onFrameAvailable(const BufferItem&amp; item) {
    Mutex::Autolock lock(mMutex);
    if (mBufferQueueLayer != nullptr) {
        mBufferQueueLayer-&gt;onFrameAvailable(item);
    }
}
```

##### BufferQueueLayer onFrameAvailable

调用BufferQueueLayer的onFrameAvailable方法，通知图像或视频帧已经可用并准备好显示：

[Android13 BufferQueueLayer onFrameAvailable流程分析-CSDN博客](/Android13%20BufferQueueLayer%20onFrameAvailable%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-CSDN%E5%8D%9A%E5%AE%A2)]


---

> 作者: [wentao](https://github.com/huang8604)  
> URL: https://huang8604.github.io/posts/android13-bufferqueueproducer-queuebuffer%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-csdn%E5%8D%9A%E5%AE%A2/  

