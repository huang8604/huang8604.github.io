---
title: Android监听UEvent之UEventObserver分析-CSDN博客
author:
  - "[[成就一亿技术人!]]"
  - "[[hope_wisdom 发出的红包]]"
created: 2025-09-18
tags:
  - clippings
  - 转载
  - blog
collections: 图形显示
source: https://blog.csdn.net/dongxianfei/article/details/128290460
date: 2025-09-18T01:35:23.307Z
lastmod: 2025-09-18T01:35:26.881Z
---
本文深入解析了Android系统中UEventObserver的工作原理，包括Java层如何监听内核事件、Kernel层如何处理事件，以及通过示例展示如何在实际应用中使用UEventObserver。

**（1）背景概述**

众所周知，在安卓系统中有状态栏，在插入外设的时候，会在顶部状态栏显示小图标。\
比如，camera设备，耳机设备，U盘，以及电池等等。这些都需要在状态栏动态显示。

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/785100576510e29bfb760670bf71a2c6.png#pic_center)\
从上面这张图片可以看出这些设备都有自己的服务一直在跑，并且都是继承了UEventObserver.java这个类去获取 kernel 的Event事件。下面将着重分析UEventObserver是如何去监听kernel的Event事件。

**（2）源码分析**

（A）Java层源码分析

```java
//frameworks/base/core/java/android/os/UEventObserver.java

/*
UEventObserver是一个从内核接收UEvents的抽象类。

子类UEventObserver实现onUEvent（UEvent事件），调用startObserving()与匹配字符串匹配，
然后UEvent线程将调用onUEvent()方法，调用stopObserving()停止接收UE事件。

每个进程只有一个UEvent线程，即使该进程具有多个UEventObserver子类实例。
UEvent线程在以下情况下启动：
在该过程中首次调用startObserving()。一旦已启动UEvent线程不会停止（尽管它可以停止通知UEventObserver通过stopObserving()）

//hide
*/

public abstract class UEventObserver {

    private static UEventThread sThread;

    private static native void nativeSetup();
    private static native String nativeWaitForNextEvent();
    private static native void nativeAddMatch(String match);
    private static native void nativeRemoveMatch(String match);

    private static UEventThread getThread() {
        synchronized (UEventObserver.class) {
            if (sThread == null) {
                sThread = new UEventThread();
                sThread.start();
            }
            return sThread;
        }
    }

    private static UEventThread peekThread() {
        synchronized (UEventObserver.class) {
            return sThread;
        }
    }

    //注释监听Observer
    public final void startObserving(String match) {
        if (match == null || match.isEmpty()) {
            throw new IllegalArgumentException("match substring must be non-empty");
        }

        final UEventThread t = getThread();
        t.addObserver(match, this);
    }

    //停止监听Observer
    public final void stopObserving() {
        final UEventThread t = peekThread();
        if (t != null) {
            t.removeObserver(this);
        }
    }

    public abstract void onUEvent(UEvent event);
java1234567891011121314151617181920212223242526272829303132333435363738394041424344454647484950515253545556575859
```

接下来看一下其使用的UEventThread。

```java
private static final class UEventThread extends Thread {
        /** Many to many mapping of string match to observer.
         *  Multimap would be better, but not available in android, so use
         *  an ArrayList where even elements are the String match and odd
         *  elements the corresponding UEventObserver observer */
        private final ArrayList<Object> mKeysAndObservers = new ArrayList<Object>();

        private final ArrayList<UEventObserver> mTempObserversToSignal =
                new ArrayList<UEventObserver>();

        public UEventThread() {
            super("UEventObserver");
        }

        @Override
        public void run() {
            nativeSetup();    //jni调用nativeSetup

            while (true) {
                String message = nativeWaitForNextEvent();    //jni调用nativeWaitForNextEvent
                if (message != null) {
                    if (DEBUG) {
                        Log.d(TAG, message);
                    }
                    sendEvent(message);
                }
            }
        }

        private void sendEvent(String message) {
            synchronized (mKeysAndObservers) {
                final int N = mKeysAndObservers.size();
                for (int i = 0; i < N; i += 2) {
                    final String key = (String)mKeysAndObservers.get(i);
                    if (message.contains(key)) {
                        final UEventObserver observer =
                                (UEventObserver)mKeysAndObservers.get(i + 1);
                        mTempObserversToSignal.add(observer);
                    }
                }
            }

            if (!mTempObserversToSignal.isEmpty()) {
                final UEvent event = new UEvent(message);
                final int N = mTempObserversToSignal.size();
                for (int i = 0; i < N; i++) {
                    final UEventObserver observer = mTempObserversToSignal.get(i);
                    observer.onUEvent(event);
                }
                mTempObserversToSignal.clear();
            }
        }

        public void addObserver(String match, UEventObserver observer) {
            synchronized (mKeysAndObservers) {
                mKeysAndObservers.add(match);
                mKeysAndObservers.add(observer);
                nativeAddMatch(match);
            }
        }

        /** Removes every key/value pair where value=observer from mObservers */
        public void removeObserver(UEventObserver observer) {
            synchronized (mKeysAndObservers) {
                for (int i = 0; i < mKeysAndObservers.size(); ) {
                    if (mKeysAndObservers.get(i + 1) == observer) {
                        mKeysAndObservers.remove(i + 1);
                        final String match = (String)mKeysAndObservers.remove(i);
                        nativeRemoveMatch(match);
                    } else {
                        i += 2;
                    }
                }
            }
        }
    }
java12345678910111213141516171819202122232425262728293031323334353637383940414243444546474849505152535455565758596061626364656667686970717273747576
```

