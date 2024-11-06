---
title: Android 14 - 绘制体系 - VSync（1）-CSDN博客
author: 
created: 2024-10-11
tags:
  - clippings
  - 转载
  - blog
collections: 图形显示
source: https://blog.csdn.net/temp7695/article/details/139250674#comments_34337980
date: 2024-11-06T06:57:54.040Z
lastmod: 2024-11-05T01:34:01.092Z
---
## **整体框架**

![8f55ed8ba868727305f113ae9a9235bf\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/67090386e6706.png)

**VsyncConfiguration**：一些基本参数的配置类，比如PhaseOffsets、WorkDuration等。

**Scheduler**：作为SF生成和派发VSync的整体调度器，主要面向SurfaceFlinger提供VSync相关接口。Scheduler包含对所有屏幕的VSync的控制。本身是MessageQueue的子类。

**Refresh\*\*\*\*RateSelector**：每个Display屏幕对应一个RefreshRateSelector，用于基于屏幕支持的刷新率范围，选择一个合适的Refresh Rate和Frame Rate，并会传递给VSyncSchedule作为软件VSync生成的重要因素。

**VsyncSchedule**：Vsync生成的调度器。每个屏幕对应一个VsyncSchedule。包含一个Tracker（VSyncPredictor）、一个Dispatch（VSyncDispatchTimerQueue）、一个Controller（VSyncReactor）。一方面对接硬件VSync信号的接收、开关，一方面对接软件VSync的计算、输出。

**VSyncPredictor** ：在VsyncSchedule的语境中为一个Tracker，是负责综合各方因素根据算法由硬件VSync计算出软件VSync周期的工具类。

**VSyncDispatchTimerQueue**：在VsyncSchedule的语境中为一个Dispather，负责将VSync分发到使用者。

**VSyncCallbackRegistration**：代表一个VSync使用者的注册。比如常见的针对上层应用的VSYNC-app、针对SurfaceFlinger合成的VSYNC-sf，都对应一个VSyncCallbackRegistration。另外在客户端，还有一个VSYNC-appSf。

**EventThread**：处理客户端应用VSYNC的一个独立线程。期内维护一个不断请求VSYNC的死循环。比如VSYNC-app，一方面，通过VSyncCallbackRegistration去申请下一次VSYNC，另一方面，当VSYNC生成，通过Connection将VSYNC发送给DisplayEventReceiver，也就是VSYNC的终端接收者。一般情况下有两个EventThread，一个是针对客户端应用侧的VSYNC-app，另一个是客户端想使用SF同步信号的VSYNC-appSf。

**DisplayEventReceiver**：通过socket机制提供了一个SF与客户端应用传递VSYNC的通道。在服务端，EventThread通过DisplayEventReceiver发送VSYNC事件，在客户端，Choreographer通过DisplayEventReceiver接收到VSYNC后，下发给ViewRootImp驱动下一次绘制渲染。

Android 13以后，VSYNC架构变化之一是给SF使用的VSYNC-sf信号，不再通过EventThread维护，而是直接在Schedule内维护。供客户端应用使用的VSYN-app仍然由EventThread维护，另外，新增了一个VSYN-appSf信号，也是由EventThread维护。主要作用是如果客户端想使用SF同步的信号，可以切换到VSYNC-appSf这个信号源来。在Android 13之前，这个机制是通过VSYNC-sf实现的，Android 13后将SF专门使用的VSYNC-sf从EventThread剥离出来直接由Schedule维护，VSYN-appSf则专门针对客户端。

### **关键\*\*\*\*参数**

在探索VSync机制前，需要对一些关键参数的概念有所了解，从dumpsys SurfaceFlinger入手：

#### **屏幕刷新率**

![f42fd1df37f94a423ebfcb64649d0fef\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/67090386ae80a.png)

ColorMode：设备支持的ColorMode；可根据系统设置或应用自身设定的颜色模式最终决定使用那个ColorMode。

deviceProductInfo： 屏幕设备信息；manufacturerPnpId=QCM为Plug and Play即插即用设备唯一识别码。

activeMode：当前使用的帧率模式

displayModes：当前屏幕支持的帧率模式。从打印的信息看支持60Hz、90Hz两种帧率

