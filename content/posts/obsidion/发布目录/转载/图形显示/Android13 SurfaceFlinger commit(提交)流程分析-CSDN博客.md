---
title: Android13 SurfaceFlinger commit(提交)流程分析_surfaceflinger::commit-CSDN博客
author: 
created: 2024-09-26
tags:
  - clippings
  - 转载
  - blog
collections: 图形显示
date: 2024-09-26T09:05:04.109Z
lastmod: 2024-09-27T03:07:57.960Z
---
SurfaceFlinger的commit方法用于将应用程序的绘制结果提交到屏幕上显示。

主要就是处理app端发起的一系列transaction的事务请求，需要对这些请求进行识别是否当前帧处理，处理过程就是把事务中的属性取出，然后更新到Layer中，偶尔buffer更新的还需要进行相关的latchbuffer操作，SurfaceFlinger的commit代码如下：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
bool SurfaceFlinger::commit(nsecs_t frameTime, int64_t vsyncId, nsecs_t expectedVsyncTime) //这里frameTime代表当前时间，expectedVsyncTime代表硬件vsync时间，即屏幕先的vsync时间
        FTL_FAKE_GUARD(kMainThreadContext) {
    // we set this once at the beginning of commit to ensure consistency throughout the whole frame
    mPowerHintSessionData.sessionEnabled = mPowerAdvisor->usePowerHintSession();
    if (mPowerHintSessionData.sessionEnabled) {
        mPowerHintSessionData.commitStart = systemTime();
    }
 
 
    // calculate the expected present time once and use the cached
    // value throughout this frame to make sure all layers are
    // seeing this same value.
    if (expectedVsyncTime >= frameTime) {
        mExpectedPresentTime = expectedVsyncTime;
    } else {
        const DisplayStatInfo stats = mScheduler->getDisplayStatInfo(frameTime);
        mExpectedPresentTime = calculateExpectedPresentTime(stats);
    }
 
 
    const nsecs_t lastScheduledPresentTime = mScheduledPresentTime;
    mScheduledPresentTime = expectedVsyncTime;
 
 
    if (mPowerHintSessionData.sessionEnabled) {
        mPowerAdvisor->setTargetWorkDuration(mExpectedPresentTime -
                                             mPowerHintSessionData.commitStart);
    }
    const auto vsyncIn = [&] {
        if (!ATRACE_ENABLED()) return 0.f;
        return (mExpectedPresentTime - systemTime()) / 1e6f;
    }();
    ATRACE_FORMAT("%s %" PRId64 " vsyncIn %.2fms%s", __func__, vsyncId, vsyncIn,
                  mExpectedPresentTime == expectedVsyncTime ? "" : " (adjusted)");
 
 
    // When Backpressure propagation is enabled we want to give a small grace period
    // for the present fence to fire instead of just giving up on this frame to handle cases
    // where present fence is just about to get signaled.
    const int graceTimeForPresentFenceMs =
            (mPropagateBackpressureClientComposition || !mHadClientComposition) ? 1 : 0;
 
 
    // Pending frames may trigger backpressure propagation.
    const TracedOrdinal<bool> framePending = {"PrevFramePending",
                                              previousFramePending(graceTimeForPresentFenceMs)};
 
 
    // Frame missed counts for metrics tracking.
    // A frame is missed if the prior frame is still pending. If no longer pending,
    // then we still count the frame as missed if the predicted present time
    // was further in the past than when the fence actually fired.
 
 
    // Add some slop to correct for drift. This should generally be
    // smaller than a typical frame duration, but should not be so small
    // that it reports reasonable drift as a missed frame.
    const DisplayStatInfo stats = mScheduler->getDisplayStatInfo(systemTime());
    const nsecs_t frameMissedSlop = stats.vsyncPeriod / 2;
    const nsecs_t previousPresentTime = previousFramePresentTime();
    const TracedOrdinal<bool> frameMissed = {"PrevFrameMissed",
                                             framePending ||
                                                     (previousPresentTime >= 0 &&
                                                      (lastScheduledPresentTime <
                                                       previousPresentTime - frameMissedSlop))};
    const TracedOrdinal<bool> hwcFrameMissed = {"PrevHwcFrameMissed",
                                                mHadDeviceComposition && frameMissed};
    const TracedOrdinal<bool> gpuFrameMissed = {"PrevGpuFrameMissed",
                                                mHadClientComposition && frameMissed};
 
 
    if (frameMissed) {
        mFrameMissedCount++;
        mTimeStats->incrementMissedFrames();
    }
 
 
    if (hwcFrameMissed) {
        mHwcFrameMissedCount++;
    }
 
 
    if (gpuFrameMissed) {
        mGpuFrameMissedCount++;
    }
 
 
    // If we are in the middle of a mode change and the fence hasn't
    // fired yet just wait for the next commit.
    // 如果我们正处于模式更改的过程中，并且围栏尚未触发，请等待下一次提交。
    if (mSetActiveModePending) {
        if (framePending) {
            mScheduler->scheduleFrame();
            return false;
        }
 
 
        // We received the present fence from the HWC, so we assume it successfully updated
        // the mode, hence we update SF.
        mSetActiveModePending = false;
        {
            Mutex::Autolock lock(mStateLock);
            updateInternalStateWithChangedMode();
        }
    }
 
 
    if (framePending) {
        if ((hwcFrameMissed && !gpuFrameMissed) || mPropagateBackpressureClientComposition) {
            scheduleCommit(FrameHint::kNone);
            return false;
        }
    }
 
 
    if (mTracingEnabledChanged) {
        mLayerTracingEnabled = mLayerTracing.isEnabled();
        mTracingEnabledChanged = false;
    }
 
 
    if (mRefreshRateOverlaySpinner) {
        Mutex::Autolock lock(mStateLock);
        if (const auto display = getDefaultDisplayDeviceLocked()) {
            display->animateRefreshRateOverlay();
        }
    }
 
 
    // Composite if transactions were committed, or if requested by HWC.
    bool mustComposite = mMustComposite.exchange(false);
    {
        mFrameTimeline->setSfWakeUp(vsyncId, frameTime, Fps::fromPeriodNsecs(stats.vsyncPeriod));
 
 
        bool needsTraversal = false;
        if (clearTransactionFlags(eTransactionFlushNeeded)) { //满足eTransactionFlushNeeded条件进入
            needsTraversal |= commitCreatedLayers(); //负责新创建的layer相关业务处理
            needsTraversal |= flushTransactionQueues(vsyncId); //这里是对前面的Transaction处理的核心部分
        }
 
 
        const bool shouldCommit =
                (getTransactionFlags() & ~eTransactionFlushNeeded) || needsTraversal;
        if (shouldCommit) {
            commitTransactions(); //进行Transaction提交主要就是交换mCurrentState和DrawingState
        }
 
 
        if (transactionFlushNeeded()) { //判断是否又要启动新vsync信号
            setTransactionFlags(eTransactionFlushNeeded);
        }
 
 
        mustComposite |= shouldCommit;
        mustComposite |= latchBuffers(); //进行核心的latchBuffer
 
 
        // This has to be called after latchBuffers because we want to include the layers that have
        // been latched in the commit callback
        if (!needsTraversal) {
            // Invoke empty transaction callbacks early.
            mTransactionCallbackInvoker.sendCallbacks(false /* onCommitOnly */);
        } else {
            // Invoke OnCommit callbacks.
            mTransactionCallbackInvoker.sendCallbacks(true /* onCommitOnly */);
        }
 
 
        updateLayerGeometry(); //对要这次vsync显示刷新的layer进行脏区设置
    }
 
 
    // Layers need to get updated (in the previous line) before we can use them for
    // choosing the refresh rate.
    // Hold mStateLock as chooseRefreshRateForContent promotes wp<Layer> to sp<Layer>
    // and may eventually call to ~Layer() if it holds the last reference
    {
        Mutex::Autolock _l(mStateLock);
        mScheduler->chooseRefreshRateForContent();
        setActiveModeInHwcIfNeeded();
    }
 
 
    updateCursorAsync(); //鼠标相关layer处理
    updateInputFlinger(); //更新触摸input下的相关的window等，这里也非常关键哈，直接影响触摸是否到app
 
 
    if (mLayerTracingEnabled && !mLayerTracing.flagIsSet(LayerTracing::TRACE_COMPOSITION)) {
        // This will block and tracing should only be enabled for debugging.
        mLayerTracing.notify(mVisibleRegionsDirty, frameTime);
    }
 
 
    persistDisplayBrightness(mustComposite);
 
 
    return mustComposite && CC_LIKELY(mBootStage != BootStage::BOOTLOADER);
}
```

上面方法主要处理如下：

1、调用SurfaceFlinger的 commitCreatedLayers，创建的layer。

2、调用SurfaceFlinger的 flushTransactionQueues 方法。

3、调用SurfaceFlinger的 commitTransactions 方法，进行Transaction提交。

4、调用SurfaceFlinger的 latchBuffers 方法，将应用程序的图形缓冲区（buffer）与显示器进行关联。

5、调用SurfaceFlinger的 updateLayerGeometry 方法，这次vsync显示刷新的layer进行脏区设置。

6、调用SurfaceFlinger的 updateInputFlinger 方法，更新输入处理。

下面分别进行分析：

## SurfaceFlinger commitCreatedLayers

调用SurfaceFlinger的commitCreatedLayers，创建的layer：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
bool SurfaceFlinger::commitCreatedLayers() {
    std::vector<LayerCreatedState> createdLayers;
    {
        std::scoped_lock<std::mutex> lock(mCreatedLayersLock);
        createdLayers = std::move(mCreatedLayers);
        mCreatedLayers.clear();
        if (createdLayers.size() == 0) {
            return false;
        }
    }
 
 
    Mutex::Autolock _l(mStateLock);
    for (const auto& createdLayer : createdLayers) {
        handleLayerCreatedLocked(createdLayer);
    }
    createdLayers.clear();
    mLayersAdded = true;
    return true;
}
```

