# Android13 SurfaceFlinger Composite(合成)流程分析-CSDN博客

SurfaceFlinger的composite方法，用于将多个窗口的图像进行合成，主要负责对相关要进行上帧的layer进行，识别排序好，然后合成，有hwc合成的会构建对应OutputLayer传递hwc，GPU合成则直接合成，再传递到hwc中，它主要完成以下几个步骤：

1. 从队列中获取所有待合成的缓冲区。
2. 将这些缓冲区按照一定的顺序进行合成，生成最终的图像。
3. 将合成后的图像提交给HWC进行显示。

SurfaceFlinger的composite方法代码如下：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
std::unique_ptr&lt;compositionengine::CompositionEngine&gt; mCompositionEngine;
void SurfaceFlinger::composite(nsecs_t frameTime, int64_t vsyncId)
        FTL_FAKE_GUARD(kMainThreadContext) {
    ATRACE_FORMAT(&#34;%s %&#34; PRId64, __func__, vsyncId);
 
 
    if (mPowerHintSessionData.sessionEnabled) {
        mPowerHintSessionData.compositeStart = systemTime();
    }
 
 
    compositionengine::CompositionRefreshArgs refreshArgs;
    const auto&amp; displays = FTL_FAKE_GUARD(mStateLock, mDisplays);
    refreshArgs.outputs.reserve(displays.size());
    for (const auto&amp; [_, display] : displays) {
        refreshArgs.outputs.push_back(display-&gt;getCompositionDisplay());
    }
    mDrawingState.traverseInZOrder([&amp;refreshArgs](Layer* layer) {
        if (auto layerFE = layer-&gt;getCompositionEngineLayerFE())
            refreshArgs.layers.push_back(layerFE);
    });
    refreshArgs.layersWithQueuedFrames.reserve(mLayersWithQueuedFrames.size());
    for (auto layer : mLayersWithQueuedFrames) {
        if (auto layerFE = layer-&gt;getCompositionEngineLayerFE())
            refreshArgs.layersWithQueuedFrames.push_back(layerFE);
    }
 
 
    refreshArgs.outputColorSetting = useColorManagement
            ? mDisplayColorSetting
            : compositionengine::OutputColorSetting::kUnmanaged;
    refreshArgs.colorSpaceAgnosticDataspace = mColorSpaceAgnosticDataspace;
    refreshArgs.forceOutputColorMode = mForceColorMode;
 
 
    refreshArgs.updatingOutputGeometryThisFrame = mVisibleRegionsDirty;
    refreshArgs.updatingGeometryThisFrame = mGeometryDirty.exchange(false) || mVisibleRegionsDirty;
    refreshArgs.blursAreExpensive = mBlursAreExpensive;
    refreshArgs.internalDisplayRotationFlags = DisplayDevice::getPrimaryDisplayRotationFlags();
 
 
    if (CC_UNLIKELY(mDrawingState.colorMatrixChanged)) {
        refreshArgs.colorTransformMatrix = mDrawingState.colorMatrix;
        mDrawingState.colorMatrixChanged = false;
    }
 
 
    refreshArgs.devOptForceClientComposition = mDebugDisableHWC;
 
 
    if (mDebugFlashDelay != 0) {
        refreshArgs.devOptForceClientComposition = true;
        refreshArgs.devOptFlashDirtyRegionsDelay = std::chrono::milliseconds(mDebugFlashDelay);
    }
 
 
    const auto expectedPresentTime = mExpectedPresentTime.load();
    const auto prevVsyncTime = mScheduler-&gt;getPreviousVsyncFrom(expectedPresentTime);
    const auto hwcMinWorkDuration = mVsyncConfiguration-&gt;getCurrentConfigs().hwcMinWorkDuration;
    refreshArgs.earliestPresentTime = prevVsyncTime - hwcMinWorkDuration;
    refreshArgs.previousPresentFence = mPreviousPresentFences[0].fenceTime;
    refreshArgs.scheduledFrameTime = mScheduler-&gt;getScheduledFrameTime();
    refreshArgs.expectedPresentTime = expectedPresentTime;
 
 
    // Store the present time just before calling to the composition engine so we could notify
    // the scheduler.
    const auto presentTime = systemTime();
 
 
    mCompositionEngine-&gt;present(refreshArgs);
 
 
    if (mPowerHintSessionData.sessionEnabled) {
        mPowerHintSessionData.presentEnd = systemTime();
    }
 
 
    mTimeStats-&gt;recordFrameDuration(frameTime, systemTime());
 
 
    if (mScheduler-&gt;onPostComposition(presentTime)) {
        scheduleComposite(FrameHint::kNone);
    }
 
 
    postFrame();
    postComposition();
 
 
    const bool prevFrameHadClientComposition = mHadClientComposition;
 
 
    mHadClientComposition = mHadDeviceComposition = mReusedClientComposition = false;
    TimeStats::ClientCompositionRecord clientCompositionRecord;
    for (const auto&amp; [_, display] : displays) {
        const auto&amp; state = display-&gt;getCompositionDisplay()-&gt;getState();
        mHadClientComposition |= state.usesClientComposition &amp;&amp; !state.reusedClientComposition;
        mHadDeviceComposition |= state.usesDeviceComposition;
        mReusedClientComposition |= state.reusedClientComposition;
        clientCompositionRecord.predicted |=
                (state.strategyPrediction != CompositionStrategyPredictionState::DISABLED);
        clientCompositionRecord.predictionSucceeded |=
                (state.strategyPrediction == CompositionStrategyPredictionState::SUCCESS);
    }
 
 
    clientCompositionRecord.hadClientComposition = mHadClientComposition;
    clientCompositionRecord.reused = mReusedClientComposition;
    clientCompositionRecord.changed = prevFrameHadClientComposition != mHadClientComposition;
    mTimeStats-&gt;pushCompositionStrategyState(clientCompositionRecord);
 
 
    // TODO: b/160583065 Enable skip validation when SF caches all client composition layers
    const bool usedGpuComposition = mHadClientComposition || mReusedClientComposition;
    modulateVsync(&amp;VsyncModulator::onDisplayRefresh, usedGpuComposition);
 
 
    mLayersWithQueuedFrames.clear();
    if (mLayerTracingEnabled &amp;&amp; mLayerTracing.flagIsSet(LayerTracing::TRACE_COMPOSITION)) {
        // This will block and should only be used for debugging.
        mLayerTracing.notify(mVisibleRegionsDirty, frameTime);
    }
 
 
    mVisibleRegionsWereDirtyThisFrame = mVisibleRegionsDirty; // Cache value for use in post-comp
    mVisibleRegionsDirty = false;
 
 
    if (mCompositionEngine-&gt;needsAnotherUpdate()) {
        scheduleCommit(FrameHint::kNone);
    }
 
 
    // calculate total render time for performance hinting if adpf cpu hint is enabled,
    if (mPowerHintSessionData.sessionEnabled) {
        const nsecs_t flingerDuration =
                (mPowerHintSessionData.presentEnd - mPowerHintSessionData.commitStart);
        mPowerAdvisor-&gt;sendActualWorkDuration(flingerDuration, mPowerHintSessionData.presentEnd);
    }
}
```

调用CompositionEngine的present，开始真正的Layer合成：

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngine/src/CompositionEngine.cpp
void CompositionEngine::present(CompositionRefreshArgs&amp; args) {
    ATRACE_CALL();
    ALOGV(__FUNCTION__);
 
 
    preComposition(args);
 
 
    {
        // latchedLayers is used to track the set of front-end layer state that
        // has been latched across all outputs for the prepare step, and is not
        // needed for anything else.
        LayerFESet latchedLayers;
 
 
        for (const auto&amp; output : args.outputs) {
            output-&gt;prepare(args, latchedLayers);
        }
    }
 
 
    updateLayerStateFromFE(args);
 
 
    for (const auto&amp; output : args.outputs) {
        output-&gt;present(args);
    }
}
```

上面方面主要处理如下：

1、调用CompositionEngine的preComposition方法，进行layer预合成。

2、调用Output的prepare方法，进行预合成。

3、调用Output的present方法，进行Layer合成。

下面分别进行分析：

## CompositionEngine preComposition

调用CompositionEngine的preComposition方法，进行合成前预处理：

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngine/src/CompositionEngine.cpp
void CompositionEngine::preComposition(CompositionRefreshArgs&amp; args) {
    ATRACE_CALL();
    ALOGV(__FUNCTION__);
 
 
    bool needsAnotherUpdate = false;
 
 
    mRefreshStartTime = systemTime(SYSTEM_TIME_MONOTONIC);
 
 
    //遍历所有layer，进行layer预合成
    for (auto&amp; layer : args.layers) {
        if (layer-&gt;onPreComposition(mRefreshStartTime)) {
            needsAnotherUpdate = true;
        }
    }
 
 
    mNeedsAnotherUpdate = needsAnotherUpdate;
}
```

### BufferLayer onPreComposition

调用Layer的onPreComposition方法，BufferLayer继承于Layer：

```cpp
//frameworks/native/services/surfaceflinger/BufferLayer.cpp
bool BufferLayer::onPreComposition(nsecs_t) {
    return hasReadyFrame();
}
```

调用BufferLayer的hasReadyFrame方法：

```cpp
//frameworks/native/services/surfaceflinger/BufferLayer.cpp
bool BufferLayer::hasReadyFrame() const {
    return hasFrameUpdate() || getSidebandStreamChanged() || getAutoRefresh();
}
```

## Output prepare

调用Output的prepare方法，进行预合成：

```cpp
/frameworks/native/services/surfaceflinger/CompositionEngin/src/Output.cpp
void Output::prepare(const compositionengine::CompositionRefreshArgs&amp; refreshArgs,
                     LayerFESet&amp; geomSnapshots) {
    ATRACE_CALL();
    ALOGV(__FUNCTION__);
 
 
    rebuildLayerStacks(refreshArgs, geomSnapshots); //Output实际上就是display， 通过调用每个Display的rebuildLayerStacks ，建立display 的 LayerStacks.
}
```

### Output rebuildLayerStacks

调用Output的rebuildLayerStacks方法：

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngin/src/Output.cpp
void Output::rebuildLayerStacks(const compositionengine::CompositionRefreshArgs&amp; refreshArgs,
                                LayerFESet&amp; layerFESet) {
    ATRACE_CALL();
    ALOGV(__FUNCTION__);
 
 
    auto&amp; outputState = editState();
 
 
    // Do nothing if this output is not enabled or there is no need to perform this update
    if (!outputState.isEnabled || CC_LIKELY(!refreshArgs.updatingOutputGeometryThisFrame)) {
        return;
    }
 
 
    // Process the layers to determine visibility and coverage
    compositionengine::Output::CoverageState coverage{layerFESet};
    collectVisibleLayers(refreshArgs, coverage);
 
 
    // Compute the resulting coverage for this output, and store it for later
    const ui::Transform&amp; tr = outputState.transform;
    Region undefinedRegion{outputState.displaySpace.getBoundsAsRect()};
    undefinedRegion.subtractSelf(tr.transform(coverage.aboveOpaqueLayers));
 
 
    outputState.undefinedRegion = undefinedRegion;
    outputState.dirtyRegion.orSelf(coverage.dirtyRegion);
}
```

## Output present

调用Output的present方法，进行Layer合成。

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngin/src/Output.cpp
void Output::present(const compositionengine::CompositionRefreshArgs&amp; refreshArgs) {
    ATRACE_CALL();
    ALOGV(__FUNCTION__);
 
 
    updateColorProfile(refreshArgs); //更新颜色配置文件
    updateCompositionState(refreshArgs); //更新合成状态
    planComposition(); //计划合成
    writeCompositionState(refreshArgs); //写入合成状态
    setColorTransform(refreshArgs); //设置颜色矩阵
    beginFrame(); //开始帧
 
 
    GpuCompositionResult result;
    const bool predictCompositionStrategy = canPredictCompositionStrategy(refreshArgs);
    if (predictCompositionStrategy) {
        result = prepareFrameAsync(refreshArgs); //准备帧数据以进行显示(Async方式)
    } else {
        prepareFrame(); //准备帧数据以进行显示
    }
 
 
    devOptRepaintFlash(refreshArgs); //处理显示输出设备的可选重绘闪烁
    finishFrame(refreshArgs, std::move(result)); //完成帧
    postFramebuffer(); //将帧缓冲区（framebuffer）的内容发送到显示设备进行显示
    renderCachedSets(refreshArgs); //进行渲染缓存设置
}
```

上面方法主要处理如下：

1、调用Output的updateColorProfile方法，更新颜色配置。

2、调用Output的updateCompositionState方法，更新合成状态。

3、调用Output的planComposition方法，计划合成。

4、调用Output的writeCompositionState方法，写入合成状态。

5、调用Output的setColorTransform方法，设置颜色矩阵。

6、调用Output的beginFrame方法，开始帧。

7、调用Output的prepareFrameAsync方法，准备帧数据以进行显示(Async方式)。

8、调用Output的prepareFrame方法，准备帧数据以进行显示。

9、调用Output的devOptRepaintFlash方法，处理显示输出设备的可选重绘闪烁。

10、调用Output的finishFrame方法，完成帧。

11、调用Output的postFramebuffer方法，将帧缓冲区（framebuffer）的内容发送到显示设备进行显示。

12、调用Output的renderCachedSets方法，进行渲染缓存设置。

下面分别进行分析：

### Output updateColorProfile

调用Output的updateColorProfile方法，更新颜色配置：

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngin/src/Output.cpp
void Output::updateColorProfile(const compositionengine::CompositionRefreshArgs&amp; refreshArgs) {
    setColorProfile(pickColorProfile(refreshArgs));
}
```

调用Output的setColorProfile方法：

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngin/src/Output.cpp
void Output::setColorProfile(const ColorProfile&amp; colorProfile) {
    ui::Dataspace targetDataspace =
            getDisplayColorProfile()-&gt;getTargetDataspace(colorProfile.mode, colorProfile.dataspace,
                                                         colorProfile.colorSpaceAgnosticDataspace);
 
 
    auto&amp; outputState = editState();
    if (outputState.colorMode == colorProfile.mode &amp;&amp;
        outputState.dataspace == colorProfile.dataspace &amp;&amp;
        outputState.renderIntent == colorProfile.renderIntent &amp;&amp;
        outputState.targetDataspace == targetDataspace) {
        return;
    }
 
 
    outputState.colorMode = colorProfile.mode;
    outputState.dataspace = colorProfile.dataspace;
    outputState.renderIntent = colorProfile.renderIntent;
    outputState.targetDataspace = targetDataspace;
 
 
    mRenderSurface-&gt;setBufferDataspace(colorProfile.dataspace); //设置缓存数据空间
 
 
    ALOGV(&#34;Set active color mode: %s (%d), active render intent: %s (%d)&#34;,
          decodeColorMode(colorProfile.mode).c_str(), colorProfile.mode,
          decodeRenderIntent(colorProfile.renderIntent).c_str(), colorProfile.renderIntent);
 
 
    dirtyEntireOutput(); //重置脏区
}
```

上面方法主要处理如下：

1、调用mRenderSurface(RenderSurface)的setBufferDataspace方法，设置缓存数据空间。

2、调用Output的dirtyEntireOutput方法，重置脏区。

下面分别进行分析：

#### RenderSurface setBufferDataspace

调用mRenderSurface(RenderSurface)的setBufferDataspace方法，设置缓存数据空间：

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngin/src/RenderSurface.cpp
void RenderSurface::setBufferDataspace(ui::Dataspace dataspace) {
    native_window_set_buffers_data_space(mNativeWindow.get(),
                                         static_cast&lt;android_dataspace&gt;(dataspace));
}
```

### Output updateCompositionState

调用Output的updateCompositionState方法，更新合成状态：

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngin/src/Output.cpp
void Output::updateCompositionState(const compositionengine::CompositionRefreshArgs&amp; refreshArgs) {
    ATRACE_CALL();
    ALOGV(__FUNCTION__);
 
 
    if (!getState().isEnabled) {
        return;
    }
 
 
    mLayerRequestingBackgroundBlur = findLayerRequestingBackgroundComposition();
    bool forceClientComposition = mLayerRequestingBackgroundBlur != nullptr;
 
 
    for (auto* layer : getOutputLayersOrderedByZ()) {
        layer-&gt;updateCompositionState(refreshArgs.updatingGeometryThisFrame,
                                      refreshArgs.devOptForceClientComposition ||
                                              forceClientComposition,
                                      refreshArgs.internalDisplayRotationFlags);
 
 
        if (mLayerRequestingBackgroundBlur == layer) {
            forceClientComposition = false;
        }
    }
}
```

#### OutputLayer updateCompositionState

调用OutputLayer的updateCompositionState方法：

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngin/src/OutputLayer.cpp
void OutputLayer::updateCompositionState(
        bool includeGeometry, bool forceClientComposition,
        ui::Transform::RotationFlags internalDisplayRotationFlags) {
    const auto* layerFEState = getLayerFE().getCompositionState();
    if (!layerFEState) {
        return;
    }
 
 
    const auto&amp; outputState = getOutput().getState();
    const auto&amp; profile = *getOutput().getDisplayColorProfile();
    auto&amp; state = editState();
 
 
    if (includeGeometry) {
        // Clear the forceClientComposition flag before it is set for any
        // reason. Note that since it can be set by some checks below when
        // updating the geometry state, we only clear it when updating the
        // geometry since those conditions for forcing client composition won&#39;t
        // go away otherwise.
        state.forceClientComposition = false;
 
 
        state.displayFrame = calculateOutputDisplayFrame();
        state.sourceCrop = calculateOutputSourceCrop(internalDisplayRotationFlags);
        state.bufferTransform = static_cast&lt;Hwc2::Transform&gt;(
                calculateOutputRelativeBufferTransform(internalDisplayRotationFlags));
 
 
        if ((layerFEState-&gt;isSecure &amp;&amp; !outputState.isSecure) ||
            (state.bufferTransform &amp; ui::Transform::ROT_INVALID)) {
            state.forceClientComposition = true;
        }
    }
 
 
    // Determine the output dependent dataspace for this layer. If it is
    // colorspace agnostic, it just uses the dataspace chosen for the output to
    // avoid the need for color conversion.
    state.dataspace = layerFEState-&gt;isColorspaceAgnostic &amp;&amp;
                    outputState.targetDataspace != ui::Dataspace::UNKNOWN
            ? outputState.targetDataspace
            : layerFEState-&gt;dataspace;
 
 
    // Override the dataspace transfer from 170M to sRGB if the device configuration requests this.
    // We do this here instead of in buffer info so that dumpsys can still report layers that are
    // using the 170M transfer. Also we only do this if the colorspace is not agnostic for the
    // layer, in case the color profile uses a 170M transfer function.
    if (outputState.treat170mAsSrgb &amp;&amp; !layerFEState-&gt;isColorspaceAgnostic &amp;&amp;
        (state.dataspace &amp; HAL_DATASPACE_TRANSFER_MASK) == HAL_DATASPACE_TRANSFER_SMPTE_170M) {
        state.dataspace = static_cast&lt;ui::Dataspace&gt;(
                (state.dataspace &amp; HAL_DATASPACE_STANDARD_MASK) |
                (state.dataspace &amp; HAL_DATASPACE_RANGE_MASK) | HAL_DATASPACE_TRANSFER_SRGB);
    }
 
 
    // For hdr content, treat the white point as the display brightness - HDR content should not be
    // boosted or dimmed.
    // If the layer explicitly requests to disable dimming, then don&#39;t dim either.
    if (isHdrDataspace(state.dataspace) ||
        getOutput().getState().displayBrightnessNits == getOutput().getState().sdrWhitePointNits ||
        getOutput().getState().displayBrightnessNits == 0.f || !layerFEState-&gt;dimmingEnabled) {
        state.dimmingRatio = 1.f;
        state.whitePointNits = getOutput().getState().displayBrightnessNits;
    } else {
        state.dimmingRatio = std::clamp(getOutput().getState().sdrWhitePointNits /
                                                getOutput().getState().displayBrightnessNits,
                                        0.f, 1.f);
        state.whitePointNits = getOutput().getState().sdrWhitePointNits;
    }
 
 
    // These are evaluated every frame as they can potentially change at any
    // time.
    if (layerFEState-&gt;forceClientComposition || !profile.isDataspaceSupported(state.dataspace) ||
        forceClientComposition) {
        state.forceClientComposition = true;
    }
}
```

### Output planComposition

调用Output的planComposition方法，计划合成：

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngin/src/Output.cpp
void Output::planComposition() {
    if (!mPlanner || !getState().isEnabled) {
        return;
    }
 
 
    ATRACE_CALL();
    ALOGV(__FUNCTION__);
 
 
    mPlanner-&gt;plan(getOutputLayersOrderedByZ());
}
```

#### Planner plan

调用Planner的plan方法：

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngin/src/Planner.cpp
void Planner::plan(
        compositionengine::Output::OutputLayersEnumerator&lt;compositionengine::Output&gt;&amp;&amp; layers) {
    ATRACE_CALL();
    std::unordered_set&lt;LayerId&gt; removedLayers;
    removedLayers.reserve(mPreviousLayers.size());
 
 
    std::transform(mPreviousLayers.begin(), mPreviousLayers.end(),
                   std::inserter(removedLayers, removedLayers.begin()),
                   [](const auto&amp; layer) { return layer.first; });
 
 
    std::vector&lt;LayerId&gt; currentLayerIds;
    for (auto layer : layers) {
        LayerId id = layer-&gt;getLayerFE().getSequence();
        if (const auto layerEntry = mPreviousLayers.find(id); layerEntry != mPreviousLayers.end()) {
            // Track changes from previous info
            LayerState&amp; state = layerEntry-&gt;second;
            ftl::Flags&lt;LayerStateField&gt; differences = state.update(layer);
            if (differences.get() == 0) {
                state.incrementFramesSinceBufferUpdate();
            } else {
                ALOGV(&#34;Layer %s changed: %s&#34;, state.getName().c_str(),
                      differences.string().c_str());
 
 
                if (differences.test(LayerStateField::Buffer)) {
                    state.resetFramesSinceBufferUpdate();
                } else {
                    state.incrementFramesSinceBufferUpdate();
                }
            }
        } else {
            LayerState state(layer);
            ALOGV(&#34;Added layer %s&#34;, state.getName().c_str());
            mPreviousLayers.emplace(std::make_pair(id, std::move(state)));
        }
 
 
        currentLayerIds.emplace_back(id);
 
 
        if (const auto found = removedLayers.find(id); found != removedLayers.end()) {
            removedLayers.erase(found);
        }
    }
 
 
    mCurrentLayers.clear();
    mCurrentLayers.reserve(currentLayerIds.size());
    std::transform(currentLayerIds.cbegin(), currentLayerIds.cend(),
                   std::back_inserter(mCurrentLayers), [this](LayerId id) {
                       LayerState* state = &amp;mPreviousLayers.at(id);
                       state-&gt;getOutputLayer()-&gt;editState().overrideInfo = {};
                       return state;
                   });
 
 
    const NonBufferHash hash = getNonBufferHash(mCurrentLayers);
    mFlattenedHash =
            mFlattener.flattenLayers(mCurrentLayers, hash, std::chrono::steady_clock::now());
    const bool layersWereFlattened = hash != mFlattenedHash;
 
 
    ALOGV(&#34;[%s] Initial hash %zx flattened hash %zx&#34;, __func__, hash, mFlattenedHash);
 
 
    if (mPredictorEnabled) {
        mPredictedPlan =
                mPredictor.getPredictedPlan(layersWereFlattened ? std::vector&lt;const LayerState*&gt;()
                                                                : mCurrentLayers,
                                            mFlattenedHash);
        if (mPredictedPlan) {
            ALOGV(&#34;[%s] Predicting plan %s&#34;, __func__, to_string(mPredictedPlan-&gt;plan).c_str());
        } else {
            ALOGV(&#34;[%s] No prediction found\n&#34;, __func__);
        }
    }
 
 
    // Clean up the set of previous layers now that the view of the LayerStates in the flattener are
    // up-to-date.
    for (LayerId removedLayer : removedLayers) {
        if (const auto layerEntry = mPreviousLayers.find(removedLayer);
            layerEntry != mPreviousLayers.end()) {
            const auto&amp; [id, state] = *layerEntry;
            ALOGV(&#34;Removed layer %s&#34;, state.getName().c_str());
            mPreviousLayers.erase(removedLayer);
        }
    }
}
```

### Output writeCompositionState

调用Output的writeCompositionState方法，写入合成状态：

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngin/src/Output.cpp
void Output::writeCompositionState(const compositionengine::CompositionRefreshArgs&amp; refreshArgs) {
    ATRACE_CALL();
    ALOGV(__FUNCTION__);
 
 
    if (!getState().isEnabled) {
        return;
    }
 
 
    editState().earliestPresentTime = refreshArgs.earliestPresentTime;
    editState().previousPresentFence = refreshArgs.previousPresentFence;
    editState().expectedPresentTime = refreshArgs.expectedPresentTime;
 
 
    compositionengine::OutputLayer* peekThroughLayer = nullptr;
    sp&lt;GraphicBuffer&gt; previousOverride = nullptr;
    bool includeGeometry = refreshArgs.updatingGeometryThisFrame;
    uint32_t z = 0;
    bool overrideZ = false;
    uint64_t outputLayerHash = 0;
    for (auto* layer : getOutputLayersOrderedByZ()) {
        if (layer == peekThroughLayer) {
            // No longer needed, although it should not show up again, so
            // resetting it is not truly needed either.
            peekThroughLayer = nullptr;
 
 
            // peekThroughLayer was already drawn ahead of its z order.
            continue;
        }
        bool skipLayer = false;
        const auto&amp; overrideInfo = layer-&gt;getState().overrideInfo;
        if (overrideInfo.buffer != nullptr) {
            if (previousOverride &amp;&amp; overrideInfo.buffer-&gt;getBuffer() == previousOverride) {
                ALOGV(&#34;Skipping redundant buffer&#34;);
                skipLayer = true;
            } else {
                // First layer with the override buffer.
                if (overrideInfo.peekThroughLayer) {
                    peekThroughLayer = overrideInfo.peekThroughLayer;
 
 
                    // Draw peekThroughLayer first.
                    overrideZ = true;
                    includeGeometry = true;
                    constexpr bool isPeekingThrough = true;
                    peekThroughLayer-&gt;writeStateToHWC(includeGeometry, false, z&#43;&#43;, overrideZ,
                                                      isPeekingThrough);
                    outputLayerHash ^= android::hashCombine(
                            reinterpret_cast&lt;uint64_t&gt;(&amp;peekThroughLayer-&gt;getLayerFE()),
                            z, includeGeometry, overrideZ, isPeekingThrough,
                            peekThroughLayer-&gt;requiresClientComposition());
                }
 
 
                previousOverride = overrideInfo.buffer-&gt;getBuffer();
            }
        }
 
 
        constexpr bool isPeekingThrough = false;
        layer-&gt;writeStateToHWC(includeGeometry, skipLayer, z&#43;&#43;, overrideZ, isPeekingThrough);
        if (!skipLayer) {
            outputLayerHash ^= android::hashCombine(
                    reinterpret_cast&lt;uint64_t&gt;(&amp;layer-&gt;getLayerFE()),
                    z, includeGeometry, overrideZ, isPeekingThrough,
                    layer-&gt;requiresClientComposition());
        }
    }
    editState().outputLayerHash = outputLayerHash;
}
```

### Output setColorTransform

调用Output的setColorTransform方法，设置颜色矩阵：

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngin/src/Output.cpp
void Output::setColorTransform(const compositionengine::CompositionRefreshArgs&amp; args) {
    auto&amp; colorTransformMatrix = editState().colorTransformMatrix;
    if (!args.colorTransformMatrix || colorTransformMatrix == args.colorTransformMatrix) {
        return;
    }
 
 
    colorTransformMatrix = *args.colorTransformMatrix;
 
 
    dirtyEntireOutput();
}
```

### Output beginFrame

调用Output的beginFrame方法，开始帧：

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngin/src/Output.cpp
void Output::beginFrame() {
    auto&amp; outputState = editState();
    const bool dirty = !getDirtyRegion().isEmpty();
    const bool empty = getOutputLayerCount() == 0;
    const bool wasEmpty = !outputState.lastCompositionHadVisibleLayers;
 
 
    // If nothing has changed (!dirty), don&#39;t recompose.
    // If something changed, but we don&#39;t currently have any visible layers,
    //   and didn&#39;t when we last did a composition, then skip it this time.
    // The second rule does two things:
    // - When all layers are removed from a display, we&#39;ll emit one black
    //   frame, then nothing more until we get new layers.
    // - When a display is created with a private layer stack, we won&#39;t
    //   emit any black frames until a layer is added to the layer stack.
    const bool mustRecompose = dirty &amp;&amp; !(empty &amp;&amp; wasEmpty);
 
 
    const char flagPrefix[] = {&#39;-&#39;, &#39;&#43;&#39;};
    static_cast&lt;void&gt;(flagPrefix);
    ALOGV_IF(&#34;%s: %s composition for %s (%cdirty %cempty %cwasEmpty)&#34;, __FUNCTION__,
             mustRecompose ? &#34;doing&#34; : &#34;skipping&#34;, getName().c_str(), flagPrefix[dirty],
             flagPrefix[empty], flagPrefix[wasEmpty]);
 
 
    mRenderSurface-&gt;beginFrame(mustRecompose);
 
 
    if (mustRecompose) {
        outputState.lastCompositionHadVisibleLayers = !empty;
    }
}
```

#### RenderSurface beginFrame

调用mRenderSurface(RenderSurface)的beginFrame方法：

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngin/src/RenderSurface.cpp
const sp&lt;DisplaySurface&gt; mDisplaySurface;
status_t RenderSurface::beginFrame(bool mustRecompose) {
    return mDisplaySurface-&gt;beginFrame(mustRecompose);
}
```

调用mDisplaySurface(DisplaySurface)的beginFrame方法，FramebufferSurface继承DisplaySurface，调用FramebufferSurface的beginFrame方法：

```cpp
//frameworks/native/services/surfaceflinger/DisplayHardware/FramebufferSurface.cpp
status_t FramebufferSurface::beginFrame(bool /*mustRecompose*/) {
    return NO_ERROR;
}
```

##### VirtualDisplaySurface beginFrame

调用mDisplaySurface(DisplaySurface)的beginFrame方法，VirtualDisplaySurface继承DisplaySurface，调用VirtualDisplaySurface的beginFrame方法：

```cpp
//frameworks/native/services/surfaceflinger/DisplayHardware/VirtualDisplaySurface.cpp
status_t VirtualDisplaySurface::beginFrame(bool mustRecompose) {
    if (GpuVirtualDisplayId::tryCast(mDisplayId)) {
        return NO_ERROR;
    }
 
 
    mMustRecompose = mustRecompose;
 
 
    VDS_LOGW_IF(mDebugState != DebugState::Idle, &#34;Unexpected %s in %s state&#34;, __func__,
                ftl::enum_string(mDebugState).c_str());
    mDebugState = DebugState::Begun;
 
 
    return refreshOutputBuffer();
}
```

调用VirtualDisplaySurface的refreshOutputBuffer方法，刷新输出缓冲区：

```cpp
//frameworks/native/services/surfaceflinger/DisplayHardware/VirtualDisplaySurface.cpp
HWComposer&amp; mHwc;
status_t VirtualDisplaySurface::refreshOutputBuffer() {
    LOG_ALWAYS_FATAL_IF(GpuVirtualDisplayId::tryCast(mDisplayId).has_value());
 
 
    if (mOutputProducerSlot &gt;= 0) {
        mSource[SOURCE_SINK]-&gt;cancelBuffer(
                mapProducer2SourceSlot(SOURCE_SINK, mOutputProducerSlot),
                mOutputFence);
    }
 
 
    int sslot;
    status_t result = dequeueBuffer(SOURCE_SINK, mOutputFormat, mOutputUsage,
            &amp;sslot, &amp;mOutputFence);
    if (result &lt; 0)
        return result;
    mOutputProducerSlot = mapSource2ProducerSlot(SOURCE_SINK, sslot);
 
 
    // On GPU-only frames, we don&#39;t have the right output buffer acquire fence
    // until after GPU calls queueBuffer(). So here we just set the buffer
    // (for use in HWC prepare) but not the fence; we&#39;ll call this again with
    // the proper fence once we have it.
    const auto halDisplayId = HalVirtualDisplayId::tryCast(mDisplayId);
    LOG_FATAL_IF(!halDisplayId);
    result = mHwc.setOutputBuffer(*halDisplayId, Fence::NO_FENCE,
                                  mProducerBuffers[mOutputProducerSlot]);
 
 
    return result;
}
```

###### HWComposer setOutputBuffer

调用mHwc(HWComposer)的setOutputBuffer方法：

```cpp
//frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp
std::unordered_map&lt;HalDisplayId, DisplayData&gt; mDisplayData;
std::unique_ptr&lt;HWC2::Display&gt; hwcDisplay;
status_t HWComposer::setOutputBuffer(HalVirtualDisplayId displayId, const sp&lt;Fence&gt;&amp; acquireFence,
                                     const sp&lt;GraphicBuffer&gt;&amp; buffer) {
    RETURN_IF_INVALID_DISPLAY(displayId, BAD_INDEX);
    const auto&amp; displayData = mDisplayData[displayId];
 
 
    auto error = displayData.hwcDisplay-&gt;setOutputBuffer(buffer, acquireFence);
    RETURN_IF_HWC_ERROR(error, displayId, UNKNOWN_ERROR);
    return NO_ERROR;
}
```

###### HWC2::Display setOutputBuffer

调用HWC2::Display的setOutputBuffer方法，之后就是HWC的处理了。

### Output prepareFrameAsync

调用Output的prepareFrameAsync方法，准备帧数据以进行显示(Async方式)：

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngin/src/Output.cpp
GpuCompositionResult Output::prepareFrameAsync(const CompositionRefreshArgs&amp; refreshArgs) {
    ATRACE_CALL();
    ALOGV(__FUNCTION__);
    auto&amp; state = editState();
    const auto&amp; previousChanges = state.previousDeviceRequestedChanges;
    std::optional&lt;android::HWComposer::DeviceRequestedChanges&gt; changes;
    resetCompositionStrategy();
    auto hwcResult = chooseCompositionStrategyAsync(&amp;changes);
    if (state.previousDeviceRequestedSuccess) {
        applyCompositionStrategy(previousChanges);
    }
    finishPrepareFrame();
 
 
    base::unique_fd bufferFence;
    std::shared_ptr&lt;renderengine::ExternalTexture&gt; buffer;
    updateProtectedContentState();
    const bool dequeueSucceeded = dequeueRenderBuffer(&amp;bufferFence, &amp;buffer);
    GpuCompositionResult compositionResult;
    if (dequeueSucceeded) {
        std::optional&lt;base::unique_fd&gt; optFd =
                composeSurfaces(Region::INVALID_REGION, refreshArgs, buffer, bufferFence);
        if (optFd) {
            compositionResult.fence = std::move(*optFd);
        }
    }
 
 
    auto chooseCompositionSuccess = hwcResult.get();
    const bool predictionSucceeded = dequeueSucceeded &amp;&amp; changes == previousChanges;
    state.strategyPrediction = predictionSucceeded ? CompositionStrategyPredictionState::SUCCESS
                                                   : CompositionStrategyPredictionState::FAIL;
    if (!predictionSucceeded) {
        ATRACE_NAME(&#34;CompositionStrategyPredictionMiss&#34;);
        resetCompositionStrategy();
        if (chooseCompositionSuccess) {
            applyCompositionStrategy(changes);
        }
        finishPrepareFrame();
        // Track the dequeued buffer to reuse so we don&#39;t need to dequeue another one.
        compositionResult.buffer = buffer;
    } else {
        ATRACE_NAME(&#34;CompositionStrategyPredictionHit&#34;);
    }
    state.previousDeviceRequestedChanges = std::move(changes);
    state.previousDeviceRequestedSuccess = chooseCompositionSuccess;
    return compositionResult;
}
```

### Output prepareFrame

调用Output的prepareFrame方法，准备帧数据以进行显示：

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngin/src/Output.cpp
void Output::prepareFrame() {
    ATRACE_CALL();
    ALOGV(__FUNCTION__);
 
 
    auto&amp; outputState = editState();
    if (!outputState.isEnabled) {
        return;
    }
 
 
    std::optional&lt;android::HWComposer::DeviceRequestedChanges&gt; changes;
    bool success = chooseCompositionStrategy(&amp;changes); //向HWC询问合成策略
    resetCompositionStrategy();
    outputState.strategyPrediction = CompositionStrategyPredictionState::DISABLED;
    outputState.previousDeviceRequestedChanges = changes;
    outputState.previousDeviceRequestedSuccess = success;
    if (success) {
        applyCompositionStrategy(changes);
    }
    finishPrepareFrame();
}
```

#### Output chooseCompositionStrategy

调用Output的chooseCompositionStrategy方法，向HWC询问合成策略：

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngin/src/Output.cpp
std::unique_ptr&lt;HwcAsyncWorker&gt; mHwComposerAsyncWorker;
std::future&lt;bool&gt; Output::chooseCompositionStrategyAsync(
        std::optional&lt;android::HWComposer::DeviceRequestedChanges&gt;* changes) {
    return mHwComposerAsyncWorker-&gt;send(
            [&amp;, changes]() { return chooseCompositionStrategy(changes); });
}
```

调用mHwComposerAsyncWorker(HwcAsyncWorker)的send方法，获取合成策略：

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngin/src/HwcAsyncWorker.cpp
std::future&lt;bool&gt; HwcAsyncWorker::send(std::function&lt;bool()&gt; task) {
    std::unique_lock&lt;std::mutex&gt; lock(mMutex);
    android::base::ScopedLockAssertion assumeLock(mMutex);
    mTask = std::packaged_task&lt;bool()&gt;([task = std::move(task)]() { return task(); });
    mTaskRequested = true;
    mCv.notify_one();
    return mTask.get_future();
}
```

返回chooseCompositionStrategy：

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngin/src/Display.cpp
bool Display::chooseCompositionStrategy(
        std::optional&lt;android::HWComposer::DeviceRequestedChanges&gt;* outChanges) {
    ATRACE_CALL();
    ALOGV(__FUNCTION__);
 
 
    if (mIsDisconnected) {
        return false;
    }
 
 
    // If we don&#39;t have a HWC display, then we are done.
    const auto halDisplayId = HalDisplayId::tryCast(mId);
    if (!halDisplayId) {
        return false;
    }
 
 
    // Get any composition changes requested by the HWC device, and apply them.
    std::optional&lt;android::HWComposer::DeviceRequestedChanges&gt; changes;
    auto&amp; hwc = getCompositionEngine().getHwComposer();
    if (status_t result =
                hwc.getDeviceCompositionChanges(*halDisplayId, anyLayersRequireClientComposition(),
                                                getState().earliestPresentTime,
                                                getState().previousPresentFence,
                                                getState().expectedPresentTime, outChanges);
        result != NO_ERROR) {
        ALOGE(&#34;chooseCompositionStrategy failed for %s: %d (%s)&#34;, getName().c_str(), result,
              strerror(-result));
        return false;
    }
 
 
    return true;
}
```

##### HWComposer getDeviceCompositionChanges

调用hwc(HWComposer)的getDeviceCompositionChanges方法，向HWC获取合成策略：

```cpp
//frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp
status_t HWComposer::getDeviceCompositionChanges(
        HalDisplayId displayId, bool frameUsesClientComposition,
        std::chrono::steady_clock::time_point earliestPresentTime,
        const std::shared_ptr&lt;FenceTime&gt;&amp; previousPresentFence, nsecs_t expectedPresentTime,
        std::optional&lt;android::HWComposer::DeviceRequestedChanges&gt;* outChanges) {
    ATRACE_CALL();
 
 
    RETURN_IF_INVALID_DISPLAY(displayId, BAD_INDEX);
 
 
    auto&amp; displayData = mDisplayData[displayId];
    auto&amp; hwcDisplay = displayData.hwcDisplay;
    if (!hwcDisplay-&gt;isConnected()) {
        return NO_ERROR;
    }
 
 
    uint32_t numTypes = 0;
    uint32_t numRequests = 0;
 
 
    hal::Error error = hal::Error::NONE;
 
 
    // First try to skip validate altogether. We can do that when
    // 1. The previous frame has not been presented yet or already passed the
    // earliest time to present. Otherwise, we may present a frame too early.
    // 2. There is no client composition. Otherwise, we first need to render the
    // client target buffer.
    const bool canSkipValidate = [&amp;] {
        // We must call validate if we have client composition
        if (frameUsesClientComposition) {
            return false;
        }
 
 
        // If composer supports getting the expected present time, we can skip
        // as composer will make sure to prevent early presentation
        if (mComposer-&gt;isSupported(Hwc2::Composer::OptionalFeature::ExpectedPresentTime)) {
            return true;
        }
 
 
        // composer doesn&#39;t support getting the expected present time. We can only
        // skip validate if we know that we are not going to present early.
        return std::chrono::steady_clock::now() &gt;= earliestPresentTime ||
                previousPresentFence-&gt;getSignalTime() == Fence::SIGNAL_TIME_PENDING;
    }();
 
 
    displayData.validateWasSkipped = false;
    if (canSkipValidate) {
        sp&lt;Fence&gt; outPresentFence;
        uint32_t state = UINT32_MAX;
        error = hwcDisplay-&gt;presentOrValidate(expectedPresentTime, &amp;numTypes, &amp;numRequests,
                                              &amp;outPresentFence, &amp;state);
        if (!hasChangesError(error)) {
            RETURN_IF_HWC_ERROR_FOR(&#34;presentOrValidate&#34;, error, displayId, UNKNOWN_ERROR);
        }
        if (state == 1) { //Present Succeeded.
            std::unordered_map&lt;HWC2::Layer*, sp&lt;Fence&gt;&gt; releaseFences;
            error = hwcDisplay-&gt;getReleaseFences(&amp;releaseFences);
            displayData.releaseFences = std::move(releaseFences);
            displayData.lastPresentFence = outPresentFence;
            displayData.validateWasSkipped = true;
            displayData.presentError = error;
            return NO_ERROR;
        }
        // Present failed but Validate ran.
    } else {
        error = hwcDisplay-&gt;validate(expectedPresentTime, &amp;numTypes, &amp;numRequests);
    }
    ALOGV(&#34;SkipValidate failed, Falling back to SLOW validate/present&#34;);
    if (!hasChangesError(error)) {
        RETURN_IF_HWC_ERROR_FOR(&#34;validate&#34;, error, displayId, BAD_INDEX);
    }
 
 
    android::HWComposer::DeviceRequestedChanges::ChangedTypes changedTypes;
    changedTypes.reserve(numTypes);
    error = hwcDisplay-&gt;getChangedCompositionTypes(&amp;changedTypes);
    RETURN_IF_HWC_ERROR_FOR(&#34;getChangedCompositionTypes&#34;, error, displayId, BAD_INDEX);
 
 
    auto displayRequests = static_cast&lt;hal::DisplayRequest&gt;(0);
    android::HWComposer::DeviceRequestedChanges::LayerRequests layerRequests;
    layerRequests.reserve(numRequests);
    error = hwcDisplay-&gt;getRequests(&amp;displayRequests, &amp;layerRequests);
    RETURN_IF_HWC_ERROR_FOR(&#34;getRequests&#34;, error, displayId, BAD_INDEX);
 
 
    DeviceRequestedChanges::ClientTargetProperty clientTargetProperty;
    error = hwcDisplay-&gt;getClientTargetProperty(&amp;clientTargetProperty);
 
 
    outChanges-&gt;emplace(DeviceRequestedChanges{std::move(changedTypes), std::move(displayRequests),
                                               std::move(layerRequests),
                                               std::move(clientTargetProperty)});
    error = hwcDisplay-&gt;acceptChanges();
    RETURN_IF_HWC_ERROR_FOR(&#34;acceptChanges&#34;, error, displayId, BAD_INDEX);
 
 
    return NO_ERROR;
}
```

### Output devOptRepaintFlash

调用Output的devOptRepaintFlash方法，处理显示输出设备的可选重绘闪烁：

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngin/src/Output.cpp
void Output::devOptRepaintFlash(const compositionengine::CompositionRefreshArgs&amp; refreshArgs) {
    if (CC_LIKELY(!refreshArgs.devOptFlashDirtyRegionsDelay)) {
        return;
    }
 
 
    if (getState().isEnabled) {
        if (const auto dirtyRegion = getDirtyRegion(); !dirtyRegion.isEmpty()) {
            base::unique_fd bufferFence;
            std::shared_ptr&lt;renderengine::ExternalTexture&gt; buffer;
            updateProtectedContentState();
            dequeueRenderBuffer(&amp;bufferFence, &amp;buffer);
            static_cast&lt;void&gt;(composeSurfaces(dirtyRegion, refreshArgs, buffer, bufferFence));
            mRenderSurface-&gt;queueBuffer(base::unique_fd());
        }
    }
 
 
    postFramebuffer();
 
 
    std::this_thread::sleep_for(*refreshArgs.devOptFlashDirtyRegionsDelay);
 
 
    prepareFrame();
}
```

#### RenderSurface queueBuffer

调用mRenderSurface(RenderSurface)的queueBuffer方法：

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngin/src/RenderSurface.cpp
void RenderSurface::queueBuffer(base::unique_fd readyFence) {
    auto&amp; state = mDisplay.getState();
 
 
    if (state.usesClientComposition || state.flipClientTarget) {
        // hasFlipClientTargetRequest could return true even if we haven&#39;t
        // dequeued a buffer before. Try dequeueing one if we don&#39;t have a
        // buffer ready.
        if (mTexture == nullptr) {
            ALOGI(&#34;Attempting to queue a client composited buffer without one &#34;
                  &#34;previously dequeued for display [%s]. Attempting to dequeue &#34;
                  &#34;a scratch buffer now&#34;,
                  mDisplay.getName().c_str());
            // We shouldn&#39;t deadlock here, since mTexture == nullptr only
            // after a successful call to queueBuffer, or if dequeueBuffer has
            // never been called.
            base::unique_fd unused;
            dequeueBuffer(&amp;unused);
        }
 
 
        if (mTexture == nullptr) {
            ALOGE(&#34;No buffer is ready for display [%s]&#34;, mDisplay.getName().c_str());
        } else {
            status_t result = mNativeWindow-&gt;queueBuffer(mNativeWindow.get(),
                                                         mTexture-&gt;getBuffer()-&gt;getNativeBuffer(),
                                                         dup(readyFence));
            if (result != NO_ERROR) {
                ALOGE(&#34;Error when queueing buffer for display [%s]: %d&#34;, mDisplay.getName().c_str(),
                      result);
                // We risk blocking on dequeueBuffer if the primary display failed
                // to queue up its buffer, so crash here.
                if (!mDisplay.isVirtual()) {
                    LOG_ALWAYS_FATAL(&#34;ANativeWindow::queueBuffer failed with error: %d&#34;, result);
                } else {
                    mNativeWindow-&gt;cancelBuffer(mNativeWindow.get(),
                                                mTexture-&gt;getBuffer()-&gt;getNativeBuffer(),
                                                dup(readyFence));
                }
            }
 
 
            mTexture = nullptr;
        }
    }
 
 
    status_t result = mDisplaySurface-&gt;advanceFrame();
    if (result != NO_ERROR) {
        ALOGE(&#34;[%s] failed pushing new frame to HWC: %d&#34;, mDisplay.getName().c_str(), result);
    }
}
```

##### FramebufferSurface advanceFrame

调用mDisplaySurface(DisplaySurface)的advanceFrame方法，FramebufferSurface继承与DisplaySurface，调用FramebufferSurface的advanceFrame方法，FramebufferSurface的advanceFrame方法用于更新帧并将其显示在屏幕上

```cpp
//frameworks/native/services/surfaceflinger/DisplayHardware/FramebufferSurface.cpp
status_t FramebufferSurface::advanceFrame() {
    uint32_t slot = 0;
    sp&lt;GraphicBuffer&gt; buf;
    sp&lt;Fence&gt; acquireFence(Fence::NO_FENCE);
    Dataspace dataspace = Dataspace::UNKNOWN;
    status_t result = nextBuffer(slot, buf, acquireFence, dataspace);
    mDataSpace = dataspace;
    if (result != NO_ERROR) {
        ALOGE(&#34;error latching next FramebufferSurface buffer: %s (%d)&#34;,
                strerror(-result), result);
    }
    return result;
}
```

调用FramebufferSurface的nextBuffer方法：

```cpp
//frameworks/native/services/surfaceflinger/DisplayHardware/FramebufferSurface.cpp
status_t FramebufferSurface::nextBuffer(uint32_t&amp; outSlot,
        sp&lt;GraphicBuffer&gt;&amp; outBuffer, sp&lt;Fence&gt;&amp; outFence,
        Dataspace&amp; outDataspace) {
    Mutex::Autolock lock(mMutex);
 
 
    BufferItem item;
    status_t err = acquireBufferLocked(&amp;item, 0);
    if (err == BufferQueue::NO_BUFFER_AVAILABLE) {
        mHwcBufferCache.getHwcBuffer(mCurrentBufferSlot, mCurrentBuffer, &amp;outSlot, &amp;outBuffer);
        return NO_ERROR;
    } else if (err != NO_ERROR) {
        ALOGE(&#34;error acquiring buffer: %s (%d)&#34;, strerror(-err), err);
        return err;
    }
 
 
    // If the BufferQueue has freed and reallocated a buffer in mCurrentSlot
    // then we may have acquired the slot we already own.  If we had released
    // our current buffer before we call acquireBuffer then that release call
    // would have returned STALE_BUFFER_SLOT, and we would have called
    // freeBufferLocked on that slot.  Because the buffer slot has already
    // been overwritten with the new buffer all we have to do is skip the
    // releaseBuffer call and we should be in the same state we&#39;d be in if we
    // had released the old buffer first.
    if (mCurrentBufferSlot != BufferQueue::INVALID_BUFFER_SLOT &amp;&amp;
        item.mSlot != mCurrentBufferSlot) {
        mHasPendingRelease = true;
        mPreviousBufferSlot = mCurrentBufferSlot;
        mPreviousBuffer = mCurrentBuffer;
    }
    mCurrentBufferSlot = item.mSlot;
    mCurrentBuffer = mSlots[mCurrentBufferSlot].mGraphicBuffer;
    mCurrentFence = item.mFence;
 
 
    outFence = item.mFence;
    mHwcBufferCache.getHwcBuffer(mCurrentBufferSlot, mCurrentBuffer, &amp;outSlot, &amp;outBuffer);
    outDataspace = static_cast&lt;Dataspace&gt;(item.mDataSpace);
    status_t result = mHwc.setClientTarget(mDisplayId, outSlot, outFence, outBuffer, outDataspace);
    if (result != NO_ERROR) {
        ALOGE(&#34;error posting framebuffer: %d&#34;, result);
        return result;
    }
 
 
    return NO_ERROR;
}
```

###### HWComposer setClientTarget

调用mHwc(HWComposer)的setClientTarget方法：

```cpp
//frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp
std::unordered_map&lt;HalDisplayId, DisplayData&gt; mDisplayData;
std::unique_ptr&lt;HWC2::Display&gt; hwcDisplay;
status_t HWComposer::setClientTarget(HalDisplayId displayId, uint32_t slot,
                                     const sp&lt;Fence&gt;&amp; acquireFence, const sp&lt;GraphicBuffer&gt;&amp; target,
                                     ui::Dataspace dataspace) {
    RETURN_IF_INVALID_DISPLAY(displayId, BAD_INDEX);
 
 
    ALOGV(&#34;%s for display %s&#34;, __FUNCTION__, to_string(displayId).c_str());
    auto&amp; hwcDisplay = mDisplayData[displayId].hwcDisplay;
    auto error = hwcDisplay-&gt;setClientTarget(slot, target, acquireFence, dataspace);
    RETURN_IF_HWC_ERROR(error, displayId, BAD_VALUE);
    return NO_ERROR;
}
```

###### HWC2::Display setOutputBuffer

调用HWC2::Display的setClientTarget方法，之后就是HWC的处理了。

### Output finishFrame

调用Output的finishFrame方法，完成帧：

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngin/src/Output.cpp
void Output::finishFrame(const CompositionRefreshArgs&amp; refreshArgs, GpuCompositionResult&amp;&amp; result) {
    ATRACE_CALL();
    ALOGV(__FUNCTION__);
    const auto&amp; outputState = getState();
    if (!outputState.isEnabled) {
        return;
    }
 
 
    std::optional&lt;base::unique_fd&gt; optReadyFence;
    std::shared_ptr&lt;renderengine::ExternalTexture&gt; buffer;
    base::unique_fd bufferFence;
    if (outputState.strategyPrediction == CompositionStrategyPredictionState::SUCCESS) {
        optReadyFence = std::move(result.fence);
    } else {
        if (result.bufferAvailable()) {
            buffer = std::move(result.buffer);
            bufferFence = std::move(result.fence);
        } else {
            updateProtectedContentState();
            if (!dequeueRenderBuffer(&amp;bufferFence, &amp;buffer)) {
                return;
            }
        }
        // Repaint the framebuffer (if needed), getting the optional fence for when
        // the composition completes.
        optReadyFence = composeSurfaces(Region::INVALID_REGION, refreshArgs, buffer, bufferFence);
    }
    if (!optReadyFence) {
        return;
    }
 
 
    // swap buffers (presentation)
    mRenderSurface-&gt;queueBuffer(std::move(*optReadyFence));
}
```

### Output postFramebuffer

调用Output的postFramebuffer方法，将帧缓冲区（framebuffer）的内容发送到显示设备进行显示：

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngin/src/Output.cpp
void Output::postFramebuffer() {
    ATRACE_CALL();
    ALOGV(__FUNCTION__);
 
 
    if (!getState().isEnabled) {
        return;
    }
 
 
    auto&amp; outputState = editState();
    outputState.dirtyRegion.clear();
    mRenderSurface-&gt;flip();
 
 
    auto frame = presentAndGetFrameFences();
 
 
    mRenderSurface-&gt;onPresentDisplayCompleted();
 
 
    for (auto* layer : getOutputLayersOrderedByZ()) {
        // The layer buffer from the previous frame (if any) is released
        // by HWC only when the release fence from this frame (if any) is
        // signaled.  Always get the release fence from HWC first.
        sp&lt;Fence&gt; releaseFence = Fence::NO_FENCE;
 
 
        if (auto hwcLayer = layer-&gt;getHwcLayer()) {
            if (auto f = frame.layerFences.find(hwcLayer); f != frame.layerFences.end()) {
                releaseFence = f-&gt;second;
            }
        }
 
 
        // If the layer was client composited in the previous frame, we
        // need to merge with the previous client target acquire fence.
        // Since we do not track that, always merge with the current
        // client target acquire fence when it is available, even though
        // this is suboptimal.
        // TODO(b/121291683): Track previous frame client target acquire fence.
        if (outputState.usesClientComposition) {
            releaseFence =
                    Fence::merge(&#34;LayerRelease&#34;, releaseFence, frame.clientTargetAcquireFence);
        }
        layer-&gt;getLayerFE().onLayerDisplayed(
                ftl::yield&lt;FenceResult&gt;(std::move(releaseFence)).share());
    }
 
 
    // We&#39;ve got a list of layers needing fences, that are disjoint with
    // OutputLayersOrderedByZ.  The best we can do is to
    // supply them with the present fence.
    for (auto&amp; weakLayer : mReleasedLayers) {
        if (const auto layer = weakLayer.promote()) {
            layer-&gt;onLayerDisplayed(ftl::yield&lt;FenceResult&gt;(frame.presentFence).share());
        }
    }
 
 
    // Clear out the released layers now that we&#39;re done with them.
    mReleasedLayers.clear();
}
```

上面方法主要处理如下：

1、调用Output的presentAndGetFrameFences方法，通过presentAndGetReleaseFences显示获取releaseFences。

2、调用BufferQueueLayer的onLayerDisplayed方法。

下面分别进行分析。

#### Output presentAndGetFrameFences

调用Output的presentAndGetFrameFences方法：

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngin/src/Output.cpp
compositionengine::Output::FrameFences Output::presentAndGetFrameFences() {
    compositionengine::Output::FrameFences result;
    if (getState().usesClientComposition) {
        result.clientTargetAcquireFence = mRenderSurface-&gt;getClientTargetAcquireFence();
    }
    return result;
}
```

调用mRenderSurface(RenderSurface)的getClientTargetAcquireFence方法：

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngin/src/RenderSurface.cpp
const sp&lt;DisplaySurface&gt; mDisplaySurface;
const sp&lt;Fence&gt;&amp; RenderSurface::getClientTargetAcquireFence() const {
    return mDisplaySurface-&gt;getClientTargetAcquireFence();
}
```

##### FramebufferSurface getClientTargetAcquireFence

调用mDisplaySurface(DisplaySurface)的prepareFrame方法，FramebufferSurface 继承于DisplaySurface，因此调用FramebufferSurface的getClientTargetAcquireFence方法：

```cpp
//frameworks/native/services/surfaceflinger/DisplayHardware/FramebufferSurface.cpp
const sp&lt;Fence&gt;&amp; FramebufferSurface::getClientTargetAcquireFence() const {
    return mCurrentFence;
}
```

#### BufferQueueLayer onLayerDisplayed

调用Layer的onLayerDisplayed方法，BufferQueueLayer继承于BuffeLayer，BuffeLayer由继承于Layer，因此调用BufferQueueLayer的onLayerDisplayed方法：

[Android13 BufferQueueLayer onLayerDisplayed流程分析-CSDN博客](/Android13%20BufferQueueLayer%20onLayerDisplayed%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-CSDN%E5%8D%9A%E5%AE%A2)

### Output renderCachedSets

调用Output的renderCachedSets方法，进行渲染缓存设置：

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngin/src/Output.cpp
void Output::renderCachedSets(const CompositionRefreshArgs&amp; refreshArgs) {
    if (mPlanner) {
        mPlanner-&gt;renderCachedSets(getState(), refreshArgs.scheduledFrameTime);
    }
}
```

### Output dirtyEntireOutput

调用Output的dirtyEntireOutput方法，重置脏区：

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngin/src/Output.cpp
OutputCompositionState outputState ;
Region dirtyRegion;
using OutputCompositionState = compositionengine::impl::OutputCompositionState;
void Output::dirtyEntireOutput() {
    auto&amp; outputState = editState();
    outputState.dirtyRegion.set(outputState.displaySpace.getBoundsAsRect());
}
```

#### Region set

调用outputState(OutputCompositionState )内成员变量dirtyRegion(Region)的set方法，设置脏区：

```cpp
//frameworks/native/libs/ui/Region.cpp
FatVector&lt;Rect&gt; mStorage;
void Region::set(const Rect&amp; r)
{
    mStorage.clear();
    mStorage.push_back(r);
}
```

### Output finishPrepareFrame

调用Output的finishPrepareFrame方法：

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngin/src/Output.cpp
void Output::finishPrepareFrame() {
    const auto&amp; state = getState();
    if (mPlanner) {
        mPlanner-&gt;reportFinalPlan(getOutputLayersOrderedByZ());
    }
    mRenderSurface-&gt;prepareFrame(state.usesClientComposition, state.usesDeviceComposition);
}
```

#### Planner reportFinalPlan

调用mPlanner(Planner)的reportFinalPlan方法：

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngin/src/Planner.cpp
void Planner::reportFinalPlan(
        compositionengine::Output::OutputLayersEnumerator&lt;compositionengine::Output&gt;&amp;&amp; layers) {
    ATRACE_CALL();
    if (!mPredictorEnabled) {
        return;
    }
 
 
    Plan finalPlan;
    const GraphicBuffer* currentOverrideBuffer = nullptr;
    bool hasSkippedLayers = false;
    for (auto layer : layers) {
        if (!layer-&gt;getState().overrideInfo.buffer) {
            continue;
        }
 
 
        const GraphicBuffer* overrideBuffer =
                layer-&gt;getState().overrideInfo.buffer-&gt;getBuffer().get();
        if (overrideBuffer != nullptr &amp;&amp; overrideBuffer == currentOverrideBuffer) {
            // Skip this layer since it is part of a previous cached set
            hasSkippedLayers = true;
            continue;
        }
 
 
        currentOverrideBuffer = overrideBuffer;
 
 
        const bool forcedOrRequestedClient =
                layer-&gt;getState().forceClientComposition || layer-&gt;requiresClientComposition();
 
 
        finalPlan.addLayerType(
                forcedOrRequestedClient
                        ? aidl::android::hardware::graphics::composer3::Composition::CLIENT
                        : layer-&gt;getLayerFE().getCompositionState()-&gt;compositionType);
    }
 
 
    mPredictor.recordResult(mPredictedPlan, mFlattenedHash, mCurrentLayers, hasSkippedLayers,
                            finalPlan);
}
```

调用mPredictor(Predictor)的recordResult方法：

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngin/src/Predictor.cpp
void Predictor::recordResult(std::optional&lt;PredictedPlan&gt; predictedPlan,
                             NonBufferHash flattenedHash,
                             const std::vector&lt;const LayerState*&gt;&amp; layers, bool hasSkippedLayers,
                             Plan result) {
    if (predictedPlan) {
        recordPredictedResult(*predictedPlan, layers, std::move(result));
        return;
    }
 
 
    &#43;&#43;mMissCount;
 
 
    if (!hasSkippedLayers &amp;&amp; findSimilarPrediction(layers, result)) {
        return;
    }
 
 
    ALOGV(&#34;[%s] Adding novel candidate %zx&#34;, __func__, flattenedHash);
    mCandidates.emplace_front(flattenedHash, Prediction(layers, result));
    if (mCandidates.size() &gt; MAX_CANDIDATES) {
        mCandidates.pop_back();
    }
}
```

#### RenderSurface prepareFrame

调用RenderSurface的prepareFrame方法：

```cpp
//frameworks/native/services/surfaceflinger/CompositionEngin/src/RenderSurface.cpp
const sp&lt;DisplaySurface&gt; mDisplaySurface;
void RenderSurface::prepareFrame(bool usesClientComposition, bool usesDeviceComposition) {
    const auto compositionType = [=] {
        using CompositionType = DisplaySurface::CompositionType;
 
 
        if (usesClientComposition &amp;&amp; usesDeviceComposition) return CompositionType::Mixed;
        if (usesClientComposition) return CompositionType::Gpu;
        if (usesDeviceComposition) return CompositionType::Hwc;
 
 
        // Nothing to do -- when turning the screen off we get a frame like
        // this. Call it a HWC frame since we won&#39;t be doing any GPU work but
        // will do a prepare/set cycle.
        return CompositionType::Hwc;
    }();
 
 
    if (status_t result = mDisplaySurface-&gt;prepareFrame(compositionType); result != NO_ERROR) {
        ALOGE(&#34;updateCompositionType failed for %s: %d (%s)&#34;, mDisplay.getName().c_str(), result,
              strerror(-result));
    }
}
```

##### VirtualDisplaySurface prepareFrame

调用mDisplaySurface(DisplaySurface)的prepareFrame方法，VirtualDisplaySurface继承于DisplaySurface，因此调用DisplaySurface的prepareFrame方法，DisplaySurface的prepareFrame方法用于准备帧

```cpp
//frameworks/native/services/surfaceflinger/DisplayHardware/VirtualDisplaySurface.cpp
status_t VirtualDisplaySurface::prepareFrame(CompositionType compositionType) {
    if (GpuVirtualDisplayId::tryCast(mDisplayId)) {
        return NO_ERROR;
    }
 
 
    VDS_LOGW_IF(mDebugState != DebugState::Begun, &#34;Unexpected %s in %s state&#34;, __func__,
                ftl::enum_string(mDebugState).c_str());
    mDebugState = DebugState::Prepared;
 
 
    mCompositionType = compositionType;
    if (mForceHwcCopy &amp;&amp; mCompositionType == CompositionType::Gpu) {
        // Some hardware can do RGB-&gt;YUV conversion more efficiently in hardware
        // controlled by HWC than in hardware controlled by the video encoder.
        // Forcing GPU-composed frames to go through an extra copy by the HWC
        // allows the format conversion to happen there, rather than passing RGB
        // directly to the consumer.
        //
        // On the other hand, when the consumer prefers RGB or can consume RGB
        // inexpensively, this forces an unnecessary copy.
        mCompositionType = CompositionType::Mixed;
    }
 
 
    if (mCompositionType != mDebugLastCompositionType) {
        VDS_LOGV(&#34;%s: composition type changed to %s&#34;, __func__,
                 toString(mCompositionType).c_str());
        mDebugLastCompositionType = mCompositionType;
    }
 
 
    if (mCompositionType != CompositionType::Gpu &amp;&amp;
        (mOutputFormat != mDefaultOutputFormat || mOutputUsage != GRALLOC_USAGE_HW_COMPOSER)) {
        // We must have just switched from GPU-only to MIXED or HWC
        // composition. Stop using the format and usage requested by the GPU
        // driver; they may be suboptimal when HWC is writing to the output
        // buffer. For example, if the output is going to a video encoder, and
        // HWC can write directly to YUV, some hardware can skip a
        // memory-to-memory RGB-to-YUV conversion step.
        //
        // If we just switched *to* GPU-only mode, we&#39;ll change the
        // format/usage and get a new buffer when the GPU driver calls
        // dequeueBuffer().
        mOutputFormat = mDefaultOutputFormat;
        mOutputUsage = GRALLOC_USAGE_HW_COMPOSER;
        refreshOutputBuffer();
    }
 
 
    return NO_ERROR;
}
```

调用VirtualDisplaySurface的refreshOutputBuffer方法：

```cpp
//frameworks/native/services/surfaceflinger/DisplayHardware/VirtualDisplaySurface.cpp
status_t VirtualDisplaySurface::refreshOutputBuffer() {
    LOG_ALWAYS_FATAL_IF(GpuVirtualDisplayId::tryCast(mDisplayId).has_value());
 
 
    if (mOutputProducerSlot &gt;= 0) {
        mSource[SOURCE_SINK]-&gt;cancelBuffer(
                mapProducer2SourceSlot(SOURCE_SINK, mOutputProducerSlot),
                mOutputFence);
    }
 
 
    int sslot;
    status_t result = dequeueBuffer(SOURCE_SINK, mOutputFormat, mOutputUsage,
            &amp;sslot, &amp;mOutputFence);
    if (result &lt; 0)
        return result;
    mOutputProducerSlot = mapSource2ProducerSlot(SOURCE_SINK, sslot);
 
 
    // On GPU-only frames, we don&#39;t have the right output buffer acquire fence
    // until after GPU calls queueBuffer(). So here we just set the buffer
    // (for use in HWC prepare) but not the fence; we&#39;ll call this again with
    // the proper fence once we have it.
    const auto halDisplayId = HalVirtualDisplayId::tryCast(mDisplayId);
    LOG_FATAL_IF(!halDisplayId);
    result = mHwc.setOutputBuffer(*halDisplayId, Fence::NO_FENCE,
                                  mProducerBuffers[mOutputProducerSlot]);
 
 
    return result;
}
```

###### HWComposer setOutputBuffer

调用HWComposer的setOutputBuffer方法：

```cpp
//frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp
std::unordered_map&lt;HalDisplayId, DisplayData&gt; mDisplayData;
std::unique_ptr&lt;HWC2::Display&gt; hwcDisplay;
status_t HWComposer::setOutputBuffer(HalVirtualDisplayId displayId, const sp&lt;Fence&gt;&amp; acquireFence,
                                     const sp&lt;GraphicBuffer&gt;&amp; buffer) {
    RETURN_IF_INVALID_DISPLAY(displayId, BAD_INDEX);
    const auto&amp; displayData = mDisplayData[displayId];
 
 
    auto error = displayData.hwcDisplay-&gt;setOutputBuffer(buffer, acquireFence);
    RETURN_IF_HWC_ERROR(error, displayId, UNKNOWN_ERROR);
    return NO_ERROR;
}
```

###### HWC2::Display setOutputBuffer

调用HWC2::Display的setOutputBuffer方法，之后就是HWC的处理了。


---

> 作者: [wentao](https://github.com/huang8604)  
> URL: https://huang8604.github.io/posts/android13-surfaceflinger-composite%E5%90%88%E6%88%90%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-csdn%E5%8D%9A%E5%AE%A2/  

