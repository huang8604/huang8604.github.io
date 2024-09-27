---
title: Android13 SurfaceFlinger onComposerHalVsync流程分析-CSDN博客
author: 
created: 2024-09-27
tags:
  - clippings
  - 转载
  - blog
collections: 图形显示
source: https://blog.csdn.net/liuning1985622/article/details/138466084
date: 2024-09-27T07:12:57.466Z
lastmod: 2024-09-27T07:30:20.557Z
---
onComposerHalVsync是一个Android系统中的一个回调函数，用于在垂直同步（Vsync）事件发生时通知应用程序。Vsync是指显示器刷新的时间点，它通常以固定的频率发生，比如60Hz。应用程序可以通过注册onComposerHalVsync回调函数来获取Vsync事件的通知。

当Vsync事件发生时，系统会调用注册了onComposerHalVsync回调函数的应用程序，并传递一个时间戳参数，表示Vsync事件发生的时间点。应用程序可以利用这个时间戳来进行一些与显示相关的操作，比如更新UI界面或者进行动画效果的计算。

onComposerHalVsync函数通常是在底层硬件抽象层（HAL）中实现的，它与具体的硬件设备相关。应用程序可以通过调用系统提供的API来注册和取消onComposerHalVsync回调函数。

SurfaceFlinger的onComposerHalVsync方法代码如下：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
void SurfaceFlinger::onComposerHalVsync(hal::HWDisplayId hwcDisplayId, int64_t timestamp,
                                        std::optional<hal::VsyncPeriodNanos> vsyncPeriod) {
    const std::string tracePeriod = [vsyncPeriod]() {
        if (ATRACE_ENABLED() && vsyncPeriod) {
            std::stringstream ss;
            ss << "(" << *vsyncPeriod << ")";
            return ss.str();
        }
        return std::string();
    }();
    ATRACE_FORMAT("onComposerHalVsync%s", tracePeriod.c_str());
 
 
    Mutex::Autolock lock(mStateLock);
    const auto displayId = getHwComposer().toPhysicalDisplayId(hwcDisplayId);
    if (displayId) {
        const auto token = getPhysicalDisplayTokenLocked(*displayId);
        const auto display = getDisplayDeviceLocked(token);
        display->onVsync(timestamp);
    }
 
 
    if (!getHwComposer().onVsync(hwcDisplayId, timestamp)) {
        return;
    }
 
 
    const bool isActiveDisplay =
            displayId && getPhysicalDisplayTokenLocked(*displayId) == mActiveDisplayToken;
    if (!isActiveDisplay) {
        // For now, we don't do anything with non active display vsyncs.
        return;
    }
 
 
    bool periodFlushed = false;
    mScheduler->addResyncSample(timestamp, vsyncPeriod, &periodFlushed);
    if (periodFlushed) {
        modulateVsync(&VsyncModulator::onRefreshRateChangeCompleted);
    }
}
```

调用modulateVsync方法：

```cpp
//frameworks/native/services/surfaceflinger/SurfaceFlinger.h
class SurfaceFlinger : public BnSurfaceComposer,
                       public PriorityDumper,
                       private IBinder::DeathRecipient,
                       private HWC2::ComposerCallback,
                       private ICompositor,
                       private scheduler::ISchedulerCallback {
    class BufferCountTracker {
    void modulateVsync(Handler handler, Args... args) {
        if (const auto config = (*mVsyncModulator.*handler)(args...)) {
            const auto vsyncPeriod = mScheduler->getVsyncPeriodFromRefreshRateConfigs();
            setVsyncConfig(*config, vsyncPeriod);
        }
    }
}
```

调用setVsyncConfig方法：

```cpp
//frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
std::unique_ptr<scheduler::Scheduler> mScheduler;
void SurfaceFlinger::setVsyncConfig(const VsyncModulator::VsyncConfig& config,
                                    nsecs_t vsyncPeriod) {
    mScheduler->setDuration(mAppConnectionHandle,
                            /*workDuration=*/config.appWorkDuration,
                            /*readyDuration=*/config.sfWorkDuration);
    mScheduler->setDuration(mSfConnectionHandle,
                            /*workDuration=*/std::chrono::nanoseconds(vsyncPeriod),
                            /*readyDuration=*/config.sfWorkDuration);
    mScheduler->setDuration(config.sfWorkDuration);
}
```

## Scheduler setDuration

调用Scheduler的setDuration方法：

```cpp
//frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
void Scheduler::setDuration(ConnectionHandle handle, std::chrono::nanoseconds workDuration,
                            std::chrono::nanoseconds readyDuration) {
    android::EventThread* thread;
    {
        std::lock_guard<std::mutex> lock(mConnectionsLock);
        RETURN_IF_INVALID_HANDLE(handle);
        thread = mConnections[handle].thread.get();
    }
    thread->setDuration(workDuration, readyDuration);
}
```

### EventThread setDuration

调用EventThread的setDuration方法：

```cpp
//frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
const std::unique_ptr<VSyncSource> mVSyncSource GUARDED_BY(mMutex);
void EventThread::setDuration(std::chrono::nanoseconds workDuration,
                              std::chrono::nanoseconds readyDuration) {
    std::lock_guard<std::mutex> lock(mMutex);
    mVSyncSource->setDuration(workDuration, readyDuration);
}
```

调用mVSyncSource(VSyncSource)的setDuration方法，DispSyncSource继承VSyncSource，调用DispSyncSource的setDuration方法：

```cpp
//frameworks/native/services/surfaceflinger/Scheduler/DispSyncSource.cpp
class DispSyncSource final : public VSyncSource {
void DispSyncSource::setDuration(std::chrono::nanoseconds workDuration,
                                 std::chrono::nanoseconds readyDuration) {
    std::lock_guard lock(mVsyncMutex);
    mWorkDuration = workDuration;
    mReadyDuration = readyDuration;
 
 
    // If we're not enabled, we don't need to mess with the listeners
    if (!mEnabled) {
        return;
    }
 
 
    mCallbackRepeater->start(mWorkDuration, mReadyDuration);
}
}
```

mCallbackRepeater的mRegistration绑定的是DispSyncSource::onVsyncCallback，因此start方法最后会执行这个onVsyncCallback：

```cpp
//frameworks/native/services/surfaceflinger/Scheduler/DispSyncSource.cpp
void DispSyncSource::onVsyncCallback(nsecs_t vsyncTime, nsecs_t targetWakeupTime,
                                     nsecs_t readyTime) {
    VSyncSource::Callback* callback;
    {
        std::lock_guard lock(mCallbackMutex);
        callback = mCallback;
    }
    if (callback != nullptr) {
        callback->onVSyncEvent(targetWakeupTime, vsyncTime, readyTime);
    }
}
```

#### EventThread onVSyncEvent

进而执行到DispSyncSource.mCallback.onVSyncEvent方法，而DispSyncSource.mCallback的callback实质是EventThread,所以回调将进入到EventThread.onVSyncEvent方法：

```cpp
//frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
void EventThread::onVSyncEvent(nsecs_t timestamp, VSyncSource::VSyncData vsyncData) {
    std::lock_guard<std::mutex> lock(mMutex);
 
 
    LOG_FATAL_IF(!mVSyncState);
    mPendingEvents.push_back(makeVSync(mVSyncState->displayId, timestamp, ++mVSyncState->count,
                                       vsyncData.expectedPresentationTime,
                                       vsyncData.deadlineTimestamp));
    mCondition.notify_all();
}
```

##### EventThread threadMain

这里我们可以看到，调用了makeVSync生成一个Event并加入到mPendingEvents后面，然后调用mCondition.notify\_all()，通知等待的线程，继续执行Event的分发方法threadMain：

[Android13 EventThread threadMain流程分析-CSDN博客](/Android13%20EventThread%20threadMain%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-CSDN%E5%8D%9A%E5%AE%A2)
