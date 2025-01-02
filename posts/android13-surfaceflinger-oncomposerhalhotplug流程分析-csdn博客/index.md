# Android13 SurfaceFlinger OnComposerHalHotplug流程分析-CSDN博客

onComposerHalHotplug是一个Android系统中的一个事件回调函数，用于处理Composer HAL（Hardware Abstraction Layer）的热插拔事件。Composer HAL是Android系统中负责处理图形渲染和显示的硬件抽象层，它与硬件驱动程序和图形服务之间进行通信。

当发生Composer HAL的热插拔事件时，系统会调用onComposerHalHotplug函数来通知相关的应用程序。这个函数可以在应用程序中实现，以便在热插拔事件发生时执行相应的操作。

SurfaceFlinger的onComposerHalHotplug方法代码如下：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
void SurfaceFlinger::onComposerHalHotplug(hal::HWDisplayId hwcDisplayId,
                                          hal::Connection connection) {
    const bool connected = connection == hal::Connection::CONNECTED;
    ALOGI(&#34;%s HAL display %&#34; PRIu64, connected ? &#34;Connecting&#34; : &#34;Disconnecting&#34;, hwcDisplayId);
 
 
    // Only lock if we&#39;re not on the main thread. This function is normally
    // called on a hwbinder thread, but for the primary display it&#39;s called on
    // the main thread with the state lock already held, so don&#39;t attempt to
    // acquire it here.
    ConditionalLock lock(mStateLock, std::this_thread::get_id() != mMainThreadId);
 
 
    mPendingHotplugEvents.emplace_back(HotplugEvent{hwcDisplayId, connection});
 
 
    if (std::this_thread::get_id() == mMainThreadId) {
        // Process all pending hot plug events immediately if we are on the main thread.
        // 如果我们在主线程上，请立即处理所有待处理的热插拔事件。
        processDisplayHotplugEventsLocked();
    }
 
 
    setTransactionFlags(eDisplayTransactionNeeded);
}
```

## SurfaceFlinger processDisplayHotplugEventsLocked

调用SurfaceFlinger的processDisplayHotplugEventsLocked方法，处理所有待处理的热插拔事件：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
void SurfaceFlinger::processDisplayHotplugEventsLocked() {
    for (const auto&amp; event : mPendingHotplugEvents) {
        std::optional&lt;DisplayIdentificationInfo&gt; info =
                getHwComposer().onHotplug(event.hwcDisplayId, event.connection);
 
 
        if (!info) {
            continue;
        }
 
 
        const auto displayId = info-&gt;id;
        const auto token = mPhysicalDisplayTokens.get(displayId);
 
 
        if (event.connection == hal::Connection::CONNECTED) {
            auto [supportedModes, activeMode] = loadDisplayModes(displayId);
 
 
            if (!token) {
                ALOGV(&#34;Creating display %s&#34;, to_string(displayId).c_str());
 
 
                DisplayDeviceState state;
                state.physical = {.id = displayId,
                                  .type = getHwComposer().getDisplayConnectionType(displayId),
                                  .hwcDisplayId = event.hwcDisplayId,
                                  .deviceProductInfo = std::move(info-&gt;deviceProductInfo),
                                  .supportedModes = std::move(supportedModes),
                                  .activeMode = std::move(activeMode)};
                state.isSecure = true; // All physical displays are currently considered secure.
                state.displayName = std::move(info-&gt;name);
 
 
                sp&lt;IBinder&gt; token = new BBinder();
                mCurrentState.displays.add(token, state);
                mPhysicalDisplayTokens.try_emplace(displayId, std::move(token));
                mInterceptor-&gt;saveDisplayCreation(state);
            } else {
                ALOGV(&#34;Recreating display %s&#34;, to_string(displayId).c_str());
 
 
                auto&amp; state = mCurrentState.displays.editValueFor(token-&gt;get());
                state.sequenceId = DisplayDeviceState{}.sequenceId; // Generate new sequenceId.
                state.physical-&gt;supportedModes = std::move(supportedModes);
                state.physical-&gt;activeMode = std::move(activeMode);
                if (getHwComposer().updatesDeviceProductInfoOnHotplugReconnect()) {
                    state.physical-&gt;deviceProductInfo = std::move(info-&gt;deviceProductInfo);
                }
            }
        } else {
            ALOGV(&#34;Removing display %s&#34;, to_string(displayId).c_str());
 
 
            if (const ssize_t index = mCurrentState.displays.indexOfKey(token-&gt;get()); index &gt;= 0) {
                const DisplayDeviceState&amp; state = mCurrentState.displays.valueAt(index);
                mInterceptor-&gt;saveDisplayDeletion(state.sequenceId);
                mCurrentState.displays.removeItemsAt(index);
            }
 
 
            mPhysicalDisplayTokens.erase(displayId);
        }
 
 
        processDisplayChangesLocked();
    }
 
 
    mPendingHotplugEvents.clear();
}
```

调用processDisplayChangesLocked方法：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
void SurfaceFlinger::processDisplayChangesLocked() {
    // here we take advantage of Vector&#39;s copy-on-write semantics to
    // improve performance by skipping the transaction entirely when
    // know that the lists are identical
    const KeyedVector&lt;wp&lt;IBinder&gt;, DisplayDeviceState&gt;&amp; curr(mCurrentState.displays);
    const KeyedVector&lt;wp&lt;IBinder&gt;, DisplayDeviceState&gt;&amp; draw(mDrawingState.displays);
    if (!curr.isIdenticalTo(draw)) {
        mVisibleRegionsDirty = true;
 
 
        // find the displays that were removed
        // (ie: in drawing state but not in current state)
        // also handle displays that changed
        // (ie: displays that are in both lists)
        for (size_t i = 0; i &lt; draw.size(); i&#43;&#43;) {
            const wp&lt;IBinder&gt;&amp; displayToken = draw.keyAt(i);
            const ssize_t j = curr.indexOfKey(displayToken);
            if (j &lt; 0) {
                // in drawing state but not in current state
                processDisplayRemoved(displayToken);
            } else {
                // this display is in both lists. see if something changed.
                const DisplayDeviceState&amp; currentState = curr[j];
                const DisplayDeviceState&amp; drawingState = draw[i];
                processDisplayChanged(displayToken, currentState, drawingState);
            }
        }
 
 
        // find displays that were added
        // (ie: in current state but not in drawing state)
        for (size_t i = 0; i &lt; curr.size(); i&#43;&#43;) {
            const wp&lt;IBinder&gt;&amp; displayToken = curr.keyAt(i);
            if (draw.indexOfKey(displayToken) &lt; 0) {
                processDisplayAdded(displayToken, curr[i]);
            }
        }
    }
 
 
    mDrawingState.displays = mCurrentState.displays;
}
```

上面方法主要处理如下：

1、调用processDisplayRemoved方法。

2、调用processDisplayChanged方法。

3、调用processDisplayAdded方法。

下面分别进行分析：

### surfaceflinger processDisplayRemoved

调用processDisplayRemoved方法：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
void SurfaceFlinger::processDisplayRemoved(const wp&lt;IBinder&gt;&amp; displayToken) {
    auto display = getDisplayDeviceLocked(displayToken);
    if (display) {
        display-&gt;disconnect();
 
 
        if (display-&gt;isVirtual()) {
            releaseVirtualDisplay(display-&gt;getVirtualId());
        } else {
            dispatchDisplayHotplugEvent(display-&gt;getPhysicalId(), false);
        }
    }
 
 
    mDisplays.erase(displayToken);
 
 
    if (display &amp;&amp; display-&gt;isVirtual()) {
        static_cast&lt;void&gt;(mScheduler-&gt;schedule([display = std::move(display)] {
            // Destroy the display without holding the mStateLock.
            // This is a temporary solution until we can manage transaction queues without
            // holding the mStateLock.
            // With blast, the IGBP that is passed to the VirtualDisplaySurface is owned by the
            // client. When the IGBP is disconnected, its buffer cache in SF will be cleared
            // via SurfaceComposerClient::doUncacheBufferTransaction. This call from the client
            // ends up running on the main thread causing a deadlock since setTransactionstate
            // will try to acquire the mStateLock. Instead we extend the lifetime of
            // DisplayDevice and destroy it in the main thread without holding the mStateLock.
            // The display will be disconnected and removed from the mDisplays list so it will
            // not be accessible.
        }));
    }
}
```

### surfaceflinger processDisplayChanged

调用processDisplayChanged方法：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
void SurfaceFlinger::processDisplayChanged(const wp&lt;IBinder&gt;&amp; displayToken,
                                           const DisplayDeviceState&amp; currentState,
                                           const DisplayDeviceState&amp; drawingState) {
    const sp&lt;IBinder&gt; currentBinder = IInterface::asBinder(currentState.surface);
    const sp&lt;IBinder&gt; drawingBinder = IInterface::asBinder(drawingState.surface);
 
 
    // Recreate the DisplayDevice if the surface or sequence ID changed.
    if (currentBinder != drawingBinder || currentState.sequenceId != drawingState.sequenceId) {
        getRenderEngine().cleanFramebufferCache();
 
 
        if (const auto display = getDisplayDeviceLocked(displayToken)) {
            display-&gt;disconnect();
            if (display-&gt;isVirtual()) {
                releaseVirtualDisplay(display-&gt;getVirtualId());
            }
        }
 
 
        mDisplays.erase(displayToken);
 
 
        if (const auto&amp; physical = currentState.physical) {
            getHwComposer().allocatePhysicalDisplay(physical-&gt;hwcDisplayId, physical-&gt;id);
        }
 
 
        processDisplayAdded(displayToken, currentState);
 
 
        if (currentState.physical) {
            const auto display = getDisplayDeviceLocked(displayToken);
            setPowerModeInternal(display, hal::PowerMode::ON);
 
 
            // TODO(b/175678251) Call a listener instead.
            if (currentState.physical-&gt;hwcDisplayId == getHwComposer().getPrimaryHwcDisplayId()) {
                updateInternalDisplayVsyncLocked(display);
            }
        }
        return;
    }
 
 
    if (const auto display = getDisplayDeviceLocked(displayToken)) {
        if (currentState.layerStack != drawingState.layerStack) {
            display-&gt;setLayerStack(currentState.layerStack);
        }
        if (currentState.flags != drawingState.flags) {
            display-&gt;setFlags(currentState.flags);
        }
        if ((currentState.orientation != drawingState.orientation) ||
            (currentState.layerStackSpaceRect != drawingState.layerStackSpaceRect) ||
            (currentState.orientedDisplaySpaceRect != drawingState.orientedDisplaySpaceRect)) {
            display-&gt;setProjection(currentState.orientation, currentState.layerStackSpaceRect,
                                   currentState.orientedDisplaySpaceRect);
            if (isDisplayActiveLocked(display)) {
                mActiveDisplayTransformHint = display-&gt;getTransformHint();
            }
        }
        if (currentState.width != drawingState.width ||
            currentState.height != drawingState.height) {
            display-&gt;setDisplaySize(currentState.width, currentState.height);
 
 
            if (isDisplayActiveLocked(display)) {
                onActiveDisplaySizeChanged(display);
            }
        }
    }
}
```

### Surfaceflinger processDisplayAdded

调用processDisplayAdded方法：

```cpp
/frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
void SurfaceFlinger::processDisplayAdded(const wp&lt;IBinder&gt;&amp; displayToken,
                                         const DisplayDeviceState&amp; state) {
    ui::Size resolution(0, 0);
    ui::PixelFormat pixelFormat = static_cast&lt;ui::PixelFormat&gt;(PIXEL_FORMAT_UNKNOWN);
    if (state.physical) {
        resolution = state.physical-&gt;activeMode-&gt;getResolution();
        pixelFormat = static_cast&lt;ui::PixelFormat&gt;(PIXEL_FORMAT_RGBA_8888);
    } else if (state.surface != nullptr) {
        int status = state.surface-&gt;query(NATIVE_WINDOW_WIDTH, &amp;resolution.width);
        ALOGE_IF(status != NO_ERROR, &#34;Unable to query width (%d)&#34;, status);
        status = state.surface-&gt;query(NATIVE_WINDOW_HEIGHT, &amp;resolution.height);
        ALOGE_IF(status != NO_ERROR, &#34;Unable to query height (%d)&#34;, status);
        int format;
        status = state.surface-&gt;query(NATIVE_WINDOW_FORMAT, &amp;format);
        ALOGE_IF(status != NO_ERROR, &#34;Unable to query format (%d)&#34;, status);
        pixelFormat = static_cast&lt;ui::PixelFormat&gt;(format);
    } else {
        // Virtual displays without a surface are dormant:
        // they have external state (layer stack, projection,
        // etc.) but no internal state (i.e. a DisplayDevice).
        return;
    }
 
 
    compositionengine::DisplayCreationArgsBuilder builder;
    if (const auto&amp; physical = state.physical) {
        builder.setId(physical-&gt;id);
    } else {
        builder.setId(acquireVirtualDisplay(resolution, pixelFormat));
    }
 
 
    builder.setPixels(resolution);
    builder.setIsSecure(state.isSecure);
    builder.setPowerAdvisor(mPowerAdvisor.get());
    builder.setName(state.displayName);
    auto compositionDisplay = getCompositionEngine().createDisplay(builder.build());
    compositionDisplay-&gt;setLayerCachingEnabled(mLayerCachingEnabled);
 
 
    sp&lt;compositionengine::DisplaySurface&gt; displaySurface;
    sp&lt;IGraphicBufferProducer&gt; producer;
    sp&lt;IGraphicBufferProducer&gt; bqProducer;
    sp&lt;IGraphicBufferConsumer&gt; bqConsumer;
    getFactory().createBufferQueue(&amp;bqProducer, &amp;bqConsumer, /*consumerIsSurfaceFlinger =*/false);
 
 
    if (state.isVirtual()) {
        const auto displayId = VirtualDisplayId::tryCast(compositionDisplay-&gt;getId());
        LOG_FATAL_IF(!displayId);
        auto surface = sp&lt;VirtualDisplaySurface&gt;::make(getHwComposer(), *displayId, state.surface,
                                                       bqProducer, bqConsumer, state.displayName);
        displaySurface = surface;
        producer = std::move(surface);
    } else {
        ALOGE_IF(state.surface != nullptr,
                 &#34;adding a supported display, but rendering &#34;
                 &#34;surface is provided (%p), ignoring it&#34;,
                 state.surface.get());
        const auto displayId = PhysicalDisplayId::tryCast(compositionDisplay-&gt;getId());
        LOG_FATAL_IF(!displayId);
        displaySurface =
                sp&lt;FramebufferSurface&gt;::make(getHwComposer(), *displayId, bqConsumer,
                                             state.physical-&gt;activeMode-&gt;getResolution(),
                                             ui::Size(maxGraphicsWidth, maxGraphicsHeight));
        producer = bqProducer;
    }
 
 
    LOG_FATAL_IF(!displaySurface);
    auto display = setupNewDisplayDeviceInternal(displayToken, std::move(compositionDisplay), state,
                                                 displaySurface, producer);
    if (display-&gt;isPrimary()) {
        initScheduler(display);
    }
    if (!state.isVirtual()) {
        dispatchDisplayHotplugEvent(display-&gt;getPhysicalId(), true);
    }
 
 
    mDisplays.try_emplace(displayToken, std::move(display));
}
```

#### SurfaceFlinger initScheduler

调用initScheduler方法，初始化Scheduler：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
std::unique_ptr&lt;scheduler::Scheduler&gt; mScheduler;
void SurfaceFlinger::initScheduler(const sp&lt;DisplayDevice&gt;&amp; display) {
    if (mScheduler) {
        // If the scheduler is already initialized, this means that we received
        // a hotplug(connected) on the primary display. In that case we should
        // update the scheduler with the most recent display information.
        ALOGW(&#34;Scheduler already initialized, updating instead&#34;);
        mScheduler-&gt;setRefreshRateConfigs(display-&gt;holdRefreshRateConfigs());
        return;
    }
    const auto currRefreshRate = display-&gt;getActiveMode()-&gt;getFps();
    mRefreshRateStats = std::make_unique&lt;scheduler::RefreshRateStats&gt;(*mTimeStats, currRefreshRate,
                                                                      hal::PowerMode::OFF);
 
 
    mVsyncConfiguration = getFactory().createVsyncConfiguration(currRefreshRate);
    mVsyncModulator = sp&lt;VsyncModulator&gt;::make(mVsyncConfiguration-&gt;getCurrentConfigs());
 
 
    using Feature = scheduler::Feature;
    scheduler::FeatureFlags features;
 
 
    if (sysprop::use_content_detection_for_refresh_rate(false)) {
        features |= Feature::kContentDetection;
    }
    if (base::GetBoolProperty(&#34;debug.sf.show_predicted_vsync&#34;s, false)) {
        features |= Feature::kTracePredictedVsync;
    }
    if (!base::GetBoolProperty(&#34;debug.sf.vsync_reactor_ignore_present_fences&#34;s, false) &amp;&amp;
        !getHwComposer().hasCapability(Capability::PRESENT_FENCE_IS_NOT_RELIABLE)) {
        features |= Feature::kPresentFences;
    }
 
 
    // 创建Scheduler对象
    mScheduler = std::make_unique&lt;scheduler::Scheduler&gt;(static_cast&lt;ICompositor&amp;&gt;(*this),
                                                        static_cast&lt;ISchedulerCallback&amp;&gt;(*this),
                                                        features);
    {
        auto configs = display-&gt;holdRefreshRateConfigs();
        if (configs-&gt;kernelIdleTimerController().has_value()) {
            features |= Feature::kKernelIdleTimer;
        }
 
 
        mScheduler-&gt;createVsyncSchedule(features);
        mScheduler-&gt;setRefreshRateConfigs(std::move(configs));
    }
    setVsyncEnabled(false);
    mScheduler-&gt;startTimers();
 
 
    const auto configs = mVsyncConfiguration-&gt;getCurrentConfigs();
    const nsecs_t vsyncPeriod = currRefreshRate.getPeriodNsecs();
    mAppConnectionHandle =
            mScheduler-&gt;createConnection(&#34;app&#34;, mFrameTimeline-&gt;getTokenManager(),
                                         /*workDuration=*/configs.late.appWorkDuration,
                                         /*readyDuration=*/configs.late.sfWorkDuration,
                                         impl::EventThread::InterceptVSyncsCallback());
    mSfConnectionHandle =
            mScheduler-&gt;createConnection(&#34;appSf&#34;, mFrameTimeline-&gt;getTokenManager(),
                                         /*workDuration=*/std::chrono::nanoseconds(vsyncPeriod),
                                         /*readyDuration=*/configs.late.sfWorkDuration,
                                         [this](nsecs_t timestamp) {
                                             mInterceptor-&gt;saveVSyncEvent(timestamp);
                                         });
 
 
    mScheduler-&gt;initVsync(mScheduler-&gt;getVsyncDispatch(), *mFrameTimeline-&gt;getTokenManager(),
                          configs.late.sfWorkDuration);
 
 
    mRegionSamplingThread =
            new RegionSamplingThread(*this, RegionSamplingThread::EnvironmentTimingTunables());
    mFpsReporter = new FpsReporter(*mFrameTimeline, *this);
    // Dispatch a mode change request for the primary display on scheduler
    // initialization, so that the EventThreads always contain a reference to a
    // prior configuration.
    //
    // This is a bit hacky, but this avoids a back-pointer into the main SF
    // classes from EventThread, and there should be no run-time binder cost
    // anyway since there are no connected apps at this point.
    mScheduler-&gt;onPrimaryDisplayModeChanged(mAppConnectionHandle, display-&gt;getActiveMode());
}
```

上面方法主要处理如下：

1、创建Scheduler对象，Scheduler是SurfaceFlinger的调度器。

2、调用mScheduler(Scheduler)的createConnection方法创建mAppConnectionHandle和mSfConnectionHandle对象。

3、调用Scheduler的initVsync方法，初始化Vsync。

下面分别进行分析：

##### new Scheduler

创建Scheduler对象，Scheduler是SurfaceFlinger的调度器，Scheduler的构造方法：

```cpp
//frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
class Scheduler : impl::MessageQueue {
Scheduler::Scheduler(ICompositor&amp; compositor, ISchedulerCallback&amp; callback, FeatureFlags features)
      : impl::MessageQueue(compositor), mFeatures(features), mSchedulerCallback(callback) {}
}
```

##### Scheduler createConnection

调用mScheduler(Scheduler)的createConnection方法创建mAppConnectionHandle和mSfConnectionHandle对象：

```cpp
//frameworks/native/services/surfaceflinger/Scheduler/Scheduler.cpp
ConnectionHandle Scheduler::createConnection(
        const char* connectionName, frametimeline::TokenManager* tokenManager,
        std::chrono::nanoseconds workDuration, std::chrono::nanoseconds readyDuration,
        impl::EventThread::InterceptVSyncsCallback interceptCallback) {
    auto vsyncSource = makePrimaryDispSyncSource(connectionName, workDuration, readyDuration);
    auto throttleVsync = makeThrottleVsyncCallback();
    auto getVsyncPeriod = makeGetVsyncPeriodFunction();
    auto eventThread = std::make_unique&lt;impl::EventThread&gt;(std::move(vsyncSource), tokenManager,
                                                           std::move(interceptCallback),
                                                           std::move(throttleVsync),
                                                           std::move(getVsyncPeriod));
    return createConnection(std::move(eventThread));
}
```

创建EventThread对象，EventThread的构造方法如下：

```cpp
//frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
class EventThread : public android::EventThread, private VSyncSource::Callback {
EventThread::EventThread(std::unique_ptr&lt;VSyncSource&gt; vsyncSource,
                         android::frametimeline::TokenManager* tokenManager,
                         InterceptVSyncsCallback interceptVSyncsCallback,
                         ThrottleVsyncCallback throttleVsyncCallback,
                         GetVsyncPeriodFunction getVsyncPeriodFunction)
      : mVSyncSource(std::move(vsyncSource)),
        mTokenManager(tokenManager),
        mInterceptVSyncsCallback(std::move(interceptVSyncsCallback)),
        mThrottleVsyncCallback(std::move(throttleVsyncCallback)),
        mGetVsyncPeriodFunction(std::move(getVsyncPeriodFunction)),
        mThreadName(mVSyncSource-&gt;getName()) {
 
 
    LOG_ALWAYS_FATAL_IF(getVsyncPeriodFunction == nullptr,
            &#34;getVsyncPeriodFunction must not be null&#34;);
 
 
    mVSyncSource-&gt;setCallback(this); //设置VsyncSource回调
 
 
    mThread = std::thread([this]() NO_THREAD_SAFETY_ANALYSIS {
        std::unique_lock&lt;std::mutex&gt; lock(mMutex); //创建一个线程
        threadMain(lock); //调用threadMain方法，接收回调消息
    });
 
 
    pthread_setname_np(mThread.native_handle(), mThreadName);
 
 
    pid_t tid = pthread_gettid_np(mThread.native_handle());
 
 
    // Use SCHED_FIFO to minimize jitter
    constexpr int EVENT_THREAD_PRIORITY = 2;
    struct sched_param param = {0};
    param.sched_priority = EVENT_THREAD_PRIORITY;
    if (pthread_setschedparam(mThread.native_handle(), SCHED_FIFO, &amp;param) != 0) {
        ALOGE(&#34;Couldn&#39;t set SCHED_FIFO for EventThread&#34;);
    }
 
 
    set_sched_policy(tid, SP_FOREGROUND);
}
}
```

在上面方法中先会设置VsyncSource回调，然后创建一个线程，调用调用threadMain方法，在threadMain方法中会循环接收回调消息，然后进行处理：

[Android13 EventThread threadMain流程分析-CSDN博客](/Android13%20EventThread%20threadMain%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-CSDN%E5%8D%9A%E5%AE%A2)


---

> 作者: [wentao](https://github.com/huang8604)  
> URL: https://huang8604.github.io/posts/android13-surfaceflinger-oncomposerhalhotplug%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-csdn%E5%8D%9A%E5%AE%A2/  

