# Android源码中学习JNI那些事--关键技巧-CSDN博客

**“JNI开发中请问如果想要一个纯native线程中执行的native方法需要调用到Java层应该要怎么做？”**\
大家注意这个问题哈，是纯native线程和方法，即没有我们正常jni调用的env环境的，正常的如果jni方法是由java层调用到jni一般都是自带了JNIEnv变量如下：\
![e3ba1293ffa2bac71e3590c27a0c643e\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/6708d3f6b3038.png)

拿getMemInfo举例子，如：\
java层：

```c
public static native void getMemInfo(long[] outSizes);
```

转变成了native层面代码就是如下：

```cpp
static void android_os_Debug_getMemInfo(JNIEnv *env, jobject clazz, jlongArray out)
```

大家看这里是不是自带了一个变量JNIEnv \*env，有了这里env后再想调用Java层面的方法就很简单，具体看参考如下：\
比如经常Java调用jni时候。

```cpp
tatic void android_os_Debug_getMemInfo(JNIEnv *env, jobject clazz, jlongArray out)
{
    char buffer[1024];
    size_t numFound = 0;

    if (out == NULL) {
        jniThrowNullPointerException(env, &#34;out == null&#34;);//抛出空指针到java
        return;
    }
    //省略。。。。
}
```

jniThrowNullPointerException方法就需要传递env，jniThrowNullPointerException最后调用到了jniThrowException：如下

```cpp
// libnativehelper/JNIHelp.cpp
extern &#34;C&#34; int jniThrowException(C_JNIEnv* env, const char* className, const char* msg) {
    JNIEnv* e = reinterpret_cast&lt;JNIEnv*&gt;(env);

    if ((*env)-&gt;ExceptionCheck(e)) {
        /* TODO: consider creating the new exception with this as &#34;cause&#34; */
        scoped_local_ref&lt;jthrowable&gt; exception(env, (*env)-&gt;ExceptionOccurred(e));
        (*env)-&gt;ExceptionClear(e);

        if (exception.get() != NULL) {
            std::string text;
            getExceptionSummary(env, exception.get(), text);
            ALOGW(&#34;Discarding pending exception (%s) to throw %s&#34;, text.c_str(), className);
        }
    }

    scoped_local_ref&lt;jclass&gt; exceptionClass(env, findClass(env, className));
    if (exceptionClass.get() == NULL) {
        ALOGE(&#34;Unable to find exception class %s&#34;, className);
        /* ClassNotFoundException now pending */
        return -1;
    }

    if ((*env)-&gt;ThrowNew(e, exceptionClass.get(), msg) != JNI_OK) {
        ALOGE(&#34;Failed throwing &#39;%s&#39; &#39;%s&#39;&#34;, className, msg);
        /* an exception, most likely OOM, will now be pending */
        return -1;
    }

    return 0;
}
```

这里就需要使用env来获取对应的java类，及对应属性，方法等

回到面试题目，人家要求没有纯native线程里面native方法，即说明这种情况下肯定不自带JNIEnv的，故这里该怎么办呢？\
![76b02ef69d1373cff8cda15b8fa67c93\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/6708d3f6b3b53.png)

不自带JNIEnv，但是调用java又需要JNIEnv，故得想方法构造出JNIEnv具体如下：\
![646d25a078bf744e0d97d3eb1416f076\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/6708d3f6b301b.png)

下面以一个Android源码中实战例子来说明：\
源码位子参考：

```bash
frameworks/base/core/jni/android/graphics/SurfaceTexture.cpp
```

1、获取JavaVM,通过AndroidRuntime::getJavaVM()获取JavaVM\
2、通过JavaVM绑定到当前线线程获取JNIEnv，通过vm-&gt;AttachCurrentThread方法中参数赋值JNIEnv

```c
JNIEnv* JNISurfaceTextureContext::getJNIEnv(bool* needsDetach) {
    *needsDetach = false;
    JNIEnv* env = AndroidRuntime::getJNIEnv();
    if (env == NULL) {
        JavaVMAttachArgs args = {
            JNI_VERSION_1_4, &#34;JNISurfaceTextureContext&#34;, NULL };
        JavaVM* vm = AndroidRuntime::getJavaVM();//获取JavaVM
        int result = vm-&gt;AttachCurrentThread(&amp;env, (void*) &amp;args);//赋值env
        if (result != JNI_OK) {
            ALOGE(&#34;thread attach failed: %#x&#34;, result);
            return NULL;
        }
        *needsDetach = true;
    }
    return env;
}
```

3、使用env：

```cpp
void JNISurfaceTextureContext::onFrameAvailable(const BufferItem&amp; /* item */)
{
    bool needsDetach = false;
    JNIEnv* env = getJNIEnv(&amp;needsDetach);
    if (env != NULL) {
        env-&gt;CallStaticVoidMethod(mClazz, fields.postEvent, mWeakThiz);//使用env调用java方法
    } else {
        ALOGW(&#34;onFrameAvailable event will not posted&#34;);
    }
    if (needsDetach) {
        detachJNI();
    }
}
```

4、不需要使用了记得与当前线程解绑定，调用JavaVM的DetachCurrentThread方法

```cpp
void JNISurfaceTextureContext::detachJNI() {
    JavaVM* vm = AndroidRuntime::getJavaVM();
    int result = vm-&gt;DetachCurrentThread();//解绑定
    if (result != JNI_OK) {
        ALOGE(&#34;thread detach failed: %#x&#34;, result);
    }
}
```


---

> 作者: [wentao](https://github.com/huang8604)  
> URL: https://huang8604.github.io/posts/android%E6%BA%90%E7%A0%81%E4%B8%AD%E5%AD%A6%E4%B9%A0jni%E9%82%A3%E4%BA%9B%E4%BA%8B--%E5%85%B3%E9%94%AE%E6%8A%80%E5%B7%A7-csdn%E5%8D%9A%E5%AE%A2/  

