# Android13 Surface DequeueBuffer流程分析-CSDN博客

Surface::dequeueBuffer是Android系统中Surface类的一个方法，用于从Surface中获取一个可用的Buffer。它通常在图形渲染或视频播放等场景中使用。

Surface::dequeueBuffer的调用地方可以有多个，具体取决于应用程序的实现和使用场景。以下是一些可能的调用地方：

1. 图形渲染引擎：在图形渲染引擎中，Surface::dequeueBuffer通常用于获取一个可用的绘制缓冲区，以便进行图形绘制操作。这样可以实现流畅的图形渲染效果。
2. 视频播放器：在视频播放器中，Surface::dequeueBuffer可以用于获取一个可用的视频帧缓冲区，以便将视频数据解码并显示在屏幕上。这样可以实现流畅的视频播放效果。
3. 图像处理应用：在图像处理应用中，Surface::dequeueBuffer可以用于获取一个可用的图像缓冲区，以便进行图像处理操作，如滤镜、特效等。这样可以实现实时的图像处理效果。
4. 游戏引擎：在游戏引擎中，Surface::dequeueBuffer可以用于获取一个可用的游戏画面缓冲区，以便进行游戏画面的渲染和更新。这样可以实现流畅的游戏画面效果。

Surface的dequeueBuffer方法的作用是应用程序一端请求绘制图像时，向BufferQueue中申请一块可用的GraphicBuffer，有了这个buffer就可以绘制图像数据了，生产者在生成内容的时候，就会有这么一个过程；

1. 生产者向 BufferQueue 中申请 slot（缓冲槽）
2. 生产者拿到 slot，但是 slot 并没有关联对应的 GraphicBuffer（缓冲区）
3. 生产者创建一个缓冲区，并将它与缓冲槽相关联。

下面从代码角度进行分析：

