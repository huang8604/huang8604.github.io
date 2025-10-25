# Android13 BufferQueueLayer OnFrameAvailable流程分析-CSDN博客

BufferQueueLayer的onFrameAvailable方法用于通知图像或视频帧已经可用并准备好显示，代码如下：

```cpp
//frameworks/native/services/surfaceflinger/BufferQueueLayer.cpp
sp<SurfaceFlinger> mFlinger;
sp<BufferLayerConsumer> mConsumer;
void BufferQueueLayer::onFrameAvailable(const BufferItem& item) {
    const int32_t layerId = getSequence();
    const uint64_t bufferId = item.mGraphicBuffer->getId();
    mFlinger->mFrameTracer->traceTimestamp(layerId, bufferId, item.mFrameNumber, systemTime(),
                                           FrameTracer::FrameEvent::QUEUE);
    mFlinger->mFrameTracer->traceFence(layerId, bufferId, item.mFrameNumber,
                                       std::make_shared<FenceTime>(item.mFence),
                                       FrameTracer::FrameEvent::ACQUIRE_FENCE);
 
 
    ATRACE_CALL();
    // Add this buffer from our internal queue tracker
    { // Autolock scope
        const nsecs_t presentTime = item.mIsAutoTimestamp ? 0 : item.mTimestamp;
 
 
        using LayerUpdateType = scheduler::LayerHistory::LayerUpdateType;
        mFlinger->mScheduler->recordLayerHistory(this, presentTime, LayerUpdateType::Buffer);
 
 
        Mutex::Autolock lock(mQueueItemLock);
        // Reset the frame number tracker when we receive the first buffer after
        // a frame number reset
        if (item.mFrameNumber == 1) {
            mLastFrameNumberReceived = 0;
        }
 
 
        // Ensure that callbacks are handled in order
        while (item.mFrameNumber != mLastFrameNumberReceived + 1) {
            status_t result = mQueueItemCondition.waitRelative(mQueueItemLock, ms2ns(500));
            if (result != NO_ERROR) {
                ALOGE("[%s] Timed out waiting on callback", getDebugName());
                break;
            }
        }
 
 
        auto surfaceFrame = createSurfaceFrameForBuffer(mFrameTimelineInfo, systemTime(), mName);
 
 
        mQueueItems.push_back({item, surfaceFrame});
        mQueuedFrames++;
 
 
        // Wake up any pending callbacks
        mLastFrameNumberReceived = item.mFrameNumber;
        mQueueItemCondition.broadcast();
    }
 
 
    mFlinger->mInterceptor->saveBufferUpdate(layerId, item.mGraphicBuffer->getWidth(),
                                             item.mGraphicBuffer->getHeight(), item.mFrameNumber);
 
 
    mFlinger->onLayerUpdate();
    mConsumer->onBufferAvailable(item);
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
        if (!mRefreshRateConfigs->canSwitch()) return;
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
    auto id = layer->getSequence();
 
 
    auto [found, layerPair] = findLayer(id);
    if (found == LayerStatus::NotFound) {
        // Offscreen layer
        ALOGV("%s: %s not registered", __func__, layer->getName().c_str());
        return;
    }
 
 
    const auto& info = layerPair->second;
    const auto layerProps = LayerInfo::LayerProps{
            .visible = layer->isVisible(),
            .bounds = layer->getBounds(),
            .transform = layer->getTransform(),
            .setFrameRateVote = layer->getFrameRateForLayerTree(),
            .frameRateSelectionPriority = layer->getFrameRateSelectionPriority(),
    };
 
 
    info->setLastPresentTime(presentTime, now, updateType, mModeChangePending, layerProps);
 
 
    // Activate layer if inactive.
    if (found == LayerStatus::LayerInInactiveMap) {
        mActiveLayerInfos.insert(
                {id, std::make_pair(layerPair->first, std::move(layerPair->second))});
        mInactiveLayerInfos.erase(id);
    }
}
```

## Layer createSurfaceFrameForBuffer

调用Layer的createSurfaceFrameForBuffer方法，创建Layer：

```cpp
//frameworks/native/services/surfaceflinger/Layer.cpp
sp<SurfaceFlinger> mFlinger;
const std::unique_ptr<frametimeline::FrameTimeline> mFrameTimeline;
std::shared_ptr<frametimeline::SurfaceFrame> Layer::createSurfaceFrameForBuffer(
        const FrameTimelineInfo& info, nsecs_t queueTime, std::string debugName) {
    auto surfaceFrame =
            mFlinger->mFrameTimeline->createSurfaceFrameForToken(info, mOwnerPid, mOwnerUid,
                                                                 getSequence(), mName, debugName,
                                                                 /*isBuffer*/ true, getGameMode());
    surfaceFrame->setActualStartTime(info.startTimeNanos);
    // For buffers, acquire fence time will set during latch.
    surfaceFrame->setActualQueueTime(queueTime);
    const auto fps = mFlinger->mScheduler->getFrameRateOverride(getOwnerUid());
    if (fps) {
        surfaceFrame->setRenderRate(*fps);
    }
    // TODO(b/178542907): Implement onSurfaceFrameCreated for BQLayer as well.
    onSurfaceFrameCreated(surfaceFrame);
    return surfaceFrame;
}
```

