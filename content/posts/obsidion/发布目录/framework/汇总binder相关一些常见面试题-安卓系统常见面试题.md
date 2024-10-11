---
title: 汇总binder相关一些常见面试题-安卓系统常见面试题-CSDN博客
author: 
created: 2024-10-10
tags:
  - clippings
  - 转载
  - blog
collections: 图形显示
date: 2024-10-10T07:09:20.792Z
lastmod: 2024-10-11T06:23:32.540Z
---
### binder讨论的一些场景面试题

##### 问题1：binder中有多个进程作为客户端通过servicemanager获取service的bpbinder，bpbinder都会有一个handle，那么多个进程获取同一个service对应的handle一样么？

> 是的，多个进程获取同一个服务对应的 `BpBinder` 的 handle 是一样的。
>
> 在 Android 的 Binder IPC 机制中，`ServiceManager` 负责管理系统中的各种服务。当一个客户端进程请求某个服务时，它会通过 `ServiceManager` 获取到服务的 `Binder` 引用（即 `BpBinder`），这个引用在内部包含了一个 handle，该 handle 是一个整数值，唯一标识了这个服务在 Binder 驱动中的位置。
>
> 由于不同进程获取的都是同一个服务的引用，所以它们获取到的 `BpBinder` 中的 handle 是相同的。这个 handle 用于在 Binder 驱动层进行通信时标识目标服务，从而确保客户端进程能够正确地与目标服务进行交互。
>
> 总结来说：
>
> * 不同进程通过 `ServiceManager` 获取同一个服务的 `BpBinder`。
> * 这些 `BpBinder` 对象中的 handle 是一样的，因为它们都指向同一个服务。
> * 这个 handle 是用于在底层 Binder 驱动中唯一标识该服务的。
>
> 希望这个回答能帮到你！如果你有更多问题，请随时提问。

##### 问题2：**请问binder通讯过程中是否可以传递bpbinder对象？**

> 是的，在 Binder 通信过程中可以传递 `BpBinder` 对象。
>
> `BpBinder` 是一个客户端侧的 Binder 引用，它在跨进程通信中可以被当作普通的 Binder 对象传递。具体过程如下：
>
> 1. **创建 `BpBinder`**: 客户端进程通过 `ServiceManager` 获取服务的 `BpBinder` 引用。
>
> 2. **传递 `BpBinder`**: 在进行 Binder 通信时，这个 `BpBinder` 可以作为参数附带在 `Parcel` 中传递给其他进程。
>
> 3. **使用 `BpBinder`**: 接收方进程会收到这个 `BpBinder` 引用，并可以通过它与原服务进行通信。
>
> 这种机制使得 Binder 能够灵活地支持对象引用的传递和跨进程交互，是 Android IPC 的核心特性之一。

##### 问题3：**请问同binder 服务端进程自己进程内部如果进行binder调用需要经过binder驱动吗，需要和不需要请详细描述？**

这个要看情况和调用方式：

###### 一 通过aidl的方式，AIDL 接口生成的时候，有区分是否是本地进程还是远程进程

Stub 的空构造函数。而Stub 本身又是Binder 的子类。调用了Binder 的attachInterface 方法。

```java
 public Stub()
    {
      this.attachInterface(this, DESCRIPTOR);
    }
```

```java
  public void attachInterface(@Nullable IInterface owner, @Nullable String descriptor) {
        mOwner = owner;
        mDescriptor = descriptor;
    }
```

分别保存了当前类的实例对象（This）和当前类接口描述符（DESCRIPTOR），其实就是

private static final java.lang.String DESCRIPTOR = "com.example.aidlapplication.IMyAidlInterface";\
当前类的全路径，这也是我们在使用aidl 的时候服务端和客户端必须要保证相同路径下的原因，因为他被保存下来作为参数用于比对当前类是否是本地类还是远程。

所以在

```java
IMyAidlInterface iMyAidlInterface = IMyAidlInterface.Stub.asInterface(service)
```

```java
public static com.example.aidlapplication.IMyAidlInterface asInterface(android.os.IBinder obj)
    {
      if ((obj==null)) {
        return null;
      }
      android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
      if (((iin!=null)&&(iin instanceof com.example.aidlapplication.IMyAidlInterface))) {
        return ((com.example.aidlapplication.IMyAidlInterface)iin);
      }
      return new com.example.aidlapplication.IMyAidlInterface.Stub.Proxy(obj);
    }
    @Override public android.os.IBinder asBinder()
    {
      return this;
    }
```

