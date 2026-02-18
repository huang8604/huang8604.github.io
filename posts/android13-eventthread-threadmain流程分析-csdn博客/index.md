# Android13 EventThread ThreadMain流程分析_eventthread::threadmain-CSDN博客

EventThread的threadMain方法无限循环处理pendingEvents,对Vsync类型的Event分发到消费者，通过往消费者的FD写数据，通知APP有Vsync信号到来。pendingEvents中的消息处理完了，分发线程等待mCondition的通知，EventThread的threadMain方法代码如下：

```cpp
//frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
std::vector<DisplayEventReceiver::Event> mPendingEvents;
void EventThread::threadMain(std::unique_lock<std::mutex>& lock) {
    // consumers 表示即将消费事件的 connection 集合
    DisplayEventConsumers consumers;
 
 
    // 状态值不等于 State::Quit 则一直循环遍历，死循环
    while (mState != State::Quit) {
        std::optional<DisplayEventReceiver::Event> event;
 
 
        // Determine next event to dispatch.
 //  确定下一个要调度的 Event
        if (!mPendingEvents.empty()) {
            event = mPendingEvents.front(); // 获取头部 Event
            mPendingEvents.pop_front(); // 将头部 Event 弹出
 
 
            switch (event->header.type) { // 根据 Event 类型分别对应处理
                case DisplayEventReceiver::DISPLAY_EVENT_HOTPLUG:  //Event类型为DISPLAY_EVENT_HOTPLUG，即当显示设备（如显示器）被插入或拔出时产生
                    if (event->hotplug.connected && !mVSyncState) {
                        mVSyncState.emplace(event->header.displayId);
                    } else if (!event->hotplug.connected && mVSyncState &&
                               mVSyncState->displayId == event->header.displayId) {
                        mVSyncState.reset();
                    }
                    break;
 
 
                case DisplayEventReceiver::DISPLAY_EVENT_VSYNC:  //Event类型为DISPLAY_EVENT_VSYNC，即Vsync（ 垂直同步）事件
                    if (mInterceptVSyncsCallback) {
                        mInterceptVSyncsCallback(event->header.timestamp);
                    }
                    break;
            }
        }
 
 
        // 标志位：是否有 VSync 请求，默认 false
        bool vsyncRequested = false;
 
 
        // Find connections that should consume this event.
 // 循环遍历存储 EventThreadConnection 的 vector 容器 mDisplayEventConnections，查找要消费事件的连接
         // begin()函数用于返回指向向量容器的第一个元素的迭代器
        auto it = mDisplayEventConnections.begin();
        while (it != mDisplayEventConnections.end()) {
            if (const auto connection = it->promote()) { // promote 下面引用有介绍
  // 如果有一个 connection 的 vsyncRequest 不为 None 则 vsyncRequested 为 true
                vsyncRequested |= connection->vsyncRequest != VSyncRequest::None;
 
 
  // event 不为空且 shouldConsumeEvent() 返回 true 则将 connection 加入到 consumers 等待消费 event
  // shouldConsumeEvent() 方法作用：对于 VSync 类型的事件，只要 VSyncRequest 的类型不是 None 就返回 true
                if (event && shouldConsumeEvent(*event, connection)) {
                    consumers.push_back(connection);
                }
 
 
                ++it;
            } else {
  // 获取不到 connection 则从 mDisplayEventConnections 移除
                it = mDisplayEventConnections.erase(it);
            }
        }
 
 
        // consumers 不为空即当前 Event 有 EventThreadConnection 来消费
        if (!consumers.empty()) {
            dispatchEvent(*event, consumers);
     // 分发完清空
            consumers.clear();
        }
 
 
        State nextState;
        if (mVSyncState && vsyncRequested) { // 有 VSync 请求
     // 调用 dispatchEvent() 方法遍历 consumers 为其每个 EventThreadConnection 分发事件
            nextState = mVSyncState->synthetic ? State::SyntheticVSync : State::VSync;
        } else {
            ALOGW_IF(!mVSyncState, "Ignoring VSYNC request while display is disconnected");
    // 显示器熄屏或没有连接、忽略 VSync 请求
            nextState = State::Idle;
        }
 
 
 // mState 值默认为 State::Idle，与 nextState 不一致，则分情况讨论
        if (mState != nextState) {
            if (mState == State::VSync) {
         // 当前状态为 State::VSync，则调用 mVSyncSource 的 setVSyncEnabled 并传入 false
                mVSyncSource->setVSyncEnabled(false);
            } else if (nextState == State::VSync) {
  // nextState 状态为 State::VSync，则调用 mVSyncSource 的 setVSyncEnabled 并传入 true
                mVSyncSource->setVSyncEnabled(true);
            }
 
 
            mState = nextState;
        }
 
 
        if (event) { // 还有事件则继续循环遍历
            continue;
        }
 
 
        // Wait for event or client registration/request.
 // 没有事件且当前状态为：State::Idle，则线程继续等待事件或客户端注册/请求
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
                    std::string debugInfo = "VsyncSource debug info:\n";
                    mVSyncSource->dump(debugInfo);
                    // Log the debug info line-by-line to avoid logcat overflow
                    auto pos = debugInfo.find('\n');
                    while (pos != std::string::npos) {
                        ALOGW("%s", debugInfo.substr(0, pos).c_str());
                        debugInfo = debugInfo.substr(pos + 1);
                        pos = debugInfo.find('\n');
                    }
                }
 
 
                LOG_FATAL_IF(!mVSyncState);
                const auto now = systemTime(SYSTEM_TIME_MONOTONIC);
                const auto deadlineTimestamp = now + timeout.count();
                const auto expectedVSyncTime = deadlineTimestamp + timeout.count();
 // 将伪造的 VSync 放入 mPendingEvents 准备分发
                mPendingEvents.push_back(makeVSync(mVSyncState->displayId, now,
                                                   ++mVSyncState->count, expectedVSyncTime,
                                                   deadlineTimestamp));
            }
        }
    }
}
```

