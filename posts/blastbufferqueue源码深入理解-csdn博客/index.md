# BLASTBufferQueue源码深入理解-CSDN博客

## BLASTBufferQueue创建部分

```cpp
BLASTBufferQueue::BLASTBufferQueue(const std::string&amp; name, bool updateDestinationFrame)
      : mSurfaceControl(nullptr),
        mSize(1, 1),
        mRequestedSize(mSize),
        mFormat(PIXEL_FORMAT_RGBA_8888),
        mTransactionReadyCallback(nullptr),
        mSyncTransaction(nullptr),
        mUpdateDestinationFrame(updateDestinationFrame) {
    //创建BufferQueue部分，并且会给mProducer，mConsumer进行赋值
    createBufferQueue(&amp;mProducer, &amp;mConsumer);
   
   //创建BLASTBufferItemConsumer部分
    mBufferItemConsumer = new BLASTBufferItemConsumer(mConsumer,
                                                      GraphicBuffer::USAGE_HW_COMPOSER |
                                                              GraphicBuffer::USAGE_HW_TEXTURE,
                                                      1, false, this);
    static int32_t id = 0;
    mName = name &#43; &#34;#&#34; &#43; std::to_string(id);
    auto consumerName = mName &#43; &#34;(BLAST Consumer)&#34; &#43; std::to_string(id);
    mQueuedBufferTrace = &#34;QueuedBuffer - &#34; &#43; mName &#43; &#34;BLAST#&#34; &#43; std::to_string(id);
    id&#43;&#43;;
    mBufferItemConsumer-&gt;setName(String8(consumerName.c_str()));
    mBufferItemConsumer-&gt;setFrameAvailableListener(this);
    mBufferItemConsumer-&gt;setBufferFreedListener(this);

    ComposerService::getComposerService()-&gt;getMaxAcquiredBufferCount(&amp;mMaxAcquiredBuffers);
    mBufferItemConsumer-&gt;setMaxAcquiredBufferCount(mMaxAcquiredBuffers);
    mCurrentMaxAcquiredBufferCount = mMaxAcquiredBuffers;
    mNumAcquired = 0;
    mNumFrameAvailable = 0;

}
```

### BufferQueue部分

下面来看看核心的方法createBufferQueue

```cpp
void BLASTBufferQueue::createBufferQueue(sp&lt;IGraphicBufferProducer&gt;* outProducer,
                                         sp&lt;IGraphicBufferConsumer&gt;* outConsumer) {
  //创建核心的BufferQueueCore
    sp&lt;BufferQueueCore&gt; core(new BufferQueueCore());

  //基于BufferQueueCore创建BBQBufferQueueProducer
    sp&lt;IGraphicBufferProducer&gt; producer(new BBQBufferQueueProducer(core));
     //基于BufferQueueCore创建BufferQueueConsumer
    sp&lt;BufferQueueConsumer&gt; consumer(new BufferQueueConsumer(core));
    consumer-&gt;setAllowExtraAcquire(true);
  
    *outProducer = producer;
    *outConsumer = consumer;
}
```

下面重点看看BufferQueueCore

```cpp
BufferQueueCore::BufferQueueCore()
      : mMutex(),//若干个成员变量初始化
        mIsAbandoned(false),
        mConsumerControlledByApp(false),
        mConsumerName(getUniqueName()),
        mConsumerListener(),
        mConsumerUsageBits(0),
        mConsumerIsProtected(false),
        mConnectedApi(NO_CONNECTED_API),
        mLinkedToDeath(),
        mConnectedProducerListener(),
        mBufferReleasedCbEnabled(false),
        mSlots(),
        mQueue(),
        mFreeSlots(),
        mFreeBuffers(),
        mUnusedSlots(),
        mActiveBuffers(),
        mDequeueCondition(),
        mDequeueBufferCannotBlock(false),
        mQueueBufferCanDrop(false),
        mLegacyBufferDrop(true),
        mDefaultBufferFormat(PIXEL_FORMAT_RGBA_8888),
        mDefaultWidth(1),
        mDefaultHeight(1),
        mDefaultBufferDataSpace(HAL_DATASPACE_UNKNOWN),
        mMaxBufferCount(BufferQueueDefs::NUM_BUFFER_SLOTS),
        mMaxAcquiredBufferCount(1),
        mMaxDequeuedBufferCount(1),
        mBufferHasBeenQueued(false),
        mFrameCounter(0),
        mTransformHint(0),
        mIsAllocating(false),
        mIsAllocatingCondition(),
        mAllowAllocation(true),
        mBufferAge(0),
        mGenerationNumber(0),
        mAsyncMode(false),
        mSharedBufferMode(false),
        mAutoRefresh(false),
        mSharedBufferSlot(INVALID_BUFFER_SLOT),
        mSharedBufferCache(Rect::INVALID_RECT, 0, NATIVE_WINDOW_SCALING_MODE_FREEZE,
                           HAL_DATASPACE_UNKNOWN),
        mLastQueuedSlot(INVALID_BUFFER_SLOT),
        mUniqueId(getUniqueId()),
        mAutoPrerotation(false),
        mTransformHintInUse(0) {
    int numStartingBuffers = getMaxBufferCountLocked();
    for (int s = 0; s &lt; numStartingBuffers; s&#43;&#43;) {
        mFreeSlots.insert(s);//初始化时候针对mFreeSlots填入了MaxBufferCount个
    }
    for (int s = numStartingBuffers; s &lt; BufferQueueDefs::NUM_BUFFER_SLOTS;
            s&#43;&#43;) {
        mUnusedSlots.push_front(s);//除了mFreeSlots部分，其他都是填入mUnusedSlots
    }
}
```

