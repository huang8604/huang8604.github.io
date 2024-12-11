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
            throw new IllegalArgumentException(&#34;action must not be null&#34;);
        }
        if (callbackType &lt; 0 || callbackType &gt; CALLBACK_LAST) {
            throw new IllegalArgumentException(&#34;callbackType is invalid&#34;);
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
            Log.d(TAG, &#34;PostCallback: type=&#34; &#43; callbackType
                    &#43; &#34;, action=&#34; &#43; action &#43; &#34;, token=&#34; &#43; token
                    &#43; &#34;, delayMillis=&#34; &#43; delayMillis);
        }
 
 
        synchronized (mLock) {
     //添加类型为callbackType的CallbackQueue（将要执行的回调封装而成）
            final long now = SystemClock.uptimeMillis();
            final long dueTime = now &#43; delayMillis;
            mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);
 
 
            if (dueTime &lt;= now) {
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
                    Log.d(TAG, &#34;Scheduling next frame on vsync.&#34;);
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
                        mLastFrameTimeNanos / TimeUtils.NANOS_PER_MS &#43; sFrameDelay, now);
                if (DEBUG_FRAMES) {
                    Log.d(TAG, &#34;Scheduling next frame in &#34; &#43; (nextFrameTime - now) &#43; &#34; ms.&#34;);
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
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, &#34;Choreographer#scheduleVsyncLocked&#34;);
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
            Log.w(TAG, &#34;Attempted to schedule a vertical sync pulse but the display event &#34;
                    &#43; &#34;receiver has already been disposed.&#34;);
        } else {
            nativeScheduleVsync(mReceiverPtr);
        }
    }
}
```

### nativeScheduleVsync

调用nativeScheduleVsync方法，nativeScheduleVsync是一个[native方法](https://so.csdn.net/so/search?q=native%E6%96%B9%E6%B3%95\&amp;spm=1001.2101.3001.7020)，在android\_view\_DisplayEventReceiver.cpp中实现：

```cpp
//frameworks/base/core/jni/android_view_DisplayEventReceiver.cpp
static void nativeScheduleVsync(JNIEnv* env, jclass clazz, jlong receiverPtr) {
    sp&lt;NativeDisplayEventReceiver&gt; receiver =
            reinterpret_cast&lt;NativeDisplayEventReceiver*&gt;(receiverPtr);
    status_t status = receiver-&gt;scheduleVsync(); //安排垂直同步
    if (status) {
        String8 message;
        message.appendFormat(&#34;Failed to schedule next vertical sync pulse.  status=%d&#34;, status);
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
        ALOGV(&#34;dispatcher %p ~ Scheduling vsync.&#34;, this);
 
 
        // Drain all pending events.
        nsecs_t vsyncTimestamp;
        PhysicalDisplayId vsyncDisplayId;
        uint32_t vsyncCount;
        VsyncEventData vsyncEventData;
        if (processPendingEvents(&amp;vsyncTimestamp, &amp;vsyncDisplayId, &amp;vsyncCount, &amp;vsyncEventData)) {
            ALOGE(&#34;dispatcher %p ~ last event processed while scheduling was for %&#34; PRId64 &#34;&#34;, this,
                  ns2ms(static_cast&lt;nsecs_t&gt;(vsyncTimestamp)));
        }
 
 
        status_t status = mReceiver.requestNextVsync();
        if (status) {
            ALOGW(&#34;Failed to request next vsync, status=%d&#34;, status);
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
    while ((n = mReceiver.getEvents(buf, EVENT_BUFFER_SIZE)) &gt; 0) {
        ALOGV(&#34;dispatcher %p ~ Read %d events.&#34;, this, int(n));
        mFrameRateOverrides.reserve(n);
        for (ssize_t i = 0; i &lt; n; i&#43;&#43;) {
            const DisplayEventReceiver::Event&amp; ev = buf[i];
            switch (ev.header.type) {
                case DisplayEventReceiver::DISPLAY_EVENT_VSYNC:
                    // Later vsync events will just overwrite the info from earlier
                    // ones. That&#39;s fine, we only care about the most recent.
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
                    ALOGW(&#34;dispatcher %p ~ ignoring unknown event type %#x&#34;, this, ev.header.type);
                    break;
            }
        }
    }
    if (n &lt; 0) {
        ALOGW(&#34;Failed to get events from display event dispatcher, status=%d&#34;, status_t(n));
    }
    return gotVsync;
}
````


---

> 作者: [wentao](https://github.com/huang8604)  
> URL: https://huang8604.github.io/posts/android13-choreographer-postcallback%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-csdn%E5%8D%9A%E5%AE%A2/  

