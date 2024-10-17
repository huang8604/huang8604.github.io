# Android13 BufferQueueLayer OnFrameAvailable流程分析-CSDN博客

BufferQueueLayer的onFrameAvailable方法用于通知图像或视频帧已经可用并准备好显示，代码如下：

```cpp
//frameworks/native/services/surfaceflinger/BufferQueueLayer.cpp
sp&lt;SurfaceFlinger&gt; mFlinger;
sp&lt;BufferLayerConsumer&gt; mConsumer;
void BufferQueueLayer::onFrameAvailable(const BufferItem&amp; item) {
    const int32_t layerId = getSequence();
    const uint64_t bufferId = item.mGraphicBuffer-&gt;getId();
    mFlinger-&gt;mFrameTracer-&gt;traceTimestamp(layerId, bufferId, item.mFrameNumber, systemTime(),
                                           FrameTracer::FrameEvent::QUEUE);
    mFlinger-&gt;mFrameTracer-&gt;traceFence(layerId, bufferId, item.mFrameNumber,
                                       std::make_shared&lt;FenceTime&gt;(item.mFence),
                                       FrameTracer::FrameEvent::ACQUIRE_FENCE);
 
 
    ATRACE_CALL();
    // Add this buffer from our internal queue tracker
    { // Autolock scope
        const nsecs_t presentTime = item.mIsAutoTimestamp ? 0 : item.mTimestamp;
 
 
        using LayerUpdateType = scheduler::LayerHistory::LayerUpdateType;
        mFlinger-&gt;mScheduler-&gt;recordLayerHistory(this, presentTime, LayerUpdateType::Buffer);
 
 
        Mutex::Autolock lock(mQueueItemLock);
        // Reset the frame number tracker when we receive the first buffer after
        // a frame number reset
        if (item.mFrameNumber == 1) {
            mLastFrameNumberReceived = 0;
        }
 
 
        // Ensure that callbacks are handled in order
        while (item.mFrameNumber != mLastFrameNumberReceived &#43; 1) {
            status_t result = mQueueItemCondition.waitRelative(mQueueItemLock, ms2ns(500));
            if (result != NO_ERROR) {
                ALOGE(&#34;[%s] Timed out waiting on callback&#34;, getDebugName());
                break;
            }
        }
 
 
        auto surfaceFrame = createSurfaceFrameForBuffer(mFrameTimelineInfo, systemTime(), mName);
 
 
        mQueueItems.push_back({item, surfaceFrame});
        mQueuedFrames&#43;&#43;;
 
 
        // Wake up any pending callbacks
        mLastFrameNumberReceived = item.mFrameNumber;
        mQueueItemCondition.broadcast();
    }
 
 
    mFlinger-&gt;mInterceptor-&gt;saveBufferUpdate(layerId, item.mGraphicBuffer-&gt;getWidth(),
                                             item.mGraphicBuffer-&gt;getHeight(), item.mFrameNumber);
 
 
    mFlinger-&gt;onLayerUpdate();
    mConsumer-&gt;onBufferAvailable(item);
}
```

上面方法主要处理如下：

1、调用mFlinger(SurfaceFlinger)内mScheduler(Scheduler)的recordLayerHistory方法。

2、调用Layer的createSurfaceFrameForBuffer方法，创建Layer。

3、调用mFlinger(SurfaceFlinger)内mInterceptor(SurfaceInterceptor)的saveBufferUpdate方法。

4、调用mFlinger(SurfaceFlinger)的onLayerUpdate方法。

5、调用mConsumer(BufferLayerConsumer)的onBufferAvailable方法。

下面分别进行分析：

## Scheduler recordLayerHistory

调用mFlinger(SurfaceFlinger)内mScheduler(Scheduler)的recordLayerHistory方法：

```cpp
//frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
LayerHistory mLayerHistory;
void Scheduler::recordLayerHistory(Layer* layer, nsecs_t presentTime,
                                   LayerHistory::LayerUpdateType updateType) {
    {
        std::scoped_lock lock(mRefreshRateConfigsLock);
        if (!mRefreshRateConfigs-&gt;canSwitch()) return;
    }
 
 
    mLayerHistory.record(layer, presentTime, systemTime(), updateType);
}
```

### LayerHistory record

调用mLayerHistory(LayerHistory)的record方法：