## SurfaceFlinger flushTransactionQueues

调用SurfaceFlinger的flushTransactionQueues方法：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
bool SurfaceFlinger::flushTransactionQueues(int64_t vsyncId) {
    // to prevent onHandleDestroyed from being called while the lock is held,
    // we must keep a copy of the transactions (specifically the composer
    // states) around outside the scope of the lock
    std::vector<TransactionState> transactions;
    // Layer handles that have transactions with buffers that are ready to be applied.
    std::unordered_map<sp<IBinder>, uint64_t, SpHash<IBinder>> bufferLayersReadyToPresent;
    std::unordered_set<sp<IBinder>, SpHash<IBinder>> applyTokensWithUnsignaledTransactions;
    {
        Mutex::Autolock _l(mStateLock);
        {
            Mutex::Autolock _l(mQueueLock);
 
 
            int lastTransactionsPendingBarrier = 0;
            int transactionsPendingBarrier = 0;
            // First collect transactions from the pending transaction queues.
            // We are not allowing unsignaled buffers here as we want to
            // collect all the transactions from applyTokens that are ready first.
     //刷新一下PendingTransactionQueues，这个主要是上次vsync中没有满足条件的Transaction放入的
            transactionsPendingBarrier =
                    flushPendingTransactionQueues(transactions, bufferLayersReadyToPresent,
                            applyTokensWithUnsignaledTransactions, /*tryApplyUnsignaled*/ false);
 
 
            // Second, collect transactions from the transaction queue.
            // Here as well we are not allowing unsignaled buffers for the same
            // reason as above.
     //开始遍历当次vsync的mTransactionQueue
            while (!mTransactionQueue.empty()) {
                auto& transaction = mTransactionQueue.front();
  //判断是否处于mPendingTransactionQueues队里里面
                const bool pendingTransactions =
                        mPendingTransactionQueues.find(transaction.applyToken) !=
                        mPendingTransactionQueues.end();
  //这里有个ready的判断，主要就是看看当前的Transaction是否已经ready，没有ready则不予进行传递Transaction
                const auto ready = [&]() REQUIRES(mStateLock) {
                    if (pendingTransactions) {
                        ATRACE_NAME("pendingTransactions");
                        return TransactionReadiness::NotReady;
                    }
 
 
                    return transactionIsReadyToBeApplied(transaction, transaction.frameTimelineInfo,
                                                         transaction.isAutoTimestamp,
                                                         transaction.desiredPresentTime,
                                                         transaction.originUid, transaction.states,
                                                         bufferLayersReadyToPresent,
                                                         transactions.size(),
                                                         /*tryApplyUnsignaled*/ false); //检查当前的事务是否准备好被应用
                }();
                ATRACE_INT("TransactionReadiness", static_cast<int>(ready));
                if (ready != TransactionReadiness::Ready) {
                    if (ready == TransactionReadiness::NotReadyBarrier) {
                        transactionsPendingBarrier++;
                    }
      //放入mPendingTransactionQueues
                    mPendingTransactionQueues[transaction.applyToken].push(std::move(transaction));
                } else {
      //已经ready则进入如下的操作
                    transaction.traverseStatesWithBuffers([&](const layer_state_t& state) {
                        const bool frameNumberChanged = state.bufferData->flags.test(
                                BufferData::BufferDataChange::frameNumberChanged);
  //会把state放入到bufferLayersReadyToPresent这个map中
                        if (frameNumberChanged) {
                            bufferLayersReadyToPresent[state.surface] = state.bufferData->frameNumber;
                        } else {
                            // Barrier function only used for BBQ which always includes a frame number.
                            // This value only used for barrier logic.
                            bufferLayersReadyToPresent[state.surface] =
                                std::numeric_limits<uint64_t>::max();
                        }
                    });
      //最重要的放入到transactions
                    transactions.emplace_back(std::move(transaction));
                }
  //从mTransactionQueue这里移除
                mTransactionQueue.pop_front();
                ATRACE_INT("TransactionQueue", mTransactionQueue.size());
            }
 
 
            // Transactions with a buffer pending on a barrier may be on a different applyToken
            // than the transaction which satisfies our barrier. In fact this is the exact use case
            // that the primitive is designed for. This means we may first process
            // the barrier dependent transaction, determine it ineligible to complete
            // and then satisfy in a later inner iteration of flushPendingTransactionQueues.
            // The barrier dependent transaction was eligible to be presented in this frame
            // but we would have prevented it without case. To fix this we continually
            // loop through flushPendingTransactionQueues until we perform an iteration
            // where the number of transactionsPendingBarrier doesn't change. This way
            // we can continue to resolve dependency chains of barriers as far as possible.
            while (lastTransactionsPendingBarrier != transactionsPendingBarrier) {
                lastTransactionsPendingBarrier = transactionsPendingBarrier;
                transactionsPendingBarrier =
                    flushPendingTransactionQueues(transactions, bufferLayersReadyToPresent,
                        applyTokensWithUnsignaledTransactions,
                        /*tryApplyUnsignaled*/ false);
            }
 
 
            // We collected all transactions that could apply without latching unsignaled buffers.
            // If we are allowing latch unsignaled of some form, now it's the time to go over the
            // transactions that were not applied and try to apply them unsignaled.
     //根据enableLatchUnsignaledConfig属性，不是disabled的话需要对前面的notready情况进行第二次的校验放过
            if (enableLatchUnsignaledConfig != LatchUnsignaledConfig::Disabled) {
                flushUnsignaledPendingTransactionQueues(transactions, bufferLayersReadyToPresent,
                                                        applyTokensWithUnsignaledTransactions);
            }
     //进行关键的applyTransactions操作
            return applyTransactions(transactions, vsyncId);
        }
    }
}
```

上面方法的主要处理如下：

1、调用SurfaceFlinger的transactionIsReadyToBeApplied方法，检查当前的事务是否准备好被应用

2、调用SurfaceFlinger的applyTransactions方法，应用事务。

下面分别进行分析：

### SurfaceFlinger transactionIsReadyToBeApplied

调用SurfaceFlinger的transactionIsReadyToBeApplied方法，检查当前的事务是否准备好被应用：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
auto SurfaceFlinger::transactionIsReadyToBeApplied(TransactionState& transaction,
        const FrameTimelineInfo& info, bool isAutoTimestamp, int64_t desiredPresentTime,
        uid_t originUid, const Vector<ComposerState>& states,
        const std::unordered_map<
            sp<IBinder>, uint64_t, SpHash<IBinder>>& bufferLayersReadyToPresent,
        size_t totalTXapplied, bool tryApplyUnsignaled) const -> TransactionReadiness {
    ATRACE_FORMAT("transactionIsReadyToBeApplied vsyncId: %" PRId64, info.vsyncId);
    const nsecs_t expectedPresentTime = mExpectedPresentTime.load();
    // Do not present if the desiredPresentTime has not passed unless it is more than one second
    // in the future. We ignore timestamps more than 1 second in the future for stability reasons.
    if (!isAutoTimestamp && desiredPresentTime >= expectedPresentTime &&
        desiredPresentTime < expectedPresentTime + s2ns(1)) {
        ATRACE_NAME("not current");
        return TransactionReadiness::NotReady;
    }
 
 
    if (!mScheduler->isVsyncValid(expectedPresentTime, originUid)) {
        ATRACE_NAME("!isVsyncValid");
        return TransactionReadiness::NotReady;
    }
 
 
    // If the client didn't specify desiredPresentTime, use the vsyncId to determine the expected
    // present time of this transaction.
    if (isAutoTimestamp && frameIsEarly(expectedPresentTime, info.vsyncId)) {
        ATRACE_NAME("frameIsEarly");
        return TransactionReadiness::NotReady;
    }
 
 
    bool fenceUnsignaled = false;
    auto queueProcessTime = systemTime();
    //关键部分对transaction的states进行挨个state遍历情况
    for (const ComposerState& state : states) {
        const layer_state_t& s = state.state;
 
 
        sp<Layer> layer = nullptr;
 //这里会判断到底有没有surface。没有的话就不会进入下面判断环节
        if (s.surface) {
            layer = fromHandle(s.surface).promote();
        } else if (s.hasBufferChanges()) {
            ALOGW("Transaction with buffer, but no Layer?");
            continue;
        }
        if (!layer) {
            continue;
        }
 
 
        if (s.hasBufferChanges() && s.bufferData->hasBarrier &&
            ((layer->getDrawingState().frameNumber) < s.bufferData->barrierFrameNumber)) {
            const bool willApplyBarrierFrame =
                (bufferLayersReadyToPresent.find(s.surface) != bufferLayersReadyToPresent.end()) &&
                (bufferLayersReadyToPresent.at(s.surface) >= s.bufferData->barrierFrameNumber);
            if (!willApplyBarrierFrame) {
                ATRACE_NAME("NotReadyBarrier");
                return TransactionReadiness::NotReadyBarrier;
            }
        }
 
 
 //注意这里会有一个标志allowLatchUnsignaled意思是是否可以latch非signaled的buffer
        const bool allowLatchUnsignaled = tryApplyUnsignaled &&
                shouldLatchUnsignaled(layer, s, states.size(), totalTXapplied);
        ATRACE_FORMAT("%s allowLatchUnsignaled=%s", layer->getName().c_str(),
                      allowLatchUnsignaled ? "true" : "false");
 
 
        const bool acquireFenceChanged = s.bufferData &&
                s.bufferData->flags.test(BufferData::BufferDataChange::fenceChanged) &&
                s.bufferData->acquireFence;
 //这里会进行关键的判断，判断state的bufferData->acquireFence是否已经signaled
        fenceUnsignaled = fenceUnsignaled ||
                (acquireFenceChanged &&
                 s.bufferData->acquireFence->getStatus() == Fence::Status::Unsignaled);
 
 
 //如果fenceUnsignaled属于fenceUnsignaled，allowLatchUnsignaled也为false
        if (fenceUnsignaled && !allowLatchUnsignaled) {
            if (!transaction.sentFenceTimeoutWarning &&
                queueProcessTime - transaction.queueTime > std::chrono::nanoseconds(4s).count()) {
                transaction.sentFenceTimeoutWarning = true;
                auto listener = s.bufferData->releaseBufferListener;
                if (listener) {
                    listener->onTransactionQueueStalled();
                }
            }
     //那么就代表不可以传递transaction，会返回NotReady
            ATRACE_NAME("fence unsignaled");
            return TransactionReadiness::NotReady;
        }
 
 
        if (s.hasBufferChanges()) {
            // If backpressure is enabled and we already have a buffer to commit, keep the
            // transaction in the queue.
            const bool hasPendingBuffer = bufferLayersReadyToPresent.find(s.surface) !=
                bufferLayersReadyToPresent.end();
     //如果前面已经有state放入了，则不在放入，会放入pending
            if (layer->backpressureEnabled() && hasPendingBuffer && isAutoTimestamp) {
                ATRACE_NAME("hasPendingBuffer");
                return TransactionReadiness::NotReady;
            }
        }
    }
    return fenceUnsignaled ? TransactionReadiness::ReadyUnsignaled : TransactionReadiness::Ready;
}
```

