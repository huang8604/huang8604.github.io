---
title: Android13 SurfaceControl创建流程分析-CSDN博客
author: 
created: 2024-09-26
tags:
  - clippings
  - 转载
  - blog
collections:
  - 图形显示
date: 2024-09-26T05:55:39.573Z
lastmod: 2024-09-26T06:23:03.423Z
---
SurfaceControl是Android系统中的一个类，用于管理和控制Surface的创建、显示和销毁，SurfaceControl的创建过程如下：

![63624b4a088029ce476c7b4bdc6508dd\_MD5](https://picgo.myjojo.fun:666/i/2024/09/26/66f4f77c8c80e.png)

下面分析WindowManagerService创建SurfaceControl的步骤：

* 首先应用进程会new一个java层SurfaceControl，什么都没做，然后传递到WMS进程，因为SurfaceControl在AIDL中是out类型，所以在WMS进程赋值。
* WMS在创建java层SurfaceControl的同时通过nativeCreate方法到native层做一系列初始化。
* 在SurfaceComposerClient的createSurfaceChecked函数中通过ISurfaceComposerClient的Bp端mClient向SurfaceFlinger进程请求创建Surface，即调用createSurface函数，而在SurfaceFlinger进程Surface对应的是Layer。
* 在第一次创建Layer的子类BufferQueueLayer的过程中，即在BufferQueueLayer的onFirstRef函数中会创建生产者消费者模型架构。[Android13 BufferQueueLayer onFirstRef流程分析-CSDN博客](/Android13%20BufferQueueLayer%20onFirstRef%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-CSDN%E5%8D%9A%E5%AE%A2)
* SurfaceFlinger进程的任务完成之后会直接new一个SurfaceControl，并将SurfaceFlinger进程创建的Layer引用和生产者保存到SurfaceControl中，最后将native层SurfaceControl指针保存到java层SurfaceControl。
* native层SurfaceControl创建好了之后就可以通过此对象创建native层的Surface对象，最后将native层Surface指针保存到java层Surface，最终java层和native层的Surface和SurfaceControl都创建完毕。

下面我们通过代码分析，首先在WindowManagerService的relayoutWindow方法中，会调用createSurfaceControl方法：

我们继续从SurfaceControl构造器(SurfaceControl.Builder)的build方法开始分析SurfaceControl的创建过程：

```java
//frameworks/base/core/java/android/view/SurfaceControl.java
public final class SurfaceControl implements Parcelable {
    public static class Builder {
        public SurfaceControl build() {
            if (mWidth < 0 || mHeight < 0) {
                throw new IllegalStateException(
                        "width and height must be positive or unset");
            }
            if ((mWidth > 0 || mHeight > 0) && (isEffectLayer() || isContainerLayer())) {
                throw new IllegalStateException(
                        "Only buffer layers can set a valid buffer size.");
            }
 
 
            if ((mFlags & FX_SURFACE_MASK) == FX_SURFACE_NORMAL) {
                setBLASTLayer();
            }
 
 
            return new SurfaceControl(
                    mSession, mName, mWidth, mHeight, mFormat, mFlags, mParent, mMetadata,
                    mLocalOwnerView, mCallsite);
        }
    }
}
```

直接通过new的方式创建SurfaceControl，接着看SurfaceControl的构造方法：

```java
// frameworks/base/core/java/android/view/SurfaceControl.java
private SurfaceControl(SurfaceSession session, String name, int w, int h, int format, int flags,
            SurfaceControl parent, SparseIntArray metadata)
                    throws OutOfResourcesException, IllegalArgumentException {
       	......
        mName = name;
        mWidth = w;
        mHeight = h;
        Parcel metaParcel = Parcel.obtain();
        try {
            if (metadata != null && metadata.size() > 0) {
                metaParcel.writeInt(metadata.size());
                for (int i = 0; i < metadata.size(); ++i) {
                    metaParcel.writeInt(metadata.keyAt(i));
                    metaParcel.writeByteArray(
                            ByteBuffer.allocate(4).order(ByteOrder.nativeOrder())
                                    .putInt(metadata.valueAt(i)).array());
                }
                metaParcel.setDataPosition(0);
            }
            mNativeObject = nativeCreate(session, name, w, h, format, flags,
                    parent != null ? parent.mNativeObject : 0, metaParcel);
        } finally {
            metaParcel.recycle();
        }
        if (mNativeObject == 0) {
            throw new OutOfResourcesException(
                    "Couldn't allocate SurfaceControl native object");
        }
 
 
        mCloseGuard.open("release");
    }
```

这里面核心就是调用nativeCreate方法，并将window的各种数据一并传递到native层处理，对应的native类是android\_view\_SurfaceControl。

```cpp
// frameworks/base/core/jni/android_view_SurfaceControl.cpp
static jlong nativeCreate(JNIEnv* env, jclass clazz, jobject sessionObj,
        jstring nameStr, jint w, jint h, jint format, jint flags, jlong parentObject,
        jobject metadataParcel) {
    ScopedUtfChars name(env, nameStr);
    //应用层访问SurfaceFlinger的client端
    sp<SurfaceComposerClient> client;
 
 
    //这里的sessionObj是java层传递下来的SurfaceSession对象，如果不为空就从此对象中获取SurfaceComposerClient，否则重新创建一个
    if (sessionObj != NULL) {
        client = android_view_SurfaceSession_getClient(env, sessionObj);
    } else {
        client = SurfaceComposerClient::getDefault();
    }
    //parentObject是java层传递下来的SurfaceControl，将其强转为native层SurfaceControl，parentObject有可能为空的
    SurfaceControl *parent = reinterpret_cast<SurfaceControl*>(parentObject);
    sp<SurfaceControl> surface;
    LayerMetadata metadata;
    Parcel* parcel = parcelForJavaObject(env, metadataParcel);
    if (parcel && !parcel->objectsCount()) {
        status_t err = metadata.readFromParcel(parcel);
        if (err != NO_ERROR) {
          jniThrowException(env, "java/lang/IllegalArgumentException",
                            "Metadata parcel has wrong format");
        }
    }
    //此函数是具体初始化SurfaceControl的函数
    status_t err = client->createSurfaceChecked(
            String8(name.c_str()), w, h, format, &surface, flags, parent, std::move(metadata));
    if (err == NAME_NOT_FOUND) {
        jniThrowException(env, "java/lang/IllegalArgumentException", NULL);
        return 0;
    } else if (err != NO_ERROR) {
        jniThrowException(env, OutOfResourcesException, NULL);
        return 0;
    }
    //增加引用计数
    surface->incStrong((void *)nativeCreate);
    //将创建完成的native层SurfaceControl返回到java层
    return reinterpret_cast<jlong>(surface.get());
}
```

在createSurfaceChecked中调用通过new的方式创建SurfaceControl：

```cpp
// frameworks/native/libs/gui/SurfaceControl.cpp
sp<ISurfaceComposerClient>  mClient;
status_t SurfaceComposerClient::createSurfaceChecked(const String8& name, uint32_t w, uint32_t h,
                                                     PixelFormat format,
                                                     sp<SurfaceControl>* outSurface, uint32_t flags,
                                                     SurfaceControl* parent,
                                                     LayerMetadata metadata) {
    sp<SurfaceControl> sur;
    status_t err = mStatus;
 
 
    if (mStatus == NO_ERROR) {
        sp<IBinder> handle;
        sp<IBinder> parentHandle;
        sp<IGraphicBufferProducer> gbp;
 
 
        if (parent != nullptr) {
            parentHandle = parent->getHandle();
        }
     
        err = mClient->createSurface(name, w, h, format, flags, parentHandle, std::move(metadata), &handle, &gbp); //创建Surface
        if (err == NO_ERROR) {
            *outSurface = new SurfaceControl(this, handle, gbp, true /* owned */); //创建SurfaceControl
        }
    }
    return err;
}
```

上面方法主要处理如下：

1、调用mClient(ISurfaceComposerClient)的createSurface方法创建Surface。

2、通过new的方式创建SurfaceControl。

下面分别进行分析：

## ISurfaceComposerClient createSurface

调用mClient(ISurfaceComposerClient)的createSurface方法创建Surface：

```cpp
//frameworks/native/libs/gui/ISurfaceComposerClient.cpp
class BpSurfaceComposerClient : public SafeBpInterface<ISurfaceComposerClient> {
    status_t createSurface(const String8& name, uint32_t width, uint32_t height, PixelFormat format,
                           uint32_t flags, const sp<IBinder>& parent, LayerMetadata metadata,
                           sp<IBinder>* handle, sp<IGraphicBufferProducer>* gbp,
                           int32_t* outLayerId, uint32_t* outTransformHint) override {
        return callRemote<decltype(&ISurfaceComposerClient::createSurface)>(Tag::CREATE_SURFACE,
                                                                            name, width, height,
                                                                            format, flags, parent,
                                                                            std::move(metadata),
                                                                            handle, gbp, outLayerId,
                                                                            outTransformHint);
    }
}
```

调用callRemote方法，之后BnSurfaceComposerClient的onTransact方法会运行：

```cpp
//frameworks/native/libs/gui/ISurfaceComposerClient.cpp
status_t BnSurfaceComposerClient::onTransact(uint32_t code, const Parcel& data, Parcel* reply,
                                             uint32_t flags) {
    if (code < IBinder::FIRST_CALL_TRANSACTION || code > static_cast<uint32_t>(Tag::LAST)) {
        return BBinder::onTransact(code, data, reply, flags);
    }
    auto tag = static_cast<Tag>(code);
    switch (tag) {
        case Tag::CREATE_SURFACE:
            return callLocal(data, reply, &ISurfaceComposerClient::createSurface);
        case Tag::CREATE_WITH_SURFACE_PARENT:
            return callLocal(data, reply, &ISurfaceComposerClient::createWithSurfaceParent);
        case Tag::CLEAR_LAYER_FRAME_STATS:
            return callLocal(data, reply, &ISurfaceComposerClient::clearLayerFrameStats);
        case Tag::GET_LAYER_FRAME_STATS:
            return callLocal(data, reply, &ISurfaceComposerClient::getLayerFrameStats);
        case Tag::MIRROR_SURFACE:
            return callLocal(data, reply, &ISurfaceComposerClient::mirrorSurface);
    }
}
```

### Client createSurface

调用Client的createSurface函数：

[Android13 Surface创建流程分析](Android13%20Surface%E5%88%9B%E5%BB%BA%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90)

## new SurfaceControl

通过new的方式创建SurfaceControl，SurfaceControl的构造函数：

```cpp
// frameworks/native/libs/gui/SurfaceControl.cpp
SurfaceControl::SurfaceControl(
        const sp<SurfaceComposerClient>& client,
        const sp<IBinder>& handle,
        const sp<IGraphicBufferProducer>& gbp,
        bool owned)
    : mClient(client), mHandle(handle), mGraphicBufferProducer(gbp), mOwned(owned),
      mExtension(mHandle, &mGraphicBufferProducer, mOwned)
{
}
```