调用mFlinger(SurfaceFlinger)中mFrameTimeline(FrameTimeline)的createSurfaceFrameForToken方法：

```cpp
//frameworks/native/services/surfaceflinger/FrameTimeline/FrameTimeline.cpp
std::shared_ptr<SurfaceFrame> FrameTimeline::createSurfaceFrameForToken(
        const FrameTimelineInfo& frameTimelineInfo, pid_t ownerPid, uid_t ownerUid, int32_t layerId,
        std::string layerName, std::string debugName, bool isBuffer, GameMode gameMode) {
    ATRACE_CALL();
    if (frameTimelineInfo.vsyncId == FrameTimelineInfo::INVALID_VSYNC_ID) {
        return std::make_shared<SurfaceFrame>(frameTimelineInfo, ownerPid, ownerUid, layerId,
                                              std::move(layerName), std::move(debugName),
                                              PredictionState::None, TimelineItem(), mTimeStats,
                                              mJankClassificationThresholds, &mTraceCookieCounter,
                                              isBuffer, gameMode);
    }
    std::optional<TimelineItem> predictions =
            mTokenManager.getPredictionsForToken(frameTimelineInfo.vsyncId);
    if (predictions) {
        return std::make_shared<SurfaceFrame>(frameTimelineInfo, ownerPid, ownerUid, layerId,
                                              std::move(layerName), std::move(debugName),
                                              PredictionState::Valid, std::move(*predictions),
                                              mTimeStats, mJankClassificationThresholds,
                                              &mTraceCookieCounter, isBuffer, gameMode);
    }
    return std::make_shared<SurfaceFrame>(frameTimelineInfo, ownerPid, ownerUid, layerId,
                                          std::move(layerName), std::move(debugName),
                                          PredictionState::Expired, TimelineItem(), mTimeStats,
                                          mJankClassificationThresholds, &mTraceCookieCounter,
                                          isBuffer, gameMode);
}
```

### new SurfaceFrame

创建SurfaceFrame对象，SurfaceFrame的构造方法如下：

```cpp
//frameworks/native/services/surfaceflinger/FrameTimeline/FrameTimeline.cpp
SurfaceFrame::SurfaceFrame(const FrameTimelineInfo& frameTimelineInfo, pid_t ownerPid,
                           uid_t ownerUid, int32_t layerId, std::string layerName,
                           std::string debugName, PredictionState predictionState,
                           frametimeline::TimelineItem&& predictions,
                           std::shared_ptr<TimeStats> timeStats,
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
    std::lock_guard<std::mutex> protoGuard(mTraceMutex);
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
    BufferUpdate* update(increment->mutable_buffer_update());
    update->set_id(layerId);
    update->set_w(width);
    update->set_h(height);
    update->set_frame_number(frameNumber);
}
```

## SurfaceFlinger onLayerUpdate

调用mFlinger(SurfaceFlinger)的onLayerUpdate方法，通知SurfaceFlinger图层更新：

[Android13 SurfaceFlinger onLayerUpdate流程分析-CSDN博客](/Android13%20SurfaceFlinger%20onLayerUpdate%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-CSDN%E5%8D%9A%E5%AE%A2)

## BufferLayerConsumer onBufferAvailable

调用mConsumer(BufferLayerConsumer)的onBufferAvailable方法。

```cpp
//frameworks/native/services/surfaceflinger/BufferLayerConsumer.cpp
void BufferLayerConsumer::onBufferAvailable(const BufferItem& item) {
    if (item.mGraphicBuffer != nullptr && item.mSlot != BufferQueue::INVALID_BUFFER_SLOT) {
        std::lock_guard<std::mutex> lock(mImagesMutex);
        const std::shared_ptr<renderengine::ExternalTexture>& oldImage = mImages[item.mSlot];
        if (oldImage == nullptr || oldImage->getBuffer() == nullptr ||
            oldImage->getBuffer()->getId() != item.mGraphicBuffer->getId()) {
            mImages[item.mSlot] = std::make_shared<
                    renderengine::impl::ExternalTexture>(item.mGraphicBuffer, mRE,
                                                         renderengine::impl::ExternalTexture::
                                                                 Usage::READABLE);
        }
    }
}
```


---

> 作者: [wentao](https://github.com/huang8604)  
> URL: https://huang8604.github.io/posts/android13-bufferqueuelayer-onframeavailable%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-csdn%E5%8D%9A%E5%AE%A2/  

