# Android13 Choreographer PostCallback流程分析_choreographer.postcallback-CSDN博客

Choreographer的postCallback()方法用于将一个任务添加到Choreographer的任务队列中，以便在下一帧绘制之前执行，代码如下：

```java
//frameworks/base/core/java/android/view/Choreographer.java
public final class Choreographer {
    public void postCallback(int callbackType, Runnable action, Object token) {
        postCallbackDelayed(callbackType, action, token, 0);
    }
}
```

调用Choreographer的postCallbackDelayed方法：

```java
//frameworks/base/core/java/android/view/Choreographer.java
public final class Choreographer {
    public void postCallbackDelayed(int callbackType,
            Runnable action, Object token, long delayMillis) {
        if (action == null) {
            throw new IllegalArgumentException("action must not be null");
        }
        if (callbackType < 0 || callbackType > CALLBACK_LAST) {
            throw new IllegalArgumentException("callbackType is invalid");
        }
 
 
        postCallbackDelayedInternal(callbackType, action, token, delayMillis);
    }
}
```

调用Choreographer的postCallbackDelayedInternal方法：

```java
//frameworks/base/core/java/android/view/Choreographer.java
public final class Choreographer {
    private void postCallbackDelayedInternal(int callbackType,
            Object action, Object token, long delayMillis) {
        if (DEBUG_FRAMES) {
            Log.d(TAG, "PostCallback: type=" + callbackType
                    + ", action=" + action + ", token=" + token
                    + ", delayMillis=" + delayMillis);
        }
 
 
        synchronized (mLock) {
     //添加类型为callbackType的CallbackQueue（将要执行的回调封装而成）
            final long now = SystemClock.uptimeMillis();
            final long dueTime = now + delayMillis;
            mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);
 
 
            if (dueTime <= now) {
  //立即执行
                scheduleFrameLocked(now);
            } else {
  //异步回调延迟执行
                Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
                msg.arg1 = callbackType;
                msg.setAsynchronous(true);
                mHandler.sendMessageAtTime(msg, dueTime);
            }
        }
    }
}
```

调用Choreographer的scheduleFrameLocked方法：

```java
//frameworks/base/core/java/android/view/Choreographer.java
public final class Choreographer {
    private void scheduleFrameLocked(long now) {
        if (!mFrameScheduled) {
            mFrameScheduled = true;
            if (USE_VSYNC) {
                if (DEBUG_FRAMES) {
                    Log.d(TAG, "Scheduling next frame on vsync.");
                }
 
 
                // If running on the Looper thread, then schedule the vsync immediately,
                // otherwise post a message to schedule the vsync from the UI thread
                // as soon as possible.
  //检测当前的Looper线程是不是主线程
                if (isRunningOnLooperThreadLocked()) {
      //当运行在Looper线程，则立刻调度vsync
                    scheduleVsyncLocked();
                } else {
      //切换到主线程，调度vsync
                    Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                    msg.setAsynchronous(true);
                    mHandler.sendMessageAtFrontOfQueue(msg);
                }
            } else {
  //如果没有VSYNC的同步，则发送消息刷新画面
                final long nextFrameTime = Math.max(
                        mLastFrameTimeNanos / TimeUtils.NANOS_PER_MS + sFrameDelay, now);
                if (DEBUG_FRAMES) {
                    Log.d(TAG, "Scheduling next frame in " + (nextFrameTime - now) + " ms.");
                }
                Message msg = mHandler.obtainMessage(MSG_DO_FRAME);
                msg.setAsynchronous(true);
                mHandler.sendMessageAtTime(msg, nextFrameTime);
            }
        }
    }
}
```

调用Choreographer的scheduleVsyncLocked方法：

```java
//frameworks/base/core/java/android/view/Choreographer.java
public final class Choreographer {
    private final FrameDisplayEventReceiver mDisplayEventReceiver;
    private void scheduleVsyncLocked() {
        try {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Choreographer#scheduleVsyncLocked");
            mDisplayEventReceiver.scheduleVsync();
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
}
```

## DisplayEventReceiver scheduleVsync

调用mDisplayEventReceiver(FrameDisplayEventReceiver)的scheduleVsync方法，FrameDisplayEventReceiver继承DisplayEventReceiver，实际调用DisplayEventReceiver的scheduleVsync方法：

```java
//frameworks/base/core/java/android/view/DisplayEventReceiver.java
public abstract class DisplayEventReceiver {
    public void scheduleVsync() {
        if (mReceiverPtr == 0) {
            Log.w(TAG, "Attempted to schedule a vertical sync pulse but the display event "
                    + "receiver has already been disposed.");
        } else {
            nativeScheduleVsync(mReceiverPtr);
        }
    }
}
```

### nativeScheduleVsync