### SurfaceFlinger applyTransactions

调用SurfaceFlinger的applyTransactions方法，应用事务：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
bool SurfaceFlinger::applyTransactions(std::vector<TransactionState>& transactions,
                                       int64_t vsyncId) {
    bool needsTraversal = false;
    // Now apply all transactions.
    //遍历每一个上面判断为ready的transaction
    for (auto& transaction : transactions) {
 //核心方法又调用到了applyTransactionState
        needsTraversal |=
                applyTransactionState(transaction.frameTimelineInfo, transaction.states,
                                      transaction.displays, transaction.flags,
                                      transaction.inputWindowCommands,
                                      transaction.desiredPresentTime, transaction.isAutoTimestamp,
                                      transaction.buffer, transaction.postTime,
                                      transaction.permissions, transaction.hasListenerCallbacks,
                                      transaction.listenerCallbacks, transaction.originPid,
                                      transaction.originUid, transaction.id);
        if (transaction.transactionCommittedSignal) {
            mTransactionCommittedSignals.emplace_back(
                    std::move(transaction.transactionCommittedSignal));
        }
    }
 
 
    if (mTransactionTracing) {
        mTransactionTracing->addCommittedTransactions(transactions, vsyncId);
    }
    return needsTraversal;
}
```

#### SurfaceFlinger applyTransactionState

调用SurfaceFlinger的applyTransactionState方法：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
bool SurfaceFlinger::applyTransactionState(const FrameTimelineInfo& frameTimelineInfo,
                                           Vector<ComposerState>& states,
                                           const Vector<DisplayState>& displays, uint32_t flags,
                                           const InputWindowCommands& inputWindowCommands,
                                           const int64_t desiredPresentTime, bool isAutoTimestamp,
                                           const client_cache_t& uncacheBuffer,
                                           const int64_t postTime, uint32_t permissions,
                                           bool hasListenerCallbacks,
                                           const std::vector<ListenerCallbacks>& listenerCallbacks,
                                           int originPid, int originUid, uint64_t transactionId) {
    uint32_t transactionFlags = 0;
    //遍历是transaction的dispplays
    for (const DisplayState& display : displays) {
 //遍历是否有dispplay相关的变化
        transactionFlags |= setDisplayStateLocked(display);
    }
 
 
    // start and end registration for listeners w/ no surface so they can get their callback.  Note
    // that listeners with SurfaceControls will start registration during setClientStateLocked
    // below.
    for (const auto& listener : listenerCallbacks) {
        mTransactionCallbackInvoker.addEmptyTransaction(listener);
    }
 
 
    uint32_t clientStateFlags = 0;
    for (int i = 0; i < states.size(); i++) { //最关键方法开始遍历一个个的states
        ComposerState& state = states.editItemAt(i);
 //调用到了setClientStateLocked方法
        clientStateFlags |= setClientStateLocked(frameTimelineInfo, state, desiredPresentTime,
                                                 isAutoTimestamp, postTime, permissions);
        if ((flags & eAnimation) && state.state.surface) {
            if (const auto layer = fromHandle(state.state.surface).promote()) {
                using LayerUpdateType = scheduler::LayerHistory::LayerUpdateType;
                mScheduler->recordLayerHistory(layer.get(),
                                               isAutoTimestamp ? 0 : desiredPresentTime,
                                               LayerUpdateType::AnimationTX);
            }
        }
    }
 
 
    transactionFlags |= clientStateFlags;
 
 
    if (permissions & layer_state_t::Permission::ACCESS_SURFACE_FLINGER) {
        transactionFlags |= addInputWindowCommands(inputWindowCommands);
    } else if (!inputWindowCommands.empty()) {
        ALOGE("Only privileged callers are allowed to send input commands.");
    }
 
 
    if (uncacheBuffer.isValid()) {
        ClientCache::getInstance().erase(uncacheBuffer);
    }
 
 
    // If a synchronous transaction is explicitly requested without any changes, force a transaction
    // anyway. This can be used as a flush mechanism for previous async transactions.
    // Empty animation transaction can be used to simulate back-pressure, so also force a
    // transaction for empty animation transactions.
    if (transactionFlags == 0 &&
            ((flags & eSynchronous) || (flags & eAnimation))) {
        transactionFlags = eTransactionNeeded;
    }
 
 
    bool needsTraversal = false;
    if (transactionFlags) {
        if (mInterceptor->isEnabled()) {
            mInterceptor->saveTransaction(states, mCurrentState.displays, displays, flags,
                                          originPid, originUid, transactionId);
        }
 
 
        // We are on the main thread, we are about to preform a traversal. Clear the traversal bit
        // so we don't have to wake up again next frame to preform an unnecessary traversal.
        if (transactionFlags & eTraversalNeeded) {
            transactionFlags = transactionFlags & (~eTraversalNeeded);
            needsTraversal = true;
        }
        if (transactionFlags) {
            setTransactionFlags(transactionFlags);
        }
 
 
        if (flags & eAnimation) {
            mAnimTransactionPending = true;
        }
    }
 
 
    return needsTraversal;
}
```

