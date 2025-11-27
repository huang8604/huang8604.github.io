---
title: Android13 SurfaceFlinger onComposerHalRefresh流程分析_android 13 surfaceflinge变化-CSDN博客
author: 
created: 2024-09-27
tags:
  - clippings
  - 转载
  - blog
collections: 图形显示
source: https://blog.csdn.net/liuning1985622/article/details/138466237?spm=1001.2014.3001.5502
date: 2025-11-21T08:01:29.000Z
lastmod: 2025-11-21T08:01:29.000Z
---
onComposerHalRefresh方法是SurfaceFlinger中的一个函数。该方法的作用是在Composer [HAL](https://so.csdn.net/so/search?q=HAL\&spm=1001.2101.3001.7020)刷新时被调用，用于更新显示内容。onComposerHalRefresh方法会在SurfaceFlinger接收到Composer HAL刷新事件时被调用。Composer HAL是硬件抽象层的一部分，负责与硬件显示设备进行通信。当Composer HAL完成一次刷新操作后，会通知SurfaceFlinger进行相应的处理。

在onComposerHalRefresh方法中，SurfaceFlinger会执行一系列操作，包括更新屏幕上的图像内容、处理显示层的合成和混合等。通过这些操作，SurfaceFlinger能够将应用程序的图像内容正确地显示在屏幕上。

SurfaceFlinger的onComposerHalRefresh方法代码如下：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
void SurfaceFlinger::onComposerHalRefresh(hal::HWDisplayId) {
    Mutex::Autolock lock(mStateLock);
    scheduleComposite(FrameHint::kNone);
}
```

## SurfaceFlinger scheduleComposite

调用SurfaceFlinger的scheduleComposite方法：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
void SurfaceFlinger::scheduleComposite(FrameHint hint) {
    mMustComposite = true;
    scheduleCommit(hint);
}
```

调用scheduleCommit方法：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
std::unique_ptr<scheduler::Scheduler> mScheduler;
void SurfaceFlinger::scheduleCommit(FrameHint hint) {
    if (hint == FrameHint::kActive) {
        mScheduler->resetIdleTimer();
    }
    mPowerAdvisor->notifyDisplayUpdateImminent();
    mScheduler->scheduleFrame();
}
```

### MessageQueue scheduleFrame

调用Scheduler的scheduleFrame方法，Scheduler继承MessageQueue，调用MessageQueue的scheduleFrame方法：

```cpp
//frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
void MessageQueue::scheduleFrame() {
    ATRACE_CALL();
 
 
    {
        std::lock_guard lock(mInjector.mutex);
        if (CC_UNLIKELY(mInjector.connection)) {
            ALOGD("%s while injecting VSYNC", __FUNCTION__);
　//请求下一个 VSync 信号
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

调用了VSyncCallbackRegistration的schedule方法：

```cpp
//frameworks/native/services/surfaceflinger/Scheduler/VSyncCallbackRegistration.cpp
std::reference_wrapper<VSyncDispatch> mDispatch;
ScheduleResult VSyncCallbackRegistration::schedule(VSyncDispatch::ScheduleTiming scheduleTiming) {
    if (!mValidToken) {
        return std::nullopt;
    }
    return mDispatch.get().schedule(mToken, scheduleTiming);
}
```

#### VSyncDispatchTimerQueue schedule

调用mDispatch(VSyncDispatch)的schedule方法，实际调用的是VSyncDispatchTimerQueue的schedule方法：

```cpp
//frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatchTimerQueue.cpp
ScheduleResult VSyncDispatchTimerQueueEntry::schedule(VSyncDispatch::ScheduleTiming timing,
                                                      VSyncTracker& tracker, nsecs_t now) {
    auto nextVsyncTime = tracker.nextAnticipatedVSyncTimeFrom(
            std::max(timing.earliestVsync, now + timing.workDuration + timing.readyDuration));
    auto nextWakeupTime = nextVsyncTime - timing.workDuration - timing.readyDuration;
 
 
    bool const wouldSkipAVsyncTarget =
            mArmedInfo && (nextVsyncTime > (mArmedInfo->mActualVsyncTime + mMinVsyncDistance));
    bool const wouldSkipAWakeup =
            mArmedInfo && ((nextWakeupTime > (mArmedInfo->mActualWakeupTime + mMinVsyncDistance)));
    if (wouldSkipAVsyncTarget && wouldSkipAWakeup) {
        return getExpectedCallbackTime(nextVsyncTime, timing);
    }
 
 
    bool const alreadyDispatchedForVsync = mLastDispatchTime &&
            ((*mLastDispatchTime + mMinVsyncDistance) >= nextVsyncTime &&
             (*mLastDispatchTime - mMinVsyncDistance) <= nextVsyncTime);
    if (alreadyDispatchedForVsync) {
        nextVsyncTime =
                tracker.nextAnticipatedVSyncTimeFrom(*mLastDispatchTime + mMinVsyncDistance);
        nextWakeupTime = nextVsyncTime - timing.workDuration - timing.readyDuration;
    }
     // 如果计时器线程即将运行，通过回调计时器重新计算应用此工作更新，以避免取消即将触发的回调。
    auto const nextReadyTime = nextVsyncTime - timing.readyDuration;
    mScheduleTiming = timing;
    mArmedInfo = {nextWakeupTime, nextVsyncTime, nextReadyTime};
    return getExpectedCallbackTime(nextVsyncTime, timing);
}
```
