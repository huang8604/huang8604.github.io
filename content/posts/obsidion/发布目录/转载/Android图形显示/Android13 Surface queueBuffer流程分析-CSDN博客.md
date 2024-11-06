---
title: Android13 Surface queueBuffer流程分析_surface::queuebuffer-CSDN博客
created: 2024-09-26
tags:
  - clippings
  - blog
  - 转载
collections:
  - 图形显示
date: 2024-11-06T06:57:52.879Z
lastmod: 2024-11-05T01:33:59.096Z
---
Surface::queueBuffer是Android系统中Surface类的一个成员函数，用于将图像数据放入Surface的缓冲区中。它的调用地方主要包括以下几个：

1. 应用程序：应用程序可以通过Surface对象的queueBuffer函数将图像数据发送给SurfaceFlinger服务。这样，SurfaceFlinger就可以将图像数据进行合成和显示。
2. SurfaceFlinger服务：SurfaceFlinger是Android系统中负责显示合成的服务。它会定期地从各个应用程序的Surface中获取图像数据，并进行合成和渲染。在合成过程中，SurfaceFlinger会调用Surface的queueBuffer函数将合成后的图像数据放入Surface的缓冲区中。
3. Hardware Composer：在一些支持硬件加速的设备上，SurfaceFlinger会将合成后的图像数据交给Hardware Composer来进行最终的渲染和显示。在这个过程中，Hardware Composer也会调用Surface的queueBuffer函数将图像数据放入硬件缓冲区中。

当对GraphicBuffer的绘制操作完成之后就需要调用queueBuffer函数将这块buffer放入BufferQueue队列中并通过回调通知消费者使用这块buffer，queueBuffer方法代码如下：

```cpp
//framework/native/libs/gui/Surface.cpp
sp<IGraphicBufferProducer> mGraphicBufferProducer;
class Surface : public ANativeObjectBase<ANativeWindow, Surface, RefBase> {
int Surface::queueBuffer(android_native_buffer_t* buffer, int fenceFd) {
    ATRACE_CALL();
    ALOGV("Surface::queueBuffer");
    Mutex::Autolock lock(mMutex);
 
 
    int i = getSlotFromBufferLocked(buffer); //找到buffer在mSlots中的下标
    if (i < 0) {
        if (fenceFd >= 0) {
            close(fenceFd);
        }
        return i;
    }
    if (mSharedBufferSlot == i && mSharedBufferHasBeenQueued) {
        if (fenceFd >= 0) {
            close(fenceFd);
        }
        return OK;
    }
 
 
    IGraphicBufferProducer::QueueBufferOutput output;
    IGraphicBufferProducer::QueueBufferInput input;
    getQueueBufferInputLocked(buffer, fenceFd, mTimestamp, &input);
    applyGrallocMetadataLocked(buffer, input);
    sp<Fence> fence = input.fence;
 
 
    nsecs_t now = systemTime();
 
 
    status_t err = mGraphicBufferProducer->queueBuffer(i, input, &output);
    mLastQueueDuration = systemTime() - now;
    if (err != OK)  {
        ALOGE("queueBuffer: error queuing buffer, %d", err);
    }
 
 
    onBufferQueuedLocked(i, fence, output);
    return err;
}
}
```

## BpGraphicBufferProducer queueBuffer

调用mGraphicBufferProducer(IGraphicBufferProducer)的queueBuffer方法，IGraphicBufferProducer是一个接口，由BpGraphicBufferProducer实现：

```cpp
//frameworks/native/libs/gui/IGraphicBufferProducer.cpp
class BpGraphicBufferProducer : public BpInterface<IGraphicBufferProducer> {
    virtual status_t queueBuffer(int buf,
            const QueueBufferInput& input, QueueBufferOutput* output) {
        Parcel data, reply;
 
 
        data.writeInterfaceToken(IGraphicBufferProducer::getInterfaceDescriptor());
        data.writeInt32(buf);
        data.write(input);
 
 
        status_t result = remote()->transact(QUEUE_BUFFER, data, &reply);
        if (result != NO_ERROR) {
            return result;
        }
 
 
        result = reply.read(*output);
        if (result != NO_ERROR) {
            return result;
        }
 
 
        result = reply.readInt32();
        return result;
    }
}
```

发送QUEUE\_BUFFER消息，发送的消息在BnGraphicBufferProducer的onTransact方法中处理：

```cpp
//frameworks/native/libs/gui/IGraphicBufferProducer.cpp
status_t BnGraphicBufferProducer::onTransact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    switch(code) {
        case QUEUE_BUFFER: {
            CHECK_INTERFACE(IGraphicBufferProducer, data, reply);
 
 
            int buf = data.readInt32();
            QueueBufferInput input(data);
            QueueBufferOutput output;
            status_t result = queueBuffer(buf, input, &output);
            reply->write(output);
            reply->writeInt32(result);
 
 
            return NO_ERROR;
        }
    }
}
```

### BufferQueueProducer queueBuffer

调用BnGraphicBufferProducer的queueBuffer方法，BnGraphicBufferProducer继承于BufferQueueProducer，调用BufferQueueProducer的queueBuffer方法，将图形缓冲区添加到队列中。当应用程序完成对图形缓冲区的绘制后，可以调用queueBuffer方法将其提交给SurfaceFlinger进行显示。

[Android13 BufferQueueProducer queueBuffer流程分析-CSDN博客](/Android13%20BufferQueueProducer%20queueBuffer%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90-CSDN%E5%8D%9A%E5%AE%A2)
