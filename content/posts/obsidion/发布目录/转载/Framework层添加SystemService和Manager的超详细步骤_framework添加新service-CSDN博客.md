---
title: Framework层添加SystemService和Manager的超详细步骤_framework添加新service-CSDN博客
source: 
tags:
  - clippings
  - 转载
  - blog
collections:
  - PMS
  - Framework
date: 2024-11-06T06:57:52.184Z
lastmod: 2024-11-05T01:33:58.411Z
---
本文适用于Android 12中增加系统服务。

**目录**

[1.总体步骤](https://blog.csdn.net/lvyaer_1122/article/details/130139795?spm=1001.2014.3001.5502/#t0)

[2、详细步骤](https://blog.csdn.net/lvyaer_1122/article/details/130139795?spm=1001.2014.3001.5502/#t1)

[2.1 创建AIDL文件](https://blog.csdn.net/lvyaer_1122/article/details/130139795?spm=1001.2014.3001.5502/#t2)

[2.2 报错修改](https://blog.csdn.net/lvyaer_1122/article/details/130139795?spm=1001.2014.3001.5502/#t3)

[2.2.1 AIDL文件自动生成的Java文件报错](https://blog.csdn.net/lvyaer_1122/article/details/130139795?spm=1001.2014.3001.5502/#t4)

[2.2.2 AIDL文件Observer/Callback/Listener命名报错修改](https://blog.csdn.net/lvyaer_1122/article/details/130139795?spm=1001.2014.3001.5502/#t5)

[2.2.3 注册方法报错](https://blog.csdn.net/lvyaer_1122/article/details/130139795?spm=1001.2014.3001.5502/#t6)

[2.2.4 避免使用枚举类enum](https://blog.csdn.net/lvyaer_1122/article/details/130139795?spm=1001.2014.3001.5502/#t7)

[2.2.5 Callback如何让APP访问到](https://blog.csdn.net/lvyaer_1122/article/details/130139795?spm=1001.2014.3001.5502/#t8)

[3. Context中定义service name](https://blog.csdn.net/lvyaer_1122/article/details/130139795?spm=1001.2014.3001.5502/#t9)

[4. 编写SystemService](https://blog.csdn.net/lvyaer_1122/article/details/130139795?spm=1001.2014.3001.5502/#t10)

[5. 在SystemServer类中注册新增的系统服务](https://blog.csdn.net/lvyaer_1122/article/details/130139795?spm=1001.2014.3001.5502/#t11)

[6. 编写系统Manager类](https://blog.csdn.net/lvyaer_1122/article/details/130139795?spm=1001.2014.3001.5502/#t12)

[7. 注册系统Manager类](https://blog.csdn.net/lvyaer_1122/article/details/130139795?spm=1001.2014.3001.5502/#t13)

[8. 应用调用](https://blog.csdn.net/lvyaer_1122/article/details/130139795?spm=1001.2014.3001.5502/#t14)

***

## 1.总体步骤

#### ![](https://i-blog.csdnimg.cn/blog_migrate/d5b84c5a1f1e84ccde72322405e3d695.png)

***

## 2、详细步骤

### **2.1 创建AIDL文件**

\*\*在framework/base/core/java/android/app下创建AIDL文件，\*\*如果业务比较复杂，可以创建模块文件夹。代码示例如下：

**framework/base/core/java/android/app/devicemanager/IDeviceManager.aidl**

```java
package android.app.devicemanager;import android.app.devicemanager.DeviceEntity;import android.app.devicemanager.IDeviceObserver;interface IDeviceManager {int bindDivice(int deviceId);    DeviceEntity getDeviceEntity(int deviceId);void registerDeviceObserver(in IDeviceObserver observer);void unregisterDeviceObserver(in IDeviceObserver observer);}
```

**framework/base/core/java/android/app/devicemanager/IDiviceObserver.aidl**

```java
package android.app.devicemanager;import android.app.devicemanager.DeviceEntity;oneway interface IDiviceObserver{void onBindChanged(int state, int deviceId);void onStateChanged(int state, int deviceId);}
```

**framework/base/core/java/android/app/devicemanager/DeviceEntity.aidl**

```java
package android.app.devicemanager;parcelable DeviceEntity;
```

### 2.2 报错修改

上述写法会报非常多的错，请参考Android 12  API规范：请参考 https://www.cnblogs.com/wanghongzhu/p/14729469.html 。

针对上述代码所犯的错误，也就是大家在Android 12版本中使用make -j32 framework-minus-apex编译framework层的报错修改。

#### 2.2.1 [AIDL](https://so.csdn.net/so/search?q=AIDL\&spm=1001.2101.3001.7020)文件自动生成的Java文件报错

这是因为AIDL自动生成的Java文件不满足Android 12 framework API的规范：framework层不能直接暴露原生AIDL文件。

修改的方式是在aidl文件上添加@hide，如下所示，这样就可以解决所有AIDL自动生成的文件。（这是扒遍国内全网都没在找到，在Google中才找到的根本解决办法）

```java
package android.app.devicemanager;import android.app.devicemanager.DeviceEntity;import android.app.devicemanager.IDeviceObserver;interface IDeviceManager {int bindDivice(int deviceId);    DeviceEntity getDeviceEntity(int deviceId);void registerDeviceObserver(in IDeviceObserver observer);void unregisterDeviceObserver(in IDeviceObserver observer);}
```

out/srcjars/android/app/devicemanager/IDiviceObserver.java:60: error: Methods calling system APIs should rethrow \`RemoteException\` as \`RuntimeException\` (but do not list it in the throws clause) \[RethrowRemoteException]

out/srcjars/android/app/devicemanager/IDiviceObserver.java:70: error: Missing nullability on method \`asBinder\` return \[MissingNullability]\
out/srcjars/android/app/devicemanager/IDiviceObserver.java:75: error: Raw AIDL interfaces must not be exposed: Stub extends Binder \[RawAidl]

#### 2.2.2 AIDL文件[Observer](https://so.csdn.net/so/search?q=Observer\&spm=1001.2101.3001.7020)/Callback/Listener命名报错修改

Observer命名报错，Android Lint 工具对callback和listener有严格的校验，不建议使用Observer作为回调，要用callback或者listener。

* 当只有一个回调方法且永远不会有其他回调方法时使用Listener，且注册监听和解注册监听的方法必须是add/remove开头，否则Android Lint编译不过。
* 当有多个回调方法时，或者有关联的常量时，应该使用Callback。Callback类可以是一个interface或者abstract class。添加callback和去掉callback应该使用register和unregister开头的方法。
* callback中的方法应该以on-开头。

请中招的小伙伴自行修改。

#### 2.2.3 注册方法报错

Registration methods should have overload that accepts delivery Executor: \`registerDeviceCallback\` \[ExecutorRegistration]

是不是一脸懵，看不明白啥意思，以前很早的framework层的manager，没啥参考价值，推荐参考新的manager和API规范，因为现有的注册回调的方法必须是两个参数，其中一个必须为Executor，如下所示。

```java
public void registerFooCallback( @NonNull @CallbackExecutor Executor executor,@NonNull FooCallback callback)public void unregisterFooCallback(@NonNull FooCallback callback) {}
```

####  2.2.4 避免使用[枚举类](https://so.csdn.net/so/search?q=%E6%9E%9A%E4%B8%BE%E7%B1%BB\&spm=1001.2101.3001.7020)enum

在Framework层使用enum会报错：Enums are discouraged in Android APIs \[Enum]，因此一般都用@intDef代替，使用新的注解表示。

#### 2.2.5 Callback如何让APP访问到

前边**2.2.1 AIDL文件自动生成的Java文件报错**，为了解决需要添加@hide，但是添加该注解后，Apps就访问不到该callback，如何实现Binder通信呢？

解决办法：

* 创建一个public abstract class DeviceCallback，对apps暴露该类，让Apps添加注册的时候创建该类的实例。
* 然后再framework层manager中实现DeviceManager类与IDeviceCallback.aidl的一一映射关系。

这样做的好处：

* 即能避免暴露原生的AIDL文件，而且Apps不用实现ICallback.aidl中所有的方法。

//创建暴露给Apps的抽象类 

```java
package android.app.devicemanager;public abstract class DiviceCallback {void onBindChanged(int state, int deviceId);void onStateChanged(int state, int deviceId);}
```

//Manager中对应的映射关系 

```java
package android.app.devicemanager;import android.app.devicemanager.IDeiceCallback;import android.app.devicemanager.DevoceCallback;import java.util.concurrent.Executor;public class DeviceManager {private ArrayMap<DeviceCallback, DeviceCallbackEntry> mCallbackMap = new ArrayMap<>();private static final class DeviceCallbackEntry extends IDeviceCallback.Stub {final DeviceCallback mCallback;final Executor mExecutor;        DeviceCallbackEntry( DeviceCallback callback, Executor executor) {            mCallback = callback;            mExecutor = executor;        }public void onBindChanged(int state, int deviceId) {            mExecutor.executor(() -> mCallback.onBindChanged(state, deviceId));        }        ...    }public void registerDeviceCallback(@Nullable Executor executor,@NonNull DeviceCallback callback) {if(callback == null || mCallbackMap.containsKey(calback)) {return;        }DeviceCallbackEntry entry = new DeviceCallbackEntry(callback, executor);        mCallbackMap.put(callback, entry);final IDeviceManager service = getService();try {            service.registerDeviceCallback(entry);        } catch (RemoteException e) {throw e.rethrowFromSystemServer();        }    }}
```

***

## **3. Context中定义service name**

**在frameworks/base/core/java/android/content/Context.java**中增加一句

```java
public static final String DEVICE_MANAGER_SERVICE = "device_manager";
```

***

## 4. 编写SystemService

\*\*在framework/base/services/core/java/com/android/service/下创建系统服务，\*\*如果业务比较复杂，可以创建模块文件夹。

**framework/base/services/core/java/com/android/service/devicemanage/DeviceManagerService.java**

* extends IDeviceManager.Stub，重写该AILD文件中方法。该AIDL文件是SystemService与系统manager进行IPC的桥梁。
* 定义静态内部类Lifecycle，extends SystemService，重写onStart()方法，把DeviceManagerService发布到ServiceManger服务中。

```java
package com.android.service.devicemanager;import android.app.devicemanager.DeviceEntity;import android.content.Context;import android.app.devicemanager.IDeviceManager;import android.app.devicemanager.IDeviceObserver;import android.os.RemoteException;public class DeviceManagerService extends IDeviceManager.Stub {private final Context mContext;public DeviceManagerService(Context mContext) {this.mContext = mContext;    }public static class Lifecycle extends SystemService {private final DeciceManagerService mService;public Lifecycle(Context context) {super(context);            mService = new DeciceManagerService(context);        }@Overridepublic void onStart() {            publishBinderService(Context.DEVICE_MANAGER_SERVICE, mService);        }    }@Overridepublic int bindDivice(int deviceId) throws RemoteException {return 0;    }@Overridepublic DeviceEntity getDeviceEntity(int deviceId) throws RemoteException {return null;    }@Overridepublic void registerDeviceObserver(IDeviceObserver observer) {    }@Overridepublic void unregisterDeviceObserver(IDeviceObserver observer) throws RemoteException {    }}
```

***

## **5. 在SystemServer类中注册新增的系统服务**

\*\*在frameworks/base/services/java/com/android/server/SystemServer.java的startOtherServices()\*\*中添加以下代码

```java
t.traceBegin("StartLocationManagerService");mSystemServiceManager.startService(DeviceManagerService.Lifecycle.class);t.traceEnd();
```

***

## 6. 编写系统Manager类

**在framework/base/core/java/android/app/devicemanager下创建DeviceManager类**

* 在DeviceManager类上加注解@SystemService()，参数是第2步中Context中定义service name。注意该参数与DeviceManagerService中的静态内部类的Lifecycle的onStart()方法中publishBinderService(Context.DEVICE\_MANAGER\_SERVICE, mService)一致。
* 实现getService()方法，返回IDeviceManager的单例；
* 利用Singleton工具类通过Binder获取DeviceManagerService的单例；
* 对外提供接口，内部实现调用service对应的实现接口。

```java
package android.app.devicemanager;import android.content.Context;import android.os.IBinder;import android.os.RemoteException;@SystemService(Context.DEVICE_MANAGER_SERVICE)public class DeviceManager {private mContext mContext;private DeviceManager(Context context) {this.mContext = context;    }public static IDeviceManager getService() {return IDeviceManagerSingleton.get();    }public static final Singleton<IDeviceManager> IDeviceManagerSingleton =            () -> {final IBinder binder = ServiceManager.getService(Context.DEVICE_MANAGER_SERVICE);return IDeviceManager.Stub.asInterface(binder);            };public int bindDivice(int deviceId) {final IDeviceManager service = getService();try {return service.bindDivice(deviceId);        } catch (RemoteException e) {throw e.rethrowFromSystemServer();        }    }}
```

## 7. 注册系统Manager类

**在framework/base/core/java/android/app/SystemServiceRegistry.java类的j静态代码块static{}中增加以下代码**

```java
registerService(Context.DEVICE_MANAGER_SERVICE, DeviceManager.class, new CachedServiceFetcher<DeviceManager>() {@Overridepublic DeviceManager createService(ContextImpl ctx) {return new DeviceManager(ctx.getOuterContext());        }});
```

## 8. 应用调用

```java
DeviceManager mDeviceManager = (DeviceManager) mContext.getSystemService(Context.DEVICE_MANAGER_SERVICE);int deviceId = 1;mDeviceManager.bindDevice(deviceId);
```

以上是在framework层添加一个完整SystemService和manager的过程。

在Android 12中添加一个系统service，简直到处踩雷，阅读了整个API规范，各种追踪源码，才把framework层的编译通过，如果你喜欢请收藏或者点赞哦！