调用nativeScheduleVsync方法，nativeScheduleVsync是一个[native方法](https://so.csdn.net/so/search?q=native%E6%96%B9%E6%B3%95\&spm=1001.2101.3001.7020)，在android\_view\_DisplayEventReceiver.cpp中实现：

```cpp
//frameworks/base/core/jni/android_view_DisplayEventReceiver.cpp
static void nativeScheduleVsync(JNIEnv* env, jclass clazz, jlong receiverPtr) {
    sp<NativeDisplayEventReceiver> receiver =
            reinterpret_cast<NativeDisplayEventReceiver*>(receiverPtr);
    status_t status = receiver->scheduleVsync(); //安排垂直同步
    if (status) {
        String8 message;
        message.appendFormat("Failed to schedule next vertical sync pulse.  status=%d", status);
        jniThrowRuntimeException(env, message.string());
    }
}
```

#### DisplayEventDispatcher scheduleVsync

调用DisplayEventDispatcher的scheduleVsync方法：

````cpp
//frameworks/native/libs/gui/DisplayEventDispatcher.cpp
status_t DisplayEventDispatcher::scheduleVsync() {
    if (!mWaitingForVsync) {
        ALOGV("dispatcher %p ~ Scheduling vsync.", this);
 
 
        // Drain all pending events.
        nsecs_t vsyncTimestamp;
        PhysicalDisplayId vsyncDisplayId;
        uint32_t vsyncCount;
        VsyncEventData vsyncEventData;
        if (processPendingEvents(&vsyncTimestamp, &vsyncDisplayId, &vsyncCount, &vsyncEventData)) {
            ALOGE("dispatcher %p ~ last event processed while scheduling was for %" PRId64 "", this,
                  ns2ms(static_cast<nsecs_t>(vsyncTimestamp)));
        }
 
 
        status_t status = mReceiver.requestNextVsync();
        if (status) {
            ALOGW("Failed to request next vsync, status=%d", status);
            return status;
        }
 
 
        mWaitingForVsync = true;
        mLastScheduleVsyncTime = systemTime(SYSTEM_TIME_MONOTONIC);
    }
    return OK;
}```

调用processPendingEvents方法：

```cpp
//frameworks/native/libs/gui/DisplayEventDispatcher.cpp
bool DisplayEventDispatcher::processPendingEvents(nsecs_t* outTimestamp,
                                                  PhysicalDisplayId* outDisplayId,
                                                  uint32_t* outCount,
                                                  VsyncEventData* outVsyncEventData) {
    bool gotVsync = false;
    DisplayEventReceiver::Event buf[EVENT_BUFFER_SIZE];
    ssize_t n;
    while ((n = mReceiver.getEvents(buf, EVENT_BUFFER_SIZE)) > 0) {
        ALOGV("dispatcher %p ~ Read %d events.", this, int(n));
        mFrameRateOverrides.reserve(n);
        for (ssize_t i = 0; i < n; i++) {
            const DisplayEventReceiver::Event& ev = buf[i];
            switch (ev.header.type) {
                case DisplayEventReceiver::DISPLAY_EVENT_VSYNC:
                    // Later vsync events will just overwrite the info from earlier
                    // ones. That's fine, we only care about the most recent.
                    gotVsync = true;
                    *outTimestamp = ev.header.timestamp;
                    *outDisplayId = ev.header.displayId;
                    *outCount = ev.vsync.count;
                    *outVsyncEventData = ev.vsync.vsyncData;
                    break;
                case DisplayEventReceiver::DISPLAY_EVENT_HOTPLUG:
                    dispatchHotplug(ev.header.timestamp, ev.header.displayId, ev.hotplug.connected);
                    break;
                case DisplayEventReceiver::DISPLAY_EVENT_MODE_CHANGE:
                    dispatchModeChanged(ev.header.timestamp, ev.header.displayId,
                                        ev.modeChange.modeId, ev.modeChange.vsyncPeriod);
                    break;
                case DisplayEventReceiver::DISPLAY_EVENT_NULL:
                    dispatchNullEvent(ev.header.timestamp, ev.header.displayId);
                    break;
                case DisplayEventReceiver::DISPLAY_EVENT_FRAME_RATE_OVERRIDE:
                    mFrameRateOverrides.emplace_back(ev.frameRateOverride);
                    break;
                case DisplayEventReceiver::DISPLAY_EVENT_FRAME_RATE_OVERRIDE_FLUSH:
                    dispatchFrameRateOverrides(ev.header.timestamp, ev.header.displayId,
                                               std::move(mFrameRateOverrides));
                    break;
                default:
                    ALOGW("dispatcher %p ~ ignoring unknown event type %#x", this, ev.header.type);
                    break;
            }
        }
    }
    if (n < 0) {
        ALOGW("Failed to get events from display event dispatcher, status=%d", status_t(n));
    }
    return gotVsync;
}
````


---

> 作者: [wentao](https://github.com/huang8604)  
> URL: https://huang8604.github.io/posts/android13-choreographer-postcallback%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-csdn%E5%8D%9A%E5%AE%A2/  