displayManagerPolicy：当前采用的帧率管理策略。primaryRanges代表的是在选取帧率时，通常采纳的范围，如果用户通过setFrameRate手动指定一个帧率，其可能超出primaryRanges的范围；appRequestRanges代表用户可以指定的帧率范围。最终的帧率可能超过primaryRanges，但绝不会超过appRequestRanges。

### **主要组件的初始化**

![c7507518965bd5c02954542624ba4d5d\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/67090386ac7ba.png)

#### **SurfaceFlinger的初始化**

SurfaceFlinger的启动源于SystemServer执行main\_surfaceflinger.cpp的main方法：

```cobol
int main(int, char**) {
    ...
    sp<SurfaceFlinger> flinger = surfaceflinger::createSurfaceFlinger();
    ...
    flinger->init();
    ...
}
```

surfaceflinger::createSurfaceFlinger()的实现在platform/frameworks/native/services/surfaceflinger/SurfaceFlingerFactory.cpp

```cobol
sp<SurfaceFlinger> createSurfaceFlinger() {
    static DefaultFactory factory;
 
    return sp<SurfaceFlinger>::make(factory);
}
```

随后，调用SurfaceFlinger的init方法：

```cpp
platform/frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SurfaceFlinger::init() FTL_FAKE_GUARD(kMainThreadContext) {
    ...
    sp<const DisplayDevice> display;
    if (const auto indexOpt = mCurrentState.getDisplayIndex(getPrimaryDisplayIdLocked())) {
        const auto& displays = mCurrentState.displays;
 
        const auto& token = displays.keyAt(*indexOpt);
        const auto& state = displays.valueAt(*indexOpt);
 
        processDisplayAdded(token, state);
        mDrawingState.displays.add(token, state);
 
        display = getDefaultDisplayDeviceLocked();
    }
    ...
    initScheduler(display);
    ...
}
```

#### **构建RefreshRateSelector**

上面的代码中调用了processDisplayAdded添加主屏幕，后续挂载一个新屏幕也会走此流程：

```cpp
void SurfaceFlinger::processDisplayAdded(const wp<IBinder>& displayToken,
                                         const DisplayDeviceState& state) {
     ...
     auto display = setupNewDisplayDeviceInternal(displayToken, std::move(compositionDisplay), state,
                                                 displaySurface, producer);
     ...
 }
 
 sp<DisplayDevice> SurfaceFlinger::setupNewDisplayDeviceInternal(
        const wp<IBinder>& displayToken,
        std::shared_ptr<compositionengine::Display> compositionDisplay,
        const DisplayDeviceState& state,
        const sp<compositionengine::DisplaySurface>& displaySurface,
        const sp<IGraphicBufferProducer>& producer) {
        ...
         creationArgs.refreshRateSelector =
                mPhysicalDisplays.get(physical->id)
                        .transform(&PhysicalDisplay::snapshotRef)
                        .transform([&](const display::DisplaySnapshot& snapshot) {
                            return std::make_shared<
                                    scheduler::RefreshRateSelector>(snapshot.displayModes(),
                                                                    creationArgs.activeModeId,
                                                                    config);
                        })
                        .value_or(nullptr);
        ...
        sp<DisplayDevice> display = getFactory().createDisplayDevice(creationArgs);
 
 }
                                             
```

上面为每个Display都构建了一个refreshRateSelector，即RefreshRateSelector

#### **初始化\*\*\*\*Scheduler**

VSync相关初始化，都在initScheduler中