当第一次启动这个线程的时候，会调用nativeSetup()方法做初始化，可以看出这个函数是native层来实现的。初始化完之后，进入一个while的死循环，不停的调用native层的nativeWaitForNextEvent()函数来获取Event事件，然后将Event事件转换成message，再通过sendEvent()将message事件传递给外设对应的Observer。

（B）Kernel层源码分析

```cpp
//frameworks/base/core/jni/android_os_UEventObserver.cpp

static void nativeSetup(JNIEnv *env, jclass clazz) {
    if (!uevent_init()) {    //kernel当中的uevent_init
        jniThrowException(env, "java/lang/RuntimeException",
                "Unable to open socket for UEventObserver");
    }
}

static jstring nativeWaitForNextEvent(JNIEnv *env, jclass clazz) {
    char buffer[1024];

    for (;;) {
        int length = uevent_next_event(buffer, sizeof(buffer) - 1);    //kernel当中的uevent_next_event
        if (length <= 0) {
            return NULL;
        }
        buffer[length] = '\0';

        ALOGV("Received uevent message: %s", buffer);

        if (isMatch(buffer, length)) {
            // Assume the message is ASCII.
            jchar message[length];
            for (int i = 0; i < length; i++) {
                message[i] = buffer[i];
            }
            return env->NewString(message, length);
        }
    }
}
cpp12345678910111213141516171819202122232425262728293031
```

进而调用内核Event相关函数uevent\_init()和uevent\_next\_event()。

```c
//hardware/libhardware_legacy/uevent.c

/* Returns 0 on failure, 1 on success */
int uevent_init()
{
    struct sockaddr_nl addr;
    int sz = 64*1024;
    int s;

    memset(&addr, 0, sizeof(addr));
    addr.nl_family = AF_NETLINK;
    addr.nl_pid = getpid();
    addr.nl_groups = 0xffffffff;

    s = socket(PF_NETLINK, SOCK_DGRAM, NETLINK_KOBJECT_UEVENT);
    if(s < 0)
        return 0;

    setsockopt(s, SOL_SOCKET, SO_RCVBUFFORCE, &sz, sizeof(sz));

    if(bind(s, (struct sockaddr *) &addr, sizeof(addr)) < 0) {
        close(s);
        return 0;
    }

    fd = s;
    return (fd > 0);
}

int uevent_next_event(char* buffer, int buffer_length)
{
    while (1) {
        struct pollfd fds;
        int nr;
    
        fds.fd = fd;
        fds.events = POLLIN;
        fds.revents = 0;
        nr = poll(&fds, 1, -1);
     
        if(nr > 0 && (fds.revents & POLLIN)) {
            int count = recv(fd, buffer, buffer_length, 0);
            if (count > 0) {
                struct uevent_handler *h;
                pthread_mutex_lock(&uevent_handler_list_lock);
                LIST_FOREACH(h, &uevent_handler_list, list)
                    h->handler(h->handler_data, buffer, buffer_length);
                pthread_mutex_unlock(&uevent_handler_list_lock);

                return count;
            } 
        }
    }
    
    // won't get here
    return 0;
}
c12345678910111213141516171819202122232425262728293031323334353637383940414243444546474849505152535455565758
```

uevent\_init：初始化 socket ，用来接收来自内核的event事件。\
uevent\_next\_event：不停的调用poll来等待socket的数据，并将数据存放在buffer中，然后返回。

