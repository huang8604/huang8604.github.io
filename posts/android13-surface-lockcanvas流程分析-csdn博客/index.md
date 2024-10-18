# Android13 Surface LockCanvas流程分析-CSDN博客

Surface的lockCanvas用于获取Canvas对象，以便进行绘图操作，代码如下：

```java
//frameworks/base/core/java/android/view/Surface.java
public class Surface implements Parcelable {
    public Canvas lockCanvas(Rect inOutDirty)
            throws Surface.OutOfResourcesException, IllegalArgumentException {
        synchronized (mLock) {
            checkNotReleasedLocked();
            if (mLockedObject != 0) {
                // Ideally, nativeLockCanvas() would throw in this situation and prevent the
                // double-lock, but that won&#39;t happen if mNativeObject was updated.  We can&#39;t
                // abandon the old mLockedObject because it might still be in use, so instead
                // we just refuse to re-lock the Surface.
                throw new IllegalArgumentException(&#34;Surface was already locked&#34;);
            }
            mLockedObject = nativeLockCanvas(mNativeObject, mCanvas, inOutDirty);
            return mCanvas;
        }
    }
}
```

调用nativeLockCanvas方法，nativeLockCanvas是一个Native方法在android\_view\_Surface.cpp中实现：

```cpp
//frameworks/base/core/jni/android_view_Surface.cpp
static jlong nativeLockCanvas(JNIEnv* env, jclass clazz,
        jlong nativeObject, jobject canvasObj, jobject dirtyRectObj) {
    sp&lt;Surface&gt; surface(reinterpret_cast&lt;Surface *&gt;(nativeObject));
 
 
    if (!isSurfaceValid(surface)) {
        jniThrowException(env, IllegalArgumentException, NULL);
        return 0;
    }
 
 
    if (!ACanvas_isSupportedPixelFormat(ANativeWindow_getFormat(surface.get()))) {
        native_window_set_buffers_format(surface.get(), PIXEL_FORMAT_RGBA_8888);
    }
 
 
    Rect dirtyRect(Rect::EMPTY_RECT);
    Rect* dirtyRectPtr = NULL;
 
 
    if (dirtyRectObj) {
        dirtyRect.left   = env-&gt;GetIntField(dirtyRectObj, gRectClassInfo.left);
        dirtyRect.top    = env-&gt;GetIntField(dirtyRectObj, gRectClassInfo.top);
        dirtyRect.right  = env-&gt;GetIntField(dirtyRectObj, gRectClassInfo.right);
        dirtyRect.bottom = env-&gt;GetIntField(dirtyRectObj, gRectClassInfo.bottom);
        dirtyRectPtr = &amp;dirtyRect;
    }
 
 
    ANativeWindow_Buffer buffer;
    status_t err = surface-&gt;lock(&amp;buffer, dirtyRectPtr);
    if (err &lt; 0) {
        const char* const exception = (err == NO_MEMORY) ?
                OutOfResourcesException : IllegalArgumentException;
        jniThrowException(env, exception, NULL);
        return 0;
    }
 
 
    graphics::Canvas canvas(env, canvasObj);
    canvas.setBuffer(&amp;buffer, static_cast&lt;int32_t&gt;(surface-&gt;getBuffersDataSpace()));
 
 
    if (dirtyRectPtr) {
        canvas.clipRect({dirtyRect.left, dirtyRect.top, dirtyRect.right, dirtyRect.bottom});
    }
 
 
    if (dirtyRectObj) {
        env-&gt;SetIntField(dirtyRectObj, gRectClassInfo.left,   dirtyRect.left);
        env-&gt;SetIntField(dirtyRectObj, gRectClassInfo.top,    dirtyRect.top);
        env-&gt;SetIntField(dirtyRectObj, gRectClassInfo.right,  dirtyRect.right);
        env-&gt;SetIntField(dirtyRectObj, gRectClassInfo.bottom, dirtyRect.bottom);
    }
 
 
    // Create another reference to the surface and return it.  This reference
    // should be passed to nativeUnlockCanvasAndPost in place of mNativeObject,
    // because the latter could be replaced while the surface is locked.
    sp&lt;Surface&gt; lockedSurface(surface);
    lockedSurface-&gt;incStrong(&amp;sRefBaseOwner);
    return (jlong) lockedSurface.get();
}
```

调用surface(Surface)的lock方法：

