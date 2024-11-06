---
title: Android13 Surface lockCanvas流程分析-CSDN博客
author: 
created: 2024-09-26
tags:
  - clippings
  - 转载
  - blog
collections:
  - 图形显示
date: 2024-11-06T06:57:52.920Z
lastmod: 2024-11-05T01:33:59.261Z
---
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
                // double-lock, but that won't happen if mNativeObject was updated.  We can't
                // abandon the old mLockedObject because it might still be in use, so instead
                // we just refuse to re-lock the Surface.
                throw new IllegalArgumentException("Surface was already locked");
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
    sp<Surface> surface(reinterpret_cast<Surface *>(nativeObject));
 
 
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
        dirtyRect.left   = env->GetIntField(dirtyRectObj, gRectClassInfo.left);
        dirtyRect.top    = env->GetIntField(dirtyRectObj, gRectClassInfo.top);
        dirtyRect.right  = env->GetIntField(dirtyRectObj, gRectClassInfo.right);
        dirtyRect.bottom = env->GetIntField(dirtyRectObj, gRectClassInfo.bottom);
        dirtyRectPtr = &dirtyRect;
    }
 
 
    ANativeWindow_Buffer buffer;
    status_t err = surface->lock(&buffer, dirtyRectPtr);
    if (err < 0) {
        const char* const exception = (err == NO_MEMORY) ?
                OutOfResourcesException : IllegalArgumentException;
        jniThrowException(env, exception, NULL);
        return 0;
    }
 
 
    graphics::Canvas canvas(env, canvasObj);
    canvas.setBuffer(&buffer, static_cast<int32_t>(surface->getBuffersDataSpace()));
 
 
    if (dirtyRectPtr) {
        canvas.clipRect({dirtyRect.left, dirtyRect.top, dirtyRect.right, dirtyRect.bottom});
    }
 
 
    if (dirtyRectObj) {
        env->SetIntField(dirtyRectObj, gRectClassInfo.left,   dirtyRect.left);
        env->SetIntField(dirtyRectObj, gRectClassInfo.top,    dirtyRect.top);
        env->SetIntField(dirtyRectObj, gRectClassInfo.right,  dirtyRect.right);
        env->SetIntField(dirtyRectObj, gRectClassInfo.bottom, dirtyRect.bottom);
    }
 
 
    // Create another reference to the surface and return it.  This reference
    // should be passed to nativeUnlockCanvasAndPost in place of mNativeObject,
    // because the latter could be replaced while the surface is locked.
    sp<Surface> lockedSurface(surface);
    lockedSurface->incStrong(&sRefBaseOwner);
    return (jlong) lockedSurface.get();
}
```

调用surface(Surface)的lock方法：

```cpp
//framework/native/libs/gui/Surface.cpp
class Surface : public ANativeObjectBase<ANativeWindow, Surface, RefBase> {
status_t Surface::lock(
        ANativeWindow_Buffer* outBuffer, ARect* inOutDirtyBounds)
{
    if (mLockedBuffer != nullptr) {
        ALOGE("Surface::lock failed, already locked");
        return INVALID_OPERATION;
    }
 
 
    if (!mConnectedToCpu) {
        int err = Surface::connect(NATIVE_WINDOW_API_CPU);
        if (err) {
            return err;
        }
        // we're intending to do software rendering from this point
        setUsage(GRALLOC_USAGE_SW_READ_OFTEN | GRALLOC_USAGE_SW_WRITE_OFTEN);
    }
 
 
    ANativeWindowBuffer* out;
    int fenceFd = -1;
    status_t err = dequeueBuffer(&out, &fenceFd);
    ALOGE_IF(err, "dequeueBuffer failed (%s)", strerror(-err));
    if (err == NO_ERROR) {
        sp<GraphicBuffer> backBuffer(GraphicBuffer::getSelf(out));
        const Rect bounds(backBuffer->width, backBuffer->height);
 
 
        Region newDirtyRegion;
        if (inOutDirtyBounds) {
            newDirtyRegion.set(static_cast<Rect const&>(*inOutDirtyBounds));
            newDirtyRegion.andSelf(bounds);
        } else {
            newDirtyRegion.set(bounds);
        }
 
 
        // figure out if we can copy the frontbuffer back
        const sp<GraphicBuffer>& frontBuffer(mPostedBuffer);
        const bool canCopyBack = (frontBuffer != nullptr &&
                backBuffer->width  == frontBuffer->width &&
                backBuffer->height == frontBuffer->height &&
                backBuffer->format == frontBuffer->format);
 
 
        if (canCopyBack) {
            // copy the area that is invalid and not repainted this round
            const Region copyback(mDirtyRegion.subtract(newDirtyRegion));
            if (!copyback.isEmpty()) {
                copyBlt(backBuffer, frontBuffer, copyback, &fenceFd);
            }
        } else {
            // if we can't copy-back anything, modify the user's dirty
            // region to make sure they redraw the whole buffer
            newDirtyRegion.set(bounds);
            mDirtyRegion.clear();
            Mutex::Autolock lock(mMutex);
            for (size_t i=0 ; i<NUM_BUFFER_SLOTS ; i++) {
                mSlots[i].dirtyRegion.clear();
            }
        }
 
 
 
 
        { // scope for the lock
            Mutex::Autolock lock(mMutex);
            int backBufferSlot(getSlotFromBufferLocked(backBuffer.get()));
            if (backBufferSlot >= 0) {
                Region& dirtyRegion(mSlots[backBufferSlot].dirtyRegion);
                mDirtyRegion.subtract(dirtyRegion);
                dirtyRegion = newDirtyRegion;
            }
        }
 
 
        mDirtyRegion.orSelf(newDirtyRegion);
        if (inOutDirtyBounds) {
            *inOutDirtyBounds = newDirtyRegion.getBounds();
        }
 
 
        void* vaddr;
        status_t res = backBuffer->lockAsync(
                GRALLOC_USAGE_SW_READ_OFTEN | GRALLOC_USAGE_SW_WRITE_OFTEN,
                newDirtyRegion.bounds(), &vaddr, fenceFd);
 
 
        ALOGW_IF(res, "failed locking buffer (handle = %p)",
                backBuffer->handle);
 
 
        if (res != 0) {
            err = INVALID_OPERATION;
        } else {
            mLockedBuffer = backBuffer;
            outBuffer->width  = backBuffer->width;
            outBuffer->height = backBuffer->height;
            outBuffer->stride = backBuffer->stride;
            outBuffer->format = backBuffer->format;
            outBuffer->bits   = vaddr;
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
