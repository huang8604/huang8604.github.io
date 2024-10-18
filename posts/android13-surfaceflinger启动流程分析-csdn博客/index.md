# Android13 SurfaceFlinger启动流程分析-CSDN博客

在Android13版本中，SurfaceFlinger是由Android.bp去启动init.rc文件，然后再解析文件去加载SurfaceFlinger。

```cobol
//frameworks/native/services/surfaceflinger/SurfaceFlinger.rc
service surfaceflinger /system/bin/surfaceflinger
    class core animation
    user system
    group graphics drmrpc readproc
    capabilities SYS_NICE
    onrestart restart --only-if-running zygote
    task_profiles HighPerformance
    socket pdx/system/vr/display/client     stream 0666 system graphics u:object_r:pdx_display_client_endpoint_socket:s0
    socket pdx/system/vr/display/manager    stream 0666 system graphics u:object_r:pdx_display_manager_endpoint_socket:s0
    socket pdx/system/vr/display/vsync      stream 0666 system graphics u:object_r:pdx_display_vsync_endpoint_socket:s0
 
 
on property:vendor.debug.sf.restart=1
     restart surfaceflinger
 
 
on property:init.svc.zygote=restarting &amp;&amp; property:debug.sf.toomanylayers=1
     restart surfaceflinger
     setprop debug.sf.toomanylayers &#34;0&#34;
```

然后以此来调用main\_SurfaceFlinger.cpp文件的main函数：

```cpp
//frameworks/native/services/surfacefilinger/main_SurfaceFlinger.cpp
int main(int, char**) {
    signal(SIGPIPE, SIG_IGN);
 
 
    hardware::configureRpcThreadpool(1 /* maxThreads */,
            false /* callerWillJoin */);
 
 
    startGraphicsAllocatorService();
 
 
    // When SF is launched in its own process, limit the number of
    // binder threads to 4.
    ProcessState::self()-&gt;setThreadPoolMaxThreadCount(4);
 
 
    // Set uclamp.min setting on all threads, maybe an overkill but we want
    // to cover important threads like RenderEngine.
    if (SurfaceFlinger::setSchedAttr(true) != NO_ERROR) {
        ALOGW(&#34;Couldn&#39;t set uclamp.min: %s\n&#34;, strerror(errno));
    }
 
 
    // The binder threadpool we start will inherit sched policy and priority
    // of (this) creating thread. We want the binder thread pool to have
    // SCHED_FIFO policy and priority 1 (lowest RT priority)
    // Once the pool is created we reset this thread&#39;s priority back to
    // original.
    int newPriority = 0;
    int origPolicy = sched_getscheduler(0);
    struct sched_param origSchedParam;
 
 
    int errorInPriorityModification = sched_getparam(0, &amp;origSchedParam);
    if (errorInPriorityModification == 0) {
        int policy = SCHED_FIFO;
        newPriority = sched_get_priority_min(policy);
 
 
        struct sched_param param;
        param.sched_priority = newPriority;
 
 
        errorInPriorityModification = sched_setscheduler(0, policy, &amp;param);
    }
 
 
    // start the thread pool
    //构建ProcessState全局对象gProcess，打开binder驱动，建立链接
    //在驱动内部创建该进程的binder_proc,binder_thread结构，保存该进程的进程信息和线程信息，并加入驱动的红黑树队列中。
    //获取驱动的版本信息，把该进程最多可同时启动的线程告诉驱动，并保存到改进程的binder_proc结构中
    //把设备文件/dev/binder映射到内存中
    sp&lt;ProcessState&gt; ps(ProcessState::self());
    ps-&gt;startThreadPool();
 
 
    // Reset current thread&#39;s policy and priority
    if (errorInPriorityModification == 0) {
        errorInPriorityModification = sched_setscheduler(0, origPolicy, &amp;origSchedParam);
    } else {
        ALOGE(&#34;Failed to set SurfaceFlinger binder threadpool priority to SCHED_FIFO&#34;);
    }
 
 
    // instantiate surfaceflinger
    // 实例化surfaceflinger
    sp&lt;SurfaceFlinger&gt; flinger = surfaceflinger::createSurfaceFlinger();
 
 
    // Set the minimum policy of surfaceflinger node to be SCHED_FIFO.
    // So any thread with policy/priority lower than {SCHED_FIFO, 1}, will run
    // at least with SCHED_FIFO policy and priority 1.
    if (errorInPriorityModification == 0) {
        flinger-&gt;setMinSchedulerPolicy(SCHED_FIFO, newPriority);
    }
 
 
    //设置优先级
    setpriority(PRIO_PROCESS, 0, PRIORITY_URGENT_DISPLAY);
 
 
    //把SF的自身调用限制在4线程
    set_sched_policy(0, SP_FOREGROUND);
 
 
    // Put most SurfaceFlinger threads in the system-background cpuset
    // Keeps us from unnecessarily using big cores
    // Do this after the binder thread pool init
    if (cpusets_enabled()) set_cpuset_policy(0, SP_SYSTEM);
 
 
    // initialize before clients can connect
    // 在客户端连接之前进行初始化
    flinger-&gt;init();
 
 
    // publish surface flinger
    // 将surfaceflinger放入servicemanager
    sp&lt;IServiceManager&gt; sm(defaultServiceManager());
    sm-&gt;addService(String16(SurfaceFlinger::getServiceName()), flinger, false,
                   IServiceManager::DUMP_FLAG_PRIORITY_CRITICAL | IServiceManager::DUMP_FLAG_PROTO);
 
 
    // publish gui::ISurfaceComposer, the new AIDL interface
    sp&lt;SurfaceComposerAIDL&gt; composerAIDL = new SurfaceComposerAIDL(flinger);
    sm-&gt;addService(String16(&#34;SurfaceFlingerAIDL&#34;), composerAIDL, false,
                   IServiceManager::DUMP_FLAG_PRIORITY_CRITICAL | IServiceManager::DUMP_FLAG_PROTO);
 
 
    startDisplayService(); // dependency on SF getting registered above
 
 
    if (SurfaceFlinger::setSchedFifo(true) != NO_ERROR) {
        ALOGW(&#34;Couldn&#39;t set to SCHED_FIFO: %s&#34;, strerror(errno));
    }
 
 
    // run surface flinger in this thread
    // 运行surfaceflinger这个线程
    flinger-&gt;run();
 
 
    return 0;
}
```

