---
title: Android13 GraphicBuffer 创建流程-CSDN博客
created: 2024-09-26
tags:
  - clippings
  - 转载
  - blog
collections:
  - 图形显示
date: 2024-11-06T06:57:53.177Z
lastmod: 2024-11-05T01:33:59.398Z
---
GraphicBuffer用于管理图形缓存数据的类，GraphicBuffer的构造方法如下：

```cpp
//frameworks/native/libs/ui/GraphicBuffer.cpp
class GraphicBuffer
    : public ANativeObjectBase<ANativeWindowBuffer, GraphicBuffer, RefBase>,
      public Flattenable<GraphicBuffer>{
	GraphicBuffer::GraphicBuffer(uint32_t inWidth, uint32_t inHeight, PixelFormat inFormat,
	                             uint32_t inLayerCount, uint64_t inUsage, std::string requestorName)
	      : GraphicBuffer() {
	    mInitCheck = initWithSize(inWidth, inHeight, inFormat, inLayerCount, inUsage,
	                              std::move(requestorName));
	}
}
```

GraphicBuffer 的构造函数非常简单, 它只是调用了一个初始化函数 initWithSize：

```cpp
//frameworks/native/libs/ui/GraphicBuffer.cpp
class GraphicBuffer
    : public ANativeObjectBase<ANativeWindowBuffer, GraphicBuffer, RefBase>,
      public Flattenable<GraphicBuffer>{
	status_t GraphicBuffer::initWithSize(uint32_t inWidth, uint32_t inHeight,
	        PixelFormat inFormat, uint32_t inLayerCount, uint64_t inUsage,
	        std::string requestorName){
	    // 获取一个 GraphicBufferAllocator 对象, 这个对象是一个单例
	    // GraphicBufferAllocator 主要负责 GraphicBuffer 的内存分配
	    GraphicBufferAllocator& allocator = GraphicBufferAllocator::get();
	    uint32_t outStride = 0;
	    // 分配一块制定宽高的 GraphicBuffer
	    status_t err = allocator.allocate(inWidth, inHeight, inFormat, inLayerCount,
	            inUsage, &handle, &outStride, mId,
	            std::move(requestorName));
	    if (err == NO_ERROR) {
	      	// 通过 GraphicBufferMapper 将这块 GraphicBuffer 的参数记录下来
	      	// GraphicBufferMapper 负责的是 GraphicBuffer 的内存映射
	        mBufferMapper.getTransportSize(handle, &mTransportNumFds, &mTransportNumInts);
	 
	 
		 // 初始化参数
	        width = static_cast<int>(inWidth);
	        height = static_cast<int>(inHeight);
	        format = inFormat;
	        layerCount = inLayerCount;
	        usage = inUsage;
	        usage_deprecated = int(usage);
	        stride = static_cast<int>(outStride);
	    }
	    return err;
	}
}
```

调用allocator(GraphicBufferAllocator)的allocate方法，分配一块制定宽高的 GraphicBuffer：

```cpp
//frameworks/native/libs/ui/GraphicBufferAllocator.cpp
class GraphicBufferAllocator : public Singleton<GraphicBufferAllocator> {
	status_t GraphicBufferAllocator::allocate(uint32_t width, uint32_t height, PixelFormat format,
	                                          uint32_t layerCount, uint64_t usage,
	                                          buffer_handle_t* handle, uint32_t* stride,
	                                          uint64_t /*graphicBufferId*/, std::string requestorName) {
	    return allocateHelper(width, height, format, layerCount, usage, handle, stride, requestorName,
	                          true);
	}
}
```

## GraphicBufferAllocator allocateHelper

调用GraphicBufferAllocator的allocateHelper方法：

