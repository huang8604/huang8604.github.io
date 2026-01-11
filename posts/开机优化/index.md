# 开机优化

![image.png](https://picgo.myjojo.fun:666/i/2024/08/30/66d15ffcd3583.png)\
根据开机的阶段，有一些优化开机速度的空间

#### 1. bootloader阶段

这边涉及的系统的稳定性，能修改的地方不多。\
最多的是去处log，对性能提升不明显，并且对后期问题的分析有比较大的影响。

或者通过属性的控制，来控制是否打印和打印的等级。

```
system/core/logd/LogBuffer.cpp
int get_log_level() {
    char buf[PROPERTY_VALUE_MAX];
    memset(buf, 0, PROPERTY_VALUE_MAX);
    property_get("persist.xxx.level", buf, "");
    return buf[0];
}
​
int loglevl = get_log_level();
  
int LogBuffer::log(log_id_t log_id, log_time realtime, uid_t uid, pid_t pid,
                   pid_t tid, const char* msg, uint16_t len) {
    if (log_id >= LOG_ID_MAX) {
        return -EINVAL;
    }
     
    // 通过属性控制log输出
     if (length > 0 && loglevl < log_id) {
        return -EINVAL;
    }
​
}


```

#### 2  kernel阶段

对于kernel 阶段的有， 对于不影响开机的modle ，可以建议延迟加载。\
可以等zygote启动后，在early-boot或boot阶段去加载ko, 让zygote早点起来\
比如蓝牙和wifi的ko驱动

#### 3  Init阶段

![image.png](https://picgo.myjojo.fun:666/i/2024/09/18/66ea39d12762f.png)

##### 3.1 zygote 在 late-init之后启动

```
# Mount filesystems and start core system services.
on late-init
    trigger early-fs

    # Mount fstab in init.{$device}.rc by mount_all command. Optional parameter
    # '--early' can be specified to skip entries with 'latemount'.
    # /system and /vendor must be mounted by the end of the fs stage,
    # while /data is optional.
    trigger fs
    trigger post-fs

    # Mount fstab in init.{$device}.rc by mount_all with '--late' parameter
    # to only mount entries with 'latemount'. This is needed if '--early' is
    # specified in the previous mount_all command on the fs stage.
    # With /system mounted and properties form /system + /factory available,
    # some services can be started.
    trigger late-fs

    # Now we can mount /data. File encryption requires keymaster to decrypt
    # /data, which in turn can only be loaded when system properties are present.
    trigger post-fs-data

    # Load persist properties and override properties (if enabled) from /data.
    trigger load_persist_props_action

    # Should be before netd, but after apex, properties and logging is available.
    trigger load_bpf_programs

    # Now we can start zygote for devices with file based encryption
    # 触发zygote启动
    trigger zygote-start

    # Remove a file to wake up anything waiting for firmware.
    trigger firmware_mounts_complete

    trigger early-boot
    trigger boot
```

##### 3.2 并行执行uevent(enable\_parallel\_restorecon)

vendor/ueventd.rc中加入parallel\_restorecon enable

```
parallel_restorecon enable
```

##### 3.3 CPU 打开性能模式

1 开机的时候，cpu开启性能模式

```
on early-init
    write /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor performance
    
# 查看当前频率
# cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
```

2 开机后关闭,改成自动适应

```
on property:sys.boot_completed=1 write /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor schedutil
```

#### 4 移除没有用到的模块

比如：\
如CI模块

```
on post-fs-data
    insmod /vendor/lib/modules/cimax-usb.ko
    insmod /vendor/lib/modules/ci.ko
```

#### 5 延迟vendor/etc/init下的非关键服务

放在early-boot或者boot阶段执行，加快zygote的启动

```
// 放在early-boot或者boot阶段执行 
on boot 
	start xxx 
service xxx /system/bin/xxx 
	user root 
	group system
```

#### 6 zygote

##### classloader懒加载，

懒加载会比直接加载更耗内存，懒加载是通过system-server发送指令给zygote做加载相关动作，在发送指令前，system-server会加载一部分自己使用的类，会和zygote中存在相同的一部分备份

```
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server --enable-lazy-preload
```

##### zygote 添加 task\_profiles ProcessCapacityHigh MaxPerformance

```
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server --enable-lazy-preload
    class main
    priority -20
    user root
    group root readproc reserved_disk
    socket zygote stream 660 root system
    socket usap_pool_primary stream 660 root system
    onrestart exec_background - system system -- /system/bin/vdc volume abort_fuse
    onrestart write /sys/power/state on
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart netd
    onrestart restart wificond
    task_profiles ProcessCapacityHigh MaxPerformance

```

##### 精简preload的classes , 比如可以去掉如下class

根据项目的时间情况\
如果需要再做一些大的裁剪，可以使用frameworks\base\config\generate-preloaded-classes.sh下的脚本来生成preloaded-classes

```
// 生物识别
android.hardware.biometrics

// 人脸识别
android.hardware.face

// 打印服务
android.hardware.fingerprint
android.print.

// 部分定位相关, 还有GPS定位相关
android.hardware.location
com.android.internal.location.GpsNetInitiatedHandler
android.location.Gnss*

// 手机通话相关
android.telephony.
android.telecom.
com.android.i18n.phonenumbers.
com.android.ims
android.hardware.radio

// nfc相关
android.nfc.
```

#### 关闭 systemServer虚拟机实列的 ClassVerify

确认是否需要Disable\_verify，不建议关闭

```
/** Turn off the verifier. */
public static final int DISABLE_VERIFIER = 1 << 9;
```

```
 parsedArgs.mRuntimeFlags |= Zygote.DISABLE_VERIFIER;
            /* Request to fork the system server process */
            pid = Zygote.forkSystemServer(
                    parsedArgs.mUid, parsedArgs.mGid,
                    parsedArgs.mGids,
                    parsedArgs.mRuntimeFlags,
                    null,
                    parsedArgs.mPermittedCapabilities,
                    parsedArgs.mEffectiveCapabilities);

```

#### systemServer

##### 7.1 调整systemServer进程，线程优先级

```
android.os.Process.setThreadPriority(-19);
android.os.Process.setThreadPriority(android.os.Process.THREAD_PRIORITY_FOREGROUND);
Process.setProcessGroup(pid, Process.THREAD_GROUP_TOP_APP);
```

```
private void run() {
    android.os.Process.setThreadPriority(-19);
    int defaultGroup = getProcessGroup(Process.myPid());
    Process.setProcessGroup(Process.myPid(), Process.THREAD_GROUP_TOP_APP);
    
    // Start services.
    try {
        t.traceBegin("StartServices");
        startBootstrapServices(t);
        startCoreServices(t);
        startOtherServices(t);
    } catch (Throwable ex) {
        Slog.e("System", "******************************************");
        Slog.e("System", "************ Failure starting system services", ex);
        throw ex;
    } finally {
        t.traceEnd(); // StartServices
    }

    android.os.Process.setThreadPriority(
        android.os.Process.THREAD_PRIORITY_FOREGROUND);
    Process.setProcessGroup(Process.myPid(), defaultGroup);
    // Loop forever.
    Looper.loop();

}
```

##### 7.2 调整SystemServerInitThreadPool线程池相关优先级

```
private SystemServerInitThreadPool() {
    final int size = Runtime.getRuntime().availableProcessors();
    Slog.i(TAG, "Creating instance with " + size + " threads");
    mService = ConcurrentUtils.newFixedThreadPool(size,
    	"system-server-init-thread", -18);//Process.THREAD_PRIORITY_FOREGROUND);
}
```

##### 7.3 裁剪TV系统中没有用到的服务，可以用feture的形式来控制开关

```
if (mPackageManager.hasSystemFeature(PackageManager.FEATURE_xxx)) { mSystemServiceManager.startService(xxx_SERVICE_CLASS); }
```

```
TelecomLoaderService
TelephonyRegistry
StatusBarManagerService
SearchManagerService
SerialService
FingerprintService
CameraService
MmsService
```

##### 7.4 延时启动presist app

```
void startPersistentApps(int matchFlags) {
    if (mFactoryTest == FactoryTest.FACTORY_TEST_LOW_LEVEL) return;

    synchronized (this) {
        try {
            final List<ApplicationInfo> apps = AppGlobals.getPackageManager()
                    .getPersistentApplications(STOCK_PM_FLAGS | matchFlags).getList();
            for (ApplicationInfo app : apps) {
                if (!"android".equals(app.packageName)) {
                    mHandler.postDelayed(() -> {
                        addAppLocked(app, null, false, null /* ABI override */,
                                ZYGOTE_POLICY_FLAG_BATCH_LAUNCH);
                    }, 1000);
                }
            }
        } catch (RemoteException ex) {
        }
    }
}
```

#### 8 Laucher

1 Laucher APK的编译方式改为 speed-profile

```
LOCAL_DEX_PREOPT_FLAGS += --compiler-filter=speed-profile // debug pm compile -m speed -f package dumpsys package package
```

2 如果是内置的launch，可以将launch设置为presist apk


---

> 作者: [wentao](https://github.com/huang8604)  
> URL: https://huang8604.github.io/posts/%E5%BC%80%E6%9C%BA%E4%BC%98%E5%8C%96/  