```cpp
//framework/native/libs/gui/Surface.cpp
class Surface : public ANativeObjectBase&lt;ANativeWindow, Surface, RefBase&gt; {
status_t Surface::lock(
        ANativeWindow_Buffer* outBuffer, ARect* inOutDirtyBounds)
{
    if (mLockedBuffer != nullptr) {
        ALOGE(&#34;Surface::lock failed, already locked&#34;);
        return INVALID_OPERATION;
    }
 
 
    if (!mConnectedToCpu) {
        int err = Surface::connect(NATIVE_WINDOW_API_CPU);
        if (err) {
            return err;
        }
        // we&#39;re intending to do software rendering from this point
        setUsage(GRALLOC_USAGE_SW_READ_OFTEN | GRALLOC_USAGE_SW_WRITE_OFTEN);
    }
 
 
    ANativeWindowBuffer* out;
    int fenceFd = -1;
    status_t err = dequeueBuffer(&amp;out, &amp;fenceFd);
    ALOGE_IF(err, &#34;dequeueBuffer failed (%s)&#34;, strerror(-err));
    if (err == NO_ERROR) {
        sp&lt;GraphicBuffer&gt; backBuffer(GraphicBuffer::getSelf(out));
        const Rect bounds(backBuffer-&gt;width, backBuffer-&gt;height);
 
 
        Region newDirtyRegion;
        if (inOutDirtyBounds) {
            newDirtyRegion.set(static_cast&lt;Rect const&amp;&gt;(*inOutDirtyBounds));
            newDirtyRegion.andSelf(bounds);
        } else {
            newDirtyRegion.set(bounds);
        }
 
 
        // figure out if we can copy the frontbuffer back
        const sp&lt;GraphicBuffer&gt;&amp; frontBuffer(mPostedBuffer);
        const bool canCopyBack = (frontBuffer != nullptr &amp;&amp;
                backBuffer-&gt;width  == frontBuffer-&gt;width &amp;&amp;
                backBuffer-&gt;height == frontBuffer-&gt;height &amp;&amp;
                backBuffer-&gt;format == frontBuffer-&gt;format);
 
 
        if (canCopyBack) {
            // copy the area that is invalid and not repainted this round
            const Region copyback(mDirtyRegion.subtract(newDirtyRegion));
            if (!copyback.isEmpty()) {
                copyBlt(backBuffer, frontBuffer, copyback, &amp;fenceFd);
            }
        } else {
            // if we can&#39;t copy-back anything, modify the user&#39;s dirty
            // region to make sure they redraw the whole buffer
            newDirtyRegion.set(bounds);
            mDirtyRegion.clear();
            Mutex::Autolock lock(mMutex);
            for (size_t i=0 ; i&lt;NUM_BUFFER_SLOTS ; i&#43;&#43;) {
                mSlots[i].dirtyRegion.clear();
            }
        }
 
 
 
 
        { // scope for the lock
            Mutex::Autolock lock(mMutex);
            int backBufferSlot(getSlotFromBufferLocked(backBuffer.get()));
            if (backBufferSlot &gt;= 0) {
                Region&amp; dirtyRegion(mSlots[backBufferSlot].dirtyRegion);
                mDirtyRegion.subtract(dirtyRegion);
                dirtyRegion = newDirtyRegion;
            }
        }
 
 
        mDirtyRegion.orSelf(newDirtyRegion);
        if (inOutDirtyBounds) {
            *inOutDirtyBounds = newDirtyRegion.getBounds();
        }
 
 
        void* vaddr;
        status_t res = backBuffer-&gt;lockAsync(
                GRALLOC_USAGE_SW_READ_OFTEN | GRALLOC_USAGE_SW_WRITE_OFTEN,
                newDirtyRegion.bounds(), &amp;vaddr, fenceFd);
 
 
        ALOGW_IF(res, &#34;failed locking buffer (handle = %p)&#34;,
                backBuffer-&gt;handle);
 
 
        if (res != 0) {
            err = INVALID_OPERATION;
        } else {
            mLockedBuffer = backBuffer;
            outBuffer-&gt;width  = backBuffer-&gt;width;
            outBuffer-&gt;height = backBuffer-&gt;height;
            outBuffer-&gt;stride = backBuffer-&gt;stride;
            outBuffer-&gt;format = backBuffer-&gt;format;
            outBuffer-&gt;bits   = vaddr;
        }
    }
    return err;
}
}
```

上面方法主要处理如下：

1、调用Surface的connect方法，建立与BufferQueueCore的连接。

2、调用Surface的dequeueBuffer方法，从Surface中获取一个可用的Buffer。

下面分别进行分析：

## Surface connect

调用Surface的connect方法，建立与SurfaceFlinger服务的连接：

[Android13 Surface connect流程分析-CSDN博客](/Android13%20Surface%20connect%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-CSDN%E5%8D%9A%E5%AE%A2)

## Surface dequeueBuffer

调用Surface的dequeueBuffer方法，从Surface中获取一个可用的Buffer：

[Android13 Surface dequeueBuffer流程分析-CSDN博客](/Android13%20Surface%20dequeueBuffer%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-CSDN%E5%8D%9A%E5%AE%A2)


---

> 作者: [wentao](https://github.com/huang8604)  
> URL: https://huang8604.github.io/posts/android13-surface-lockcanvas%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-csdn%E5%8D%9A%E5%AE%A2/  

