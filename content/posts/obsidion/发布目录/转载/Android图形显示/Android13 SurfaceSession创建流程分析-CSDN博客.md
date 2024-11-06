---
title: Android13 SurfaceSession创建流程分析_android surfacesession-CSDN博客
author: 
created: 2024-09-26
tags:
  - clippings
  - 转载
  - blog
collections:
  - 图形显示
date: 2024-11-06T06:57:53.011Z
lastmod: 2024-11-05T01:33:59.290Z
---
SurfaceSession是Android系统中与图形表面相关的一个关键类，它提供了与SurfaceFlinger服务通信以创建和管理图形表面连接的API，SurfaceSession在WindowManagerService的addWindow时创建，构造方法如下：

```java
//frameworks/base/core/java/android/view/SurfaceSession.java
public final class SurfaceSession {
    public SurfaceSession() {
        mNativeClient = nativeCreate();
    }
}
```

调用nativeCreate方法，nativeCreate是个Native方法，通过查表调用android\_view\_SurfaceSession.cpp的nativeCreate方法：

```cpp
//frameworks/base/core/jni/android_view_SurfaceSession.cpp
static jlong nativeCreate(JNIEnv* env, jclass clazz) {
    SurfaceComposerClient* client = new SurfaceComposerClient(); //创建SurfaceComposerClient对象
    client->incStrong((void*)nativeCreate);
    return reinterpret_cast<jlong>(client);
}
```

通过new的方式创建SurfaceComposerClient对象，SurfaceComposerClient的构造方法如下：

```cpp
//frameworks/native/libs/gui/SurfaceComposerClient.cpp
class SurfaceComposerClient : public RefBase
	SurfaceComposerClient::SurfaceComposerClient() : mStatus(NO_INIT) {}
}
```