参考这篇：[android Binder queryLocalInterface 本地与远程-CSDN博客](https://blog.csdn.net/u013179982/article/details/105920378)

###### 二 通过bind service

这种方式可能还是要通过binder去调用，但在binder驱动内部，可能还是有一定的优化流程。 #TODO

##### 问题4：**Android App进程天生支持binder通讯的原理是什么，刚开始初始化时候自带了几个binder线程？**

zygote 启动的时候就支持了binder通讯，所有后面的app孵化之后，也会自带binder通讯。

zygote 在 init的时候：

###### 1. 启动 ProcessState  **ProcessState::self()**

###### 2. 启动线程池：**proc->startThreadPool();**

```java
virtual void onZygoteInit()
    {
        sp<ProcessState> proc = ProcessState::self();
        ALOGV("App process: starting thread pool.\n");
        proc->startThreadPool();
    }

```

**new ProcessState(kDefaultDriver);**

```java
sp<ProcessState> ProcessState::self()
{
    Mutex::Autolock _l(gProcessMutex);
    if (gProcess != nullptr) {
        return gProcess;
    }
    gProcess = new ProcessState(kDefaultDriver);
    return gProcess;
}

```

**mDriverFD(open\_driver(driver))**   打开驱动\
**mMaxThreads(DEFAULT\_MAX\_BINDER\_THREADS)**  最大线程数\
**mmap(0, BINDER\_VM\_SIZE, PROT\_READ, MAP\_PRIVATE | MAP\_NORESERVE, mDriverFD, 0);**  映射内存 BINDER\_VM\_SIZE 默认 1M- 2PageSize

```java
ProcessState::ProcessState(const char *driver)
    : mDriverName(String8(driver))
    , mDriverFD(open_driver(driver))
    , mVMStart(MAP_FAILED)
    , mThreadCountLock(PTHREAD_MUTEX_INITIALIZER)
    , mThreadCountDecrement(PTHREAD_COND_INITIALIZER)
    , mExecutingThreadsCount(0)
    , mMaxThreads(DEFAULT_MAX_BINDER_THREADS)
    , mStarvationStartTimeMs(0)
    , mManagesContexts(false)
    , mBinderContextCheckFunc(NULL)
    , mBinderContextUserData(NULL)
    , mThreadPoolStarted(false)
    , mThreadPoolSeq(1)
{
    if (mDriverFD >= 0) {
        // mmap the binder, providing a chunk of virtual address space to receive transactions.
        mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
        if (mVMStart == MAP_FAILED) {
            // *sigh*
            ALOGE("Using /dev/binder failed: unable to mmap transaction memory.\n");
            close(mDriverFD);
            mDriverFD = -1;
            mDriverName.clear();
        }
    }

    LOG_ALWAYS_FATAL_IF(mDriverFD < 0, "Binder driver could not be opened.  Terminating.");
}

```

```java
void IPCThreadState::joinThreadPool(bool isMain)
{
    LOG_THREADPOOL("**** THREAD %p (PID %d) IS JOINING THE THREAD POOL\n", (void*)pthread_self(), getpid());

    mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);

    status_t result;
    do {
        processPendingDerefs();
        // now get the next command to be processed, waiting if necessary
        result = getAndExecuteCommand();

        if (result < NO_ERROR && result != TIMED_OUT && result != -ECONNREFUSED && result != -EBADF) {
            ALOGE("getAndExecuteCommand(fd=%d) returned unexpected error %d, aborting",
                  mProcess->mDriverFD, result);
            abort();
        }

        // Let this thread exit the thread pool if it is no longer
        // needed and it is not the main process thread.
        if(result == TIMED_OUT && !isMain) {
            break;
        }
    } while (result != -ECONNREFUSED && result != -EBADF);

    LOG_THREADPOOL("**** THREAD %p (PID %d) IS LEAVING THE THREAD POOL err=%d\n",
        (void*)pthread_self(), getpid(), result);

    mOut.writeInt32(BC_EXIT_LOOPER);
    talkWithDriver(false);
}

```

app刚开始启动Binder时候：\
1、打开binder驱动\
2、binder映射对应的内存\
3、启动binder线程\
4、获取IPCThread对象\
5、通知binder驱动已经进入循环

这部分内容在zygote中已经干完了，所以app在fork出来后，会自带binder功能。

##### 问题5：**请描述一下binder通讯起来后，需要新开binder线程的创建是怎么一个过程？**

\[\[#2. 启动线程池：**proc->startThreadPool();**]]

```java
void ProcessState::startThreadPool()
{
    AutoMutex _l(mLock);
    if (!mThreadPoolStarted) {
        mThreadPoolStarted = true;
        spawnPooledThread(true);
    }
}


```

```java
void ProcessState::spawnPooledThread(bool isMain)
{
    if (mThreadPoolStarted) {
        String8 name = makeBinderThreadName();
        ALOGV("Spawning new pooled thread, name=%s\n", name.string());
        sp<Thread> t = new PoolThread(isMain);
        t->run(name.string());
    }
}


```

```java
class PoolThread : public Thread
{
public:
    explicit PoolThread(bool isMain)
        : mIsMain(isMain)
    {
    }
    
protected:
    virtual bool threadLoop()
    {
        IPCThreadState::self()->joinThreadPool(mIsMain);
        return false;
    }
    
    const bool mIsMain;
};


```

IPCThreadState::self() 构造之后，启动线程池

```java
void IPCThreadState::joinThreadPool(bool isMain)
{
    LOG_THREADPOOL("**** THREAD %p (PID %d) IS JOINING THE THREAD POOL\n", (void*)pthread_self(), getpid());

    mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);

    status_t result;
    do {
        processPendingDerefs();
        // now get the next command to be processed, waiting if necessary
        result = getAndExecuteCommand();

        if (result < NO_ERROR && result != TIMED_OUT && result != -ECONNREFUSED && result != -EBADF) {
            ALOGE("getAndExecuteCommand(fd=%d) returned unexpected error %d, aborting",
                  mProcess->mDriverFD, result);
            abort();
        }

        // Let this thread exit the thread pool if it is no longer
        // needed and it is not the main process thread.
        if(result == TIMED_OUT && !isMain) {
            break;
        }
    } while (result != -ECONNREFUSED && result != -EBADF);

    LOG_THREADPOOL("**** THREAD %p (PID %d) IS LEAVING THE THREAD POOL err=%d\n",
        (void*)pthread_self(), getpid(), result);

    mOut.writeInt32(BC_EXIT_LOOPER);
    talkWithDriver(false);
}


```

参考：\
[Binder之ProcessState和IPCThreadState类分析-CSDN博客](https://blog.csdn.net/jiejingguo/article/details/86518401)

##### 问题6：请问binder的如何使用异步请求，优势是什么，但频繁使用相关风险？

在 Android 的 Binder IPC（进程间通信）机制中，异步请求的使用可以提高应用的响应性和用户体验。下面是异步请求的基本使用方法、优势及其频繁使用的风险。

如何使用异步请求

1. **AIDL 接口定义**：在定义 AIDL 接口时，可以使用 `oneway` 修饰符来标记某个方法为异步调用。例如：
   ```java
   oneway void asyncMethod();
   ```

2. **实现异步方法**：在服务端实现该方法时，可以将其放入一个新的线程中执行，从而不会阻塞主线程：
   ```java
   public void asyncMethod() {
       new Thread(new Runnable() {
           @Override
           public void run() {
               // 执行耗时操作
           }
       }).start();
   }
   ```

3. **客户端调用**：客户端调用该异步方法时，不需要等待结果，调用将立即返回。

4. **回调接口**：为了获取结果，通常会在 AIDL 接口中定义一个回调接口，客户端在调用时传入实现了该接口的对象。服务端在处理完成后，通过该回调将结果返回：
   ```java
   interface Callback {
       void onResult(ResultType result);
   }

   void asyncMethod(Callback callback);
   ```

优势

1. **提高响应性**：异步请求不会阻塞主线程，用户界面能够保持响应，提供更好的用户体验。

2. **资源利用率**：在执行耗时操作时，可以更有效地利用 CPU 资源，不至于造成单一线程的瓶颈。

3. **更好的扩展性**：对于需要频繁通信的场景，异步请求可以有效地处理并发请求，提高了系统的处理能力。

风险

1. **复杂性**：异步编程增加了代码的复杂性，尤其是在管理状态和错误处理时，可能导致难以追踪的 bug。

2. **线程管理**：频繁创建和销毁线程可能导致性能问题或资源消耗过大，应该合理管理线程池的大小。

3. **回调地狱**：使用回调接口时，可能会导致“回调地狱”问题，代码的可读性和可维护性下降。

4. **数据一致性**：在异步操作中，如果多个线程同时修改同一数据，可能会导致数据不一致的问题，需要适当的同步机制。

5. **内存泄漏**：如果持有对 Activity 或 Context 的引用而未能及时清除，可能会导致内存泄漏。使用弱引用或在 Activity 销毁时清除引用是必要的。

总的来说，虽然异步请求在 Android Binder 通信中具有诸多优势，但需要在实现时注意潜在的风险和复杂性。合理设计的异步机制能够提高应用的性能和用户体验。

##### 问题7：**跨进程通讯一般都需要通过ServiceManager进行getService获取bpbinder，但普通app经常使用系统组件Service的bindService进行跨进程通讯也可以获取bpbinder，这个是啥原理？**

客户端 bindService

```java
   @Override
    public boolean bindService(Intent service, ServiceConnection conn,
            int flags) {
        return mBase.bindService(service, conn, flags);
    }

```

```java
    @Override
    public boolean bindService(Intent service, ServiceConnection conn, int flags) {
        warnIfCallingFromSystemProcess();
        return bindServiceCommon(service, conn, flags, null, mMainThread.getHandler(), null,
                getUser());
    }


```

会直接去通过AMS 调用bindIsolatedService

```java
private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags,
            String instanceName, Handler handler, Executor executor, UserHandle user) {
        //省略部分
        try {
            IBinder token = getActivityToken();
            if (token == null && (flags&BIND_AUTO_CREATE) == 0 && mPackageInfo != null
                    && mPackageInfo.getApplicationInfo().targetSdkVersion
                    < android.os.Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
                flags |= BIND_WAIVE_PRIORITY;
            }
            service.prepareToLeaveProcess(this);
            //这里是核心，调用了AMS的bindIsolatedService
            int res = ActivityManager.getService().bindIsolatedService(
                mMainThread.getApplicationThread(), getActivityToken(), service,
                service.resolveTypeIfNeeded(getContentResolver()),
                sd, flags, instanceName, getOpPackageName(), user.getIdentifier());
            if (res < 0) {
                throw new SecurityException(
                        "Not allowed to bind to service " + service);
            }
            return res != 0;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

```

```java
    @UnsupportedAppUsage
    public static IActivityManager getService() {
        return IActivityManagerSingleton.get();
    }

    @UnsupportedAppUsage
    private static final Singleton<IActivityManager> IActivityManagerSingleton =
            new Singleton<IActivityManager>() {
                @Override
                protected IActivityManager create() {
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                    final IActivityManager am = IActivityManager.Stub.asInterface(b);
                    return am;
                }
            };
```

跨进程拿到AMS的binder 去调用bindIsolatedService

```java
 public int bindIsolatedService(IApplicationThread caller, IBinder token, Intent service,
            String resolvedType, IServiceConnection connection, int flags, String instanceName,
            String callingPackage, int userId) throws TransactionTooLargeException {
       //省略部分

        synchronized(this) {
	        //mServices是ActiveServices类型
            return mServices.bindServiceLocked(caller, token, service,
                    resolvedType, connection, flags, instanceName, callingPackage, userId);
        }
    }

int bindServiceLocked(IApplicationThread caller, IBinder token, Intent service,
            String resolvedType, final IServiceConnection connection, int flags,
            String instanceName, String callingPackage, final int userId)
            throws TransactionTooLargeException {
      //省略部分
      //AMS中调用retrieveServiceLocked把对应ServiceRecord创建
        ServiceLookupResult res =
            retrieveServiceLocked(service, instanceName, resolvedType, callingPackage,
                    Binder.getCallingPid(), Binder.getCallingUid(), userId, true,
                    callerFg, isBindExternal, allowInstant);
       //省略部分
            if ((flags&Context.BIND_AUTO_CREATE) != 0) {
                s.lastActivity = SystemClock.uptimeMillis();
                //重要方法，会调用到真正Service对应的服务端进程进行onCreate onBind
                if (bringUpServiceLocked(s, service.getFlags(), callerFg, false,
                        permissionsReviewRequired) != null) {
                    return 0;
                }
            }
              //省略部分
        return 1;
    }

private String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
            boolean whileRestarting, boolean permissionsReviewRequired)
            throws TransactionTooLargeException {
                //省略部分
        final boolean isolated = (r.serviceInfo.flags&ServiceInfo.FLAG_ISOLATED_PROCESS) != 0;
        final String procName = r.processName;
        HostingRecord hostingRecord = new HostingRecord("service", r.instanceName);
        ProcessRecord app;

        if (!isolated) {
            app = mAm.getProcessRecordLocked(procName, r.appInfo.uid, false);
            if (app != null && app.thread != null) {
                try {
                    app.addPackage(r.appInfo.packageName, r.appInfo.longVersionCode, mAm.mProcessStats);
                    //核心方法
                    realStartServiceLocked(r, app, execInFg);
                    return null;
                } 
            }
        } 
        //省略部分

        return null;
    }



 private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app, boolean execInFg) throws RemoteException {
       //省略部分
            mAm.notifyPackageUse(r.serviceInfo.packageName,
                                 PackageManager.NOTIFY_PACKAGE_USE_SERVICE);
            app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
            //调用远端进程的scheduleCreateService
            app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackage(r.serviceInfo.applicationInfo),
                    app.getReportedProcState());
            r.postNotification();
            created = true;
        }
         //省略部分
       当Service Create调用后，会调用要求bind的方法
        requestServiceBindingsLocked(r, execInFg);

       //省略部分
    }


    private final void requestServiceBindingsLocked(ServiceRecord r, boolean execInFg)
            throws TransactionTooLargeException {
        for (int i=r.bindings.size()-1; i>=0; i--) {
            IntentBindRecord ibr = r.bindings.valueAt(i);
            if (!requestServiceBindingLocked(r, ibr, execInFg, false)) {
                break;
            }
        }
    }

private final boolean requestServiceBindingLocked(ServiceRecord r, IntentBindRecord i,
            boolean execInFg, boolean rebind) throws TransactionTooLargeException {
         //省略部分
        if ((!i.requested || rebind) && i.apps.size() > 0) {
            try {
                bumpServiceExecutingLocked(r, execInFg, "bind");
                r.app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
                //调用到Service进程执行bind修改
                r.app.thread.scheduleBindService(r, i.intent.getIntent(), rebind,
                        r.app.getReportedProcState());
                if (!rebind) {
                    i.requested = true;
                }
                i.hasBound = true;
                i.doRebind = false;
            }
            //省略部分
        }
        return true;
    }


让远端执行onBind相关
void publishServiceLocked(ServiceRecord r, Intent intent, IBinder service) {
   //省略大部分
                    //会调用到最开始bindService那个客户端进程，而且带有了IBinder service参数
                                c.conn.connected(r.name, service, false);
                           //省略大部分
    }



```

远端服务如果没有启动，前面还有一个oncreate的过程。是在 scheduleCreateService 的时候调用。调用后才会去bind 相关服务

create 过程

```java
 public final void scheduleCreateService(IBinder token,
                ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
            updateProcessState(processState, false);
            CreateServiceData s = new CreateServiceData();
            s.token = token;
            s.info = info;
            s.compatInfo = compatInfo;

            sendMessage(H.CREATE_SERVICE, s);
        }
    
   case CREATE_SERVICE:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, ("serviceCreate: " + String.valueOf(msg.obj)));
                    handleCreateService((CreateServiceData)msg.obj);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;



private void handleCreateService(CreateServiceData data) {
        //省略部分
            ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
            context.setOuterContext(service);

            Application app = packageInfo.makeApplication(false, mInstrumentation);
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManager.getService());
            service.onCreate();
            mServices.put(data.token, service);
            try {
                ActivityManager.getService().serviceDoneExecuting(
                        data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        } 
          //省略部分
    }


```

调用 service.onCreate();

```
      case BIND_SERVICE:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "serviceBind");
                    handleBindService((BindServiceData)msg.obj);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;


 private void handleBindService(BindServiceData data) {
       //省略
                    if (!data.rebind) {
                        IBinder binder = s.onBind(data.intent);
                        ActivityManager.getService().publishService(
                                data.token, data.intent, binder);
             //省略
    }



```