## EventThread dispatchEvent

调用EventThread的dispatchEvent方法：

```cpp
//frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
void EventThread::dispatchEvent(const DisplayEventReceiver::Event& event,
                                const DisplayEventConsumers& consumers) {
    // 循环获取 DisplayEventConsumers 中保存每个 EventThreadConnection
    for (const auto& consumer : consumers) {
        DisplayEventReceiver::Event copy = event;
        if (event.header.type == DisplayEventReceiver::DISPLAY_EVENT_VSYNC) {
     // 获取 VSync 信号周期即：帧间隔
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

### EventThreadConnection postEvent

分发之调用的consumer->postEvent(copy)，这个consumer就是上面加入的EventThreadConnection对象。它的postEvent如下：

```cpp
//frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
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

#### DisplayEventReceiver sendEvents

mChannel 的类型是BitTube，保存的是EventThreadConnection中记录的文件描述符,这个文件描述符会复制到连接过来的app进程.然后调用DisplayEventReceiver::sendEvents(\&mChannel, \&event, 1)方法：

```cpp
//frameworks/native/libs/gui/DisplayEventReceiver.cpp
ssize_t DisplayEventReceiver::sendEvents(gui::BitTube* dataChannel,
        Event const* events, size_t count)
{
    return gui::BitTube::sendObjects(dataChannel, events, count);
}
```

##### BitTube sendObjects

调用BitTube调用sendObjects方法：

```cpp
//frameworks/native/libs/gui/BitTube.cpp
ssize_t BitTube::sendObjects(BitTube* tube, void const* events, size_t count, size_t objSize) {
    const char* vaddr = reinterpret_cast<const char*>(events);
    // 调用 BitTube 的 write() 方法来写入数据
    ssize_t size = tube->write(vaddr, count * objSize);
 
 
    // should never happen because of SOCK_SEQPACKET
    LOG_ALWAYS_FATAL_IF((size >= 0) && (size % static_cast<ssize_t>(objSize)),
                        "BitTube::sendObjects(count=%zu, size=%zu), res=%zd (partial events were "
                        "sent!)",
                        count, objSize, size);
 
 
    // ALOGE_IF(size<0, "error %d sending %d events", size, count);
    return size < 0 ? size : size / static_cast<ssize_t>(objSize);
}
```

调用BitTube的write方法：

```cpp
//frameworks/native/libs/gui/BitTube.cpp
ssize_t BitTube::write(void const* vaddr, size_t size) {
    ssize_t err, len;
    do {
 // 通过文件描述符 mSendFd 关联的 socket 管道将数据发送到 vaddr 指定的一段
 // size 大小的缓冲区中，并返回实际发送的数据大小：len
        len = ::send(mSendFd, vaddr, size, MSG_DONTWAIT | MSG_NOSIGNAL);
        // cannot return less than size, since we're using SOCK_SEQPACKET
        err = len < 0 ? errno : 0;
    } while (err == EINTR);
    return err == 0 ? len : -err;
}
```

通过调用系统函数send向与消费者关联的文件描述符FD发送信号，于是完成Vsync的分发，分发的消息在Choreographer的postCallback方法中处理：

[Android13 Choreographer postCallback流程分析-CSDN博客](/Android13%20Choreographer%20postCallback%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-CSDN%E5%8D%9A%E5%AE%A2)


---

> 作者: [wentao](https://github.com/huang8604)  
> URL: https://huang8604.github.io/posts/android13-eventthread-threadmain%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-csdn%E5%8D%9A%E5%AE%A2/  