```cpp
//framework/native/libs/gui/Surface.cpp
sp&lt;IGraphicBufferProducer&gt; mGraphicBufferProducer;
class Surface : public ANativeObjectBase&lt;ANativeWindow, Surface, RefBase&gt; {
int Surface::dequeueBuffer(android_native_buffer_t** buffer, int* fenceFd) {
    ATRACE_CALL();
    ALOGV(&#34;Surface::dequeueBuffer&#34;);
 
 
    IGraphicBufferProducer::DequeueBufferInput dqInput;
    {
        Mutex::Autolock lock(mMutex);
        if (mReportRemovedBuffers) {
            mRemovedBuffers.clear();
        }
 
 
        getDequeueBufferInputLocked(&amp;dqInput);
 
 
        if (mSharedBufferMode &amp;&amp; mAutoRefresh &amp;&amp; mSharedBufferSlot !=
                BufferItem::INVALID_BUFFER_SLOT) {
            sp&lt;GraphicBuffer&gt;&amp; gbuf(mSlots[mSharedBufferSlot].buffer);
            if (gbuf != nullptr) {
                *buffer = gbuf.get();
                *fenceFd = -1;
                return OK;
            }
        }
    } // Drop the lock so that we can still touch the Surface while blocking in IGBP::dequeueBuffer
 
 
    int buf = -1;
    sp&lt;Fence&gt; fence;
    nsecs_t startTime = systemTime();
 
 
    FrameEventHistoryDelta frameTimestamps;
    //这里尝试去dequeueBuffer,因为这时SurfaceFlinger对应Layer的slot还没有分配buffer,这时SurfaceFlinger会回复的flag会有BUFFER_NEEDS_REALLOCATIO
    status_t result = mGraphicBufferProducer-&gt;dequeueBuffer(&amp;buf, &amp;fence, dqInput.width,
                                                            dqInput.height, dqInput.format,
                                                            dqInput.usage, &amp;mBufferAge,
                                                            dqInput.getTimestamps ?
                                                                    &amp;frameTimestamps : nullptr);
    mLastDequeueDuration = systemTime() - startTime;
 
 
    if (result &lt; 0) {
        ALOGV(&#34;dequeueBuffer: IGraphicBufferProducer::dequeueBuffer&#34;
                &#34;(%d, %d, %d, %#&#34; PRIx64 &#34;) failed: %d&#34;,
                dqInput.width, dqInput.height, dqInput.format, dqInput.usage, result);
        return result;
    }
 
 
    if (buf &lt; 0 || buf &gt;= NUM_BUFFER_SLOTS) {
        ALOGE(&#34;dequeueBuffer: IGraphicBufferProducer returned invalid slot number %d&#34;, buf);
        android_errorWriteLog(0x534e4554, &#34;36991414&#34;); // SafetyNet logging
        return FAILED_TRANSACTION;
    }
 
 
    Mutex::Autolock lock(mMutex);
 
 
    // Write this while holding the mutex
    mLastDequeueStartTime = startTime;
 
 
    sp&lt;GraphicBuffer&gt;&amp; gbuf(mSlots[buf].buffer);
 
 
    // this should never happen
    ALOGE_IF(fence == nullptr, &#34;Surface::dequeueBuffer: received null Fence! buf=%d&#34;, buf);
 
 
    if (CC_UNLIKELY(atrace_is_tag_enabled(ATRACE_TAG_GRAPHICS))) {
        static FenceMonitor hwcReleaseThread(&#34;HWC release&#34;);
        hwcReleaseThread.queueFence(fence);
    }
 
 
    if (result &amp; IGraphicBufferProducer::RELEASE_ALL_BUFFERS) {
        freeAllBuffers();
    }
 
 
    if (dqInput.getTimestamps) {
         mFrameEventHistory-&gt;applyDelta(frameTimestamps);
    }
 
 
    if ((result &amp; IGraphicBufferProducer::BUFFER_NEEDS_REALLOCATION) || gbuf == nullptr) {
        if (mReportRemovedBuffers &amp;&amp; (gbuf != nullptr)) {
            mRemovedBuffers.push_back(gbuf);
        }
 //这里检查到dequeueBuffer返回的结果里带有BUFFER_NEEDS_REALLOCATION标志就会发出一次requestBuffer
        result = mGraphicBufferProducer-&gt;requestBuffer(buf, &amp;gbuf);
        if (result != NO_ERROR) {
            ALOGE(&#34;dequeueBuffer: IGraphicBufferProducer::requestBuffer failed: %d&#34;, result);
            mGraphicBufferProducer-&gt;cancelBuffer(buf, fence);
            return result;
        }
    }
 
 
    if (fence-&gt;isValid()) {
        *fenceFd = fence-&gt;dup();
        if (*fenceFd == -1) {
            ALOGE(&#34;dequeueBuffer: error duping fence: %d&#34;, errno);
            // dup() should never fail; something is badly wrong. Soldier on
            // and hope for the best; the worst that should happen is some
            // visible corruption that lasts until the next frame.
        }
    } else {
        *fenceFd = -1;
    }
 
 
    *buffer = gbuf.get();
 
 
    if (mSharedBufferMode &amp;&amp; mAutoRefresh) {
        mSharedBufferSlot = buf;
        mSharedBufferHasBeenQueued = false;
    } else if (mSharedBufferSlot == buf) {
        mSharedBufferSlot = BufferItem::INVALID_BUFFER_SLOT;
        mSharedBufferHasBeenQueued = false;
    }
 
 
    mDequeuedSlots.insert(buf);
 
 
    return OK;
}
}
```

上面方法的主要处理如下：

1、调用mGraphicBufferProducer(IGraphicBufferProducer)的dequeueBuffer方法，尝试去dequeueBuffer。

2、检查到dequeueBuffer返回的结果里带有BUFFER\_NEEDS\_REALLOCATION标志，如果带有就调用mGraphicBufferProducer(IGraphicBufferProducer)的requestBuffer方法，去requestBuffer。

下面分别进行分析：

## BpGraphicBufferProducer dequeueBuffer

调用mGraphicBufferProducer(IGraphicBufferProducer)的dequeueBuffer方法，IGraphicBufferProducer是一个接口，由BpGraphicBufferProducer实现：