##### SurfaceFlinger setClientStateLocked

调用SurfaceFlinger的setClientStateLocked方法：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
uint32_t SurfaceFlinger::setClientStateLocked(const FrameTimelineInfo& frameTimelineInfo,
                                              ComposerState& composerState,
                                              int64_t desiredPresentTime, bool isAutoTimestamp,
                                              int64_t postTime, uint32_t permissions) {
    layer_state_t& s = composerState.state;
    s.sanitize(permissions);
 
 
    std::vector<ListenerCallbacks> filteredListeners;
    for (auto& listener : s.listeners) {
        // Starts a registration but separates the callback ids according to callback type. This
        // allows the callback invoker to send on latch callbacks earlier.
        // note that startRegistration will not re-register if the listener has
        // already be registered for a prior surface control
 
 
        ListenerCallbacks onCommitCallbacks = listener.filter(CallbackId::Type::ON_COMMIT);
        if (!onCommitCallbacks.callbackIds.empty()) {
            filteredListeners.push_back(onCommitCallbacks);
        }
 
 
        ListenerCallbacks onCompleteCallbacks = listener.filter(CallbackId::Type::ON_COMPLETE);
        if (!onCompleteCallbacks.callbackIds.empty()) {
            filteredListeners.push_back(onCompleteCallbacks);
        }
    }
 
 
    const uint64_t what = s.what;
    uint32_t flags = 0;
    sp<Layer> layer = nullptr;
    if (s.surface) {
        layer = fromHandle(s.surface).promote();
    } else {
        // The client may provide us a null handle. Treat it as if the layer was removed.
        ALOGW("Attempt to set client state with a null layer handle");
    }
    if (layer == nullptr) {
        for (auto& [listener, callbackIds] : s.listeners) {
            mTransactionCallbackInvoker.registerUnpresentedCallbackHandle(
                    new CallbackHandle(listener, callbackIds, s.surface));
        }
        return 0;
    }
 
 
    // Only set by BLAST adapter layers
    if (what & layer_state_t::eProducerDisconnect) {
        layer->onDisconnect();
    }
 
 
    if (what & layer_state_t::ePositionChanged) {
        if (layer->setPosition(s.x, s.y)) {
            flags |= eTraversalNeeded;
        }
    }
    if (what & layer_state_t::eLayerChanged) {
        // NOTE: index needs to be calculated before we update the state
        const auto& p = layer->getParent();
        if (p == nullptr) {
            ssize_t idx = mCurrentState.layersSortedByZ.indexOf(layer);
            if (layer->setLayer(s.z) && idx >= 0) {
                mCurrentState.layersSortedByZ.removeAt(idx);
                mCurrentState.layersSortedByZ.add(layer);
                // we need traversal (state changed)
                // AND transaction (list changed)
                flags |= eTransactionNeeded|eTraversalNeeded;
            }
        } else {
            if (p->setChildLayer(layer, s.z)) {
                flags |= eTransactionNeeded|eTraversalNeeded;
            }
        }
    }
    if (what & layer_state_t::eRelativeLayerChanged) {
        // NOTE: index needs to be calculated before we update the state
        const auto& p = layer->getParent();
        const auto& relativeHandle = s.relativeLayerSurfaceControl ?
                s.relativeLayerSurfaceControl->getHandle() : nullptr;
        if (p == nullptr) {
            ssize_t idx = mCurrentState.layersSortedByZ.indexOf(layer);
            if (layer->setRelativeLayer(relativeHandle, s.z) &&
                idx >= 0) {
                mCurrentState.layersSortedByZ.removeAt(idx);
                mCurrentState.layersSortedByZ.add(layer);
                // we need traversal (state changed)
                // AND transaction (list changed)
                flags |= eTransactionNeeded|eTraversalNeeded;
            }
        } else {
            if (p->setChildRelativeLayer(layer, relativeHandle, s.z)) {
                flags |= eTransactionNeeded|eTraversalNeeded;
            }
        }
    }
    if (what & layer_state_t::eSizeChanged) {
        if (layer->setSize(s.w, s.h)) {
            flags |= eTraversalNeeded;
        }
    }
    if (what & layer_state_t::eAlphaChanged) {
        if (layer->setAlpha(s.alpha))
            flags |= eTraversalNeeded;
    }
    if (what & layer_state_t::eColorChanged) {
        if (layer->setColor(s.color))
            flags |= eTraversalNeeded;
    }
    if (what & layer_state_t::eColorTransformChanged) {
        if (layer->setColorTransform(s.colorTransform)) {
            flags |= eTraversalNeeded;
        }
    }
    if (what & layer_state_t::eBackgroundColorChanged) {
        if (layer->setBackgroundColor(s.color, s.bgColorAlpha, s.bgColorDataspace)) {
            flags |= eTraversalNeeded;
        }
    }
    if (what & layer_state_t::eMatrixChanged) {
        if (layer->setMatrix(s.matrix)) flags |= eTraversalNeeded;
    }
    if (what & layer_state_t::eTransparentRegionChanged) {
        if (layer->setTransparentRegionHint(s.transparentRegion))
            flags |= eTraversalNeeded;
    }
    if (what & layer_state_t::eFlagsChanged) {
        if (layer->setFlags(s.flags, s.mask)) flags |= eTraversalNeeded;
    }
    if (what & layer_state_t::eCornerRadiusChanged) {
        if (layer->setCornerRadius(s.cornerRadius))
            flags |= eTraversalNeeded;
    }
    if (what & layer_state_t::eBackgroundBlurRadiusChanged && mSupportsBlur) {
        if (layer->setBackgroundBlurRadius(s.backgroundBlurRadius)) flags |= eTraversalNeeded;
    }
    if (what & layer_state_t::eBlurRegionsChanged) {
        if (layer->setBlurRegions(s.blurRegions)) flags |= eTraversalNeeded;
    }
    if (what & layer_state_t::eLayerStackChanged) {
        ssize_t idx = mCurrentState.layersSortedByZ.indexOf(layer);
        // We only allow setting layer stacks for top level layers,
        // everything else inherits layer stack from its parent.
        if (layer->hasParent()) {
            ALOGE("Attempt to set layer stack on layer with parent (%s) is invalid",
                  layer->getDebugName());
        } else if (idx < 0) {
            ALOGE("Attempt to set layer stack on layer without parent (%s) that "
                  "that also does not appear in the top level layer list. Something"
                  " has gone wrong.",
                  layer->getDebugName());
        } else if (layer->setLayerStack(s.layerStack)) {
            mCurrentState.layersSortedByZ.removeAt(idx);
            mCurrentState.layersSortedByZ.add(layer);
            // we need traversal (state changed)
            // AND transaction (list changed)
            flags |= eTransactionNeeded | eTraversalNeeded | eTransformHintUpdateNeeded;
        }
    }
    if (what & layer_state_t::eTransformChanged) {
        if (layer->setTransform(s.transform)) flags |= eTraversalNeeded;
    }
    if (what & layer_state_t::eTransformToDisplayInverseChanged) {
        if (layer->setTransformToDisplayInverse(s.transformToDisplayInverse))
            flags |= eTraversalNeeded;
    }
    if (what & layer_state_t::eCropChanged) {
        if (layer->setCrop(s.crop)) flags |= eTraversalNeeded;
    }
    if (what & layer_state_t::eDataspaceChanged) {
        if (layer->setDataspace(s.dataspace)) flags |= eTraversalNeeded;
    }
    if (what & layer_state_t::eHdrMetadataChanged) {
        if (layer->setHdrMetadata(s.hdrMetadata)) flags |= eTraversalNeeded;
    }
    if (what & layer_state_t::eSurfaceDamageRegionChanged) {
        if (layer->setSurfaceDamageRegion(s.surfaceDamageRegion)) flags |= eTraversalNeeded;
    }
    if (what & layer_state_t::eApiChanged) {
        if (layer->setApi(s.api)) flags |= eTraversalNeeded;
    }
    if (what & layer_state_t::eSidebandStreamChanged) {
        if (layer->setSidebandStream(s.sidebandStream)) flags |= eTraversalNeeded;
    }
    if (what & layer_state_t::eInputInfoChanged) {
        layer->setInputInfo(*s.windowInfoHandle->getInfo());
        flags |= eTraversalNeeded;
    }
    std::optional<nsecs_t> dequeueBufferTimestamp;
    if (what & layer_state_t::eMetadataChanged) {
        dequeueBufferTimestamp = s.metadata.getInt64(METADATA_DEQUEUE_TIME);
 
 
        if (const int32_t gameMode = s.metadata.getInt32(METADATA_GAME_MODE, -1); gameMode != -1) {
            // The transaction will be received on the Task layer and needs to be applied to all
            // child layers. Child layers that are added at a later point will obtain the game mode
            // info through addChild().
            layer->setGameModeForTree(static_cast<GameMode>(gameMode));
        }
 
 
        if (layer->setMetadata(s.metadata)) flags |= eTraversalNeeded;
    }
    if (what & layer_state_t::eColorSpaceAgnosticChanged) {
        if (layer->setColorSpaceAgnostic(s.colorSpaceAgnostic)) {
            flags |= eTraversalNeeded;
        }
    }
    if (what & layer_state_t::eShadowRadiusChanged) {
        if (layer->setShadowRadius(s.shadowRadius)) flags |= eTraversalNeeded;
    }
    if (what & layer_state_t::eFrameRateSelectionPriority) {
        if (layer->setFrameRateSelectionPriority(s.frameRateSelectionPriority)) {
            flags |= eTraversalNeeded;
        }
    }
    if (what & layer_state_t::eFrameRateChanged) {
        const auto compatibility =
            Layer::FrameRate::convertCompatibility(s.frameRateCompatibility);
        const auto strategy =
            Layer::FrameRate::convertChangeFrameRateStrategy(s.changeFrameRateStrategy);
 
 
        if (layer->setFrameRate(
                Layer::FrameRate(Fps::fromValue(s.frameRate), compatibility, strategy))) {
          flags |= eTraversalNeeded;
        }
    }
    if (what & layer_state_t::eFixedTransformHintChanged) {
        if (layer->setFixedTransformHint(s.fixedTransformHint)) {
            flags |= eTraversalNeeded | eTransformHintUpdateNeeded;
        }
    }
    if (what & layer_state_t::eAutoRefreshChanged) {
        layer->setAutoRefresh(s.autoRefresh);
    }
    if (what & layer_state_t::eDimmingEnabledChanged) {
        if (layer->setDimmingEnabled(s.dimmingEnabled)) flags |= eTraversalNeeded;
    }
    if (what & layer_state_t::eTrustedOverlayChanged) {
        if (layer->setTrustedOverlay(s.isTrustedOverlay)) {
            flags |= eTraversalNeeded;
        }
    }
    if (what & layer_state_t::eStretchChanged) {
        if (layer->setStretchEffect(s.stretchEffect)) {
            flags |= eTraversalNeeded;
        }
    }
    if (what & layer_state_t::eBufferCropChanged) {
        if (layer->setBufferCrop(s.bufferCrop)) {
            flags |= eTraversalNeeded;
        }
    }
    if (what & layer_state_t::eDestinationFrameChanged) {
        if (layer->setDestinationFrame(s.destinationFrame)) {
            flags |= eTraversalNeeded;
        }
    }
    if (what & layer_state_t::eDropInputModeChanged) {
        if (layer->setDropInputMode(s.dropInputMode)) {
            flags |= eTraversalNeeded;
            mInputInfoChanged = true;
        }
    }
    // This has to happen after we reparent children because when we reparent to null we remove
    // child layers from current state and remove its relative z. If the children are reparented in
    // the same transaction, then we have to make sure we reparent the children first so we do not
    // lose its relative z order.
    if (what & layer_state_t::eReparent) {
        bool hadParent = layer->hasParent();
        auto parentHandle = (s.parentSurfaceControlForChild)
                ? s.parentSurfaceControlForChild->getHandle()
                : nullptr;
        if (layer->reparent(parentHandle)) {
            if (!hadParent) {
                layer->setIsAtRoot(false);
                mCurrentState.layersSortedByZ.remove(layer);
            }
            flags |= eTransactionNeeded | eTraversalNeeded;
        }
    }
    std::vector<sp<CallbackHandle>> callbackHandles;
    if ((what & layer_state_t::eHasListenerCallbacksChanged) && (!filteredListeners.empty())) {
        for (auto& [listener, callbackIds] : filteredListeners) {
            callbackHandles.emplace_back(new CallbackHandle(listener, callbackIds, s.surface));
        }
    }
 
 
    if (what & layer_state_t::eBufferChanged) {  //如果发现有buffer变化
        std::shared_ptr<renderengine::ExternalTexture> buffer =
                getExternalTextureFromBufferData(*s.bufferData, layer->getDebugName());
 //把state的bufferData相关信息设置到了layer中去
        if (layer->setBuffer(buffer, *s.bufferData, postTime, desiredPresentTime, isAutoTimestamp,
                             dequeueBufferTimestamp, frameTimelineInfo)) {
            flags |= eTraversalNeeded;
        }
    } else if (frameTimelineInfo.vsyncId != FrameTimelineInfo::INVALID_VSYNC_ID) {
        layer->setFrameTimelineVsyncForBufferlessTransaction(frameTimelineInfo, postTime);
    }
 
 
    if (layer->setTransactionCompletedListeners(callbackHandles)) flags |= eTraversalNeeded;
    // Do not put anything that updates layer state or modifies flags after
    // setTransactionCompletedListener
    return flags;
}
```

## SurfaceFlinger commitTransactions

调用SurfaceFlinger的commitTransactions方法，进行Transaction提交：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
void SurfaceFlinger::commitTransactions() {
    ATRACE_CALL();
 
 
    // Keep a copy of the drawing state (that is going to be overwritten
    // by commitTransactionsLocked) outside of mStateLock so that the side
    // effects of the State assignment don't happen with mStateLock held,
    // which can cause deadlocks.
    State drawingState(mDrawingState);
 
 
    Mutex::Autolock lock(mStateLock);
    mDebugInTransaction = systemTime();
 
 
    // Here we're guaranteed that some transaction flags are set
    // so we can call commitTransactionsLocked unconditionally.
    // We clear the flags with mStateLock held to guarantee that
    // mCurrentState won't change until the transaction is committed.
    modulateVsync(&VsyncModulator::onTransactionCommit);
    commitTransactionsLocked(clearTransactionFlags(eTransactionMask));
 
 
    mDebugInTransaction = 0;
}
```