```cpp
void SurfaceFlinger::initScheduler(const sp<const DisplayDevice>& display) {
    const auto activeMode = display->refreshRateSelector().getActiveMode();
    const Fps activeRefreshRate = activeMode.fps;
    // 创建配置
    mVsyncConfiguration = getFactory().createVsyncConfiguration(activeRefreshRate);
    // 创建Scheduler
    mScheduler = std::make_unique<Scheduler>(static_cast<ICompositor&>(*this),
                                             static_cast<ISchedulerCallback&>(*this), features,
                                             std::move(modulatorPtr));
    // 注册屏幕，为每个屏幕构建一个VsyncSchedule对象
    mScheduler->registerDisplay(display->getPhysicalId(), display->holdRefreshRateSelector());
    mScheduler->startTimers();
    // 创建VSYNC-app对应的EventThread
    mAppConnectionHandle =
		mScheduler->createEventThread(Scheduler::Cycle::Render,
									  mFrameTimeline->getTokenManager(),
									  /* workDuration */ configs.late.appWorkDuration,
									  /* readyDuration */ configs.late.sfWorkDuration);
    // 创建VSYNC-appSf对应的EventThread
mSfConnectionHandle =
		mScheduler->createEventThread(Scheduler::Cycle::LastComposite,
									  mFrameTimeline->getTokenManager(),
									  /* workDuration */ activeRefreshRate.getPeriod(),
									  /* readyDuration */ configs.late.sfWorkDuration);
    // 创建VSYNC-sf对应的调度器
    mScheduler->initVsync(mScheduler->getVsyncSchedule()->getDispatch(),
                          *mFrameTimeline->getTokenManager(), configs.late.sfWorkDuration);
      ...
}
```

VsyncSchedule的创建在Scheduler::registerDisplay中

```cpp
platform/frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
 
void Scheduler::registerDisplay(PhysicalDisplayId displayId, RefreshRateSelectorPtr selectorPtr) {
    auto schedulePtr = std::make_shared<VsyncSchedule>(displayId, mFeatures,
                                                       [this](PhysicalDisplayId id, bool enable) {
                                                           onHardwareVsyncRequest(id, enable);
                                                       });
 
    registerDisplayInternal(displayId, std::move(selectorPtr), std::move(schedulePtr));
}
```

VsyncSchedule构造函数的初始化：

```cpp
platform/frameworks/native/services/surfaceflinger/Scheduler/VsyncSchedule.cpp
 
VsyncSchedule::VsyncSchedule(PhysicalDisplayId id, FeatureFlags features,
                             RequestHardwareVsync requestHardwareVsync)
      : mId(id),
        mRequestHardwareVsync(std::move(requestHardwareVsync)),
        mTracker(createTracker(id)),
        mDispatch(createDispatch(mTracker)),
        mController(createController(id, *mTracker, features)),
        mTracer(features.test(Feature::kTracePredictedVsync)
                        ? std::make_unique<PredictedVsyncTracer>(mDispatch)
                        : nullptr) {}
```

VsyncSchedule中，构造了几个重要组件：

```cpp
VsyncSchedule::VsyncSchedule(PhysicalDisplayId id, FeatureFlags features,
                             RequestHardwareVsync requestHardwareVsync)
      : mId(id),
        mRequestHardwareVsync(std::move(requestHardwareVsync)),
        mTracker(createTracker(id)),
        mDispatch(createDispatch(mTracker)),
        mController(createController(id, *mTracker, features)),
        mTracer(features.test(Feature::kTracePredictedVsync)
                        ? std::make_unique<PredictedVsyncTracer>(mDispatch)
                        : nullptr) {}
```

其中，mTracker，本质上是VSyncPredictor：

```cpp
VsyncSchedule::TrackerPtr VsyncSchedule::createTracker(PhysicalDisplayId id) {
    // TODO(b/144707443): Tune constants.
    constexpr nsecs_t kInitialPeriod = (60_Hz).getPeriodNsecs();
    constexpr size_t kHistorySize = 20;
    constexpr size_t kMinSamplesForPrediction = 6;
    constexpr uint32_t kDiscardOutlierPercent = 20;
 
    return std::make_unique<VSyncPredictor>(id, kInitialPeriod, kHistorySize,
                                            kMinSamplesForPrediction, kDiscardOutlierPercent);
}
```

mDispatch本质上是VSyncDispatchTimerQueue

```cpp
VsyncSchedule::DispatchPtr VsyncSchedule::createDispatch(TrackerPtr tracker) {
    using namespace std::chrono_literals;
 
    // TODO(b/144707443): Tune constants.
    constexpr std::chrono::nanoseconds kGroupDispatchWithin = 500us;
    constexpr std::chrono::nanoseconds kSnapToSameVsyncWithin = 3ms;
 
    return std::make_unique<VSyncDispatchTimerQueue>(std::make_unique<Timer>(), std::move(tracker),
                                                     kGroupDispatchWithin.count(),
                                                     kSnapToSameVsyncWithin.count());
}
```

