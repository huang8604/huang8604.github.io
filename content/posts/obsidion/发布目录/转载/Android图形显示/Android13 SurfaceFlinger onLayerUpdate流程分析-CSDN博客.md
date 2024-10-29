---
title: Android13 SurfaceFlinger onLayerUpdate流程分析-CSDN博客
author: 
created: 2024-09-26
tags:
  - clippings
  - 转载
  - blog
collections: 图形显示
date: 2024-09-26T09:00:49.006Z
lastmod: 2024-09-27T03:27:07.000Z
---
SurfaceFlinger的onLayerUpdate方法用于在图形层更新时进行相应的处理，代码如下：

```cpp
void SurfaceFlinger::onLayerUpdate() {
    scheduleCommit(FrameHint::kActive);
}
```

调用SurfaceFlinger的scheduleCommit方法：

```cpp
//frameworks/native/service/surfaceflinger/SurfaeFlinger.cpp
std::unique_ptr<scheduler::Scheduler> mScheduler;
void SurfaceFlinger::scheduleCommit(FrameHint hint) {
    if (hint == FrameHint::kActive) {
        mScheduler->resetIdleTimer();
    }
    mPowerAdvisor->notifyDisplayUpdateImminent();
    mScheduler->scheduleFrame();
}
```

调用mScheduler(Scheduler)的scheduleFrame方法，Scheduler继承于MessageQueue，调用MessageQueue的scheduleFrame方法：

```cpp
//frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
Vsync mVsync;
std::unique_ptr<scheduler::VSyncCallbackRegistration> registration;
void MessageQueue::scheduleFrame() {
    ATRACE_CALL();
 
 
    {
        std::lock_guard lock(mInjector.mutex);
        if (CC_UNLIKELY(mInjector.connection)) {
            ALOGD("%s while injecting VSYNC", __FUNCTION__);
            mInjector.connection->requestNextVsync();
            return;
        }
    }
 
 
    std::lock_guard lock(mVsync.mutex);
    mVsync.scheduledFrameTime =
            mVsync.registration->schedule({.workDuration = mVsync.workDuration.get().count(),
                                           .readyDuration = 0,
                                           .earliestVsync = mVsync.lastCallbackTime.count()});
}
```

调用mVsync(Vsync)成员变量registration的schedule方法，而mVsync.registration在MessageQueue的initVsync中注册：

```cpp
//frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
void MessageQueue::initVsync(scheduler::VSyncDispatch& dispatch,
                             frametimeline::TokenManager& tokenManager,
                             std::chrono::nanoseconds workDuration) {
    setDuration(workDuration);
    mVsync.tokenManager = &tokenManager;
    mVsync.registration = std::make_unique<
            scheduler::VSyncCallbackRegistration>(dispatch,
                                                  std::bind(&MessageQueue::vsyncCallback, this,
                                                            std::placeholders::_1,
                                                            std::placeholders::_2,
                                                            std::placeholders::_3),
                                                  "sf");
}
```

因此会调用到MessageQueue的vsyncCallback方法：

```cpp
//frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
void MessageQueue::vsyncCallback(nsecs_t vsyncTime, nsecs_t targetWakeupTime, nsecs_t readyTime) {
    ATRACE_CALL();
    // Trace VSYNC-sf
    mVsync.value = (mVsync.value + 1) % 2;
 
 
    {
        std::lock_guard lock(mVsync.mutex);
        mVsync.lastCallbackTime = std::chrono::nanoseconds(vsyncTime);
        mVsync.scheduledFrameTime.reset();
    }
 
 
    const auto vsyncId = mVsync.tokenManager->generateTokenForPredictions(
            {targetWakeupTime, readyTime, vsyncTime});
 
 
    mHandler->dispatchFrame(vsyncId, vsyncTime);
}
```

调用Handler的dispatchFrame方法：

```cpp
//frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
void MessageQueue::Handler::dispatchFrame(int64_t vsyncId, nsecs_t expectedVsyncTime) {
    if (!mFramePending.exchange(true)) {
        mVsyncId = vsyncId;
        mExpectedVsyncTime = expectedVsyncTime;
        mQueue.mLooper->sendMessage(this, Message());
    }
}
```

发送Message，消息在Handler的handleMessage方法中处理：

```cpp
//frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
ICompositor& mCompositor;
void MessageQueue::Handler::handleMessage(const Message&) {
    mFramePending.store(false);
 
 
    const nsecs_t frameTime = systemTime();
    auto& compositor = mQueue.mCompositor;
 
 
    if (!compositor.commit(frameTime, mVsyncId, mExpectedVsyncTime)) {
        return;
    }
 
 
    compositor.composite(frameTime, mVsyncId);
    compositor.sample();
}
```

上面方法主要处理如下：

1、调用mCompositor(ICompositor)的commit方法。

2、调用mCompositor(ICompositor)的composite方法。

下面分别进行分析：

## SurfaceFlinger commit

调用mCompositor(ICompositor)的commit方法，ICompositor是一个接口，由SurfaceFlinger实现，SurfaceFlinger的commit方法用于将应用程序的绘制结果提交到屏幕上显示：

[Android13 SurfaceFlinger commit(提交)流程分析-CSDN博客](/Android13%20SurfaceFlinger%20commit\(%E6%8F%90%E4%BA%A4\)%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-CSDN%E5%8D%9A%E5%AE%A2)

## SurfaceFlinger composite

调用mCompositor(ICompositor)的commit方法，ICompositor是一个接口，由SurfaceFlinger实现，SurfaceFlinger的composite用于将多个窗口的图像进行合成：

[Android13 SurfaceFlinger composite(合成)流程分析-CSDN博客](/Android13%20SurfaceFlinger%20composite\(%E5%90%88%E6%88%90\)%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-CSDN%E5%8D%9A%E5%AE%A2)