```cpp
//frameworks/native/services/surfaceflinger/Scheduler/LayerHistory.cpp
void LayerHistory::record(Layer* layer, nsecs_t presentTime, nsecs_t now,
                          LayerUpdateType updateType) {
    std::lock_guard lock(mLock);
    auto id = layer-&gt;getSequence();
 
 
    auto [found, layerPair] = findLayer(id);
    if (found == LayerStatus::NotFound) {
        // Offscreen layer
        ALOGV(&#34;%s: %s not registered&#34;, __func__, layer-&gt;getName().c_str());
        return;
    }
 
 
    const auto&amp; info = layerPair-&gt;second;
    const auto layerProps = LayerInfo::LayerProps{
            .visible = layer-&gt;isVisible(),
            .bounds = layer-&gt;getBounds(),
            .transform = layer-&gt;getTransform(),
            .setFrameRateVote = layer-&gt;getFrameRateForLayerTree(),
            .frameRateSelectionPriority = layer-&gt;getFrameRateSelectionPriority(),
    };
 
 
    info-&gt;setLastPresentTime(presentTime, now, updateType, mModeChangePending, layerProps);
 
 
    // Activate layer if inactive.
    if (found == LayerStatus::LayerInInactiveMap) {
        mActiveLayerInfos.insert(
                {id, std::make_pair(layerPair-&gt;first, std::move(layerPair-&gt;second))});
        mInactiveLayerInfos.erase(id);
    }
}
```

## Layer createSurfaceFrameForBuffer

调用Layer的createSurfaceFrameForBuffer方法，创建Layer：

```cpp
//frameworks/native/services/surfaceflinger/Layer.cpp
sp&lt;SurfaceFlinger&gt; mFlinger;
const std::unique_ptr&lt;frametimeline::FrameTimeline&gt; mFrameTimeline;
std::shared_ptr&lt;frametimeline::SurfaceFrame&gt; Layer::createSurfaceFrameForBuffer(
        const FrameTimelineInfo&amp; info, nsecs_t queueTime, std::string debugName) {
    auto surfaceFrame =
            mFlinger-&gt;mFrameTimeline-&gt;createSurfaceFrameForToken(info, mOwnerPid, mOwnerUid,
                                                                 getSequence(), mName, debugName,
                                                                 /*isBuffer*/ true, getGameMode());
    surfaceFrame-&gt;setActualStartTime(info.startTimeNanos);
    // For buffers, acquire fence time will set during latch.
    surfaceFrame-&gt;setActualQueueTime(queueTime);
    const auto fps = mFlinger-&gt;mScheduler-&gt;getFrameRateOverride(getOwnerUid());
    if (fps) {
        surfaceFrame-&gt;setRenderRate(*fps);
    }
    // TODO(b/178542907): Implement onSurfaceFrameCreated for BQLayer as well.
    onSurfaceFrameCreated(surfaceFrame);
    return surfaceFrame;
}
```

调用mFlinger(SurfaceFlinger)中mFrameTimeline(FrameTimeline)的createSurfaceFrameForToken方法：

```cpp
//frameworks/native/services/surfaceflinger/FrameTimeline/FrameTimeline.cpp
std::shared_ptr&lt;SurfaceFrame&gt; FrameTimeline::createSurfaceFrameForToken(
        const FrameTimelineInfo&amp; frameTimelineInfo, pid_t ownerPid, uid_t ownerUid, int32_t layerId,
        std::string layerName, std::string debugName, bool isBuffer, GameMode gameMode) {
    ATRACE_CALL();
    if (frameTimelineInfo.vsyncId == FrameTimelineInfo::INVALID_VSYNC_ID) {
        return std::make_shared&lt;SurfaceFrame&gt;(frameTimelineInfo, ownerPid, ownerUid, layerId,
                                              std::move(layerName), std::move(debugName),
                                              PredictionState::None, TimelineItem(), mTimeStats,
                                              mJankClassificationThresholds, &amp;mTraceCookieCounter,
                                              isBuffer, gameMode);
    }
    std::optional&lt;TimelineItem&gt; predictions =
            mTokenManager.getPredictionsForToken(frameTimelineInfo.vsyncId);
    if (predictions) {
        return std::make_shared&lt;SurfaceFrame&gt;(frameTimelineInfo, ownerPid, ownerUid, layerId,
                                              std::move(layerName), std::move(debugName),
                                              PredictionState::Valid, std::move(*predictions),
                                              mTimeStats, mJankClassificationThresholds,
                                              &amp;mTraceCookieCounter, isBuffer, gameMode);
    }
    return std::make_shared&lt;SurfaceFrame&gt;(frameTimelineInfo, ownerPid, ownerUid, layerId,
                                          std::move(layerName), std::move(debugName),
                                          PredictionState::Expired, TimelineItem(), mTimeStats,
                                          mJankClassificationThresholds, &amp;mTraceCookieCounter,
                                          isBuffer, gameMode);
}
```