mController本质上是VSyncReactor

```cpp
VsyncSchedule::ControllerPtr VsyncSchedule::createController(PhysicalDisplayId id,
                                                             VsyncTracker& tracker,
                                                             FeatureFlags features) {
    // TODO(b/144707443): Tune constants.
    constexpr size_t kMaxPendingFences = 20;
    const bool hasKernelIdleTimer = features.test(Feature::kKernelIdleTimer);
 
    auto reactor = std::make_unique<VSyncReactor>(id, std::make_unique<SystemClock>(), tracker,
                                                  kMaxPendingFences, hasKernelIdleTimer);
 
    reactor->setIgnorePresentFences(!features.test(Feature::kPresentFences));
    return reactor;
}
```

VSYNC-app和VSYNC-appSf这两个信号是在EventThread维护的。看下createEventThread：

```cpp
platform/frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
 
ConnectionHandle Scheduler::createEventThread(Cycle cycle,
                                              frametimeline::TokenManager* tokenManager,
                                              std::chrono::nanoseconds workDuration,
                                              std::chrono::nanoseconds readyDuration) {
  // 根据cycle参数不同，选择是app还是appSf
    auto eventThread = std::make_unique<impl::EventThread>(cycle == Cycle::Render ? "app" : "appSf",
                                                           getVsyncSchedule(), tokenManager,
                                                           makeThrottleVsyncCallback(),
                                                           makeGetVsyncPeriodFunction(),
                                                           workDuration, readyDuration);
 
    auto& handle = cycle == Cycle::Render ? mAppConnectionHandle : mSfConnectionHandle;
    // 创建Connection
    handle = createConnection(std::move(eventThread));
    return handle;
}
```

EventThread的构造函数：

```cpp
EventThread::EventThread(const char* name, std::shared_ptr<scheduler::VsyncSchedule> vsyncSchedule,
                         android::frametimeline::TokenManager* tokenManager,
                         ThrottleVsyncCallback throttleVsyncCallback,
                         GetVsyncPeriodFunction getVsyncPeriodFunction,
                         std::chrono::nanoseconds workDuration,
                         std::chrono::nanoseconds readyDuration)
      : mThreadName(name),
      // ATRACE名称
        mVsyncTracer(base::StringPrintf("VSYNC-%s", name), 0),
        // VSYNC计算时使用的偏移量
        mWorkDuration(base::StringPrintf("VsyncWorkDuration-%s", name), workDuration),
        // VSYNC计算时使用的偏移量
        mReadyDuration(readyDuration),
        mVsyncSchedule(std::move(vsyncSchedule)),
        // 一个VSYNC请求者的具体处理类
        mVsyncRegistration(mVsyncSchedule->getDispatch(), createDispatchCallback()/*VSync生成后的回调函数*/, name),
        mTokenManager(tokenManager),
        mThrottleVsyncCallback(std::move(throttleVsyncCallback)),
        mGetVsyncPeriodFunction(std::move(getVsyncPeriodFunction)) {
    LOG_ALWAYS_FATAL_IF(getVsyncPeriodFunction == nullptr,
            "getVsyncPeriodFunction must not be null");
 
    mThread = std::thread([this]() NO_THREAD_SAFETY_ANALYSIS {
        std::unique_lock<std::mutex> lock(mMutex);
        threadMain(lock);
    });
```

在上面的方法中，通过createConnection创建了VSync事件派发的通道：

```cpp
platform/frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
 
ConnectionHandle Scheduler::createConnection(std::unique_ptr<EventThread> eventThread) {
    const ConnectionHandle handle = ConnectionHandle{mNextConnectionHandleId++};
    ALOGV("Creating a connection handle with ID %" PRIuPTR, handle.id);
 
    auto connection = createConnectionInternal(eventThread.get());
 
    std::lock_guard<std::mutex> lock(mConnectionsLock);
    mConnections.emplace(handle, Connection{connection, std::move(eventThread)});
    return handle;
}
 
sp<EventThreadConnection> Scheduler::createConnectionInternal(
        EventThread* eventThread, EventRegistrationFlags eventRegistration,
        const sp<IBinder>& layerHandle) {
    int32_t layerId = static_cast<int32_t>(LayerHandle::getLayerId(layerHandle));
    auto connection = eventThread->createEventConnection([&] { resync(); }, eventRegistration);
    mLayerHistory.attachChoreographer(layerId, connection);
    return connection;
}
```

