---
tags:
  - clippings
  - 转载
  - WMS
  - blog
collections:
  - WMS
  - Framework
title: Vsync之app层面深入分析  vsynceventdata-CSDN博客
date: 2024-09-25T07:24:04.939Z
lastmod: 2024-09-25T07:54:20.000Z
---
### 背景

前面文章和视频课程都是直接从SurfaceFlinger层面开始讲解Vsync部分的，当然vsync的主要核心逻辑也确实在SurfaceFlinger，但是一般vsync都是由app层面发起请求的，这一部分也还是有必要带大家了解清楚

### java层面的分析和堆栈：

在Activity进行Resume时候，会addView,这个时候会对ViewRootImpl进行够着，构建出一个Choreographer，在构造时候会构造方法里面又会对应的FrameDisplayEventReceiver，FrameDisplayEventReceiver本身继承DisplayEventReceiver\
这里的DisplayEventReceiver就是核心部分，它负责和sf进行双向通讯，不过这里双向不是一种ipc通讯方式，涉及到两个方式\
![e868c7a9e8b03a790bcc24748e61a83a\_MD5](https://picgo.myjojo.fun:666/i/2024/09/25/66f3c152969c1.png)

app主动发起请求一般都是直接使用binder调用，比如常见的如下几个接口：

```cpp
interface IDisplayEventConnection {
    /*
     * stealReceiveChannel() returns a BitTube to receive events from. Only the receive file
     * descriptor of outChannel will be initialized, and this effectively "steals" the receive
     * channel from the remote end (such that the remote end can only use its send channel).
     */
    void stealReceiveChannel(out BitTube outChannel);

    /*
     * setVsyncRate() sets the vsync event delivery rate. A value of 1 returns every vsync event.
     * A value of 2 returns every other event, etc. A value of 0 returns no event unless
     * requestNextVsync() has been called.
     */
    void setVsyncRate(in int count);

    /*
     * requestNextVsync() schedules the next vsync event. It has no effect if the vsync rate is > 0.
     */
    oneway void requestNextVsync(); // Asynchronous

    /*
     * getLatestVsyncEventData() gets the latest vsync event data.
     */
    ParcelableVsyncEventData getLatestVsyncEventData();
}
```

SurfaceFlinger进程也需要与app进行通讯，比如把vsync来临这种通知调用：\
![62252d47f74aa498190dffbad848ebbc\_MD5](https://picgo.myjojo.fun:666/i/2024/09/25/66f3c1533f76d.png)

***这里为啥sf要是有socket呢？这里主要还是为了性能考虑，socket相比延时阻塞情况比binder好，vsync通知这种属于实时性较强的操作。***

下面接着看看app层面FrameDisplayEventReceiver构造接下来干了啥

```cpp
  public FrameDisplayEventReceiver(Looper looper, int vsyncSource) {
            super(looper, vsyncSource, 0);//直接调用了父类的构造
        }

   /**
     * Creates a display event receiver.
     *
     * @param looper The looper to use when invoking callbacks.
     * @param vsyncSource The source of the vsync tick. Must be on of the VSYNC_SOURCE_* values.
     * @param eventRegistration Which events to dispatch. Must be a bitfield consist of the
     * EVENT_REGISTRATION_*_FLAG values.
     */
    public DisplayEventReceiver(Looper looper, int vsyncSource, int eventRegistration) {
        mMessageQueue = looper.getQueue();
        mReceiverPtr = nativeInit(new WeakReference<DisplayEventReceiver>(this), mMessageQueue,
                vsyncSource, eventRegistration);
    }
```

这里调用了nativeInit，接下来代码就到了native层面了\
具体堆栈如下：\
![b4742751a5faa8e30ce847a35042b753\_MD5](https://picgo.myjojo.fun:666/i/2024/09/25/66f3c15398bf1.png)

### native层面的分析和堆栈：

接上面的nativeInit\
frameworks/base/core/[jni](https://so.csdn.net/so/search?q=jni\&spm=1001.2101.3001.7020)/android\_view\_DisplayEventReceiver.cpp

```cpp
static jlong nativeInit(JNIEnv* env, jclass clazz, jobject receiverWeak, jobject messageQueueObj,
                        jint vsyncSource, jint eventRegistration) {
    //省略部分
    sp<NativeDisplayEventReceiver> receiver =
            new NativeDisplayEventReceiver(env, receiverWeak, messageQueue, vsyncSource,
                                           eventRegistration);
    status_t status = receiver->initialize();
  //省略部分
    return reinterpret_cast<jlong>(receiver.get());
}
```

构造 NativeDisplayEventReceiver类：

```cpp
NativeDisplayEventReceiver::NativeDisplayEventReceiver(JNIEnv* env, jobject receiverWeak,
                                                       const sp<MessageQueue>& messageQueue,
                                                       jint vsyncSource, jint eventRegistration)
      : DisplayEventDispatcher(messageQueue->getLooper(),
                               static_cast<ISurfaceComposer::VsyncSource>(vsyncSource),
                               static_cast<ISurfaceComposer::EventRegistration>(eventRegistration)),
        mReceiverWeakGlobal(env->NewGlobalRef(receiverWeak)),
        mMessageQueue(messageQueue) {
}
```

注意这里的NativeDisplayEventReceiver继承DisplayEventDispatcher

```cpp
DisplayEventDispatcher::DisplayEventDispatcher(
        const sp<Looper>& looper, ISurfaceComposer::VsyncSource vsyncSource,
        ISurfaceComposer::EventRegistrationFlags eventRegistration)
      : mLooper(looper), mReceiver(vsyncSource, eventRegistration), mWaitingForVsync(false),
        mLastVsyncCount(0), mLastScheduleVsyncTime(0) {
    ALOGV("dispatcher %p ~ Initializing display event dispatcher.", this);
}
```

注意这里的DisplayEventDispatcher构造也会mReceiver也构造， mReceiver是DisplayEventReceiver 类型，构造方法如下：

```cpp
DisplayEventReceiver::DisplayEventReceiver(
        ISurfaceComposer::VsyncSource vsyncSource,
        ISurfaceComposer::EventRegistrationFlags eventRegistration) {
    sp<ISurfaceComposer> sf(ComposerService::getComposerService());
    if (sf != nullptr) {
       //会与sf进行跨进程通讯，让创建对应connection
        mEventConnection = sf->createDisplayEventConnection(vsyncSource, eventRegistration);
        if (mEventConnection != nullptr) {
            mDataChannel = std::make_unique<gui::BitTube>();
            //connection创建成功，这里就会调用mEventConnection的stealReceiveChannel获取通讯的socket的fd
            const auto status = mEventConnection->stealReceiveChannel(mDataChannel.get());
        }
    }
}
```

这里可以看出来和sf通讯开始用mEventConnection啦

initialize主要干的事如下：

```cpp
status_t DisplayEventDispatcher::initialize() {
    if (mLooper != nullptr) {
        int rc = mLooper->addFd(mReceiver.getFd(), 0, Looper::EVENT_INPUT, this, NULL);
        if (rc < 0) {
            return UNKNOWN_ERROR;
        }
    }

    return OK;
}
```

把对应的服务端sf传递socket的fd进行输入事件监听，这样sf有往socket写入数据，app既可以接受到了

### SurfaceFlinger端的相关方法分析

createDisplayEventConnection方法,这个方法app发起跨进程调用后会到服务端BnSurfaceComposer，这个SurfaceFlinger是继承这个BnSurfaceComposer的

```cpp
class BnSurfaceComposer: public BnInterface<ISurfaceComposer> 

class SurfaceFlinger : public BnSurfaceComposer,
                       public PriorityDumper,
                       private IBinder::DeathRecipient,
                       private HWC2::ComposerCallback,
                       private ICompositor,
                       private scheduler::ISchedulerCallback {
```

所以这里直接看SurfaceFlinger的createDisplayEventConnection

```cpp
sp<IDisplayEventConnection> SurfaceFlinger::createDisplayEventConnection(
        ISurfaceComposer::VsyncSource vsyncSource,
        ISurfaceComposer::EventRegistrationFlags eventRegistration) {
        //这里参数有一个vsyncSource就是app还是appSf两个，以前没有appSf，这个参数来选着哪个EventThread
    const auto& handle =
            vsyncSource == eVsyncSourceSurfaceFlinger ? mSfConnectionHandle : mAppConnectionHandle;

    return mScheduler->createDisplayEventConnection(handle, eventRegistration);
}
```

再看看Scheduler::createDisplayEventConnection

```cpp
sp<IDisplayEventConnection> Scheduler::createDisplayEventConnection(
        ConnectionHandle handle, ISurfaceComposer::EventRegistrationFlags eventRegistration) {
    return createConnectionInternal(mConnections[handle].thread.get(), eventRegistration);
}

sp<EventThreadConnection> Scheduler::createConnectionInternal(
        EventThread* eventThread, ISurfaceComposer::EventRegistrationFlags eventRegistration) {
    return eventThread->createEventConnection([&] { resync(); }, eventRegistration);
}

sp<EventThreadConnection> EventThread::createEventConnection(
        ResyncCallback resyncCallback,
        ISurfaceComposer::EventRegistrationFlags eventRegistration) const {
        //创建对应的EventThreadConnection对象
    return new EventThreadConnection(const_cast<EventThread*>(this),
                                     IPCThreadState::self()->getCallingUid(),
                                     std::move(resyncCallback), eventRegistration);
}
```

可以看到最后其实是构造了一个EventThreadConnection对象

```cpp
class EventThreadConnection : public gui::BnDisplayEventConnection {
public:
    EventThreadConnection(EventThread*, uid_t callingUid, ResyncCallback,
                          ISurfaceComposer::EventRegistrationFlags eventRegistration = {});

    virtual status_t postEvent(const DisplayEventReceiver::Event& event);

    binder::Status stealReceiveChannel(gui::BitTube* outChannel) override;
    binder::Status setVsyncRate(int rate) override;
    binder::Status requestNextVsync() override; // asynchronous
    binder::Status getLatestVsyncEventData(ParcelableVsyncEventData* outVsyncEventData) override;

};
```

上面就是它几个主要方法，就是app和sf通过这个EventThreadConnection进行通讯的接口方法

同时注意一下EventThreadConnection的onFirstRef方法

```cpp
void EventThreadConnection::onFirstRef() {
    mEventThread->registerDisplayEventConnection(this);
}
```

这里调用了EventThread的registerDisplayEventConnection方法

```cpp
status_t EventThread::registerDisplayEventConnection(const sp<EventThreadConnection>& connection) {
    std::lock_guard<std::mutex> lock(mMutex);
    mDisplayEventConnections.push_back(connection);
    mCondition.notify_all();
    return NO_ERROR;
}
```

这里主要就是把每个app的connection都放到了mDisplayEventConnections这个变量中

接下来看看对应的stealReceiveChannel的接口方法

```cpp
binder::Status EventThreadConnection::stealReceiveChannel(gui::BitTube* outChannel) {
      outChannel->setReceiveFd(mChannel.moveReceiveFd());
    outChannel->setSendFd(base::unique_fd(dup(mChannel.getSendFd())));
    return binder::Status::ok();
}
```

可以看出这里主要是把EventThreadConnection的mChannel的fd搞到outChannel的fd具体mChannel其实是个BitTube类型

```cpp
BitTube::BitTube(size_t bufsize)
    : mSendFd(-1), mReceiveFd(-1)
{
    init(bufsize, bufsize);//调用是init方法
}
void BitTube::init(size_t rcvbuf, size_t sndbuf) {
    int sockets[2];
    //其实本质是有一对socketpair
    if (socketpair(AF_UNIX, SOCK_SEQPACKET, 0, sockets) == 0) {
        size_t size = DEFAULT_SOCKET_BUFFER_SIZE;
        setsockopt(sockets[0], SOL_SOCKET, SO_RCVBUF, &rcvbuf, sizeof(rcvbuf));
        setsockopt(sockets[1], SOL_SOCKET, SO_SNDBUF, &sndbuf, sizeof(sndbuf));
        // sine we don't use the "return channel", we keep it small...
        setsockopt(sockets[0], SOL_SOCKET, SO_SNDBUF, &size, sizeof(size));
        setsockopt(sockets[1], SOL_SOCKET, SO_RCVBUF, &size, sizeof(size));
        fcntl(sockets[0], F_SETFL, O_NONBLOCK);
        fcntl(sockets[1], F_SETFL, O_NONBLOCK);
        mReceiveFd = sockets[0];
        mSendFd = sockets[1];
    } else {
        mReceiveFd = -errno;
        ALOGE("BitTube: pipe creation failed (%s)", strerror(-mReceiveFd));
    }
}
```

sf总结图如下：\
![537cecdafe17aef5ed3148bb737b240b\_MD5](https://picgo.myjojo.fun:666/i/2024/09/25/66f3c1531616b.png)