```cpp
//frameworks/native/libs/ui/GraphicBufferAllocator.cpp
class GraphicBufferAllocator : public Singleton<GraphicBufferAllocator> {
std::unique_ptr<const GrallocAllocator> mAllocator;
status_t GraphicBufferAllocator::allocateHelper(uint32_t width, uint32_t height, PixelFormat format,
                                                uint32_t layerCount, uint64_t usage,
                                                buffer_handle_t* handle, uint32_t* stride,
                                                std::string requestorName, bool importBuffer) {
    ATRACE_CALL();
 
 
    // make sure to not allocate a N x 0 or 0 x N buffer, since this is
    // allowed from an API stand-point allocate a 1x1 buffer instead.
    // 如果宽或者高为0, 则将宽高设置为1
    if (!width || !height)
        width = height = 1;
 
 
    const uint32_t bpp = bytesPerPixel(format);
    if (std::numeric_limits<size_t>::max() / width / height < static_cast<size_t>(bpp)) {
        ALOGE("Failed to allocate (%u x %u) layerCount %u format %d "
              "usage %" PRIx64 ": Requesting too large a buffer size",
              width, height, layerCount, format, usage);
        return BAD_VALUE;
    }
 
 
    // Ensure that layerCount is valid.
    // 如果图层的数量少于1, 则将图层的数量设置为1
    if (layerCount < 1) {
        layerCount = 1;
    }
 
 
    // TODO(b/72323293, b/72703005): Remove these invalid bits from callers
    usage &= ~static_cast<uint64_t>((1 << 10) | (1 << 13));
 
 
    // 分配内存，使用的是 GrallocAllocator 指针，根据不同的版本有哦不同的实现，这里我们假设它的实现是 Gralloc3Allocator
    status_t error = mAllocator->allocate(requestorName, width, height, format, layerCount, usage,
                                          1, stride, handle, importBuffer);
    if (error != NO_ERROR) {
        ALOGE("Failed to allocate (%u x %u) layerCount %u format %d "
              "usage %" PRIx64 ": %d",
              width, height, layerCount, format, usage, error);
        return error;
    }
 
 
    if (!importBuffer) {
        return NO_ERROR;
    }
    size_t bufSize;
 
 
    // if stride has no meaning or is too large,
    // approximate size with the input width instead
    if ((*stride) != 0 &&
        std::numeric_limits<size_t>::max() / height / (*stride) < static_cast<size_t>(bpp)) {
        bufSize = static_cast<size_t>(width) * height * bpp;
    } else {
        bufSize = static_cast<size_t>((*stride)) * height * bpp;
    }
 
 
    // 初始化参数
    Mutex::Autolock _l(sLock);
    KeyedVector<buffer_handle_t, alloc_rec_t>& list(sAllocList);
    alloc_rec_t rec;
    rec.width = width;
    rec.height = height;
    rec.stride = *stride;
    rec.format = format;
    rec.layerCount = layerCount;
    rec.usage = usage;
    rec.size = bufSize;
    rec.requestorName = std::move(requestorName);
    list.add(*handle, rec);
 
 
    return NO_ERROR;
}
}
```

### Gralloc3Allocator allocate

GrallocAllocator有多个实现版本，Gralloc3Allocator是其中一个版本，调用Gralloc3Allocator的allocate方法：

```cpp
//framewrks/native/libs/ui/Gralloc3.cpp
sp<hardware::graphics::allocator::V3_0::IAllocator> mAllocator;
class Gralloc3Allocator : public GrallocAllocator {
status_t Gralloc3Allocator::allocate(std::string /*requestorName*/, uint32_t width, uint32_t height,
                                     android::PixelFormat format, uint32_t layerCount,
                                     uint64_t usage, uint32_t bufferCount, uint32_t* outStride,
                                     buffer_handle_t* outBufferHandles, bool importBuffers) const {
    IMapper::BufferDescriptorInfo descriptorInfo;
    sBufferDescriptorInfo(width, height, format, layerCount, usage, &descriptorInfo);
 
 
    BufferDescriptor descriptor;
    status_t error = mMapper.createDescriptor(static_cast<void*>(&descriptorInfo),
                                              static_cast<void*>(&descriptor));
    if (error != NO_ERROR) {
        return error;
    }
 
 
    auto ret = mAllocator->allocate(descriptor, bufferCount,
                                    [&](const auto& tmpError, const auto& tmpStride,
                                        const auto& tmpBuffers) {
                                        error = static_cast<status_t>(tmpError);
                                        if (tmpError != Error::NONE) {
                                            return;
                                        }
 
 
                                        if (importBuffers) {
                                            for (uint32_t i = 0; i < bufferCount; i++) {
                                                error = mMapper.importBuffer(tmpBuffers[i],
                                                                             &outBufferHandles[i]);
                                                if (error != NO_ERROR) {
                                                    for (uint32_t j = 0; j < i; j++) {
                                                        mMapper.freeBuffer(outBufferHandles[j]);
                                                        outBufferHandles[j] = nullptr;
                                                    }
                                                    return;
                                                }
                                            }
                                        } else {
                                            for (uint32_t i = 0; i < bufferCount; i++) {
                                                outBufferHandles[i] = native_handle_clone(
                                                        tmpBuffers[i].getNativeHandle());
                                                if (!outBufferHandles[i]) {
                                                    for (uint32_t j = 0; j < i; j++) {
                                                        auto buffer = const_cast<native_handle_t*>(
                                                                outBufferHandles[j]);
                                                        native_handle_close(buffer);
                                                        native_handle_delete(buffer);
                                                        outBufferHandles[j] = nullptr;
                                                    }
                                                }
                                            }
                                        }
                                        *outStride = tmpStride;
                                    });
 
 
    // make sure the kernel driver sees BC_FREE_BUFFER and closes the fds now
    hardware::IPCThreadState::self()->flushCommands();
 
 
    return (ret.isOk()) ? error : static_cast<status_t>(kTransactionError);
}
}
```