最终是由EventThread创建Connection：

```cpp
platform/frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
 
sp<EventThreadConnection> EventThread::createEventConnection(
        ResyncCallback resyncCallback, EventRegistrationFlags eventRegistration) const {
    return sp<EventThreadConnection>::make(const_cast<EventThread*>(this),
                                           IPCThreadState::self()->getCallingUid(),
                                           std::move(resyncCallback), eventRegistration);
}
```

## **VSYNC-app的请求流程**

VSYNC-app由EventThread维护。VSYNC-app的触发有多种因素，包括新屏幕挂载、客户端主动请求、EVentThread线程内部自发。

以EventThread内部自发为例：

```cpp
void EventThread::threadMain(std::unique_lock<std::mutex>& lock) {
    DisplayEventConsumers consumers;
 
    while (mState != State::Quit) {
        std::optional<DisplayEventReceiver::Event> event;
 
        // Determine next event to dispatch.
        if (!mPendingEvents.empty()) {
            event = mPendingEvents.front();
            mPendingEvents.pop_front();
 
            if (event->header.type == DisplayEventReceiver::DISPLAY_EVENT_HOTPLUG) {
                if (event->hotplug.connected && !mVSyncState) {
                    mVSyncState.emplace(event->header.displayId);
                } else if (!event->hotplug.connected && mVSyncState &&
                           mVSyncState->displayId == event->header.displayId) {
                    mVSyncState.reset();
                }
            }
        }
 
        bool vsyncRequested = false;
 
        // Find connections that should consume this event.
        auto it = mDisplayEventConnections.begin();
        while (it != mDisplayEventConnections.end()) {
            if (const auto connection = it->promote()) {
                if (event && shouldConsumeEvent(*event, connection)) {
                    consumers.push_back(connection);
                }
 
                vsyncRequested |= connection->vsyncRequest != VSyncRequest::None;
 
                ++it;
            } else {
                it = mDisplayEventConnections.erase(it);
            }
        }
 
        if (!consumers.empty()) {
            dispatchEvent(*event, consumers);
            consumers.clear();
        }
 
        if (mVSyncState && vsyncRequested) {
            mState = mVSyncState->synthetic ? State::SyntheticVSync : State::VSync;
        } else {
            ALOGW_IF(!mVSyncState, "Ignoring VSYNC request while display is disconnected");
            mState = State::Idle;
        }
 
        if (mState == State::VSync) {
            const auto scheduleResult =
                    mVsyncRegistration.schedule({.workDuration = mWorkDuration.get().count(),
                                                 .readyDuration = mReadyDuration.count(),
                                                 .earliestVsync = mLastVsyncCallbackTime.ns()});
            LOG_ALWAYS_FATAL_IF(!scheduleResult, "Error scheduling callback");
        } else {
            mVsyncRegistration.cancel();
        }
 
        if (!mPendingEvents.empty()) {
            continue;
        }
 
        // Wait for event or client registration/request.
        if (mState == State::Idle) {
            mCondition.wait(lock);
        } else {
            // Generate a fake VSYNC after a long timeout in case the driver stalls. When the
            // display is off, keep feeding clients at 60 Hz.
            const std::chrono::nanoseconds timeout =
                    mState == State::SyntheticVSync ? 16ms : 1000ms;
            if (mCondition.wait_for(lock, timeout) == std::cv_status::timeout) {
                if (mState == State::VSync) {
                    ALOGW("Faking VSYNC due to driver stall for thread %s", mThreadName);
                }
 
                LOG_FATAL_IF(!mVSyncState);
                const auto now = systemTime(SYSTEM_TIME_MONOTONIC);
                const auto deadlineTimestamp = now + timeout.count();
                const auto expectedVSyncTime = deadlineTimestamp + timeout.count();
                mPendingEvents.push_back(makeVSync(mVSyncState->displayId, now,
                                                   ++mVSyncState->count, expectedVSyncTime,
                                                   deadlineTimestamp));
            }
        }
    }
    // cancel any pending vsync event before exiting
    mVsyncRegistration.cancel();
}
```