调用SurfaceFlinger的commitTransactionsLocked方法：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
void SurfaceFlinger::commitTransactionsLocked(uint32_t transactionFlags) {
    // Commit display transactions.
    const bool displayTransactionNeeded = transactionFlags & eDisplayTransactionNeeded;
    if (displayTransactionNeeded) { //如果需要显示
        processDisplayChangesLocked(); //处理显示变化相关
        processDisplayHotplugEventsLocked();
    }
    mForceTransactionDisplayChange = displayTransactionNeeded;
 
 
    if (mSomeChildrenChanged) {
        mVisibleRegionsDirty = true;
        mSomeChildrenChanged = false;
    }
 
 
    // Update transform hint.
    if (transactionFlags & (eTransformHintUpdateNeeded | eDisplayTransactionNeeded)) {
        // Layers and/or displays have changed, so update the transform hint for each layer.
        //
        // NOTE: we do this here, rather than when presenting the display so that
        // the hint is set before we acquire a buffer from the surface texture.
        //
        // NOTE: layer transactions have taken place already, so we use their
        // drawing state. However, SurfaceFlinger's own transaction has not
        // happened yet, so we must use the current state layer list
        // (soon to become the drawing state list).
        //
        sp<const DisplayDevice> hintDisplay;
        ui::LayerStack layerStack;
 
 
        mCurrentState.traverse([&](Layer* layer) REQUIRES(mStateLock) {
            // NOTE: we rely on the fact that layers are sorted by
            // layerStack first (so we don't have to traverse the list
            // of displays for every layer).
            if (const auto filter = layer->getOutputFilter(); layerStack != filter.layerStack) {
                layerStack = filter.layerStack;
                hintDisplay = nullptr;
 
 
                // Find the display that includes the layer.
                for (const auto& [token, display] : mDisplays) {
                    if (!display->getCompositionDisplay()->includesLayer(filter)) {
                        continue;
                    }
 
 
                    // Pick the primary display if another display mirrors the layer.
                    if (hintDisplay) {
                        hintDisplay = nullptr;
                        break;
                    }
 
 
                    hintDisplay = display;
                }
            }
 
 
            if (!hintDisplay) {
                // NOTE: TEMPORARY FIX ONLY. Real fix should cause layers to
                // redraw after transform hint changes. See bug 8508397.
 
 
                // could be null when this layer is using a layerStack
                // that is not visible on any display. Also can occur at
                // screen off/on times.
                hintDisplay = getDefaultDisplayDeviceLocked();
            }
 
 
            layer->updateTransformHint(hintDisplay->getTransformHint());
        });
    }
 
 
    if (mLayersAdded) {
        mLayersAdded = false;
        // Layers have been added.
        mVisibleRegionsDirty = true;
    }
 
 
    // some layers might have been removed, so
    // we need to update the regions they're exposing.
    if (mLayersRemoved) {
        mLayersRemoved = false;
        mVisibleRegionsDirty = true;
        mDrawingState.traverseInZOrder([&](Layer* layer) {
            if (mLayersPendingRemoval.indexOf(layer) >= 0) {
                // this layer is not visible anymore
                Region visibleReg;
                visibleReg.set(layer->getScreenBounds());
                invalidateLayerStack(layer, visibleReg);
            }
        });
    }
 
 
    doCommitTransactions();
    signalSynchronousTransactions(CountDownLatch::eSyncTransaction);
    mAnimTransactionPending = false;
}
```

上面方法主要处理如下：

1、调用SurfaceFlinger的processDisplayChangesLocked方法，处理显示器变化相关。

2、调用SurfaceFlinger的doCommitTransactions方法，执行提交事务。

下面分别进行分析：

### SurfaceFlinger processDisplayChangesLocked

调用SurfaceFlinger的processDisplayChangesLocked方法：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
void SurfaceFlinger::processDisplayChangesLocked() {
    // here we take advantage of Vector's copy-on-write semantics to
    // improve performance by skipping the transaction entirely when
    // know that the lists are identical
    const KeyedVector<wp<IBinder>, DisplayDeviceState>& curr(mCurrentState.displays);
    const KeyedVector<wp<IBinder>, DisplayDeviceState>& draw(mDrawingState.displays);
    if (!curr.isIdenticalTo(draw)) {
        mVisibleRegionsDirty = true;
 
 
        // find the displays that were removed
        // (ie: in drawing state but not in current state)
        // also handle displays that changed
        // (ie: displays that are in both lists)
        for (size_t i = 0; i < draw.size(); i++) {
            const wp<IBinder>& displayToken = draw.keyAt(i);
            const ssize_t j = curr.indexOfKey(displayToken);
            if (j < 0) {
                // in drawing state but not in current state
                // 处于绘图状态，但不在当前状态
                processDisplayRemoved(displayToken);
            } else {
                // this display is in both lists. see if something changed.
 // 此显示在两个列表中。看看是否有变化。
                const DisplayDeviceState& currentState = curr[j];
                const DisplayDeviceState& drawingState = draw[i];
                processDisplayChanged(displayToken, currentState, drawingState);
            }
        }
 
 
        // find displays that were added
        // 查找已添加的显示
        // (ie: in current state but not in drawing state)
        for (size_t i = 0; i < curr.size(); i++) {
            const wp<IBinder>& displayToken = curr.keyAt(i);
            if (draw.indexOfKey(displayToken) < 0) {
                processDisplayAdded(displayToken, curr[i]);
            }
        }
    }
 
 
    mDrawingState.displays = mCurrentState.displays;
}
```