```cpp
//frameworks/native/libs/gui/IGraphicBufferProducer.cpp
class BpGraphicBufferProducer : public BpInterface&lt;IGraphicBufferProducer&gt; {
    virtual status_t dequeueBuffer(int* buf, sp&lt;Fence&gt;* fence, uint32_t width, uint32_t height,
                                   PixelFormat format, uint64_t usage, uint64_t* outBufferAge,
                                   FrameEventHistoryDelta* outTimestamps) {
        Parcel data, reply;
        bool getFrameTimestamps = (outTimestamps != nullptr);
 
 
        data.writeInterfaceToken(IGraphicBufferProducer::getInterfaceDescriptor());
        data.writeUint32(width);
        data.writeUint32(height);
        data.writeInt32(static_cast&lt;int32_t&gt;(format));
        data.writeUint64(usage);
        data.writeBool(getFrameTimestamps);
 
 
        status_t result = remote()-&gt;transact(DEQUEUE_BUFFER, data, &amp;reply);
        if (result != NO_ERROR) {
            return result;
        }
 
 
        *buf = reply.readInt32();
        *fence = new Fence();
        result = reply.read(**fence);
        if (result != NO_ERROR) {
            fence-&gt;clear();
            return result;
        }
        if (outBufferAge) {
            result = reply.readUint64(outBufferAge);
        } else {
            // Read the value even if outBufferAge is nullptr:
            uint64_t bufferAge;
            result = reply.readUint64(&amp;bufferAge);
        }
        if (result != NO_ERROR) {
            ALOGE(&#34;IGBP::dequeueBuffer failed to read buffer age: %d&#34;, result);
            return result;
        }
        if (getFrameTimestamps) {
            result = reply.read(*outTimestamps);
            if (result != NO_ERROR) {
                ALOGE(&#34;IGBP::dequeueBuffer failed to read timestamps: %d&#34;,
                        result);
                return result;
            }
        }
        result = reply.readInt32();
        return result;
    }
}
```

发送DEQUEUE\_BUFFER消息，发送的消息在BnGraphicBufferProducer的onTransact方法中处理：

```cpp
//frameworks/native/libs/gui/IGraphicBufferProducer.cpp
status_t BnGraphicBufferProducer::onTransact(
    uint32_t code, const Parcel&amp; data, Parcel* reply, uint32_t flags)
{
    switch(code) {
        case DEQUEUE_BUFFER: {
            CHECK_INTERFACE(IGraphicBufferProducer, data, reply);
            uint32_t width = data.readUint32();
            uint32_t height = data.readUint32();
            PixelFormat format = static_cast&lt;PixelFormat&gt;(data.readInt32());
            uint64_t usage = data.readUint64();
            uint64_t bufferAge = 0;
            bool getTimestamps = data.readBool();
 
 
            int buf = 0;
            sp&lt;Fence&gt; fence = Fence::NO_FENCE;
            FrameEventHistoryDelta frameTimestamps;
            int result = dequeueBuffer(&amp;buf, &amp;fence, width, height, format, usage, &amp;bufferAge,
                                       getTimestamps ? &amp;frameTimestamps : nullptr);
 
 
            if (fence == nullptr) {
                ALOGE(&#34;dequeueBuffer returned a NULL fence, setting to Fence::NO_FENCE&#34;);
                fence = Fence::NO_FENCE;
            }
            reply-&gt;writeInt32(buf);
            reply-&gt;write(*fence);
            reply-&gt;writeUint64(bufferAge);
            if (getTimestamps) {
                reply-&gt;write(frameTimestamps);
            }
            reply-&gt;writeInt32(result);
            return NO_ERROR;
        }
    }
}
```

### BufferQueueProducer dequeueBuffer

调用BnGraphicBufferProducer的dequeueBuffer方法，BnGraphicBufferProducer继承于BufferQueueProducer，调用BufferQueueProducer的dequeueBuffer方法：

[Android13 BufferQueueProducer dequeueBuffer流程分析-CSDN博客](/Android13%20BufferQueueProducer%20dequeueBuffer%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-CSDN%E5%8D%9A%E5%AE%A2)

## BpGraphicBufferProducer requestBuffer

在SurfaceFlinger这端，第一次收到dequeueBuffer时发现分配出来的slot没有GraphicBuffer， 这时会去申请对应的buffer:

调用mGraphicBufferProducer(IGraphicBufferProducer)的requestBuffer方法，IGraphicBufferProducer是一个接口，由BpGraphicBufferProducer实现：