触发VSYNC请求的是：

const auto scheduleResult =

mVsyncRegistration.schedule({.workDuration = mWorkDuration.get().count(),

.readyDuration = mReadyDuration.count(),

.earliestVsync = mLastVsyncCallbackTime.ns()});

LOG\_ALWAYS\_FATAL\_IF(!scheduleResult, "Error scheduling callback");

```cpp
platform/frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatchTimerQueue.cpp
 
ScheduleResult VSyncCallbackRegistration::schedule(VSyncDispatch::ScheduleTiming scheduleTiming) {
    if (!mToken) {
        return std::nullopt;
    }
    return mDispatch->schedule(*mToken, scheduleTiming);
}
```

mDispatch是VSyncDispatchTimerQueue，在同一个文件里：

```cpp
platform/frameworks/native/services/surfaceflinger/Scheduler/VSyncDispatchTimerQueue.cpp
 
ScheduleResult VSyncDispatchTimerQueue::schedule(CallbackToken token,
                                                 ScheduleTiming scheduleTiming) {
    std::lock_guard lock(mMutex);
    return scheduleLocked(token, scheduleTiming);
}
 
```

```rust
ScheduleResult VSyncDispatchTimerQueue::scheduleLocked(CallbackToken token,
                                                       ScheduleTiming scheduleTiming) {
    auto it = mCallbacks.find(token);
    if (it == mCallbacks.end()) {
        return {};
    }
    auto& callback = it->second;
    auto const now = mTimeKeeper->now();
 
    /* If the timer thread will run soon, we'll apply this work update via the callback
     * timer recalculation to avoid cancelling a callback that is about to fire. */
    auto const rearmImminent = now > mIntendedWakeupTime;
    if (CC_UNLIKELY(rearmImminent)) {
        callback->addPendingWorkloadUpdate(scheduleTiming);
        return getExpectedCallbackTime(*mTracker, now, scheduleTiming);
    }
 
    const ScheduleResult result = callback->schedule(scheduleTiming, *mTracker, now);
    if (!result.has_value()) {
        return {};
    }
 
    if (callback->wakeupTime() < mIntendedWakeupTime - mTimerSlack) {
        rearmTimerSkippingUpdateFor(now, it);
    }
 
    return result;
}
```

这里的callback是VSyncDispatchTimerQueueEntry，具体实现也在VSyncDispatchTimerQueue.cpp文件里：

```cobol
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
 
    nextVsyncTime = adjustVsyncIfNeeded(tracker, nextVsyncTime);
    nextWakeupTime = nextVsyncTime - timing.workDuration - timing.readyDuration;
 
    auto const nextReadyTime = nextVsyncTime - timing.readyDuration;
    mScheduleTiming = timing;
    mArmedInfo = {nextWakeupTime, nextVsyncTime, nextReadyTime};
    return getExpectedCallbackTime(nextVsyncTime, timing);
}
```

这里就是计算下一次VSync的地方了。tracker就是VSyncPredictor，是计算VSync的核心类。

在上面的VSyncDispatchTimerQueue::scheduleLocked中，schedule计算完时间后，会调用

rearmTimerSkippingUpdateFor，而rearmTimerSkippingUpdateFor设置了一个Alarm，定时执行VSync的回调：

```cobol
void VSyncDispatchTimerQueue::rearmTimerSkippingUpdateFor(
        nsecs_t now, CallbackMap::iterator const& skipUpdateIt) {
        ...
        setTimer(*min, now);
        ...
}
 
void VSyncDispatchTimerQueue::setTimer(nsecs_t targetTime, nsecs_t /*now*/) {
    mIntendedWakeupTime = targetTime;
    mTimeKeeper->alarmAt(std::bind(&VSyncDispatchTimerQueue::timerCallback, this),
                         mIntendedWakeupTime);
    mLastTimerSchedule = mTimeKeeper->now();
}
```

这里的timerCallback，是EventThread里的这个方法：