最后附上流程 [时序图](https://so.csdn.net/so/search?q=%E6%97%B6%E5%BA%8F%E5%9B%BE\&spm=1001.2101.3001.7020) ：\
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/955b66d04dec689e3e60df453f61adef.jpeg#pic_center)\
**（3）简单测试用例**

（A）UsbDeviceManager

```java
//frameworks/base/services/usb/java/com/android/server/usb/UsbDeviceManager.java

private static final String USB_STATE_MATCH = "DEVPATH=/devices/virtual/android_usb/android0";
private static final String ACCESSORY_START_MATCH = "DEVPATH=/devices/virtual/misc/usb_accessory";

private final UEventObserver mUEventObserver;

// Watch for USB configuration changes
mUEventObserver = new UsbUEventObserver();
mUEventObserver.startObserving(USB_STATE_MATCH);
mUEventObserver.startObserving(ACCESSORY_START_MATCH);

/*
     * Listens for uevent messages from the kernel to monitor the USB state
     */
    private final class UsbUEventObserver extends UEventObserver {
        @Override
        public void onUEvent(UEventObserver.UEvent event) {
            Slog.v(TAG, "USB UEVENT: " + event.toString());
            String state = event.get("USB_STATE");
            String accessory = event.get("ACCESSORY");
            
            if (state != null) {
                mHandler.updateState(state);
            } else if ("GETPROTOCOL".equals(accessory)) {
                //...
            } else if ("SENDSTRING".equals(accessory)) {
                //...
            } else if ("START".equals(accessory)) {
                //...
            }
        }
    }
java123456789101112131415161718192021222324252627282930313233
```

（B）监听摄像头的打开和关闭

在上层服务中监听摄像头的打开和关闭，并作相应处理。需要内核摄像头驱动中也要发出event事件才行，所以改动分为内核和上层两部分。

```java
//kernel drv

static atomic_t g_CamHWOpend;     //camera是否打开成功的标志，其它地方赋值
struct device* sensor_device = NULL;
static void set_camera_status()
{
  char *envp[2];
  int ret = atomic_read(&g_CamHWOpend)? 1 : 0;
    if(ret)
        envp[0] = "STATUS=OPEN";
    else
        envp[0] = "STATUS=CLOSE";
    envp[1] = NULL;
    kobject_uevent_env(&sensor_device->kobj, KOBJ_CHANGE, envp);    //将envp通过kobject上报
    return;
}

//上层监听

    m_CameraStatusObserver.startObserving("DEVPATH=/devices/virtual/sensordrv/kd_camera_hw");

    private UEventObserver m_CameraStatusObserver = new UEventObserver(){
        public void onUEvent(UEvent event){
            String status = event.get("STATUS");    //没有取特定长度字符串，直接取=前面的子串
            if( "OPEN".equals(status)){
                Log.i(TAG,"camera app open");            
            }
            else if ("CLOSE".equals(status)){
                Log.i(TAG,"camera app close");
            }
        }
    };
java1234567891011121314151617181920212223242526272829303132
```

打赏作者

¥1 ¥2 ¥4 ¥6 ¥10 ¥20

扫码支付： ¥1

获取中

扫码支付

您的余额不足，请更换扫码支付或 [充值](https://i.csdn.net/#/wallet/balance/recharge?utm_source=RewardVip)

打赏作者

实付 元

[使用余额支付](https://blog.csdn.net/dongxianfei/article/details/)

点击重新获取

扫码支付

钱包余额 0

抵扣说明：

1.余额是钱包充值的虚拟货币，按照1:1的比例进行支付金额的抵扣。\
2.余额无法直接购买下载，可以购买VIP、付费专栏及课程。

[余额充值](https://i.csdn.net/#/wallet/balance/recharge)

举报

[![](https://csdnimg.cn/release/blogv2/dist/pc/img/toolbar/Group-dark.png) 点击体验\
DeepSeekR1满血版](https://ai.csdn.net/?utm_source=cknow_pc_blogdetail\&spm=1001.2101.3001.10583) 隐藏侧栏 ![程序员都在用的中文IT技术交流社区](https://g.csdnimg.cn/side-toolbar/3.6/images/qr_app.png)

程序员都在用的中文IT技术交流社区

![专业的中文 IT 技术社区，与千万技术人共成长](https://g.csdnimg.cn/side-toolbar/3.6/images/qr_wechat.png)

专业的中文 IT 技术社区，与千万技术人共成长

![关注【CSDN】视频号，行业资讯、技术分享精彩不断，直播好礼送不停！](https://g.csdnimg.cn/side-toolbar/3.6/images/qr_video.png)

关注【CSDN】视频号，行业资讯、技术分享精彩不断，直播好礼送不停！

返回顶部

![](https://i-blog.csdnimg.cn/blog_migrate/785100576510e29bfb760670bf71a2c6.png#pic_center) ![](https://i-blog.csdnimg.cn/blog_migrate/955b66d04dec689e3e60df453f61adef.jpeg#pic_center)
