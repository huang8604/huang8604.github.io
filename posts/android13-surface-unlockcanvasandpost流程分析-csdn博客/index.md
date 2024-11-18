# Android13 Surface UnlockCanvasAndPost流程分析-CSDN博客

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
            throw new IllegalArgumentException(&#34;canvas object must be the same instance that &#34;
                    &#43; &#34;was previously returned by lockCanvas&#34;);
        }
        if (mNativeObject != mLockedObject) {
            Log.w(TAG, &#34;WARNING: Surface&#39;s mNativeObject (0x&#34; &#43;
                    Long.toHexString(mNativeObject) &#43; &#34;) != mLockedObject (0x&#34; &#43;
                    Long.toHexString(mLockedObject) &#43;&#34;)&#34;);
        }
        if (mLockedObject == 0) {
            throw new IllegalStateException(&#34;Surface was not locked&#34;);
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
    sp&lt;Surface&gt; surface(reinterpret_cast&lt;Surface *&gt;(nativeObject));
    if (!isSurfaceValid(surface)) {
        return;
    }
 
 
    // detach the canvas from the surface
    graphics::Canvas canvas(env, canvasObj);
    canvas.setBuffer(nullptr, ADATASPACE_UNKNOWN);
 
 
    // unlock surface
    status_t err = surface-&gt;unlockAndPost();
    if (err &lt; 0) {
        jniThrowException(env, IllegalArgumentException, NULL);
    }
}
```

调用surface(Surface)的unlockAndPost方法：

```cpp
//framework/native/libs/gui/Surface.cpp
class Surface : public ANativeObjectBase&lt;ANativeWindow, Surface, RefBase&gt; {
status_t Surface::unlockAndPost()
{
    if (mLockedBuffer == nullptr) {
        ALOGE(&#34;Surface::unlockAndPost failed, no locked buffer&#34;);
        return INVALID_OPERATION;
    }
 
 
    int fd = -1;
    status_t err = mLockedBuffer-&gt;unlockAsync(&amp;fd);
    ALOGE_IF(err, &#34;failed unlocking buffer (%p)&#34;, mLockedBuffer-&gt;handle);
 
 
    err = queueBuffer(mLockedBuffer.get(), fd);
    ALOGE_IF(err, &#34;queueBuffer (handle=%p) failed (%s)&#34;,
            mLockedBuffer-&gt;handle, strerror(-err));
 
 
    mPostedBuffer = mLockedBuffer;
    mLockedBuffer = nullptr;
    return err;
}
}
```

## Surface queueBuffer

调用Surface的queueBuffer方法，将图像数据放入Surface的缓冲区中：

[Android13 Surface queueBuffer流程分析-CSDN博客](/Android13%20Surface%20queueBuffer%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-CSDN%E5%8D%9A%E5%AE%A2)


---

> 作者: [wentao](https://github.com/huang8604)  
> URL: https://huang8604.github.io/posts/android13-surface-unlockcanvasandpost%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-csdn%E5%8D%9A%E5%AE%A2/  