```cpp
scheduler::VSyncDispatch::Callback EventThread::createDispatchCallback() {
    return [this](nsecs_t vsyncTime, nsecs_t wakeupTime, nsecs_t readyTime) {
        onVsync(vsyncTime, wakeupTime, readyTime);
    };
}
```

```cobol
void EventThread::onVsync(nsecs_t vsyncTime, nsecs_t wakeupTime, nsecs_t readyTime) {
    std::lock_guard<std::mutex> lock(mMutex);
    mLastVsyncCallbackTime = TimePoint::fromNs(vsyncTime);
 
    LOG_FATAL_IF(!mVSyncState);
    mVsyncTracer = (mVsyncTracer + 1) % 2;
    mPendingEvents.push_back(makeVSync(mVSyncState->displayId, wakeupTime, ++mVSyncState->count,
                                       vsyncTime, readyTime));
    mCondition.notify_all();
}
```

Vsync信号将会封装成Event放入mPendingEvents

## **VSYNC-app的分发过程**

在之前的threadMain方法中，有代码：

```cpp
if (!mPendingEvents.empty()) {
            event = mPendingEvents.front();
            mPendingEvents.pop_front();
 
            if (event->header.type == DisplayEventReceiver::DISPLAY_EVENT_HOTPLUG) {
                if (event->hotplug.connected && !mVSyncState) {
                    mVSyncState.emplace(event->header.displayId);
                } else if (!event->hotplug.connected && mVSyncState &&
                           mVSyncState->displayId == event->header.displayId) {
                    mVSyncState.reset();
                }
            }
        }
...
if (!consumers.empty()) {
    dispatchEvent(*event, consumers);
    consumers.clear();
}
```

```cpp
void EventThread::dispatchEvent(const DisplayEventReceiver::Event& event,
                                const DisplayEventConsumers& consumers) {
    for (const auto& consumer : consumers) {
        DisplayEventReceiver::Event copy = event;
        if (event.header.type == DisplayEventReceiver::DISPLAY_EVENT_VSYNC) {
            const int64_t frameInterval = mGetVsyncPeriodFunction(consumer->mOwnerUid);
            copy.vsync.vsyncData.frameInterval = frameInterval;
            generateFrameTimeline(copy.vsync.vsyncData, frameInterval, copy.header.timestamp,
                                  event.vsync.vsyncData.preferredExpectedPresentationTime(),
                                  event.vsync.vsyncData.preferredDeadlineTimestamp());
        }
        switch (consumer->postEvent(copy)) {
            case NO_ERROR:
                break;
 
            case -EAGAIN:
                // TODO: Try again if pipe is full.
                ALOGW("Failed dispatching %s for %s", toString(event).c_str(),
                      toString(*consumer).c_str());
                break;
 
            default:
                // Treat EPIPE and other errors as fatal.
                removeDisplayEventConnectionLocked(consumer);
        }
    }
}
```

这里的consumer为EventThreadConnection

```cpp
platform/frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
 
status_t EventThreadConnection::postEvent(const DisplayEventReceiver::Event& event) {
    constexpr auto toStatus = [](ssize_t size) {
        return size < 0 ? status_t(size) : status_t(NO_ERROR);
    };
 
    if (event.header.type == DisplayEventReceiver::DISPLAY_EVENT_FRAME_RATE_OVERRIDE ||
        event.header.type == DisplayEventReceiver::DISPLAY_EVENT_FRAME_RATE_OVERRIDE_FLUSH) {
        mPendingEvents.emplace_back(event);
        if (event.header.type == DisplayEventReceiver::DISPLAY_EVENT_FRAME_RATE_OVERRIDE) {
            return status_t(NO_ERROR);
        }
 
        auto size = DisplayEventReceiver::sendEvents(&mChannel, mPendingEvents.data(),
                                                     mPendingEvents.size());
        mPendingEvents.clear();
        return toStatus(size);
    }
 
    auto size = DisplayEventReceiver::sendEvents(&mChannel, &event, 1);
    return toStatus(size);
}
```

DisplayEventReceiver::sendEvents最终发给客户端。

