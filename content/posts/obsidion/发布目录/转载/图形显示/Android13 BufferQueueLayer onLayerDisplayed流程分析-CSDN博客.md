---
title: Android13 BufferQueueLayer onLayerDisplayed流程分析-CSDN博客
author: 
created: 2024-09-27
tags:
  - clippings
  - 转载
  - blog
collections: 图形显示
source: https://blog.csdn.net/liuning1985622/article/details/138467476?spm=1001.2014.3001.5502
date: 2024-09-27T03:23:47.462Z
lastmod: 2024-09-27T03:24:27.685Z
---
BufferQueueLayer的onLayerDisplayed方法在BufferQueueLayer被显示时调用，代码如下：

```cpp
//frameworks/native/services/surfaceflinger/BufferQueueLayer.cpp
sp<SurfaceFlinger> mFlinger;
const std::unique_ptr<FrameTracer> mFrameTracer;
void BufferQueueLayer::onLayerDisplayed(ftl::SharedFuture<FenceResult> futureFenceResult) {
    const sp<Fence> releaseFence = futureFenceResult.get().value_or(Fence::NO_FENCE);
    mConsumer->setReleaseFence(releaseFence);
 
 
    // Prevent tracing the same release multiple times.
    if (mPreviousFrameNumber != mPreviousReleasedFrameNumber) {
        mFlinger->mFrameTracer->traceFence(getSequence(), mPreviousBufferId, mPreviousFrameNumber,
                                           std::make_shared<FenceTime>(releaseFence),
                                           FrameTracer::FrameEvent::RELEASE_FENCE);
        mPreviousReleasedFrameNumber = mPreviousFrameNumber;
    }
}
```

## FrameTracer traceFence

调用FrameTracer的traceFence方法，用于分析和调试界面渲染的性能问题：

```cpp
//frameworks/native/services/surfaceflinger/FrameTracer/FrameTracer.cpp
void FrameTracer::traceFence(int32_t layerId, uint64_t bufferID, uint64_t frameNumber,
                             const std::shared_ptr<FenceTime>& fence,
                             FrameEvent::BufferEventType type, nsecs_t startTime) {
    FrameTracerDataSource::Trace([this, layerId, bufferID, frameNumber, &fence, type,
                                  startTime](FrameTracerDataSource::TraceContext ctx) {
        const nsecs_t signalTime = fence->getSignalTime();
        if (signalTime != Fence::SIGNAL_TIME_INVALID) {
            std::lock_guard<std::mutex> lock(mTraceMutex);
            if (mTraceTracker.find(layerId) == mTraceTracker.end()) {
                return;
            }
 
 
            // Handle any pending fences for this buffer.
            tracePendingFencesLocked(ctx, layerId, bufferID);
 
 
            if (signalTime != Fence::SIGNAL_TIME_PENDING) {
                traceSpanLocked(ctx, layerId, bufferID, frameNumber, type, startTime, signalTime);
            } else {
                mTraceTracker[layerId].pendingFences[bufferID].push_back(
                        {.frameNumber = frameNumber,
                         .type = type,
                         .fence = fence,
                         .startTime = startTime});
            }
        }
    });
}
```
