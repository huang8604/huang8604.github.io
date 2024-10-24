---
title: Android13 Surface unlockCanvasAndPost流程分析-CSDN博客
created: 2024-09-26
tags:
  - clippings
  - 转载
  - blog
collections:
  - 图形显示
date: 2024-09-26T08:34:35.161Z
lastmod: 2024-09-26T08:55:50.000Z
---
Surface的unlockCanvasAndPost用于解锁并提交Surface上的画布内容，代码如下：

```java
//frameworks/base/core/java/android/view/Surface.java
public class Surface implements Parcelable {
    public void unlockCanvasAndPost(Canvas canvas) {
        synchronized (mLock) {
            checkNotReleasedLocked();
 
 
            if (mHwuiContext != null) {
                mHwuiContext.unlockAndPost(canvas);
            } else {
                unlockSwCanvasAndPost(canvas);
            }
        }
    }
}
```

调用Surface的unlockSwCanvasAndPost方法：

```cpp
//frameworks/base/core/java/android/view/Surface.java
public class Surface implements Parcelable {
    private void unlockSwCanvasAndPost(Canvas canvas) {
        if (canvas != mCanvas) {
            throw new IllegalArgumentException("canvas object must be the same instance that "
                    + "was previously returned by lockCanvas");
        }
        if (mNativeObject != mLockedObject) {
            Log.w(TAG, "WARNING: Surface's mNativeObject (0x" +
                    Long.toHexString(mNativeObject) + ") != mLockedObject (0x" +
                    Long.toHexString(mLockedObject) +")");
        }
        if (mLockedObject == 0) {
            throw new IllegalStateException("Surface was not locked");
        }
        try {
            nativeUnlockCanvasAndPost(mLockedObject, canvas);
        } finally {
            nativeRelease(mLockedObject);
            mLockedObject = 0;
        }
    }
}
```

调用nativeUnlockCanvasAndPost方法，nativeLockCanvas是一个Native方法在android\_view\_Surface.cpp中实现：

```cpp
//frameworks/base/core/jni/android_view_Surface.cpp
static void nativeUnlockCanvasAndPost(JNIEnv* env, jclass clazz,
        jlong nativeObject, jobject canvasObj) {
    sp<Surface> surface(reinterpret_cast<Surface *>(nativeObject));
    if (!isSurfaceValid(surface)) {
        return;
    }
 
 
    // detach the canvas from the surface
    graphics::Canvas canvas(env, canvasObj);
    canvas.setBuffer(nullptr, ADATASPACE_UNKNOWN);
 
 
    // unlock surface
    status_t err = surface->unlockAndPost();
    if (err < 0) {
        jniThrowException(env, IllegalArgumentException, NULL);
    }
}
```

调用surface(Surface)的unlockAndPost方法：

```cpp
//framework/native/libs/gui/Surface.cpp
class Surface : public ANativeObjectBase<ANativeWindow, Surface, RefBase> {
status_t Surface::unlockAndPost()
{
    if (mLockedBuffer == nullptr) {
        ALOGE("Surface::unlockAndPost failed, no locked buffer");
        return INVALID_OPERATION;
    }
 
 
    int fd = -1;
    status_t err = mLockedBuffer->unlockAsync(&fd);
    ALOGE_IF(err, "failed unlocking buffer (%p)", mLockedBuffer->handle);
 
 
    err = queueBuffer(mLockedBuffer.get(), fd);
    ALOGE_IF(err, "queueBuffer (handle=%p) failed (%s)",
            mLockedBuffer->handle, strerror(-err));
 
 
    mPostedBuffer = mLockedBuffer;
    mLockedBuffer = nullptr;
    return err;
}
}
```

## Surface queueBuffer

调用Surface的queueBuffer方法，将图像数据放入Surface的缓冲区中：

[Android13 Surface queueBuffer流程分析-CSDN博客](/Android13%20Surface%20queueBuffer%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-CSDN%E5%8D%9A%E5%AE%A2)