上面的构造中有以下几个[成员变量](https://so.csdn.net/so/search?q=%E6%88%90%E5%91%98%E5%8F%98%E9%87%8F\&amp;spm=1001.2101.3001.7020)非常关键需要重点介绍

```cpp
// mSlots is an array of buffer slots that must be mirrored on the producer
    // side. This allows buffer ownership to be transferred between the producer
    // and consumer without sending a GraphicBuffer over Binder. The entire
    // array is initialized to NULL at construction time, and buffers are
    // allocated for a slot when requestBuffer is called with that slot&#39;s index.
    BufferQueueDefs::SlotsType mSlots;

    // mQueue is a FIFO of queued buffers used in synchronous mode.
    Fifo mQueue;

    // mFreeSlots contains all of the slots which are FREE and do not currently
    // have a buffer attached.
    std::set&lt;int&gt; mFreeSlots;

    // mFreeBuffers contains all of the slots which are FREE and currently have
    // a buffer attached.
    std::list&lt;int&gt; mFreeBuffers;

    // mUnusedSlots contains all slots that are currently unused. They should be
    // free and not have a buffer attached.
    std::list&lt;int&gt; mUnusedSlots;

    // mActiveBuffers contains all slots which have a non-FREE buffer attached.
    std::set&lt;int&gt; mActiveBuffers;
```

mQueue —&gt; 存放BufferItem的[FIFO](https://so.csdn.net/so/search?q=FIFO\&amp;spm=1001.2101.3001.7020)队列\
mSlots —&gt; BufferSlot[结构体数组](https://so.csdn.net/so/search?q=%E7%BB%93%E6%9E%84%E4%BD%93%E6%95%B0%E7%BB%84\&amp;spm=1001.2101.3001.7020)，数组长度固定为64\
mFreeSlots —&gt; BufferSlot状态为FREE，且没有GraphicBuffer与之相绑定的slot集合\
mFreeBuffers —&gt; BufferSlot状态为FREE，且有GraphicBuffer与之相绑定的slot集合\
mActiveBuffers —&gt; BufferSlot状态不为FREE（即DEQUEUED、QUEUED、ACQUIRED、SHARED）的slot集合。既然状态不是FREE，那么该BufferSlot必然有一个GraphicBuffer与之相绑定\
mUnusedSlots —&gt; 未参与使用的slot集合，由 mMaxBufferCount 决定

注意mFreeSlots，mFreeBuffers，mActiveBuffers，mUnusedSlots都其实是一个int集合，内容就是mSlots数组的一个个index

详细剖析一下mSlots这个变量：

```cpp
namespace BufferQueueDefs {
        typedef BufferSlot SlotsType[NUM_BUFFER_SLOTS];
    } // namespace BufferQueueDefs
    
struct BufferSlot {

    BufferSlot()
    : mGraphicBuffer(nullptr),
      mEglDisplay(EGL_NO_DISPLAY),
      mBufferState(),
      mRequestBufferCalled(false),
      mFrameNumber(0),
      mEglFence(EGL_NO_SYNC_KHR),
      mFence(Fence::NO_FENCE),
      mAcquireCalled(false),
      mNeedsReallocation(false) {
    }
      }
```

可以看到mSlots其实BufferSlot的一个数组，BufferSlot重要包含了mGraphicBuffer和mBufferState两个核心变量，下面来看看核心变量mBufferState，类型是BufferState

```cpp
// BufferState tracks the states in which a buffer slot can be.
struct BufferState {

    // All slots are initially FREE (not dequeued, queued, acquired, or shared).
    BufferState()
    : mDequeueCount(0),
      mQueueCount(0),
      mAcquireCount(0),
      mShared(false) {
    }

    uint32_t mDequeueCount;
    uint32_t mQueueCount;
    uint32_t mAcquireCount;
    bool mShared;

    // A buffer can be in one of five states, represented as below:
    //
    //         | mShared | mDequeueCount | mQueueCount | mAcquireCount |
    // --------|---------|---------------|-------------|---------------|
    // FREE    |  false  |       0       |      0      |       0       |
    // DEQUEUED|  false  |       1       |      0      |       0       |
    // QUEUED  |  false  |       0       |      1      |       0       |
    // ACQUIRED|  false  |       0       |      0      |       1       |
    // SHARED  |  true   |      any      |     any     |      any      |
    //
    // FREE indicates that the buffer is available to be dequeued by the
    // producer. The slot is &#34;owned&#34; by BufferQueue. It transitions to DEQUEUED
    // when dequeueBuffer is called.
    //
    // DEQUEUED indicates that the buffer has been dequeued by the producer, but
    // has not yet been queued or canceled. The producer may modify the
    // buffer&#39;s contents as soon as the associated release fence is signaled.
    // The slot is &#34;owned&#34; by the producer. It can transition to QUEUED (via
    // queueBuffer or attachBuffer) or back to FREE (via cancelBuffer or
    // detachBuffer).
    //
    // QUEUED indicates that the buffer has been filled by the producer and
    // queued for use by the consumer. The buffer contents may continue to be
    // modified for a finite time, so the contents must not be accessed until
    // the associated fence is signaled. The slot is &#34;owned&#34; by BufferQueue. It
    // can transition to ACQUIRED (via acquireBuffer) or to FREE (if another
    // buffer is queued in asynchronous mode).
    //
    // ACQUIRED indicates that the buffer has been acquired by the consumer. As
    // with QUEUED, the contents must not be accessed by the consumer until the
    // acquire fence is signaled. The slot is &#34;owned&#34; by the consumer. It
    // transitions to FREE when releaseBuffer (or detachBuffer) is called. A
    // detached buffer can also enter the ACQUIRED state via attachBuffer.
    //
    // SHARED indicates that this buffer is being used in shared buffer
    // mode. It can be in any combination of the other states at the same time,
    // except for FREE (since that excludes being in any other state). It can
    // also be dequeued, queued, or acquired multiple times.
```

可以看到这里的BufferState就是我们常说的一下几个状态\
// FREE\
// DEQUEUED\
// QUEUED\
// ACQUIRED\
// SHARED\
但是大家可以看到，每一个状态并不是用类似enum这种变量的,而是用对对应整形计数变量表示，比如：\
inline void dequeue() {\
mDequeueCount&#43;&#43;;\
}\
inline bool isDequeued() const {\
return mDequeueCount &gt; 0;\
}

isDequeued（）方法就是代表状态，dequeue就是操作变量，简单就是改变状态，都是使用各自计数来的

所有的状态方法如下：

```cpp
//是否为Free状态就是判断一下其他三个状态计数是否为0
    inline bool isFree() const {
        return !isAcquired() &amp;&amp; !isDequeued() &amp;&amp; !isQueued();
    }

    inline bool isDequeued() const {
        return mDequeueCount &gt; 0;
    }

    inline bool isQueued() const {
        return mQueueCount &gt; 0;
    }

    inline bool isAcquired() const {
        return mAcquireCount &gt; 0;
    }

    inline bool isShared() const {
        return mShared;
    }

    inline void reset() {
        *this = BufferState();
    }

    const char* string() const;

    inline void dequeue() {
        //dequeue操作就是简单的把mDequeueCount进行加操作
        mDequeueCount&#43;&#43;;
    }

    inline void detachProducer() {
        if (mDequeueCount &gt; 0) {
            mDequeueCount--;
        }
    }

    inline void attachProducer() {
        mDequeueCount&#43;&#43;;
    }

    inline void queue() {
        //会对mDequeueCount先进行减
        if (mDequeueCount &gt; 0) {
            mDequeueCount--;
        }
        //再对mQueueCount进行加操作
        mQueueCount&#43;&#43;;
    }

    inline void cancel() {
        if (mDequeueCount &gt; 0) {
            mDequeueCount--;
        }
    }

    inline void freeQueued() {
        if (mQueueCount &gt; 0) {
            mQueueCount--;
        }
    }

    inline void acquire() {
        //首先对mQueueCount会进行减操作
        if (mQueueCount &gt; 0) {
            mQueueCount--;
        }
        //然后再是对mAcquireCount进行加操作
        mAcquireCount&#43;&#43;;
    }

    inline void acquireNotInQueue() {
        mAcquireCount&#43;&#43;;
    }

    inline void release() {//release只对mAcquireCount进行了减操作
        if (mAcquireCount &gt; 0) {
            mAcquireCount--;
        }
    }

    inline void detachConsumer() {
        if (mAcquireCount &gt; 0) {
            mAcquireCount--;
        }
    }

    inline void attachConsumer() {
        mAcquireCount&#43;&#43;;
    }
};
```

总结图：

![50bc4396c75da53dd274ea74ac7912e5\_MD5](https://picgo.myjojo.fun:666/i/2024/09/29/66f92f3a6569a.png)

补充一下几个状态转移图：\
![d6f32a1e45a4b0524b0b3930413fdac4\_MD5](https://picgo.myjojo.fun:666/i/2024/09/29/66f92f3a23a8f.png)\
–&gt;1、一开始如果什么都没发生那肯定BufferSlot对应BufferState是FREE状态\
\----&gt;2、生产者app一般先进行dequue操作，这个时候状态就变成了 DEQUEUE状态\
\-------&gt;3、生产者app dequeue获取了graphicbuffer后，绘制完成进行queue操作，变成了QUEUED状态\
\------------&gt;4、queuebuffer后通知消费者，消费者接受通知后，会进行acquire操作，这个时候状态变成了ACQUIRED\
\---------------&gt;5、消费者对buffer消费完成后就可以对bufferslot进行relaese操作，这个时候就变成FREE状态

#### dequeueBuffer() --生产者方法

```cpp
status_t BufferQueueProducer::dequeueBuffer(int* outSlot, sp&lt;android::Fence&gt;* outFence,
                                            uint32_t width, uint32_t height, PixelFormat format,
                                            uint64_t usage, uint64_t* outBufferAge,
                                            FrameEventHistoryDelta* outTimestamps) {
  //省略
        int found = BufferItem::INVALID_BUFFER_SLOT;//作为入参found
        while (found == BufferItem::INVALID_BUFFER_SLOT) {
        		//核心方法从mSlots数组找到一个可以用的BufferSlot,把相关的index赋值到found这个变量
            status_t status = waitForFreeSlotThenRelock(FreeSlotCaller::Dequeue, lock, &amp;found);
        //省略
        }
        ATRACE_FORMAT(&#34;dequeueBuffer found = %d&#34;,found);
        const sp&lt;GraphicBuffer&gt;&amp; buffer(mSlots[found].mGraphicBuffer);
        
        *outSlot = found;
        ATRACE_BUFFER_INDEX(found);

        attachedByConsumer = mSlots[found].mNeedsReallocation;
        mSlots[found].mNeedsReallocation = false;

        mSlots[found].mBufferState.dequeue();//把状态变成Dequeue

        if ((buffer == nullptr) ||
                buffer-&gt;needsReallocation(width, height, format, BQ_LAYER_COUNT, usage))
        {
         //如果需要进行buffer申请
            returnFlags |= BUFFER_NEEDS_REALLOCATION;
        } else {
            // We add 1 because that will be the frame number when this buffer
            // is queued
            mCore-&gt;mBufferAge = mCore-&gt;mFrameCounter &#43; 1 - mSlots[found].mFrameNumber;
        }
//省略部分

        if (!(returnFlags &amp; BUFFER_NEEDS_REALLOCATION)) {//不需要allocation
            if (mCore-&gt;mConsumerListener != nullptr) {//通知回调一下consumeronFrameDequeued
                mCore-&gt;mConsumerListener-&gt;onFrameDequeued(mSlots[*outSlot].mGraphicBuffer-&gt;getId());
            }
        }
    } // Autolock scope

    if (returnFlags &amp; BUFFER_NEEDS_REALLOCATION) {
     //开始申请GraphicBuffer
        sp&lt;GraphicBuffer&gt; graphicBuffer = new GraphicBuffer(
                width, height, format, BQ_LAYER_COUNT, usage,
                {mConsumerName.string(), mConsumerName.size()});

        status_t error = graphicBuffer-&gt;initCheck();
        //申请GraphicBuffer成功
            if (error == NO_ERROR &amp;&amp; !mCore-&gt;mIsAbandoned) {
                graphicBuffer-&gt;setGenerationNumber(mCore-&gt;mGenerationNumber);
                //把申请成功的buffer赋值到了mSlots的mGraphicBuffer中
                mSlots[*outSlot].mGraphicBuffer = graphicBuffer;
                if (mCore-&gt;mConsumerListener != nullptr) {
                    mCore-&gt;mConsumerListener-&gt;onFrameDequeued(
                            mSlots[*outSlot].mGraphicBuffer-&gt;getId());
                }
            }
        } // Autolock scope
    }
 //省略
    return returnFlags;
}
```

接下来看看核心方法waitForFreeSlotThenRelock

```cpp
status_t BufferQueueProducer::waitForFreeSlotThenRelock(FreeSlotCaller caller,
        std::unique_lock&lt;std::mutex&gt;&amp; lock, int* found) const {
  
    while (tryAgain) {
        int dequeuedCount = 0;
        int acquiredCount = 0;
        for (int s : mCore-&gt;mActiveBuffers) {
            if (mSlots[s].mBufferState.isDequeued()) {
                &#43;&#43;dequeuedCount;
            }
            if (mSlots[s].mBufferState.isAcquired()) {
                &#43;&#43;acquiredCount;
            }
        }
        *found = BufferQueueCore::INVALID_BUFFER_SLOT;
//省略部分
        if (tooManyBuffers) {
        
        } else {
            // If in shared buffer mode and a shared buffer exists, always
            // return it.
            if (mCore-&gt;mSharedBufferMode &amp;&amp; mCore-&gt;mSharedBufferSlot !=
                    BufferQueueCore::INVALID_BUFFER_SLOT) {
                *found = mCore-&gt;mSharedBufferSlot;
            } else {
                if (caller == FreeSlotCaller::Dequeue) {//如果dequeue方法
                    // If we&#39;re calling this from dequeue, prefer free buffers
                    int slot = getFreeBufferLocked();//先从FreeBuffer的数组中获取
                    if (slot != BufferQueueCore::INVALID_BUFFER_SLOT) {
                        *found = slot;
                    } else if (mCore-&gt;mAllowAllocation) {//如果没有找到则到FreeSlot数组中找
                        *found = getFreeSlotLocked();
                    }
                } else {
                    // If we&#39;re calling this from attach, prefer free slots
                    int slot = getFreeSlotLocked();//先从FreeSlot数组中找
                    if (slot != BufferQueueCore::INVALID_BUFFER_SLOT) {
                        *found = slot;
                    } else {//没找到再去FreeBuffer中找
                        *found = getFreeBufferLocked();
                    }
                }
            }
        }
//省略

    return NO_ERROR;
}
int BufferQueueProducer::getFreeSlotLocked() const {
    if (mCore-&gt;mFreeSlots.empty()) {
        return BufferQueueCore::INVALID_BUFFER_SLOT;
    }
    int slot = *(mCore-&gt;mFreeSlots.begin());
    mCore-&gt;mFreeSlots.erase(slot);
    return slot;
}
```

上面waitForFreeSlotThenRelock就是其实就是去FreeBuffer或者FreeSlot中寻找一个BufferSlot，返回这个slot的在数组的index,赋值给found

dequeueBuffer主要干的事情如下：\
1、通过waitForFreeSlotThenRelock寻找到Free状态的BufferSlot\
2、判断这个BufferSlot是否需要重新申请buffer，如果需要则进行构造新的GraphicBuffer

#### requestBuffer()–生产者方法

主要就是实现了根据传递进来的slot这个index，然后吧对应mSlots的mGraphicBuffer赋值给参数buf指针

```cpp
status_t BufferQueueProducer::requestBuffer(int slot, sp&lt;GraphicBuffer&gt;* buf) {
    mSlots[slot].mRequestBufferCalled = true;
    *buf = mSlots[slot].mGraphicBuffer;
    return NO_ERROR;
}
```

#### queueBuffer()–生产者方法

```cpp
//注意传递参数里面有一个关键的slot即mSlots中的index
status_t BufferQueueProducer::queueBuffer(int slot,
        const QueueBufferInput &amp;input, QueueBufferOutput *output) {
  
    BufferItem item;
    {

        mSlots[slot].mFence = acquireFence;
        mSlots[slot].mBufferState.queue();//这个地方就会调用mBufferState的queue，状态就改变了
        
				//开始把mSlots[slot]挨个赋值给BufferItem
        item.mAcquireCalled = mSlots[slot].mAcquireCalled;
        item.mGraphicBuffer = mSlots[slot].mGraphicBuffer;
        item.mCrop = crop;
        item.mTransform = transform &amp;
                ~static_cast&lt;uint32_t&gt;(NATIVE_WINDOW_TRANSFORM_INVERSE_DISPLAY);
        item.mTransformToDisplayInverse =
                (transform &amp; NATIVE_WINDOW_TRANSFORM_INVERSE_DISPLAY) != 0;
        item.mScalingMode = static_cast&lt;uint32_t&gt;(scalingMode);
        item.mTimestamp = requestedPresentTimestamp;
        item.mIsAutoTimestamp = isAutoTimestamp;
        item.mDataSpace = dataSpace;
        item.mHdrMetadata = hdrMetadata;
        item.mFrameNumber = currentFrameNumber;
        item.mSlot = slot;
        item.mFence = acquireFence;
        item.mFenceTime = acquireFenceTime;
        item.mIsDroppable = mCore-&gt;mAsyncMode ||
                (mConsumerIsSurfaceFlinger &amp;&amp; mCore-&gt;mQueueBufferCanDrop) ||
                (mCore-&gt;mLegacyBufferDrop &amp;&amp; mCore-&gt;mQueueBufferCanDrop) ||
                (mCore-&gt;mSharedBufferMode &amp;&amp; mCore-&gt;mSharedBufferSlot == slot);
        item.mSurfaceDamage = surfaceDamage;
        item.mQueuedBuffer = true;
        item.mAutoRefresh = mCore-&gt;mSharedBufferMode &amp;&amp; mCore-&gt;mAutoRefresh;
        item.mApi = mCore-&gt;mConnectedApi;

        if (mCore-&gt;mQueue.empty()) {//如果mQueue为空
       
            mCore-&gt;mQueue.push_back(item);//直接push到mQueue的
            frameAvailableListener = mCore-&gt;mConsumerListener;
        } else {
            // When the queue is not empty, we need to look at the last buffer
            // in the queue to see if we need to replace it
            const BufferItem&amp; last = mCore-&gt;mQueue.itemAt(
                    mCore-&gt;mQueue.size() - 1);//获取末尾的Item
             //判断末尾Item是不是可以被丢弃的
            if (last.mIsDroppable) {
			//省略
            } else {
                mCore-&gt;mQueue.push_back(item);//直接push
                frameAvailableListener = mCore-&gt;mConsumerListener;
            }
        }

        if (frameAvailableListener != nullptr) {
            frameAvailableListener-&gt;onFrameAvailable(item);//这里核心部分，会调用到消费者的onFrameAvailable
        } else if (frameReplacedListener != nullptr) {
            frameReplacedListener-&gt;onFrameReplaced(item);
        }

    return NO_ERROR;
}
```

上面queueBuffer主要干了以下几件事：\
1、根据入参slot，把mSlots中的BufferState变成Queued状态\
2、根据BufferSlot相关变量构建出一个新的BufferItem对象，且塞入mQueue集合\
3、最后通过onFrameAvailable通知消费者

#### onFrameAvailable

```cpp
void BLASTBufferQueue::onFrameAvailable(const BufferItem&amp; item) {
    std::function&lt;void(SurfaceComposerClient::Transaction*)&gt; prevCallback = nullptr;
    SurfaceComposerClient::Transaction* prevTransaction = nullptr;
    bool waitForTransactionCallback = !mSyncedFrameNumbers.empty();
        ATRACE_CALL();
    {
        BBQ_TRACE();
        std::unique_lock _lock{mMutex};
        const bool syncTransactionSet = mTransactionReadyCallback != nullptr;
        BQA_LOGV(&#34;onFrameAvailable-start syncTransactionSet=%s&#34;, boolToString(syncTransactionSet));

        if (syncTransactionSet) {
            bool mayNeedToWaitForBuffer = true;
            // If we are going to re-use the same mSyncTransaction, release the buffer that may
            // already be set in the Transaction. This is to allow us a free slot early to continue
            // processing a new buffer.
            if (!mAcquireSingleBuffer) {
                auto bufferData = mSyncTransaction-&gt;getAndClearBuffer(mSurfaceControl);
                if (bufferData) {
                    BQA_LOGD(&#34;Releasing previous buffer when syncing: framenumber=%&#34; PRIu64,
                             bufferData-&gt;frameNumber);
                    releaseBuffer(bufferData-&gt;generateReleaseCallbackId(),
                                  bufferData-&gt;acquireFence);
                    // Because we just released a buffer, we know there&#39;s no need to wait for a free
                    // buffer.
                    mayNeedToWaitForBuffer = false;
                }
            }

            if (mayNeedToWaitForBuffer) {
                flushAndWaitForFreeBuffer(_lock);
            }
        }

        // add to shadow queue
        mNumFrameAvailable&#43;&#43;;
        if (waitForTransactionCallback &amp;&amp; mNumFrameAvailable &gt;= 2) {
            acquireAndReleaseBuffer();
        }
        ATRACE_INT(mQueuedBufferTrace.c_str(),
                   mNumFrameAvailable &#43; mNumAcquired - mPendingRelease.size());

        BQA_LOGV(&#34;onFrameAvailable framenumber=%&#34; PRIu64 &#34; syncTransactionSet=%s&#34;,
                 item.mFrameNumber, boolToString(syncTransactionSet));
        {
            ATRACE_FORMAT(&#34;onFrameAvailable syncTransactionSet = %d&#34;,syncTransactionSet);
        }
        if (syncTransactionSet) {
            acquireNextBufferLocked(mSyncTransaction);

            // Only need a commit callback when syncing to ensure the buffer that&#39;s synced has been
            // sent to SF
            incStrong((void*)transactionCommittedCallbackThunk);
            mSyncTransaction-&gt;addTransactionCommittedCallback(transactionCommittedCallbackThunk,
                                                              static_cast&lt;void*&gt;(this));
            mSyncedFrameNumbers.emplace(item.mFrameNumber);
            if (mAcquireSingleBuffer) {
                prevCallback = mTransactionReadyCallback;
                prevTransaction = mSyncTransaction;
                mTransactionReadyCallback = nullptr;
                mSyncTransaction = nullptr;
            }
        } else if (!waitForTransactionCallback) {
            acquireNextBufferLocked(std::nullopt);
        }
    }
    if (prevCallback) {
        prevCallback(prevTransaction);
    }
}
```

#### acquireNextBufferLocked

```cpp
void BLASTBufferQueue::acquireNextBufferLocked(
        const std::optional&lt;SurfaceComposerClient::Transaction*&gt; transaction) {

    BufferItem bufferItem;

    status_t status =
            mBufferItemConsumer-&gt;acquireBuffer(&amp;bufferItem, 0 /* expectedPresent */, false);//消费者中获取bufferItem

    auto buffer = bufferItem.mGraphicBuffer;
 
    auto releaseBufferCallback =
            std::bind(releaseBufferCallbackThunk, wp&lt;BLASTBufferQueue&gt;(this) /* callbackContext */,
                      std::placeholders::_1, std::placeholders::_2, std::placeholders::_3);
    sp&lt;Fence&gt; fence = bufferItem.mFence ? new Fence(bufferItem.mFence-&gt;dup()) : Fence::NO_FENCE;
    //最为关键吧buffer设置到了事务中
    t-&gt;setBuffer(mSurfaceControl, buffer, fence, bufferItem.mFrameNumber, releaseBufferCallback);
    t-&gt;setDataspace(mSurfaceControl, static_cast&lt;ui::Dataspace&gt;(bufferItem.mDataSpace));
    t-&gt;setHdrMetadata(mSurfaceControl, bufferItem.mHdrMetadata);
    t-&gt;setSurfaceDamageRegion(mSurfaceControl, bufferItem.mSurfaceDamage);
    t-&gt;addTransactionCompletedCallback(transactionCallbackThunk, static_cast&lt;void*&gt;(this));
//省略
}
```

#### acquireBuffer()–消费者方法

```cpp
//注意这有个入参outBuffer
status_t BufferQueueConsumer::acquireBuffer(BufferItem* outBuffer,
        nsecs_t expectedPresent, uint64_t maxFrameNumber) {

        BufferQueueCore::Fifo::iterator front(mCore-&gt;mQueue.begin());//注意这个front就是mQueue的第一个
//省略
        if (sharedBufferAvailable &amp;&amp; mCore-&gt;mQueue.empty()) {
 
        } else if (acquireNonDroppableBuffer &amp;&amp; front-&gt;mIsDroppable) {
            return NO_BUFFER_AVAILABLE;
        } else {
            //把front的slot及指向对象赋值给outBuffer这个入参
            slot = front-&gt;mSlot;
            *outBuffer = *front;
        }

        if (!outBuffer-&gt;mIsStale) {
            mSlots[slot].mAcquireCalled = true;
            if (mCore-&gt;mQueue.empty()) {
                mSlots[slot].mBufferState.acquireNotInQueue();
            } else {
            //下面就是BufferState进行相关的改变，调用acquire方法
                mSlots[slot].mBufferState.acquire();
            }
            mSlots[slot].mFence = Fence::NO_FENCE;
        }

			//mQueue中移除掉这个front
        mCore-&gt;mQueue.erase(front);

    return NO_ERROR;
}
```

可以看到acquireBuffer主要干的事情如：\
1、mQueue中取出最顶部的BufferItem

2、把顶部的BufferItem的slot获取，且让入参outBuffer指向该BufferItem

3、把mSlots中对应的BufferState变成acquired，且mQueue移除掉front

#### releaseBuffer()–消费者方法

```cpp
status_t BufferQueueConsumer::releaseBuffer(int slot, uint64_t frameNumber,
        const sp&lt;Fence&gt;&amp; releaseFence, EGLDisplay eglDisplay,
        EGLSyncKHR eglFence) {

    sp&lt;IProducerListener&gt; listener;
    { // Autolock scope
        std::lock_guard&lt;std::mutex&gt; lock(mCore-&gt;mMutex);
 //省略部分
        mSlots[slot].mEglDisplay = eglDisplay;
        mSlots[slot].mEglFence = eglFence;
        mSlots[slot].mFence = releaseFence;
        
     //最重要调用了mBufferState的release方法
        mSlots[slot].mBufferState.release();

     //省略部分
     //下面就把slot从 mCore-&gt;mActiveBuffers移除，放入到  mCore-&gt;mFreeBuffers
        if (!mSlots[slot].mBufferState.isShared()) {
            mCore-&gt;mActiveBuffers.erase(slot);
            mCore-&gt;mFreeBuffers.push_back(slot);
        }
 //省略部分

    return NO_ERROR;
}
```

整个release方法就干了2件事\
1、调用了mBufferState的release方法，让状态就变成了FREE

2、从 mCore-&gt;mActiveBuffers移除，放入到 mCore-&gt;mFreeBuffers

相关release触发流程，这里以perffetto来结合代码看\
每次的sf在合成时候都会触发上一帧画面的releaseBuffer操作\
![894b67906bcef20ff825d08965c410bb\_MD5](https://picgo.myjojo.fun:666/i/2024/09/29/66f92f3a56dde.png)

但是刚启动第一帧明显没有看到有releaseBuffer，第二帧才看见\
第一帧，只见到了onTransactionCompleted：\
![1b24821c946d36574a0877a9eba868a5\_MD5](https://picgo.myjojo.fun:666/i/2024/09/29/66f92f3917f5b.png)\
第二帧，见到了onTransactionCompleted同时，也见到了releaseBuffer相关，因为这次release是上一帧的buffer：\
![1ac3796f46f844888ecc16b81807dbae\_MD5](https://picgo.myjojo.fun:666/i/2024/09/29/66f92f392ea0e.png)\
看看前后两次差异到底在哪里？为啥第一次就没有releaseBuffer相关调用，第二次就有了呢？\
看一下onTransactionCompleted方法的差异化执行：

```cpp
void TransactionCompletedListener::onTransactionCompleted(ListenerStats listenerStats) {
    std::unordered_map&lt;CallbackId, CallbackTranslation, CallbackIdHash&gt; callbacksMap;
    std::multimap&lt;int32_t, sp&lt;JankDataListener&gt;&gt; jankListenersMap;
        ATRACE_CALL();
    
        // handle on complete callbacks
        for (auto callbackId : transactionStats.callbackIds) {
            if (callbackId.type != CallbackId::Type::ON_COMPLETE) {
                continue;
            }
            auto&amp; [callbackFunction, callbackSurfaceControls] = callbacksMap[callbackId];
            if (!callbackFunction) {
                ALOGE(&#34;cannot call null callback function, skipping&#34;);
                continue;
            }
            std::vector&lt;SurfaceControlStats&gt; surfaceControlStats;
            for (const auto&amp; surfaceStats : transactionStats.surfaceStats) {
                surfaceControlStats
                        .emplace_back(callbacksMap[callbackId]
                                              .surfaceControls[surfaceStats.surfaceControl],
                                      transactionStats.latchTime, surfaceStats.acquireTimeOrFence,
                                      transactionStats.presentFence,
                                      surfaceStats.previousReleaseFence, surfaceStats.transformHint,
                                      surfaceStats.eventStats,
                                      surfaceStats.currentMaxAcquiredBufferCount);
                if (callbacksMap[callbackId].surfaceControls[surfaceStats.surfaceControl]) {
                    callbacksMap[callbackId]
                            .surfaceControls[surfaceStats.surfaceControl]
                            -&gt;setTransformHint(surfaceStats.transformHint);
                }
                //这里就是是否会执行releaseBuffer的关键，靠传递过来的previousReleaseCallbackId是不是有效
                if (surfaceStats.previousReleaseCallbackId != ReleaseCallbackId::INVALID_ID) {
                
                    ReleaseBufferCallback callback;
                    {
                        std::scoped_lock&lt;std::mutex&gt; lock(mMutex);
                        callback = popReleaseBufferCallbackLocked(
                                surfaceStats.previousReleaseCallbackId);
                    }
                    if (callback) {
                        callback(surfaceStats.previousReleaseCallbackId,
                                 surfaceStats.previousReleaseFence
                                         ? surfaceStats.previousReleaseFence
                                         : Fence::NO_FENCE,
                                 surfaceStats.currentMaxAcquiredBufferCount);
                    }
                }
            }

            callbackFunction(transactionStats.latchTime, transactionStats.presentFence,
                             surfaceControlStats);
        }
//省略
}
```

这里的previousReleaseCallbackId设置其实服务端sf进行的设置，下面会进行详细介绍。

### sf如何触发的release呢？

触发流程如下：\
sf端是在线程中发起的触发跨进程onTransactionCompleted调用\
![849c2feac850bd4f7b02a98054624cc5\_MD5](https://picgo.myjojo.fun:666/i/2024/09/29/66f92f39d75ba.png)这里sf子线程跨进程调用是由主线程sendCallbacks方法触发的

![210fc5be8b42e6219b9cf2a73756fc55\_MD5](https://picgo.myjojo.fun:666/i/2024/09/29/66f92f3a59508.png)

#### sendCallbacks方法

```cpp
void TransactionCallbackInvoker::sendCallbacks(bool onCommitOnly) {
    ATRACE_CALL();
    // For each listener
    //获取全局的mCompletedTransactions变量再进行遍历
    auto completedTransactionsItr = mCompletedTransactions.begin();
    BackgroundExecutor::Callbacks callbacks;
    while (completedTransactionsItr != mCompletedTransactions.end()) {
    //省略部分
        // If the listener has completed transactions
        if (!listenerStats.transactionStats.empty()) {
            // If the listener is still alive
            if (listener-&gt;isBinderAlive()) {
            //塞入callbacks集合
                callbacks.emplace_back([stats = std::move(listenerStats)]() {
                        ATRACE_NAME(&#34;ITransactionCompletedListener onTransactionCompleted&#34;);
                    interface_cast&lt;ITransactionCompletedListener&gt;(stats.listener)
                            -&gt;onTransactionCompleted(stats);
                });
            }
        }
        completedTransactionsItr&#43;&#43;;
    }
    BackgroundExecutor::getInstance().sendCallbacks(std::move(callbacks));
}
```

重点看看mCompletedTransactions这个集合的哪来的，通过查询主要是findOrCreateTransactionStats方法进行加入相关的元素，追踪后发现实际上这里主要是看addCallbackHandles方法会填入对应，最终发现是在releasePendingBuffer方法中塞入：

```cpp
void BufferStateLayer::releasePendingBuffer(nsecs_t dequeueReadyTime) {
  for (const auto&amp; handle : mDrawingState.callbackHandles) {
        handle-&gt;transformHint = mTransformHint;
        handle-&gt;dequeueReadyTime = dequeueReadyTime;
        handle-&gt;currentMaxAcquiredBufferCount =
                mFlinger-&gt;getMaxAcquiredBufferCountForCurrentRefreshRate(mOwnerUid);
    }
//注意这里是把mPreviousReleaseCallbackId变量赋值给  handle-&gt;previousReleaseCallbackId 
    for (auto&amp; handle : mDrawingState.callbackHandles) {
        if (handle-&gt;releasePreviousBuffer &amp;&amp;
            mDrawingState.releaseBufferEndpoint == handle-&gt;listener) {
            handle-&gt;previousReleaseCallbackId = mPreviousReleaseCallbackId;
            break;
        }
    }
 //调用addCallbackHandles
    mFlinger-&gt;getTransactionCallbackInvoker().addCallbackHandles(
            mDrawingState.callbackHandles, jankData);
}
```

这里的mPreviousReleaseCallbackId是哪里来的呢？那么就需要看updateActiveBuffer

#### updateActiveBuffer

updateActiveBuffer方法会对mPreviousReleaseCallbackId这个变量进行赋值，大家注意这里的为啥叫做前一帧的CallbackId，因为下面这个updateActiveBuffer就是赋值是先进行的mPreviousReleaseCallbackId赋值，然后才进行的新buffer的赋值，所以这个mPreviousReleaseCallbackId其实上一个的buffer的Id，不是当前这次的\
![d737d59c99c0b3a2082b4aefdcb0ca4d\_MD5](https://picgo.myjojo.fun:666/i/2024/09/29/66f92f3a78c9f.png)

综上既可以解释releaseBuffer是在下一帧上帧后才由sf调用到app层面让其releaseBuffer。


---

> 作者: [wentao](https://github.com/huang8604)  
> URL: https://huang8604.github.io/posts/blastbufferqueue%E6%BA%90%E7%A0%81%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3-csdn%E5%8D%9A%E5%AE%A2/  