上面方法主要处理如下：

1、调用surfaceflinger的createSurfaceFlinger方法，实例化surfaceflinger。

2、调用flinger(surfaceflinger)的init()方法，在客户端连接之前进行初始化。

3、调用flinger(surfaceflinger)的run()方法，运行surfaceflinger这个线程。

下面分别进行分析：

## surfaceflinger createSurfaceFlinger

通过调用surfaceflinger的createSurfaceFlinger方法实例化surfaceflinger：

```cpp
//frameworks/native/services/surfaceflinger/SurfaceflingerFactory.cpp
sp&lt;SurfaceFlinger&gt; createSurfaceFlinger() {
    static DefaultFactory factory;
    return new SurfaceFlinger(factory);
}
```

### new surfaceflinger

通过new的方式创建surfaceflinger对象，surfaceflinger的构造方法如下：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
SurfaceFlinger::SurfaceFlinger(Factory&amp; factory) : SurfaceFlinger(factory, SkipInitialization) {
    ALOGI(&#34;SurfaceFlinger is starting&#34;);
 
 
    hasSyncFramework = running_without_sync_framework(true);
 
 
    dispSyncPresentTimeOffset = present_time_offset_from_vsync_ns(0);
 
 
    useHwcForRgbToYuv = force_hwc_copy_for_virtual_displays(false);
 
 
    maxFrameBufferAcquiredBuffers = max_frame_buffer_acquired_buffers(2);
 
 
    maxGraphicsWidth = std::max(max_graphics_width(0), 0);
    maxGraphicsHeight = std::max(max_graphics_height(0), 0);
 
 
    hasWideColorDisplay = has_wide_color_display(false);
    mDefaultCompositionDataspace =
            static_cast&lt;ui::Dataspace&gt;(default_composition_dataspace(Dataspace::V0_SRGB));
    mWideColorGamutCompositionDataspace = static_cast&lt;ui::Dataspace&gt;(wcg_composition_dataspace(
            hasWideColorDisplay ? Dataspace::DISPLAY_P3 : Dataspace::V0_SRGB));
    defaultCompositionDataspace = mDefaultCompositionDataspace;
    wideColorGamutCompositionDataspace = mWideColorGamutCompositionDataspace;
    defaultCompositionPixelFormat = static_cast&lt;ui::PixelFormat&gt;(
            default_composition_pixel_format(ui::PixelFormat::RGBA_8888));
    wideColorGamutCompositionPixelFormat =
            static_cast&lt;ui::PixelFormat&gt;(wcg_composition_pixel_format(ui::PixelFormat::RGBA_8888));
 
 
    mColorSpaceAgnosticDataspace =
            static_cast&lt;ui::Dataspace&gt;(color_space_agnostic_dataspace(Dataspace::UNKNOWN));
 
 
    mLayerCachingEnabled = [] {
        const bool enable =
                android::sysprop::SurfaceFlingerProperties::enable_layer_caching().value_or(false);
        return base::GetBoolProperty(std::string(&#34;debug.sf.enable_layer_caching&#34;), enable);
    }();
 
 
    useContextPriority = use_context_priority(true);
 
 
    mInternalDisplayPrimaries = sysprop::getDisplayNativePrimaries();
 
 
    // debugging stuff...
    char value[PROPERTY_VALUE_MAX];
 
 
    property_get(&#34;ro.bq.gpu_to_cpu_unsupported&#34;, value, &#34;0&#34;);
    mGpuToCpuSupported = !atoi(value);
 
 
    property_get(&#34;ro.build.type&#34;, value, &#34;user&#34;);
    mIsUserBuild = strcmp(value, &#34;user&#34;) == 0;
 
 
    mDebugFlashDelay = base::GetUintProperty(&#34;debug.sf.showupdates&#34;s, 0u);
 
 
    // DDMS debugging deprecated (b/120782499)
    property_get(&#34;debug.sf.ddms&#34;, value, &#34;0&#34;);
    int debugDdms = atoi(value);
    ALOGI_IF(debugDdms, &#34;DDMS debugging not supported&#34;);
 
 
    property_get(&#34;debug.sf.enable_gl_backpressure&#34;, value, &#34;0&#34;);
    mPropagateBackpressureClientComposition = atoi(value);
    ALOGI_IF(mPropagateBackpressureClientComposition,
             &#34;Enabling backpressure propagation for Client Composition&#34;);
 
 
    property_get(&#34;ro.surface_flinger.supports_background_blur&#34;, value, &#34;0&#34;);
    bool supportsBlurs = atoi(value);
    mSupportsBlur = supportsBlurs;
    ALOGI_IF(!mSupportsBlur, &#34;Disabling blur effects, they are not supported.&#34;);
    property_get(&#34;ro.sf.blurs_are_expensive&#34;, value, &#34;0&#34;);
    mBlursAreExpensive = atoi(value);
 
 
    const size_t defaultListSize = ISurfaceComposer::MAX_LAYERS;
    auto listSize = property_get_int32(&#34;debug.sf.max_igbp_list_size&#34;, int32_t(defaultListSize));
    mMaxGraphicBufferProducerListSize = (listSize &gt; 0) ? size_t(listSize) : defaultListSize;
    mGraphicBufferProducerListSizeLogThreshold =
            std::max(static_cast&lt;int&gt;(0.95 *
                                      static_cast&lt;double&gt;(mMaxGraphicBufferProducerListSize)),
                     1);
 
 
    property_get(&#34;debug.sf.luma_sampling&#34;, value, &#34;1&#34;);
    mLumaSampling = atoi(value);
 
 
    property_get(&#34;debug.sf.disable_client_composition_cache&#34;, value, &#34;0&#34;);
    mDisableClientCompositionCache = atoi(value);
 
 
    property_get(&#34;debug.sf.predict_hwc_composition_strategy&#34;, value, &#34;1&#34;);
    mPredictCompositionStrategy = atoi(value);
 
 
    property_get(&#34;debug.sf.treat_170m_as_sRGB&#34;, value, &#34;0&#34;);
    mTreat170mAsSrgb = atoi(value);
 
 
    // We should be reading &#39;persist.sys.sf.color_saturation&#39; here
    // but since /data may be encrypted, we need to wait until after vold
    // comes online to attempt to read the property. The property is
    // instead read after the boot animation
 
 
    if (base::GetBoolProperty(&#34;debug.sf.treble_testing_override&#34;s, false)) {
        // Without the override SurfaceFlinger cannot connect to HIDL
        // services that are not listed in the manifests.  Considered
        // deriving the setting from the set service name, but it
        // would be brittle if the name that&#39;s not &#39;default&#39; is used
        // for production purposes later on.
        ALOGI(&#34;Enabling Treble testing override&#34;);
        android::hardware::details::setTrebleTestingOverride(true);
    }
 
 
    mRefreshRateOverlaySpinner = property_get_bool(&#34;sf.debug.show_refresh_rate_overlay_spinner&#34;, 0);
 
 
    if (!mIsUserBuild &amp;&amp; base::GetBoolProperty(&#34;debug.sf.enable_transaction_tracing&#34;s, true)) {
        mTransactionTracing.emplace();
    }
 
 
    mIgnoreHdrCameraLayers = ignore_hdr_camera_layers(false);
}
```

SurfaceFlinger继承BnSurfaceComposer，实现ISurfaceComposer接口，实现ComposerCallback，PriorityDumper是一个辅助类，提供了SurfaceFlinger的dump信息。ISurfaceComposer 是Client端对SurfaceFlinger进程的binder接口调用。

ComposerCallback，这个是HWC模块的回调，这个包含了三个很关键的回调函数，onComposerHotplug函数表示显示屏热插拔事件， onComposerHalRefresh函数表示Refresh事件，onComposerHalVsync表示Vsync信号事件。

## surfaceflinger init

调用flinger的init()方法，在客户端连接之前进行初始化：

在SurfaceFlingger初始化时，会向HWComposer注册回调，HWComposer会通过HWBinder向硬件测注册回调。

SurfaceFlinger搭建好处理Vsync的基础设施，初始化Scheduler，DispVsyncSoure以及最重要的EventThread。 EventThread的threadMain方法无限循环处理pendingEvents,对Vsync类型的Event分发到消费者，通过往消费者的FD写数据，通知APP有Vsync信号到来。pendingEvents中的消息处理完了，分发线程等待mCondition的通知。

当HWComposer收到硬件通过HWBinder回调onVSyncEvent时，会通过SurfaceFlinger的onVSyncEvent最终调用到EventThread的onVSyncEvent，于是调用makeVSync生成Vsync的Event, 添加到pendingEvents,然后再调用mCondition.notify\_all()唤起等待中的分发线程。

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
void SurfaceFlinger::init() {
    ALOGI(  &#34;SurfaceFlinger&#39;s main thread ready to run. &#34;
            &#34;Initializing graphics H/W...&#34;);
    Mutex::Autolock _l(mStateLock);
 
 
    // Get a RenderEngine for the given display / config (can&#39;t fail)
    // TODO(b/77156734): We need to stop casting and use HAL types when possible.
    // Sending maxFrameBufferAcquiredBuffers as the cache size is tightly tuned to single-display.
    //获取给定显示/配置的 RenderEngine，并且设置一些参数
    auto builder = renderengine::RenderEngineCreationArgs::Builder()
                           .setPixelFormat(static_cast&lt;int32_t&gt;(defaultCompositionPixelFormat))
                           .setImageCacheSize(maxFrameBufferAcquiredBuffers)
                           .setUseColorManagerment(useColorManagement)
                           .setEnableProtectedContext(enable_protected_contents(false))
                           .setPrecacheToneMapperShaderOnly(false)
                           .setSupportsBackgroundBlur(mSupportsBlur)
                           .setContextPriority(
                                   useContextPriority
                                           ? renderengine::RenderEngine::ContextPriority::REALTIME
                                           : renderengine::RenderEngine::ContextPriority::MEDIUM);
    if (auto type = chooseRenderEngineTypeViaSysProp()) {
        builder.setRenderEngineType(type.value());
    }
    mCompositionEngine-&gt;setRenderEngine(renderengine::RenderEngine::create(builder.build()));
    mMaxRenderTargetSize =
            std::min(getRenderEngine().getMaxTextureSize(), getRenderEngine().getMaxViewportDims());
 
 
    // Set SF main policy after initializing RenderEngine which has its own policy.
    if (!SetTaskProfiles(0, {&#34;SFMainPolicy&#34;})) {
        ALOGW(&#34;Failed to set main task profile&#34;);
    }
 
 
    mCompositionEngine-&gt;setTimeStats(mTimeStats);
    mCompositionEngine-&gt;setHwComposer(getFactory().createHWComposer(mHwcServiceName));
    mCompositionEngine-&gt;getHwComposer().setCallback(*this);
    ClientCache::getInstance().setRenderEngine(&amp;getRenderEngine());
 
 
    enableLatchUnsignaledConfig = getLatchUnsignaledConfig();
 
 
    if (base::GetBoolProperty(&#34;debug.sf.enable_hwc_vds&#34;s, false)) {
        enableHalVirtualDisplays(true);
    }
 
 
    // Process any initial hotplug and resulting display changes.
    processDisplayHotplugEventsLocked();
    const auto display = getDefaultDisplayDeviceLocked();
    LOG_ALWAYS_FATAL_IF(!display, &#34;Missing primary display after registering composer callback.&#34;);
    const auto displayId = display-&gt;getPhysicalId();
    LOG_ALWAYS_FATAL_IF(!getHwComposer().isConnected(displayId),
                        &#34;Primary display is disconnected.&#34;);
 
 
    // initialize our drawing state
    mDrawingState = mCurrentState;
 
 
    // set initial conditions (e.g. unblank default device)
    initializeDisplays();
 
 
    mPowerAdvisor-&gt;init();
 
 
    char primeShaderCache[PROPERTY_VALUE_MAX];
    property_get(&#34;service.sf.prime_shader_cache&#34;, primeShaderCache, &#34;1&#34;);
    if (atoi(primeShaderCache)) {
        if (setSchedFifo(false) != NO_ERROR) {
            ALOGW(&#34;Can&#39;t set SCHED_OTHER for primeCache&#34;);
        }
 
 
        mRenderEnginePrimeCacheFuture = getRenderEngine().primeCache();
 
 
        if (setSchedFifo(true) != NO_ERROR) {
            ALOGW(&#34;Can&#39;t set SCHED_OTHER for primeCache&#34;);
        }
    }
 
 
    onActiveDisplaySizeChanged(display);
 
 
    // Inform native graphics APIs whether the present timestamp is supported:
 
 
    const bool presentFenceReliable =
            !getHwComposer().hasCapability(Capability::PRESENT_FENCE_IS_NOT_RELIABLE);
    mStartPropertySetThread = getFactory().createStartPropertySetThread(presentFenceReliable);
 
 
    if (mStartPropertySetThread-&gt;Start() != NO_ERROR) {
        ALOGE(&#34;Run StartPropertySetThread failed!&#34;);
    }
 
 
    ALOGV(&#34;Done initializing&#34;);
}
```

调用processDisplayHotplugEventsLocked方法：

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
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
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

调用mScheduler(Scheduler)的createConnection方法创建mAppConnectionHandle和mSfConnectionHandle对象，一个名为&#34;app&#34;，用于APP绘制图像，一个名为&#34;sf&#34;，用于SurfaceFlinger合成：

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

待补充

##### Scheduler initVsync

调用Scheduler的initVsync方法：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
InjectVSyncSource* mVSyncInjector = nullptr;
bool Scheduler::injectVSync(nsecs_t when, nsecs_t expectedVSyncTime, nsecs_t deadlineTimestamp) {
    if (!mInjectVSyncs || !mVSyncInjector) {
        return false;
    }
 
 
    mVSyncInjector-&gt;onInjectSyncEvent(when, expectedVSyncTime, deadlineTimestamp);
    return true;
}
```

调用InjectVSyncSource的onInjectSyncEvent方法：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
class InjectVSyncSource final : public VSyncSource {
    void onInjectSyncEvent(nsecs_t when, nsecs_t expectedVSyncTimestamp,
                           nsecs_t deadlineTimestamp) {
        std::lock_guard&lt;std::mutex&gt; lock(mCallbackMutex);
        if (mCallback) {
            mCallback-&gt;onVSyncEvent(when, {expectedVSyncTimestamp, deadlineTimestamp});
        }
    }
}
```

## surfaceflinger run

调用flinger的run()方法，运行surfaceflinger这个线程：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
std::unique_ptr&lt;scheduler::Scheduler&gt; mScheduler;
void SurfaceFlinger::run() {
    mScheduler-&gt;run();
}
```

### Scheduler run

调用Scheduler的run方法：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
void Scheduler::run() {
    while (true) {
        waitMessage(); //等待消息
    }
}
```


---

> 作者: [wentao](https://github.com/huang8604)  
> URL: https://huang8604.github.io/posts/android13-surfaceflinger%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-csdn%E5%8D%9A%E5%AE%A2/  

