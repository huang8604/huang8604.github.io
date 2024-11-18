# Android13 SurfaceFlinger Commit(提交)流程分析_surfaceflinger::commit-CSDN博客

SurfaceFlinger的commit方法用于将应用程序的绘制结果提交到屏幕上显示。

主要就是处理app端发起的一系列transaction的事务请求，需要对这些请求进行识别是否当前帧处理，处理过程就是把事务中的属性取出，然后更新到Layer中，偶尔buffer更新的还需要进行相关的latchbuffer操作，SurfaceFlinger的commit代码如下：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
bool SurfaceFlinger::commit(nsecs_t frameTime, int64_t vsyncId, nsecs_t expectedVsyncTime) //这里frameTime代表当前时间，expectedVsyncTime代表硬件vsync时间，即屏幕先的vsync时间
        FTL_FAKE_GUARD(kMainThreadContext) {
    // we set this once at the beginning of commit to ensure consistency throughout the whole frame
    mPowerHintSessionData.sessionEnabled = mPowerAdvisor-&gt;usePowerHintSession();
    if (mPowerHintSessionData.sessionEnabled) {
        mPowerHintSessionData.commitStart = systemTime();
    }
 
 
    // calculate the expected present time once and use the cached
    // value throughout this frame to make sure all layers are
    // seeing this same value.
    if (expectedVsyncTime &gt;= frameTime) {
        mExpectedPresentTime = expectedVsyncTime;
    } else {
        const DisplayStatInfo stats = mScheduler-&gt;getDisplayStatInfo(frameTime);
        mExpectedPresentTime = calculateExpectedPresentTime(stats);
    }
 
 
    const nsecs_t lastScheduledPresentTime = mScheduledPresentTime;
    mScheduledPresentTime = expectedVsyncTime;
 
 
    if (mPowerHintSessionData.sessionEnabled) {
        mPowerAdvisor-&gt;setTargetWorkDuration(mExpectedPresentTime -
                                             mPowerHintSessionData.commitStart);
    }
    const auto vsyncIn = [&amp;] {
        if (!ATRACE_ENABLED()) return 0.f;
        return (mExpectedPresentTime - systemTime()) / 1e6f;
    }();
    ATRACE_FORMAT(&#34;%s %&#34; PRId64 &#34; vsyncIn %.2fms%s&#34;, __func__, vsyncId, vsyncIn,
                  mExpectedPresentTime == expectedVsyncTime ? &#34;&#34; : &#34; (adjusted)&#34;);
 
 
    // When Backpressure propagation is enabled we want to give a small grace period
    // for the present fence to fire instead of just giving up on this frame to handle cases
    // where present fence is just about to get signaled.
    const int graceTimeForPresentFenceMs =
            (mPropagateBackpressureClientComposition || !mHadClientComposition) ? 1 : 0;
 
 
    // Pending frames may trigger backpressure propagation.
    const TracedOrdinal&lt;bool&gt; framePending = {&#34;PrevFramePending&#34;,
                                              previousFramePending(graceTimeForPresentFenceMs)};
 
 
    // Frame missed counts for metrics tracking.
    // A frame is missed if the prior frame is still pending. If no longer pending,
    // then we still count the frame as missed if the predicted present time
    // was further in the past than when the fence actually fired.
 
 
    // Add some slop to correct for drift. This should generally be
    // smaller than a typical frame duration, but should not be so small
    // that it reports reasonable drift as a missed frame.
    const DisplayStatInfo stats = mScheduler-&gt;getDisplayStatInfo(systemTime());
    const nsecs_t frameMissedSlop = stats.vsyncPeriod / 2;
    const nsecs_t previousPresentTime = previousFramePresentTime();
    const TracedOrdinal&lt;bool&gt; frameMissed = {&#34;PrevFrameMissed&#34;,
                                             framePending ||
                                                     (previousPresentTime &gt;= 0 &amp;&amp;
                                                      (lastScheduledPresentTime &lt;
                                                       previousPresentTime - frameMissedSlop))};
    const TracedOrdinal&lt;bool&gt; hwcFrameMissed = {&#34;PrevHwcFrameMissed&#34;,
                                                mHadDeviceComposition &amp;&amp; frameMissed};
    const TracedOrdinal&lt;bool&gt; gpuFrameMissed = {&#34;PrevGpuFrameMissed&#34;,
                                                mHadClientComposition &amp;&amp; frameMissed};
 
 
    if (frameMissed) {
        mFrameMissedCount&#43;&#43;;
        mTimeStats-&gt;incrementMissedFrames();
    }
 
 
    if (hwcFrameMissed) {
        mHwcFrameMissedCount&#43;&#43;;
    }
 
 
    if (gpuFrameMissed) {
        mGpuFrameMissedCount&#43;&#43;;
    }
 
 
    // If we are in the middle of a mode change and the fence hasn&#39;t
    // fired yet just wait for the next commit.
    // 如果我们正处于模式更改的过程中，并且围栏尚未触发，请等待下一次提交。
    if (mSetActiveModePending) {
        if (framePending) {
            mScheduler-&gt;scheduleFrame();
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
        if ((hwcFrameMissed &amp;&amp; !gpuFrameMissed) || mPropagateBackpressureClientComposition) {
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
            display-&gt;animateRefreshRateOverlay();
        }
    }
 
 
    // Composite if transactions were committed, or if requested by HWC.
    bool mustComposite = mMustComposite.exchange(false);
    {
        mFrameTimeline-&gt;setSfWakeUp(vsyncId, frameTime, Fps::fromPeriodNsecs(stats.vsyncPeriod));
 
 
        bool needsTraversal = false;
        if (clearTransactionFlags(eTransactionFlushNeeded)) { //满足eTransactionFlushNeeded条件进入
            needsTraversal |= commitCreatedLayers(); //负责新创建的layer相关业务处理
            needsTraversal |= flushTransactionQueues(vsyncId); //这里是对前面的Transaction处理的核心部分
        }
 
 
        const bool shouldCommit =
                (getTransactionFlags() &amp; ~eTransactionFlushNeeded) || needsTraversal;
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
    // Hold mStateLock as chooseRefreshRateForContent promotes wp&lt;Layer&gt; to sp&lt;Layer&gt;
    // and may eventually call to ~Layer() if it holds the last reference
    {
        Mutex::Autolock _l(mStateLock);
        mScheduler-&gt;chooseRefreshRateForContent();
        setActiveModeInHwcIfNeeded();
    }
 
 
    updateCursorAsync(); //鼠标相关layer处理
    updateInputFlinger(); //更新触摸input下的相关的window等，这里也非常关键哈，直接影响触摸是否到app
 
 
    if (mLayerTracingEnabled &amp;&amp; !mLayerTracing.flagIsSet(LayerTracing::TRACE_COMPOSITION)) {
        // This will block and tracing should only be enabled for debugging.
        mLayerTracing.notify(mVisibleRegionsDirty, frameTime);
    }
 
 
    persistDisplayBrightness(mustComposite);
 
 
    return mustComposite &amp;&amp; CC_LIKELY(mBootStage != BootStage::BOOTLOADER);
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
    std::vector&lt;LayerCreatedState&gt; createdLayers;
    {
        std::scoped_lock&lt;std::mutex&gt; lock(mCreatedLayersLock);
        createdLayers = std::move(mCreatedLayers);
        mCreatedLayers.clear();
        if (createdLayers.size() == 0) {
            return false;
        }
    }
 
 
    Mutex::Autolock _l(mStateLock);
    for (const auto&amp; createdLayer : createdLayers) {
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
    std::vector&lt;TransactionState&gt; transactions;
    // Layer handles that have transactions with buffers that are ready to be applied.
    std::unordered_map&lt;sp&lt;IBinder&gt;, uint64_t, SpHash&lt;IBinder&gt;&gt; bufferLayersReadyToPresent;
    std::unordered_set&lt;sp&lt;IBinder&gt;, SpHash&lt;IBinder&gt;&gt; applyTokensWithUnsignaledTransactions;
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
                auto&amp; transaction = mTransactionQueue.front();
  //判断是否处于mPendingTransactionQueues队里里面
                const bool pendingTransactions =
                        mPendingTransactionQueues.find(transaction.applyToken) !=
                        mPendingTransactionQueues.end();
  //这里有个ready的判断，主要就是看看当前的Transaction是否已经ready，没有ready则不予进行传递Transaction
                const auto ready = [&amp;]() REQUIRES(mStateLock) {
                    if (pendingTransactions) {
                        ATRACE_NAME(&#34;pendingTransactions&#34;);
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
                ATRACE_INT(&#34;TransactionReadiness&#34;, static_cast&lt;int&gt;(ready));
                if (ready != TransactionReadiness::Ready) {
                    if (ready == TransactionReadiness::NotReadyBarrier) {
                        transactionsPendingBarrier&#43;&#43;;
                    }
      //放入mPendingTransactionQueues
                    mPendingTransactionQueues[transaction.applyToken].push(std::move(transaction));
                } else {
      //已经ready则进入如下的操作
                    transaction.traverseStatesWithBuffers([&amp;](const layer_state_t&amp; state) {
                        const bool frameNumberChanged = state.bufferData-&gt;flags.test(
                                BufferData::BufferDataChange::frameNumberChanged);
  //会把state放入到bufferLayersReadyToPresent这个map中
                        if (frameNumberChanged) {
                            bufferLayersReadyToPresent[state.surface] = state.bufferData-&gt;frameNumber;
                        } else {
                            // Barrier function only used for BBQ which always includes a frame number.
                            // This value only used for barrier logic.
                            bufferLayersReadyToPresent[state.surface] =
                                std::numeric_limits&lt;uint64_t&gt;::max();
                        }
                    });
      //最重要的放入到transactions
                    transactions.emplace_back(std::move(transaction));
                }
  //从mTransactionQueue这里移除
                mTransactionQueue.pop_front();
                ATRACE_INT(&#34;TransactionQueue&#34;, mTransactionQueue.size());
            }
 
 
            // Transactions with a buffer pending on a barrier may be on a different applyToken
            // than the transaction which satisfies our barrier. In fact this is the exact use case
            // that the primitive is designed for. This means we may first process
            // the barrier dependent transaction, determine it ineligible to complete
            // and then satisfy in a later inner iteration of flushPendingTransactionQueues.
            // The barrier dependent transaction was eligible to be presented in this frame
            // but we would have prevented it without case. To fix this we continually
            // loop through flushPendingTransactionQueues until we perform an iteration
            // where the number of transactionsPendingBarrier doesn&#39;t change. This way
            // we can continue to resolve dependency chains of barriers as far as possible.
            while (lastTransactionsPendingBarrier != transactionsPendingBarrier) {
                lastTransactionsPendingBarrier = transactionsPendingBarrier;
                transactionsPendingBarrier =
                    flushPendingTransactionQueues(transactions, bufferLayersReadyToPresent,
                        applyTokensWithUnsignaledTransactions,
                        /*tryApplyUnsignaled*/ false);
            }
 
 
            // We collected all transactions that could apply without latching unsignaled buffers.
            // If we are allowing latch unsignaled of some form, now it&#39;s the time to go over the
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
auto SurfaceFlinger::transactionIsReadyToBeApplied(TransactionState&amp; transaction,
        const FrameTimelineInfo&amp; info, bool isAutoTimestamp, int64_t desiredPresentTime,
        uid_t originUid, const Vector&lt;ComposerState&gt;&amp; states,
        const std::unordered_map&lt;
            sp&lt;IBinder&gt;, uint64_t, SpHash&lt;IBinder&gt;&gt;&amp; bufferLayersReadyToPresent,
        size_t totalTXapplied, bool tryApplyUnsignaled) const -&gt; TransactionReadiness {
    ATRACE_FORMAT(&#34;transactionIsReadyToBeApplied vsyncId: %&#34; PRId64, info.vsyncId);
    const nsecs_t expectedPresentTime = mExpectedPresentTime.load();
    // Do not present if the desiredPresentTime has not passed unless it is more than one second
    // in the future. We ignore timestamps more than 1 second in the future for stability reasons.
    if (!isAutoTimestamp &amp;&amp; desiredPresentTime &gt;= expectedPresentTime &amp;&amp;
        desiredPresentTime &lt; expectedPresentTime &#43; s2ns(1)) {
        ATRACE_NAME(&#34;not current&#34;);
        return TransactionReadiness::NotReady;
    }
 
 
    if (!mScheduler-&gt;isVsyncValid(expectedPresentTime, originUid)) {
        ATRACE_NAME(&#34;!isVsyncValid&#34;);
        return TransactionReadiness::NotReady;
    }
 
 
    // If the client didn&#39;t specify desiredPresentTime, use the vsyncId to determine the expected
    // present time of this transaction.
    if (isAutoTimestamp &amp;&amp; frameIsEarly(expectedPresentTime, info.vsyncId)) {
        ATRACE_NAME(&#34;frameIsEarly&#34;);
        return TransactionReadiness::NotReady;
    }
 
 
    bool fenceUnsignaled = false;
    auto queueProcessTime = systemTime();
    //关键部分对transaction的states进行挨个state遍历情况
    for (const ComposerState&amp; state : states) {
        const layer_state_t&amp; s = state.state;
 
 
        sp&lt;Layer&gt; layer = nullptr;
 //这里会判断到底有没有surface。没有的话就不会进入下面判断环节
        if (s.surface) {
            layer = fromHandle(s.surface).promote();
        } else if (s.hasBufferChanges()) {
            ALOGW(&#34;Transaction with buffer, but no Layer?&#34;);
            continue;
        }
        if (!layer) {
            continue;
        }
 
 
        if (s.hasBufferChanges() &amp;&amp; s.bufferData-&gt;hasBarrier &amp;&amp;
            ((layer-&gt;getDrawingState().frameNumber) &lt; s.bufferData-&gt;barrierFrameNumber)) {
            const bool willApplyBarrierFrame =
                (bufferLayersReadyToPresent.find(s.surface) != bufferLayersReadyToPresent.end()) &amp;&amp;
                (bufferLayersReadyToPresent.at(s.surface) &gt;= s.bufferData-&gt;barrierFrameNumber);
            if (!willApplyBarrierFrame) {
                ATRACE_NAME(&#34;NotReadyBarrier&#34;);
                return TransactionReadiness::NotReadyBarrier;
            }
        }
 
 
 //注意这里会有一个标志allowLatchUnsignaled意思是是否可以latch非signaled的buffer
        const bool allowLatchUnsignaled = tryApplyUnsignaled &amp;&amp;
                shouldLatchUnsignaled(layer, s, states.size(), totalTXapplied);
        ATRACE_FORMAT(&#34;%s allowLatchUnsignaled=%s&#34;, layer-&gt;getName().c_str(),
                      allowLatchUnsignaled ? &#34;true&#34; : &#34;false&#34;);
 
 
        const bool acquireFenceChanged = s.bufferData &amp;&amp;
                s.bufferData-&gt;flags.test(BufferData::BufferDataChange::fenceChanged) &amp;&amp;
                s.bufferData-&gt;acquireFence;
 //这里会进行关键的判断，判断state的bufferData-&gt;acquireFence是否已经signaled
        fenceUnsignaled = fenceUnsignaled ||
                (acquireFenceChanged &amp;&amp;
                 s.bufferData-&gt;acquireFence-&gt;getStatus() == Fence::Status::Unsignaled);
 
 
 //如果fenceUnsignaled属于fenceUnsignaled，allowLatchUnsignaled也为false
        if (fenceUnsignaled &amp;&amp; !allowLatchUnsignaled) {
            if (!transaction.sentFenceTimeoutWarning &amp;&amp;
                queueProcessTime - transaction.queueTime &gt; std::chrono::nanoseconds(4s).count()) {
                transaction.sentFenceTimeoutWarning = true;
                auto listener = s.bufferData-&gt;releaseBufferListener;
                if (listener) {
                    listener-&gt;onTransactionQueueStalled();
                }
            }
     //那么就代表不可以传递transaction，会返回NotReady
            ATRACE_NAME(&#34;fence unsignaled&#34;);
            return TransactionReadiness::NotReady;
        }
 
 
        if (s.hasBufferChanges()) {
            // If backpressure is enabled and we already have a buffer to commit, keep the
            // transaction in the queue.
            const bool hasPendingBuffer = bufferLayersReadyToPresent.find(s.surface) !=
                bufferLayersReadyToPresent.end();
     //如果前面已经有state放入了，则不在放入，会放入pending
            if (layer-&gt;backpressureEnabled() &amp;&amp; hasPendingBuffer &amp;&amp; isAutoTimestamp) {
                ATRACE_NAME(&#34;hasPendingBuffer&#34;);
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
bool SurfaceFlinger::applyTransactions(std::vector&lt;TransactionState&gt;&amp; transactions,
                                       int64_t vsyncId) {
    bool needsTraversal = false;
    // Now apply all transactions.
    //遍历每一个上面判断为ready的transaction
    for (auto&amp; transaction : transactions) {
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
        mTransactionTracing-&gt;addCommittedTransactions(transactions, vsyncId);
    }
    return needsTraversal;
}
```

#### SurfaceFlinger applyTransactionState

调用SurfaceFlinger的applyTransactionState方法：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
bool SurfaceFlinger::applyTransactionState(const FrameTimelineInfo&amp; frameTimelineInfo,
                                           Vector&lt;ComposerState&gt;&amp; states,
                                           const Vector&lt;DisplayState&gt;&amp; displays, uint32_t flags,
                                           const InputWindowCommands&amp; inputWindowCommands,
                                           const int64_t desiredPresentTime, bool isAutoTimestamp,
                                           const client_cache_t&amp; uncacheBuffer,
                                           const int64_t postTime, uint32_t permissions,
                                           bool hasListenerCallbacks,
                                           const std::vector&lt;ListenerCallbacks&gt;&amp; listenerCallbacks,
                                           int originPid, int originUid, uint64_t transactionId) {
    uint32_t transactionFlags = 0;
    //遍历是transaction的dispplays
    for (const DisplayState&amp; display : displays) {
 //遍历是否有dispplay相关的变化
        transactionFlags |= setDisplayStateLocked(display);
    }
 
 
    // start and end registration for listeners w/ no surface so they can get their callback.  Note
    // that listeners with SurfaceControls will start registration during setClientStateLocked
    // below.
    for (const auto&amp; listener : listenerCallbacks) {
        mTransactionCallbackInvoker.addEmptyTransaction(listener);
    }
 
 
    uint32_t clientStateFlags = 0;
    for (int i = 0; i &lt; states.size(); i&#43;&#43;) { //最关键方法开始遍历一个个的states
        ComposerState&amp; state = states.editItemAt(i);
 //调用到了setClientStateLocked方法
        clientStateFlags |= setClientStateLocked(frameTimelineInfo, state, desiredPresentTime,
                                                 isAutoTimestamp, postTime, permissions);
        if ((flags &amp; eAnimation) &amp;&amp; state.state.surface) {
            if (const auto layer = fromHandle(state.state.surface).promote()) {
                using LayerUpdateType = scheduler::LayerHistory::LayerUpdateType;
                mScheduler-&gt;recordLayerHistory(layer.get(),
                                               isAutoTimestamp ? 0 : desiredPresentTime,
                                               LayerUpdateType::AnimationTX);
            }
        }
    }
 
 
    transactionFlags |= clientStateFlags;
 
 
    if (permissions &amp; layer_state_t::Permission::ACCESS_SURFACE_FLINGER) {
        transactionFlags |= addInputWindowCommands(inputWindowCommands);
    } else if (!inputWindowCommands.empty()) {
        ALOGE(&#34;Only privileged callers are allowed to send input commands.&#34;);
    }
 
 
    if (uncacheBuffer.isValid()) {
        ClientCache::getInstance().erase(uncacheBuffer);
    }
 
 
    // If a synchronous transaction is explicitly requested without any changes, force a transaction
    // anyway. This can be used as a flush mechanism for previous async transactions.
    // Empty animation transaction can be used to simulate back-pressure, so also force a
    // transaction for empty animation transactions.
    if (transactionFlags == 0 &amp;&amp;
            ((flags &amp; eSynchronous) || (flags &amp; eAnimation))) {
        transactionFlags = eTransactionNeeded;
    }
 
 
    bool needsTraversal = false;
    if (transactionFlags) {
        if (mInterceptor-&gt;isEnabled()) {
            mInterceptor-&gt;saveTransaction(states, mCurrentState.displays, displays, flags,
                                          originPid, originUid, transactionId);
        }
 
 
        // We are on the main thread, we are about to preform a traversal. Clear the traversal bit
        // so we don&#39;t have to wake up again next frame to preform an unnecessary traversal.
        if (transactionFlags &amp; eTraversalNeeded) {
            transactionFlags = transactionFlags &amp; (~eTraversalNeeded);
            needsTraversal = true;
        }
        if (transactionFlags) {
            setTransactionFlags(transactionFlags);
        }
 
 
        if (flags &amp; eAnimation) {
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
uint32_t SurfaceFlinger::setClientStateLocked(const FrameTimelineInfo&amp; frameTimelineInfo,
                                              ComposerState&amp; composerState,
                                              int64_t desiredPresentTime, bool isAutoTimestamp,
                                              int64_t postTime, uint32_t permissions) {
    layer_state_t&amp; s = composerState.state;
    s.sanitize(permissions);
 
 
    std::vector&lt;ListenerCallbacks&gt; filteredListeners;
    for (auto&amp; listener : s.listeners) {
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
    sp&lt;Layer&gt; layer = nullptr;
    if (s.surface) {
        layer = fromHandle(s.surface).promote();
    } else {
        // The client may provide us a null handle. Treat it as if the layer was removed.
        ALOGW(&#34;Attempt to set client state with a null layer handle&#34;);
    }
    if (layer == nullptr) {
        for (auto&amp; [listener, callbackIds] : s.listeners) {
            mTransactionCallbackInvoker.registerUnpresentedCallbackHandle(
                    new CallbackHandle(listener, callbackIds, s.surface));
        }
        return 0;
    }
 
 
    // Only set by BLAST adapter layers
    if (what &amp; layer_state_t::eProducerDisconnect) {
        layer-&gt;onDisconnect();
    }
 
 
    if (what &amp; layer_state_t::ePositionChanged) {
        if (layer-&gt;setPosition(s.x, s.y)) {
            flags |= eTraversalNeeded;
        }
    }
    if (what &amp; layer_state_t::eLayerChanged) {
        // NOTE: index needs to be calculated before we update the state
        const auto&amp; p = layer-&gt;getParent();
        if (p == nullptr) {
            ssize_t idx = mCurrentState.layersSortedByZ.indexOf(layer);
            if (layer-&gt;setLayer(s.z) &amp;&amp; idx &gt;= 0) {
                mCurrentState.layersSortedByZ.removeAt(idx);
                mCurrentState.layersSortedByZ.add(layer);
                // we need traversal (state changed)
                // AND transaction (list changed)
                flags |= eTransactionNeeded|eTraversalNeeded;
            }
        } else {
            if (p-&gt;setChildLayer(layer, s.z)) {
                flags |= eTransactionNeeded|eTraversalNeeded;
            }
        }
    }
    if (what &amp; layer_state_t::eRelativeLayerChanged) {
        // NOTE: index needs to be calculated before we update the state
        const auto&amp; p = layer-&gt;getParent();
        const auto&amp; relativeHandle = s.relativeLayerSurfaceControl ?
                s.relativeLayerSurfaceControl-&gt;getHandle() : nullptr;
        if (p == nullptr) {
            ssize_t idx = mCurrentState.layersSortedByZ.indexOf(layer);
            if (layer-&gt;setRelativeLayer(relativeHandle, s.z) &amp;&amp;
                idx &gt;= 0) {
                mCurrentState.layersSortedByZ.removeAt(idx);
                mCurrentState.layersSortedByZ.add(layer);
                // we need traversal (state changed)
                // AND transaction (list changed)
                flags |= eTransactionNeeded|eTraversalNeeded;
            }
        } else {
            if (p-&gt;setChildRelativeLayer(layer, relativeHandle, s.z)) {
                flags |= eTransactionNeeded|eTraversalNeeded;
            }
        }
    }
    if (what &amp; layer_state_t::eSizeChanged) {
        if (layer-&gt;setSize(s.w, s.h)) {
            flags |= eTraversalNeeded;
        }
    }
    if (what &amp; layer_state_t::eAlphaChanged) {
        if (layer-&gt;setAlpha(s.alpha))
            flags |= eTraversalNeeded;
    }
    if (what &amp; layer_state_t::eColorChanged) {
        if (layer-&gt;setColor(s.color))
            flags |= eTraversalNeeded;
    }
    if (what &amp; layer_state_t::eColorTransformChanged) {
        if (layer-&gt;setColorTransform(s.colorTransform)) {
            flags |= eTraversalNeeded;
        }
    }
    if (what &amp; layer_state_t::eBackgroundColorChanged) {
        if (layer-&gt;setBackgroundColor(s.color, s.bgColorAlpha, s.bgColorDataspace)) {
            flags |= eTraversalNeeded;
        }
    }
    if (what &amp; layer_state_t::eMatrixChanged) {
        if (layer-&gt;setMatrix(s.matrix)) flags |= eTraversalNeeded;
    }
    if (what &amp; layer_state_t::eTransparentRegionChanged) {
        if (layer-&gt;setTransparentRegionHint(s.transparentRegion))
            flags |= eTraversalNeeded;
    }
    if (what &amp; layer_state_t::eFlagsChanged) {
        if (layer-&gt;setFlags(s.flags, s.mask)) flags |= eTraversalNeeded;
    }
    if (what &amp; layer_state_t::eCornerRadiusChanged) {
        if (layer-&gt;setCornerRadius(s.cornerRadius))
            flags |= eTraversalNeeded;
    }
    if (what &amp; layer_state_t::eBackgroundBlurRadiusChanged &amp;&amp; mSupportsBlur) {
        if (layer-&gt;setBackgroundBlurRadius(s.backgroundBlurRadius)) flags |= eTraversalNeeded;
    }
    if (what &amp; layer_state_t::eBlurRegionsChanged) {
        if (layer-&gt;setBlurRegions(s.blurRegions)) flags |= eTraversalNeeded;
    }
    if (what &amp; layer_state_t::eLayerStackChanged) {
        ssize_t idx = mCurrentState.layersSortedByZ.indexOf(layer);
        // We only allow setting layer stacks for top level layers,
        // everything else inherits layer stack from its parent.
        if (layer-&gt;hasParent()) {
            ALOGE(&#34;Attempt to set layer stack on layer with parent (%s) is invalid&#34;,
                  layer-&gt;getDebugName());
        } else if (idx &lt; 0) {
            ALOGE(&#34;Attempt to set layer stack on layer without parent (%s) that &#34;
                  &#34;that also does not appear in the top level layer list. Something&#34;
                  &#34; has gone wrong.&#34;,
                  layer-&gt;getDebugName());
        } else if (layer-&gt;setLayerStack(s.layerStack)) {
            mCurrentState.layersSortedByZ.removeAt(idx);
            mCurrentState.layersSortedByZ.add(layer);
            // we need traversal (state changed)
            // AND transaction (list changed)
            flags |= eTransactionNeeded | eTraversalNeeded | eTransformHintUpdateNeeded;
        }
    }
    if (what &amp; layer_state_t::eTransformChanged) {
        if (layer-&gt;setTransform(s.transform)) flags |= eTraversalNeeded;
    }
    if (what &amp; layer_state_t::eTransformToDisplayInverseChanged) {
        if (layer-&gt;setTransformToDisplayInverse(s.transformToDisplayInverse))
            flags |= eTraversalNeeded;
    }
    if (what &amp; layer_state_t::eCropChanged) {
        if (layer-&gt;setCrop(s.crop)) flags |= eTraversalNeeded;
    }
    if (what &amp; layer_state_t::eDataspaceChanged) {
        if (layer-&gt;setDataspace(s.dataspace)) flags |= eTraversalNeeded;
    }
    if (what &amp; layer_state_t::eHdrMetadataChanged) {
        if (layer-&gt;setHdrMetadata(s.hdrMetadata)) flags |= eTraversalNeeded;
    }
    if (what &amp; layer_state_t::eSurfaceDamageRegionChanged) {
        if (layer-&gt;setSurfaceDamageRegion(s.surfaceDamageRegion)) flags |= eTraversalNeeded;
    }
    if (what &amp; layer_state_t::eApiChanged) {
        if (layer-&gt;setApi(s.api)) flags |= eTraversalNeeded;
    }
    if (what &amp; layer_state_t::eSidebandStreamChanged) {
        if (layer-&gt;setSidebandStream(s.sidebandStream)) flags |= eTraversalNeeded;
    }
    if (what &amp; layer_state_t::eInputInfoChanged) {
        layer-&gt;setInputInfo(*s.windowInfoHandle-&gt;getInfo());
        flags |= eTraversalNeeded;
    }
    std::optional&lt;nsecs_t&gt; dequeueBufferTimestamp;
    if (what &amp; layer_state_t::eMetadataChanged) {
        dequeueBufferTimestamp = s.metadata.getInt64(METADATA_DEQUEUE_TIME);
 
 
        if (const int32_t gameMode = s.metadata.getInt32(METADATA_GAME_MODE, -1); gameMode != -1) {
            // The transaction will be received on the Task layer and needs to be applied to all
            // child layers. Child layers that are added at a later point will obtain the game mode
            // info through addChild().
            layer-&gt;setGameModeForTree(static_cast&lt;GameMode&gt;(gameMode));
        }
 
 
        if (layer-&gt;setMetadata(s.metadata)) flags |= eTraversalNeeded;
    }
    if (what &amp; layer_state_t::eColorSpaceAgnosticChanged) {
        if (layer-&gt;setColorSpaceAgnostic(s.colorSpaceAgnostic)) {
            flags |= eTraversalNeeded;
        }
    }
    if (what &amp; layer_state_t::eShadowRadiusChanged) {
        if (layer-&gt;setShadowRadius(s.shadowRadius)) flags |= eTraversalNeeded;
    }
    if (what &amp; layer_state_t::eFrameRateSelectionPriority) {
        if (layer-&gt;setFrameRateSelectionPriority(s.frameRateSelectionPriority)) {
            flags |= eTraversalNeeded;
        }
    }
    if (what &amp; layer_state_t::eFrameRateChanged) {
        const auto compatibility =
            Layer::FrameRate::convertCompatibility(s.frameRateCompatibility);
        const auto strategy =
            Layer::FrameRate::convertChangeFrameRateStrategy(s.changeFrameRateStrategy);
 
 
        if (layer-&gt;setFrameRate(
                Layer::FrameRate(Fps::fromValue(s.frameRate), compatibility, strategy))) {
          flags |= eTraversalNeeded;
        }
    }
    if (what &amp; layer_state_t::eFixedTransformHintChanged) {
        if (layer-&gt;setFixedTransformHint(s.fixedTransformHint)) {
            flags |= eTraversalNeeded | eTransformHintUpdateNeeded;
        }
    }
    if (what &amp; layer_state_t::eAutoRefreshChanged) {
        layer-&gt;setAutoRefresh(s.autoRefresh);
    }
    if (what &amp; layer_state_t::eDimmingEnabledChanged) {
        if (layer-&gt;setDimmingEnabled(s.dimmingEnabled)) flags |= eTraversalNeeded;
    }
    if (what &amp; layer_state_t::eTrustedOverlayChanged) {
        if (layer-&gt;setTrustedOverlay(s.isTrustedOverlay)) {
            flags |= eTraversalNeeded;
        }
    }
    if (what &amp; layer_state_t::eStretchChanged) {
        if (layer-&gt;setStretchEffect(s.stretchEffect)) {
            flags |= eTraversalNeeded;
        }
    }
    if (what &amp; layer_state_t::eBufferCropChanged) {
        if (layer-&gt;setBufferCrop(s.bufferCrop)) {
            flags |= eTraversalNeeded;
        }
    }
    if (what &amp; layer_state_t::eDestinationFrameChanged) {
        if (layer-&gt;setDestinationFrame(s.destinationFrame)) {
            flags |= eTraversalNeeded;
        }
    }
    if (what &amp; layer_state_t::eDropInputModeChanged) {
        if (layer-&gt;setDropInputMode(s.dropInputMode)) {
            flags |= eTraversalNeeded;
            mInputInfoChanged = true;
        }
    }
    // This has to happen after we reparent children because when we reparent to null we remove
    // child layers from current state and remove its relative z. If the children are reparented in
    // the same transaction, then we have to make sure we reparent the children first so we do not
    // lose its relative z order.
    if (what &amp; layer_state_t::eReparent) {
        bool hadParent = layer-&gt;hasParent();
        auto parentHandle = (s.parentSurfaceControlForChild)
                ? s.parentSurfaceControlForChild-&gt;getHandle()
                : nullptr;
        if (layer-&gt;reparent(parentHandle)) {
            if (!hadParent) {
                layer-&gt;setIsAtRoot(false);
                mCurrentState.layersSortedByZ.remove(layer);
            }
            flags |= eTransactionNeeded | eTraversalNeeded;
        }
    }
    std::vector&lt;sp&lt;CallbackHandle&gt;&gt; callbackHandles;
    if ((what &amp; layer_state_t::eHasListenerCallbacksChanged) &amp;&amp; (!filteredListeners.empty())) {
        for (auto&amp; [listener, callbackIds] : filteredListeners) {
            callbackHandles.emplace_back(new CallbackHandle(listener, callbackIds, s.surface));
        }
    }
 
 
    if (what &amp; layer_state_t::eBufferChanged) {  //如果发现有buffer变化
        std::shared_ptr&lt;renderengine::ExternalTexture&gt; buffer =
                getExternalTextureFromBufferData(*s.bufferData, layer-&gt;getDebugName());
 //把state的bufferData相关信息设置到了layer中去
        if (layer-&gt;setBuffer(buffer, *s.bufferData, postTime, desiredPresentTime, isAutoTimestamp,
                             dequeueBufferTimestamp, frameTimelineInfo)) {
            flags |= eTraversalNeeded;
        }
    } else if (frameTimelineInfo.vsyncId != FrameTimelineInfo::INVALID_VSYNC_ID) {
        layer-&gt;setFrameTimelineVsyncForBufferlessTransaction(frameTimelineInfo, postTime);
    }
 
 
    if (layer-&gt;setTransactionCompletedListeners(callbackHandles)) flags |= eTraversalNeeded;
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
    // effects of the State assignment don&#39;t happen with mStateLock held,
    // which can cause deadlocks.
    State drawingState(mDrawingState);
 
 
    Mutex::Autolock lock(mStateLock);
    mDebugInTransaction = systemTime();
 
 
    // Here we&#39;re guaranteed that some transaction flags are set
    // so we can call commitTransactionsLocked unconditionally.
    // We clear the flags with mStateLock held to guarantee that
    // mCurrentState won&#39;t change until the transaction is committed.
    modulateVsync(&amp;VsyncModulator::onTransactionCommit);
    commitTransactionsLocked(clearTransactionFlags(eTransactionMask));
 
 
    mDebugInTransaction = 0;
}
```

调用SurfaceFlinger的commitTransactionsLocked方法：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
void SurfaceFlinger::commitTransactionsLocked(uint32_t transactionFlags) {
    // Commit display transactions.
    const bool displayTransactionNeeded = transactionFlags &amp; eDisplayTransactionNeeded;
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
    if (transactionFlags &amp; (eTransformHintUpdateNeeded | eDisplayTransactionNeeded)) {
        // Layers and/or displays have changed, so update the transform hint for each layer.
        //
        // NOTE: we do this here, rather than when presenting the display so that
        // the hint is set before we acquire a buffer from the surface texture.
        //
        // NOTE: layer transactions have taken place already, so we use their
        // drawing state. However, SurfaceFlinger&#39;s own transaction has not
        // happened yet, so we must use the current state layer list
        // (soon to become the drawing state list).
        //
        sp&lt;const DisplayDevice&gt; hintDisplay;
        ui::LayerStack layerStack;
 
 
        mCurrentState.traverse([&amp;](Layer* layer) REQUIRES(mStateLock) {
            // NOTE: we rely on the fact that layers are sorted by
            // layerStack first (so we don&#39;t have to traverse the list
            // of displays for every layer).
            if (const auto filter = layer-&gt;getOutputFilter(); layerStack != filter.layerStack) {
                layerStack = filter.layerStack;
                hintDisplay = nullptr;
 
 
                // Find the display that includes the layer.
                for (const auto&amp; [token, display] : mDisplays) {
                    if (!display-&gt;getCompositionDisplay()-&gt;includesLayer(filter)) {
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
 
 
            layer-&gt;updateTransformHint(hintDisplay-&gt;getTransformHint());
        });
    }
 
 
    if (mLayersAdded) {
        mLayersAdded = false;
        // Layers have been added.
        mVisibleRegionsDirty = true;
    }
 
 
    // some layers might have been removed, so
    // we need to update the regions they&#39;re exposing.
    if (mLayersRemoved) {
        mLayersRemoved = false;
        mVisibleRegionsDirty = true;
        mDrawingState.traverseInZOrder([&amp;](Layer* layer) {
            if (mLayersPendingRemoval.indexOf(layer) &gt;= 0) {
                // this layer is not visible anymore
                Region visibleReg;
                visibleReg.set(layer-&gt;getScreenBounds());
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
                // 处于绘图状态，但不在当前状态
                processDisplayRemoved(displayToken);
            } else {
                // this display is in both lists. see if something changed.
 // 此显示在两个列表中。看看是否有变化。
                const DisplayDeviceState&amp; currentState = curr[j];
                const DisplayDeviceState&amp; drawingState = draw[i];
                processDisplayChanged(displayToken, currentState, drawingState);
            }
        }
 
 
        // find displays that were added
        // 查找已添加的显示
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

上面方法根据不同的调节调用如下方法：

1、调用processDisplayRemoved方法，处理显示器删除。

2、调用processDisplayChanged方法，处理显示器变更。

3、调用processDisplayAdded方法，处理显示器添加。

下面分别进行分析：

#### SurfaceFlinger processDisplayRemoved

调用processDisplayRemoved方法，处理显示器删除：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
void SurfaceFlinger::processDisplayRemoved(const wp&lt;IBinder&gt;&amp; displayToken) {
    auto display = getDisplayDeviceLocked(displayToken);
    if (display) {
        display-&gt;disconnect(); //与Display断开连接
 
 
        if (display-&gt;isVirtual()) {
            releaseVirtualDisplay(display-&gt;getVirtualId()); //如果是虚拟显示器，就进行资源释放
        } else {
            dispatchDisplayHotplugEvent(display-&gt;getPhysicalId(), false);  //分配显示器热插拔事件
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

#### SurfaceFlinger processDisplayChanged

调用processDisplayChanged方法，处理显示器变更：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
void SurfaceFlinger::processDisplayChanged(const wp&lt;IBinder&gt;&amp; displayToken,
                                           const DisplayDeviceState&amp; currentState,
                                           const DisplayDeviceState&amp; drawingState) {
    const sp&lt;IBinder&gt; currentBinder = IInterface::asBinder(currentState.surface);
    const sp&lt;IBinder&gt; drawingBinder = IInterface::asBinder(drawingState.surface);
 
 
    // Recreate the DisplayDevice if the surface or sequence ID changed.
    // 如果图面或序列 ID 发生更改，请重新创建 DisplayDevice。
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
            display-&gt;setLayerStack(currentState.layerStack); //设置显示器层堆栈
        }
        if (currentState.flags != drawingState.flags) {
            display-&gt;setFlags(currentState.flags); //设置显示器标志
        }
        if ((currentState.orientation != drawingState.orientation) ||
            (currentState.layerStackSpaceRect != drawingState.layerStackSpaceRect) ||
            (currentState.orientedDisplaySpaceRect != drawingState.orientedDisplaySpaceRect)) {
            display-&gt;setProjection(currentState.orientation, currentState.layerStackSpaceRect,
                                   currentState.orientedDisplaySpaceRect); //设置显示器投影
            if (isDisplayActiveLocked(display)) {
                mActiveDisplayTransformHint = display-&gt;getTransformHint();
            }
        }
        if (currentState.width != drawingState.width ||
            currentState.height != drawingState.height) {
            display-&gt;setDisplaySize(currentState.width, currentState.height); //设置显示器尺寸
 
 
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

### SurfaceFlinger doCommitTransactions

调用SurfaceFlinger的doCommitTransactions方法：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
void SurfaceFlinger::doCommitTransactions() {
    ATRACE_CALL();
 
 
    if (!mLayersPendingRemoval.isEmpty()) {
        // Notify removed layers now that they can&#39;t be drawn from
        for (const auto&amp; l : mLayersPendingRemoval) {
            // Ensure any buffers set to display on any children are released.
            if (l-&gt;isRemovedFromCurrentState()) {
                l-&gt;latchAndReleaseBuffer();
            }
 
 
            // If a layer has a parent, we allow it to out-live it&#39;s handle
            // with the idea that the parent holds a reference and will eventually
            // be cleaned up. However no one cleans up the top-level so we do so
            // here.
            if (l-&gt;isAtRoot()) {
                l-&gt;setIsAtRoot(false);
                mCurrentState.layersSortedByZ.remove(l);
            }
 
 
            // If the layer has been removed and has no parent, then it will not be reachable
            // when traversing layers on screen. Add the layer to the offscreenLayers set to
            // ensure we can copy its current to drawing state.
            if (!l-&gt;getParent()) {
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
    // clear the &#34;changed&#34; flags in current state
    mCurrentState.colorMatrixChanged = false;
 
 
    if (mVisibleRegionsDirty) {
        for (const auto&amp; rootLayer : mDrawingState.layersSortedByZ) {
            rootLayer-&gt;commitChildList();
        }
    }
 
 
    commitOffscreenLayers();
    if (mNumClones &gt; 0) {
        mDrawingState.traverse([&amp;](Layer* layer) { layer-&gt;updateMirrorInfo(); });
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
    // Display is now waiting on Layer 1&#39;s frame, which is behind layer 0&#39;s
    // second frame. But layer 0&#39;s second frame could be waiting on display.
    mDrawingState.traverse([&amp;](Layer* layer) {
        if (layer-&gt;clearTransactionFlags(eTransactionNeeded) || mForceTransactionDisplayChange) {
            const uint32_t flags = layer-&gt;doTransaction(0);
            if (flags &amp; Layer::eVisibleRegion) {
                mVisibleRegionsDirty = true;
            }
        }
 
 
 //判断是否有buffer
        if (layer-&gt;hasReadyFrame()) {
            frameQueued = true;
     //是否应该显示
            if (layer-&gt;shouldPresentNow(expectedPresentTime)) {
  //如果要显示放入到mLayersWithQueuedFrames队列
                mLayersWithQueuedFrames.emplace(layer);
            } else {
                ATRACE_NAME(&#34;!layer-&gt;shouldPresentNow()&#34;);
                layer-&gt;useEmptyDamage();
            }
        } else {
            layer-&gt;useEmptyDamage();
        }
    });
    //上面主要是为了从sf的mDrawingState遍历寻找出有buffer的layer
 
 
    mForceTransactionDisplayChange = false;
 
 
    // The client can continue submitting buffers for offscreen layers, but they will not
    // be shown on screen. Therefore, we need to latch and release buffers of offscreen
    // layers to ensure dequeueBuffer doesn&#39;t block indefinitely.
    for (Layer* offscreenLayer : mOffscreenLayers) {
        offscreenLayer-&gt;traverse(LayerVector::StateSet::Drawing,
                                         [&amp;](Layer* l) { l-&gt;latchAndReleaseBuffer(); });
    }
 
 
    //如果buffer队列不为空
    if (!mLayersWithQueuedFrames.empty()) {
        // mStateLock is needed for latchBuffer as LayerRejecter::reject()
        // writes to Layer current state. See also b/119481871
        Mutex::Autolock lock(mStateLock);
 
 
        for (const auto&amp; layer : mLayersWithQueuedFrames) {
     //进行关键的latchBuffer操作,然后newDataLatched设置成了true
            if (layer-&gt;latchBuffer(visibleRegions, latchTime, expectedPresentTime)) {
                mLayersPendingRefresh.push_back(layer); /这里有latchBuffer说明有buffer刷新，放入mLayersPendingRefresh
                newDataLatched = true;
            }
            layer-&gt;useSurfaceDamage();
        }
    }
 
 
    mVisibleRegionsDirty |= visibleRegions;
 
 
    // If we will need to wake up at some time in the future to deal with a
    // queued frame that shouldn&#39;t be displayed during this vsync period, wake
    // up during the next vsync period to check again.
    if (frameQueued &amp;&amp; (mLayersWithQueuedFrames.empty() || !newDataLatched)) {
        scheduleCommit(FrameHint::kNone);
    }
 
 
    // enter boot animation on first buffer latch
    if (CC_UNLIKELY(mBootStage == BootStage::BOOTLOADER &amp;&amp; newDataLatched)) {
        ALOGI(&#34;Enter boot animation&#34;);
        mBootStage = BootStage::BOOTANIMATION;
    }
 
 
    if (mNumClones &gt; 0) {
        mDrawingState.traverse([&amp;](Layer* layer) { layer-&gt;updateCloneBufferInfo(); });
    }
 
 
    // Only continue with the refresh if there is actually new work to do
    return !mLayersWithQueuedFrames.empty() &amp;&amp; newDataLatched;
}
```

### BufferLayer latchBuffer

调用layer的latchBuffer方法：

```cpp
//frameworks/native/services/surfaceflinger/BufferLayer.cpp
bool BufferLayer::latchBuffer(bool&amp; recomputeVisibleRegions, nsecs_t latchTime,
                              nsecs_t expectedPresentTime) {
    ATRACE_CALL();
 
 
    bool refreshRequired = latchSidebandStream(recomputeVisibleRegions);
 
 
    if (refreshRequired) {
        return refreshRequired;
    }
 
 
    // If the head buffer&#39;s acquire fence hasn&#39;t signaled yet, return and
    // try again later
    // 如果头部缓冲区的采集围栏尚未发出信号，请返回并稍后重试
    if (!fenceHasSignaled()) {
        ATRACE_NAME(&#34;!fenceHasSignaled()&#34;);
        mFlinger-&gt;onLayerUpdate(); // (692) SurfaceFlinger onLayerUpdate流程分析 | 知识管理 - PingCode 
        return false;
    }
 
 
    // Capture the old state of the layer for comparisons later
    const State&amp; s(getDrawingState());
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
        uint32_t bufWidth = mBufferInfo.mBuffer-&gt;getWidth();
        uint32_t bufHeight = mBufferInfo.mBuffer-&gt;getHeight();
        if (bufWidth != oldBufferInfo.mBuffer-&gt;getWidth() ||
            bufHeight != oldBufferInfo.mBuffer-&gt;getHeight()) {
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
    for (auto&amp; layer : mLayersPendingRefresh) {
        Region visibleReg;
        visibleReg.set(layer-&gt;getScreenBounds());
        invalidateLayerStack(layer, visibleReg); //刷新一下display的dirtyRegion
    }
    mLayersPendingRefresh.clear();
}
```

### SurfaceFlinger invalidateLayerStack

调用SurfaceFlinger的invalidateLayerStack方法：

```cpp
//frameworks/native/services/surfaceflinger/Surfaceflinger.cpp
void SurfaceFlinger::invalidateLayerStack(const sp&lt;const Layer&gt;&amp; layer, const Region&amp; dirty) {
    for (const auto&amp; [token, displayDevice] : FTL_FAKE_GUARD(mStateLock, mDisplays)) {
        auto display = displayDevice-&gt;getCompositionDisplay();
        if (display-&gt;includesLayer(layer-&gt;getOutputFilter())) { //寻找到对应的display
            display-&gt;editState().dirtyRegion.orSelf(dirty); //设置到了display-&gt;editState中
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
 
 
    std::vector&lt;WindowInfo&gt; windowInfos;
    std::vector&lt;DisplayInfo&gt; displayInfos;
    bool updateWindowInfo = false;
    if (mVisibleRegionsDirty || mInputInfoChanged) {
        mInputInfoChanged = false;
        updateWindowInfo = true;
 //进行遍历整个系统的layer和display转变成windowInfos，displayInfos信息
        buildWindowInfos(windowInfos, displayInfos);
    }
    if (!updateWindowInfo &amp;&amp; mInputWindowCommands.empty()) {
        return;
    }
    //这里放入子线程进行相对应跨进程通讯
    BackgroundExecutor::getInstance().sendCallbacks({[updateWindowInfo,
                                                      windowInfos = std::move(windowInfos),
                                                      displayInfos = std::move(displayInfos),
                                                      inputWindowCommands =
                                                              std::move(mInputWindowCommands),
                                                      inputFlinger = mInputFlinger, this]() {
        ATRACE_NAME(&#34;BackgroundExecutor::updateInputFlinger&#34;);
        if (updateWindowInfo) {
     //这里调用是mWindowInfosListenerInvoker进行的windowInfos, displayInfos进行跨进程传递
            mWindowInfosListenerInvoker-&gt;windowInfosChanged(windowInfos, displayInfos,
                                                            inputWindowCommands.syncInputWindows);
        } else if (inputWindowCommands.syncInputWindows) {
            // If the caller requested to sync input windows, but there are no
            // changes to input windows, notify immediately.
            windowInfosReported();
        }
        for (const auto&amp; focusRequest : inputWindowCommands.focusRequests) {
     //直接调用inputFlinger的bpbinder跨进程设置setFocusedWindow
            inputFlinger-&gt;setFocusedWindow(focusRequest);
        }
    }});
 
 
    mInputWindowCommands.clear();
}
```


---

> 作者: [wentao](https://github.com/huang8604)  
> URL: https://huang8604.github.io/posts/android13-surfaceflinger-commit%E6%8F%90%E4%BA%A4%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-csdn%E5%8D%9A%E5%AE%A2/  