上面方法根据不同的调节调用如下方法：

1、调用processDisplayRemoved方法，处理显示器删除。

2、调用processDisplayChanged方法，处理显示器变更。

3、调用processDisplayAdded方法，处理显示器添加。

下面分别进行分析：

#### SurfaceFlinger processDisplayRemoved

调用processDisplayRemoved方法，处理显示器删除：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
void SurfaceFlinger::processDisplayRemoved(const wp<IBinder>& displayToken) {
    auto display = getDisplayDeviceLocked(displayToken);
    if (display) {
        display->disconnect(); //与Display断开连接
 
 
        if (display->isVirtual()) {
            releaseVirtualDisplay(display->getVirtualId()); //如果是虚拟显示器，就进行资源释放
        } else {
            dispatchDisplayHotplugEvent(display->getPhysicalId(), false);  //分配显示器热插拔事件
        }
    }
 
 
    mDisplays.erase(displayToken);
 
 
    if (display && display->isVirtual()) {
        static_cast<void>(mScheduler->schedule([display = std::move(display)] {
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

#### SurfaceFlinger processDisplayChanged

调用processDisplayChanged方法，处理显示器变更：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
void SurfaceFlinger::processDisplayChanged(const wp<IBinder>& displayToken,
                                           const DisplayDeviceState& currentState,
                                           const DisplayDeviceState& drawingState) {
    const sp<IBinder> currentBinder = IInterface::asBinder(currentState.surface);
    const sp<IBinder> drawingBinder = IInterface::asBinder(drawingState.surface);
 
 
    // Recreate the DisplayDevice if the surface or sequence ID changed.
    // 如果图面或序列 ID 发生更改，请重新创建 DisplayDevice。
    if (currentBinder != drawingBinder || currentState.sequenceId != drawingState.sequenceId) {
        getRenderEngine().cleanFramebufferCache();
 
 
        if (const auto display = getDisplayDeviceLocked(displayToken)) {
            display->disconnect();
            if (display->isVirtual()) {
                releaseVirtualDisplay(display->getVirtualId());
            }
        }
 
 
        mDisplays.erase(displayToken);
 
 
        if (const auto& physical = currentState.physical) {
            getHwComposer().allocatePhysicalDisplay(physical->hwcDisplayId, physical->id);
        }
 
 
        processDisplayAdded(displayToken, currentState);
 
 
        if (currentState.physical) {
            const auto display = getDisplayDeviceLocked(displayToken);
            setPowerModeInternal(display, hal::PowerMode::ON);
 
 
            // TODO(b/175678251) Call a listener instead.
            if (currentState.physical->hwcDisplayId == getHwComposer().getPrimaryHwcDisplayId()) {
                updateInternalDisplayVsyncLocked(display);
            }
        }
        return;
    }
 
 
    if (const auto display = getDisplayDeviceLocked(displayToken)) {
        if (currentState.layerStack != drawingState.layerStack) {
            display->setLayerStack(currentState.layerStack); //设置显示器层堆栈
        }
        if (currentState.flags != drawingState.flags) {
            display->setFlags(currentState.flags); //设置显示器标志
        }
        if ((currentState.orientation != drawingState.orientation) ||
            (currentState.layerStackSpaceRect != drawingState.layerStackSpaceRect) ||
            (currentState.orientedDisplaySpaceRect != drawingState.orientedDisplaySpaceRect)) {
            display->setProjection(currentState.orientation, currentState.layerStackSpaceRect,
                                   currentState.orientedDisplaySpaceRect); //设置显示器投影
            if (isDisplayActiveLocked(display)) {
                mActiveDisplayTransformHint = display->getTransformHint();
            }
        }
        if (currentState.width != drawingState.width ||
            currentState.height != drawingState.height) {
            display->setDisplaySize(currentState.width, currentState.height); //设置显示器尺寸
 
 
            if (isDisplayActiveLocked(display)) {
                onActiveDisplaySizeChanged(display);
            }
        }
    }
}
```

#### SurfaceFlinger processDisplayAdded

调用processDisplayAdded方法，处理显示器添加：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
void SurfaceFlinger::processDisplayAdded(const wp<IBinder>& displayToken,
                                         const DisplayDeviceState& state) {
    ui::Size resolution(0, 0);
    ui::PixelFormat pixelFormat = static_cast<ui::PixelFormat>(PIXEL_FORMAT_UNKNOWN);
    if (state.physical) {
        resolution = state.physical->activeMode->getResolution();
        pixelFormat = static_cast<ui::PixelFormat>(PIXEL_FORMAT_RGBA_8888);
    } else if (state.surface != nullptr) {
        int status = state.surface->query(NATIVE_WINDOW_WIDTH, &resolution.width);
        ALOGE_IF(status != NO_ERROR, "Unable to query width (%d)", status);
        status = state.surface->query(NATIVE_WINDOW_HEIGHT, &resolution.height);
        ALOGE_IF(status != NO_ERROR, "Unable to query height (%d)", status);
        int format;
        status = state.surface->query(NATIVE_WINDOW_FORMAT, &format);
        ALOGE_IF(status != NO_ERROR, "Unable to query format (%d)", status);
        pixelFormat = static_cast<ui::PixelFormat>(format);
    } else {
        // Virtual displays without a surface are dormant:
        // they have external state (layer stack, projection,
        // etc.) but no internal state (i.e. a DisplayDevice).
        return;
    }
 
 
    compositionengine::DisplayCreationArgsBuilder builder;
    if (const auto& physical = state.physical) {
        builder.setId(physical->id);
    } else {
        builder.setId(acquireVirtualDisplay(resolution, pixelFormat));
    }
 
 
    builder.setPixels(resolution);
    builder.setIsSecure(state.isSecure);
    builder.setPowerAdvisor(mPowerAdvisor.get());
    builder.setName(state.displayName);
    auto compositionDisplay = getCompositionEngine().createDisplay(builder.build());
    compositionDisplay->setLayerCachingEnabled(mLayerCachingEnabled);
 
 
    sp<compositionengine::DisplaySurface> displaySurface;
    sp<IGraphicBufferProducer> producer;
    sp<IGraphicBufferProducer> bqProducer;
    sp<IGraphicBufferConsumer> bqConsumer;
    getFactory().createBufferQueue(&bqProducer, &bqConsumer, /*consumerIsSurfaceFlinger =*/false);
 
 
    if (state.isVirtual()) {
        const auto displayId = VirtualDisplayId::tryCast(compositionDisplay->getId());
        LOG_FATAL_IF(!displayId);
        auto surface = sp<VirtualDisplaySurface>::make(getHwComposer(), *displayId, state.surface,
                                                       bqProducer, bqConsumer, state.displayName);
        displaySurface = surface;
        producer = std::move(surface);
    } else {
        ALOGE_IF(state.surface != nullptr,
                 "adding a supported display, but rendering "
                 "surface is provided (%p), ignoring it",
                 state.surface.get());
        const auto displayId = PhysicalDisplayId::tryCast(compositionDisplay->getId());
        LOG_FATAL_IF(!displayId);
        displaySurface =
                sp<FramebufferSurface>::make(getHwComposer(), *displayId, bqConsumer,
                                             state.physical->activeMode->getResolution(),
                                             ui::Size(maxGraphicsWidth, maxGraphicsHeight));
        producer = bqProducer;
    }
 
 
    LOG_FATAL_IF(!displaySurface);
    auto display = setupNewDisplayDeviceInternal(displayToken, std::move(compositionDisplay), state,
                                                 displaySurface, producer);
    if (display->isPrimary()) {
        initScheduler(display);
    }
    if (!state.isVirtual()) {
        dispatchDisplayHotplugEvent(display->getPhysicalId(), true);
    }
 
 
    mDisplays.try_emplace(displayToken, std::move(display));
}
```

### SurfaceFlinger doCommitTransactions

调用SurfaceFlinger的doCommitTransactions方法：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
void SurfaceFlinger::doCommitTransactions() {
    ATRACE_CALL();
 
 
    if (!mLayersPendingRemoval.isEmpty()) {
        // Notify removed layers now that they can't be drawn from
        for (const auto& l : mLayersPendingRemoval) {
            // Ensure any buffers set to display on any children are released.
            if (l->isRemovedFromCurrentState()) {
                l->latchAndReleaseBuffer();
            }
 
 
            // If a layer has a parent, we allow it to out-live it's handle
            // with the idea that the parent holds a reference and will eventually
            // be cleaned up. However no one cleans up the top-level so we do so
            // here.
            if (l->isAtRoot()) {
                l->setIsAtRoot(false);
                mCurrentState.layersSortedByZ.remove(l);
            }
 
 
            // If the layer has been removed and has no parent, then it will not be reachable
            // when traversing layers on screen. Add the layer to the offscreenLayers set to
            // ensure we can copy its current to drawing state.
            if (!l->getParent()) {
                mOffscreenLayers.emplace(l.get());
            }
        }
        mLayersPendingRemoval.clear();
    }
 
 
    // If this transaction is part of a window animation then the next frame
    // we composite should be considered an animation as well.
    //最关键mCurrentState赋值给了mDrawingState
    mAnimCompositionPending = mAnimTransactionPending;
 
 
    mDrawingState = mCurrentState;
    // clear the "changed" flags in current state
    mCurrentState.colorMatrixChanged = false;
 
 
    if (mVisibleRegionsDirty) {
        for (const auto& rootLayer : mDrawingState.layersSortedByZ) {
            rootLayer->commitChildList();
        }
    }
 
 
    commitOffscreenLayers();
    if (mNumClones > 0) {
        mDrawingState.traverse([&](Layer* layer) { layer->updateMirrorInfo(); });
    }
}
```

## SurfaceFlinger latchBuffers

调用SurfaceFlinger的latchBuffers方法，将应用程序的图形缓冲区（buffer）与显示器进行关联：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
bool SurfaceFlinger::latchBuffers() {
    ATRACE_CALL();
 
 
    const nsecs_t latchTime = systemTime();
 
 
    bool visibleRegions = false;
    bool frameQueued = false;
    bool newDataLatched = false;
 
 
    const nsecs_t expectedPresentTime = mExpectedPresentTime.load();
 
 
    // Store the set of layers that need updates. This set must not change as
    // buffers are being latched, as this could result in a deadlock.
    // Example: Two producers share the same command stream and:
    // 1.) Layer 0 is latched
    // 2.) Layer 0 gets a new frame
    // 2.) Layer 1 gets a new frame
    // 3.) Layer 1 is latched.
    // Display is now waiting on Layer 1's frame, which is behind layer 0's
    // second frame. But layer 0's second frame could be waiting on display.
    mDrawingState.traverse([&](Layer* layer) {
        if (layer->clearTransactionFlags(eTransactionNeeded) || mForceTransactionDisplayChange) {
            const uint32_t flags = layer->doTransaction(0);
            if (flags & Layer::eVisibleRegion) {
                mVisibleRegionsDirty = true;
            }
        }
 
 
 //判断是否有buffer
        if (layer->hasReadyFrame()) {
            frameQueued = true;
     //是否应该显示
            if (layer->shouldPresentNow(expectedPresentTime)) {
  //如果要显示放入到mLayersWithQueuedFrames队列
                mLayersWithQueuedFrames.emplace(layer);
            } else {
                ATRACE_NAME("!layer->shouldPresentNow()");
                layer->useEmptyDamage();
            }
        } else {
            layer->useEmptyDamage();
        }
    });
    //上面主要是为了从sf的mDrawingState遍历寻找出有buffer的layer
 
 
    mForceTransactionDisplayChange = false;
 
 
    // The client can continue submitting buffers for offscreen layers, but they will not
    // be shown on screen. Therefore, we need to latch and release buffers of offscreen
    // layers to ensure dequeueBuffer doesn't block indefinitely.
    for (Layer* offscreenLayer : mOffscreenLayers) {
        offscreenLayer->traverse(LayerVector::StateSet::Drawing,
                                         [&](Layer* l) { l->latchAndReleaseBuffer(); });
    }
 
 
    //如果buffer队列不为空
    if (!mLayersWithQueuedFrames.empty()) {
        // mStateLock is needed for latchBuffer as LayerRejecter::reject()
        // writes to Layer current state. See also b/119481871
        Mutex::Autolock lock(mStateLock);
 
 
        for (const auto& layer : mLayersWithQueuedFrames) {
     //进行关键的latchBuffer操作,然后newDataLatched设置成了true
            if (layer->latchBuffer(visibleRegions, latchTime, expectedPresentTime)) {
                mLayersPendingRefresh.push_back(layer); /这里有latchBuffer说明有buffer刷新，放入mLayersPendingRefresh
                newDataLatched = true;
            }
            layer->useSurfaceDamage();
        }
    }
 
 
    mVisibleRegionsDirty |= visibleRegions;
 
 
    // If we will need to wake up at some time in the future to deal with a
    // queued frame that shouldn't be displayed during this vsync period, wake
    // up during the next vsync period to check again.
    if (frameQueued && (mLayersWithQueuedFrames.empty() || !newDataLatched)) {
        scheduleCommit(FrameHint::kNone);
    }
 
 
    // enter boot animation on first buffer latch
    if (CC_UNLIKELY(mBootStage == BootStage::BOOTLOADER && newDataLatched)) {
        ALOGI("Enter boot animation");
        mBootStage = BootStage::BOOTANIMATION;
    }
 
 
    if (mNumClones > 0) {
        mDrawingState.traverse([&](Layer* layer) { layer->updateCloneBufferInfo(); });
    }
 
 
    // Only continue with the refresh if there is actually new work to do
    return !mLayersWithQueuedFrames.empty() && newDataLatched;
}
```

### BufferLayer latchBuffer

调用layer的latchBuffer方法：

```cpp
//frameworks/native/services/surfaceflinger/BufferLayer.cpp
bool BufferLayer::latchBuffer(bool& recomputeVisibleRegions, nsecs_t latchTime,
                              nsecs_t expectedPresentTime) {
    ATRACE_CALL();
 
 
    bool refreshRequired = latchSidebandStream(recomputeVisibleRegions);
 
 
    if (refreshRequired) {
        return refreshRequired;
    }
 
 
    // If the head buffer's acquire fence hasn't signaled yet, return and
    // try again later
    // 如果头部缓冲区的采集围栏尚未发出信号，请返回并稍后重试
    if (!fenceHasSignaled()) {
        ATRACE_NAME("!fenceHasSignaled()");
        mFlinger->onLayerUpdate(); // (692) SurfaceFlinger onLayerUpdate流程分析 | 知识管理 - PingCode 
        return false;
    }
 
 
    // Capture the old state of the layer for comparisons later
    const State& s(getDrawingState());
    const bool oldOpacity = isOpaque(s);
 
 
    BufferInfo oldBufferInfo = mBufferInfo;
 
 
    //调用到BufferStateLayer中的updateTexImage，这里业务主要返回对应的recomputeVisibleRegions
    status_t err = updateTexImage(recomputeVisibleRegions, latchTime, expectedPresentTime);
    if (err != NO_ERROR) {
        return false;
    }
 
 
    //这里主要吧state的相关buffer数据赋值给mBufferInfo
    err = updateActiveBuffer();
    if (err != NO_ERROR) {
        return false;
    }
 
 
    //赋值一下framenumber
    err = updateFrameNumber();
    if (err != NO_ERROR) {
        return false;
    }
 
 
    //这里又一次吧mDrawingState中BufferInfo需要的大部分数据进行赋值
    gatherBufferInfo();
 
 
    if (oldBufferInfo.mBuffer == nullptr) {
        // the first time we receive a buffer, we need to trigger a
        // geometry invalidation.
        recomputeVisibleRegions = true;
    }
 
 
    if ((mBufferInfo.mCrop != oldBufferInfo.mCrop) ||
        (mBufferInfo.mTransform != oldBufferInfo.mTransform) ||
        (mBufferInfo.mScaleMode != oldBufferInfo.mScaleMode) ||
        (mBufferInfo.mTransformToDisplayInverse != oldBufferInfo.mTransformToDisplayInverse)) {
        recomputeVisibleRegions = true;
    }
 
 
    if (oldBufferInfo.mBuffer != nullptr) {
        uint32_t bufWidth = mBufferInfo.mBuffer->getWidth();
        uint32_t bufHeight = mBufferInfo.mBuffer->getHeight();
        if (bufWidth != oldBufferInfo.mBuffer->getWidth() ||
            bufHeight != oldBufferInfo.mBuffer->getHeight()) {
            recomputeVisibleRegions = true;
        }
    }
 
 
    if (oldOpacity != isOpaque(s)) {
        recomputeVisibleRegions = true;
    }
 
 
    return true;
}
```

#### BufferQueueLayer updateTexImage

调用BufferLayer的updateTexImage方法，BufferQueueLayer继承与于BufferLayer，调用BufferQueueLayer的updateTexImage方法：

[Android13 BufferQueueLayer updateTexImage流程分析-CSDN博客](/Android13%20BufferQueueLayer%20updateTexImage%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-CSDN%E5%8D%9A%E5%AE%A2)

## SurfaceFlinger updateLayerGeometry

调用SurfaceFlinger的updateLayerGeometry方法，这次vsync显示刷新的layer进行脏区设置：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
void SurfaceFlinger::updateLayerGeometry() {
    ATRACE_CALL();
 
 
    if (mVisibleRegionsDirty) {
        computeLayerBounds(); //触发各个layer计算对于的bound
    }
 
 
    //前面有buffer的集合mLayersPendingRefresh进行对应display的dirtyRegion更新
    for (auto& layer : mLayersPendingRefresh) {
        Region visibleReg;
        visibleReg.set(layer->getScreenBounds());
        invalidateLayerStack(layer, visibleReg); //刷新一下display的dirtyRegion
    }
    mLayersPendingRefresh.clear();
}
```

### SurfaceFlinger invalidateLayerStack

调用SurfaceFlinger的invalidateLayerStack方法：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
void SurfaceFlinger::invalidateLayerStack(const sp<const Layer>& layer, const Region& dirty) {
    for (const auto& [token, displayDevice] : FTL_FAKE_GUARD(mStateLock, mDisplays)) {
        auto display = displayDevice->getCompositionDisplay();
        if (display->includesLayer(layer->getOutputFilter())) { //寻找到对应的display
            display->editState().dirtyRegion.orSelf(dirty); //设置到了display->editState中
        }
    }
}
```

上面主要就是干了一件事，根据这次有buffer的Layer进行遍历，刷新对应display的dirtyRegion。

## SurfaceFlinger updateInputFlinger

调用SurfaceFlinger的updateInputFlinger方法，更新输入处理：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
void SurfaceFlinger::updateInputFlinger() {
    ATRACE_CALL();
    if (!mInputFlinger) {
        return;
    }
 
 
    std::vector<WindowInfo> windowInfos;
    std::vector<DisplayInfo> displayInfos;
    bool updateWindowInfo = false;
    if (mVisibleRegionsDirty || mInputInfoChanged) {
        mInputInfoChanged = false;
        updateWindowInfo = true;
 //进行遍历整个系统的layer和display转变成windowInfos，displayInfos信息
        buildWindowInfos(windowInfos, displayInfos);
    }
    if (!updateWindowInfo && mInputWindowCommands.empty()) {
        return;
    }
    //这里放入子线程进行相对应跨进程通讯
    BackgroundExecutor::getInstance().sendCallbacks({[updateWindowInfo,
                                                      windowInfos = std::move(windowInfos),
                                                      displayInfos = std::move(displayInfos),
                                                      inputWindowCommands =
                                                              std::move(mInputWindowCommands),
                                                      inputFlinger = mInputFlinger, this]() {
        ATRACE_NAME("BackgroundExecutor::updateInputFlinger");
        if (updateWindowInfo) {
     //这里调用是mWindowInfosListenerInvoker进行的windowInfos, displayInfos进行跨进程传递
            mWindowInfosListenerInvoker->windowInfosChanged(windowInfos, displayInfos,
                                                            inputWindowCommands.syncInputWindows);
        } else if (inputWindowCommands.syncInputWindows) {
            // If the caller requested to sync input windows, but there are no
            // changes to input windows, notify immediately.
            windowInfosReported();
        }
        for (const auto& focusRequest : inputWindowCommands.focusRequests) {
     //直接调用inputFlinger的bpbinder跨进程设置setFocusedWindow
            inputFlinger->setFocusedWindow(focusRequest);
        }
    }});
 
 
    mInputWindowCommands.clear();
}
```