```cpp
//frameworks/native/libs/gui/IGraphicBufferProducer.cpp
class BpGraphicBufferProducer : public BpInterface&lt;IGraphicBufferProducer&gt; {
    virtual status_t requestBuffer(int bufferIdx, sp&lt;GraphicBuffer&gt;* buf) {
        Parcel data, reply;
        data.writeInterfaceToken(IGraphicBufferProducer::getInterfaceDescriptor());
        data.writeInt32(bufferIdx);
        status_t result = remote()-&gt;transact(REQUEST_BUFFER, data, &amp;reply);
        if (result != NO_ERROR) {
            return result;
        }
        bool nonNull = reply.readInt32();
        if (nonNull) {
            *buf = new GraphicBuffer();
            result = reply.read(**buf);
            if(result != NO_ERROR) {
                (*buf).clear();
                return result;
            }
        }
        result = reply.readInt32();
        return result;
    }
}
```

发送REQUEST\_BUFFER消息，发送的消息在BnGraphicBufferProducer的onTransact方法中处理：

```cpp
//frameworks/native/libs/gui/IGraphicBufferProducer.cpp
status_t BnGraphicBufferProducer::onTransact(
    uint32_t code, const Parcel&amp; data, Parcel* reply, uint32_t flags)
{
    switch(code) {
        case REQUEST_BUFFER: {
            CHECK_INTERFACE(IGraphicBufferProducer, data, reply);
            int bufferIdx = data.readInt32();
            sp&lt;GraphicBuffer&gt; buffer;
            int result = requestBuffer(bufferIdx, &amp;buffer);
            reply-&gt;writeInt32(buffer != nullptr);
            if (buffer != nullptr) {
                reply-&gt;write(*buffer);
            }
            reply-&gt;writeInt32(result);
            return NO_ERROR;
        }
    }
    return BBinder::onTransact(code, data, reply, flags);
}
```

### BufferQueueProducer requestBuffer

调用BnGraphicBufferProducer的requestBuffer方法，BnGraphicBufferProducer继承于BufferQueueProducer，调用BufferQueueProducer的requestBuffer方法：

```cpp
//frameworks/native/libs/gui/BufferQueueProducer.cpp
class BufferQueueProducer : public BnGraphicBufferProducer {
status_t BufferQueueProducer::requestBuffer(int slot, sp&lt;GraphicBuffer&gt;* buf) {
    ATRACE_CALL();
    BQ_LOGV(&#34;requestBuffer: slot %d&#34;, slot);
    std::lock_guard&lt;std::mutex&gt; lock(mCore-&gt;mMutex);
 
 
    if (mCore-&gt;mIsAbandoned) {
        BQ_LOGE(&#34;requestBuffer: BufferQueue has been abandoned&#34;);
        return NO_INIT;
    }
 
 
    if (mCore-&gt;mConnectedApi == BufferQueueCore::NO_CONNECTED_API) {
        BQ_LOGE(&#34;requestBuffer: BufferQueue has no connected producer&#34;);
        return NO_INIT;
    }
 
 
    if (slot &lt; 0 || slot &gt;= BufferQueueDefs::NUM_BUFFER_SLOTS) {
        BQ_LOGE(&#34;requestBuffer: slot index %d out of range [0, %d)&#34;,
                slot, BufferQueueDefs::NUM_BUFFER_SLOTS);
        return BAD_VALUE;
    } else if (!mSlots[slot].mBufferState.isDequeued()) {
        BQ_LOGE(&#34;requestBuffer: slot %d is not owned by the producer &#34;
                &#34;(state = %s)&#34;, slot, mSlots[slot].mBufferState.string());
        return BAD_VALUE;
    }
 
 
    mSlots[slot].mRequestBufferCalled = true;
    *buf = mSlots[slot].mGraphicBuffer;
    return NO_ERROR;
}
}
```

requestBuffer的主要作用就是把GraphicBuffer传递到应用侧。


---

> 作者: [wentao](https://github.com/huang8604)  
> URL: https://huang8604.github.io/posts/android13-surface-dequeuebuffer%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-csdn%E5%8D%9A%E5%AE%A2/  