```cpp
ssize_t DisplayEventReceiver::sendEvents(gui::BitTube* dataChannel,
        Event const* events, size_t count)
{
    return gui::BitTube::sendObjects(dataChannel, events, count);
}
```

BitTube通过socket将事件分发到客户端的Choreographer。

## **VSYNC-sf的请求和分发**

VSYNC-sf的触发点在SurfaceFlinger::initScheduler调用MessageQueue::initVsync

```cpp
platform/frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
 
void MessageQueue::initVsync(std::shared_ptr<scheduler::VSyncDispatch> dispatch,
                             frametimeline::TokenManager& tokenManager,
                             std::chrono::nanoseconds workDuration) {
    std::unique_ptr<scheduler::VSyncCallbackRegistration> oldRegistration;
    {
        std::lock_guard lock(mVsync.mutex);
        mVsync.workDuration = workDuration;
        mVsync.tokenManager = &tokenManager;
        oldRegistration = onNewVsyncScheduleLocked(std::move(dispatch));
    }
 
    // See comments in onNewVsyncSchedule. Today, oldRegistration should be
    // empty, but nothing prevents us from calling initVsync multiple times, so
    // go ahead and destruct it outside the lock for safety.
    oldRegistration.reset();
}
```

```cpp
platform/frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
 
std::unique_ptr<scheduler::VSyncCallbackRegistration> MessageQueue::onNewVsyncScheduleLocked(
        std::shared_ptr<scheduler::VSyncDispatch> dispatch) {
    const bool reschedule = mVsync.registration &&
            mVsync.registration->cancel() == scheduler::CancelResult::Cancelled;
    auto oldRegistration = std::move(mVsync.registration);
    mVsync.registration = std::make_unique<
            scheduler::VSyncCallbackRegistration>(std::move(dispatch),
                                                  std::bind(&MessageQueue::vsyncCallback, this,
                                                            std::placeholders::_1,
                                                            std::placeholders::_2,
                                                            std::placeholders::_3),
                                                  "sf");
    if (reschedule) {
        mVsync.scheduledFrameTime =
                mVsync.registration->schedule({.workDuration = mVsync.workDuration.get().count(),
                                               .readyDuration = 0,
                                               .earliestVsync = mVsync.lastCallbackTime.ns()});
    }
    return oldRegistration;
}
```

可以看到，这里对VSYNC信号命名为“sf”。mVsync.registration->schedule此处调用VSync的生成过程，与前面的VSync-app相同。但这里的回调函数是vsyncCallback

```cpp
platform/frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
 
void MessageQueue::vsyncCallback(nsecs_t vsyncTime, nsecs_t targetWakeupTime, nsecs_t readyTime) {
    ATRACE_CALL();
    // Trace VSYNC-sf
    mVsync.value = (mVsync.value + 1) % 2;
 
    const auto expectedVsyncTime = TimePoint::fromNs(vsyncTime);
    {
        std::lock_guard lock(mVsync.mutex);
        mVsync.lastCallbackTime = expectedVsyncTime;
        mVsync.scheduledFrameTime.reset();
    }
 
    const auto vsyncId = VsyncId{mVsync.tokenManager->generateTokenForPredictions(
            {targetWakeupTime, readyTime, vsyncTime})};
 
    mHandler->dispatchFrame(vsyncId, expectedVsyncTime);
}
```

MessageQueue本身是一个消息队列的实现类。这里看到收到VSync回调后，通过mHandler->dispatchFrame抛到队列里去了。最终会执行

```cpp
platform/frameworks/native/services/surfaceflinger/Scheduler/MessageQueue.cpp
 
void MessageQueue::Handler::handleMessage(const Message&) {
    mFramePending.store(false);
    mQueue.onFrameSignal(mQueue.mCompositor, mVsyncId, mExpectedVsyncTime);
}
```

最后回调

```cpp
platform/frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
 
void Scheduler::onFrameSignal(ICompositor& compositor, VsyncId vsyncId,
                              TimePoint expectedVsyncTime) {
          ...
          if (!compositor.commit(pacesetterId, targets)) return;
          ...
          const auto resultsPerDisplay = compositor.composite(pacesetterId, targeters);
  }
```

这里的compositor是SurfaceFlinger。接下来就是由SF进行合成工作了。