### new SurfaceFrame

创建SurfaceFrame对象，SurfaceFrame的构造方法如下：

```cpp
//frameworks/native/services/surfaceflinger/FrameTimeline/FrameTimeline.cpp
SurfaceFrame::SurfaceFrame(const FrameTimelineInfo&amp; frameTimelineInfo, pid_t ownerPid,
                           uid_t ownerUid, int32_t layerId, std::string layerName,
                           std::string debugName, PredictionState predictionState,
                           frametimeline::TimelineItem&amp;&amp; predictions,
                           std::shared_ptr&lt;TimeStats&gt; timeStats,
                           JankClassificationThresholds thresholds,
                           TraceCookieCounter* traceCookieCounter, bool isBuffer, GameMode gameMode)
      : mToken(frameTimelineInfo.vsyncId),
        mInputEventId(frameTimelineInfo.inputEventId),
        mOwnerPid(ownerPid),
        mOwnerUid(ownerUid),
        mLayerName(std::move(layerName)),
        mDebugName(std::move(debugName)),
        mLayerId(layerId),
        mPresentState(PresentState::Unknown),
        mPredictionState(predictionState),
        mPredictions(predictions),
        mActuals({0, 0, 0}),
        mTimeStats(timeStats),
        mJankClassificationThresholds(thresholds),
        mTraceCookieCounter(*traceCookieCounter),
        mIsBuffer(isBuffer),
        mGameMode(gameMode) {}
```

## SurfaceInterceptor saveBufferUpdate

调用mFlinger(SurfaceFlinger)内mInterceptor(SurfaceInterceptor)的saveBufferUpdate方法：

```cpp
//frameworks/native/services/surfaceflinger/SurfaceInterceptor.cpp
void SurfaceInterceptor::saveBufferUpdate(int32_t layerId, uint32_t width,
        uint32_t height, uint64_t frameNumber)
{
    if (!mEnabled) {
        return;
    }
    ATRACE_CALL();
    std::lock_guard&lt;std::mutex&gt; protoGuard(mTraceMutex);
    addBufferUpdateLocked(createTraceIncrementLocked(), layerId, width, height, frameNumber);
}
```

### SurfaceInterceptor addBufferUpdateLocked

调用SurfaceInterceptor的addBufferUpdateLocked方法：

```cpp
//frameworks/native/services/surfaceflinger/SurfaceInterceptor.cpp
void SurfaceInterceptor::addBufferUpdateLocked(Increment* increment, int32_t layerId,
        uint32_t width, uint32_t height, uint64_t frameNumber)
{
    BufferUpdate* update(increment-&gt;mutable_buffer_update());
    update-&gt;set_id(layerId);
    update-&gt;set_w(width);
    update-&gt;set_h(height);
    update-&gt;set_frame_number(frameNumber);
}
```

## SurfaceFlinger onLayerUpdate

调用mFlinger(SurfaceFlinger)的onLayerUpdate方法，通知SurfaceFlinger图层更新：

[Android13 SurfaceFlinger onLayerUpdate流程分析-CSDN博客](/Android13%20SurfaceFlinger%20onLayerUpdate%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-CSDN%E5%8D%9A%E5%AE%A2)

## BufferLayerConsumer onBufferAvailable

调用mConsumer(BufferLayerConsumer)的onBufferAvailable方法。

```cpp
//frameworks/native/services/surfaceflinger/BufferLayerConsumer.cpp
void BufferLayerConsumer::onBufferAvailable(const BufferItem&amp; item) {
    if (item.mGraphicBuffer != nullptr &amp;&amp; item.mSlot != BufferQueue::INVALID_BUFFER_SLOT) {
        std::lock_guard&lt;std::mutex&gt; lock(mImagesMutex);
        const std::shared_ptr&lt;renderengine::ExternalTexture&gt;&amp; oldImage = mImages[item.mSlot];
        if (oldImage == nullptr || oldImage-&gt;getBuffer() == nullptr ||
            oldImage-&gt;getBuffer()-&gt;getId() != item.mGraphicBuffer-&gt;getId()) {
            mImages[item.mSlot] = std::make_shared&lt;
                    renderengine::impl::ExternalTexture&gt;(item.mGraphicBuffer, mRE,
                                                         renderengine::impl::ExternalTexture::
                                                                 Usage::READABLE);
        }
    }
}
```


---

> 作者: [wentao](https://github.com/huang8604)  
> URL: https://huang8604.github.io/posts/android13-bufferqueuelayer-onframeavailable%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-csdn%E5%8D%9A%E5%AE%A2/  

