---
title: Android TV开机优化_android 关闭class verify-CSDN博客
source: https://blog.csdn.net/he980725866/article/details/123030906
author: 
published: 
created: 2024-11-06
description: 文章浏览阅读4.6k次，点赞5次，收藏26次。Android开机优化_android 关闭class verify
tags:
  - clippings
  - blog
date: 2024-11-06T06:57:52.891Z
lastmod: 2024-11-06T04:26:04.890Z
---
## Android开机优化相关点

### 1 关键路径

bootloader > kernel > init > zygote > system server > [launcher](https://so.csdn.net/so/search?q=launcher\&spm=1001.2101.3001.7020)

![be3124bd9693d56ef1297d95ee3f3a3f\_MD5](https://picgo.myjojo.fun:666/i/2024/11/06/672aef703a575.png)

### 2 打印优化

2.1 关闭bootloader打印

2.2 关闭 kernel打印

2.3 提高Android log打印等级\
system/core/logd/LogBuffer.cpp

```
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

### 3 uboot

**1 修改emmc速度提升读写速度加快uboot启动时间**

**2 提高uboot cpu的频率**

### 4 kernel

如果不是非常紧急的驱动延迟初始化，可以等zygote启动后，在early-boot或boot阶段去加载ko, 让zygote早点起来，zygote 在late-init阶段后面起来

比如wifi和蓝牙驱动

```
on boot
    chown bluetooth bluetooth /proc/bluetooth/sleep/btwrite
    chown bluetooth bluetooth /proc/bluetooth/sleep/lpm
    chmod 0660 /proc/bluetooth/sleep/btwrite
    chmod 0660 /proc/bluetooth/sleep/lpm

    insmod /vendor/lib/modules/btusb.ko
```

### 5 init

SecondStageMain 触发的各个阶段

![9f6b193f80bae6c84c77b9f39add1be2\_MD5](https://picgo.myjojo.fun:666/i/2024/11/06/672aef703a575.png)

zygote在late-init稍后的阶段启动

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

#### 5.1 并行执行uevent(enable\_parallel\_restorecon)

vendor/ueventd.rc中加入parallel\_restorecon enable

```
parallel_restorecon enable
```

#### 5.2 CPU开启性能模式

1 开机的时候，cpu开启性能模式

```bash
on early-init
    write /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor performance
    
# 查看当前频率
# cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
```

2 开机完成后，CPU频率变成自适应

```
on property:sys.boot_completed=1
    write /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor schedutil
```

#### 5.3 读写IO调整

read\_ahead\_kb: 事先预读数据的Kb数

nr\_requests: 默认IO请求队列的长度

加大read\_ahead\_kb和nr\_requests 大小，优化时间不明显

```
on late-fs
    # boot time fs tune
    write /sys/block/mmcblk0/queue/iostats 0
    write /sys/block/mmcblk0/queue/read_ahead_kb 2048
    write /sys/block/mmcblk0/queue/nr_requests 256
    
on property:sys.boot_completed=1
    # end boot time fs tune
    write /sys/block/mmcblk0/queue/read_ahead_kb 128
    
```

#### 5.4 移除没有用到的模块

如CI模块

```
on post-fs-data
    insmod /vendor/lib/modules/cimax-usb.ko
    insmod /vendor/lib/modules/ci.ko
```

#### 5.5 延迟vendor/etc/init下的非关键服务，放在early-boot或者boot阶段执行，加快zygote的启动

```java
// 放在early-boot或者boot阶段执行
on boot
    start xxx

service xxx /system/bin/xxx
    user root
    group system
```

### 6 zygote

#### 6.1 lazy-preload

classloader懒加载，懒加载会比直接加载更耗内存，懒加载是通过system-server发送指令给zygote做加载相关动作，在发送指令前，system-server会加载一部分自己使用的类，会和zygote中存在相同的一部分备份

```
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server --enable-lazy-preload
```

#### 6.2 zygote 添加 task\_profiles ProcessCapacityHigh MaxPerformance

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

#### 6.3 精简preload的classes , TV系统可以去掉如下class

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

#### 6.4 关闭 systemServer虚拟机实列的 ClassVerify

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

### 7 systemServer

#### 7.1 调整systemServer进程，线程优先级

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

#### 7.2 调整SystemServerInitThreadPool线程池相关优先级

```
private SystemServerInitThreadPool() {
    final int size = Runtime.getRuntime().availableProcessors();
    Slog.i(TAG, "Creating instance with " + size + " threads");
    mService = ConcurrentUtils.newFixedThreadPool(size,
    	"system-server-init-thread", -18);//Process.THREAD_PRIORITY_FOREGROUND);
}
```

#### 7.3 裁剪TV系统中没有用到的服务，可以用feture的形式来控制开关

```
if (mPackageManager.hasSystemFeature(PackageManager.FEATURE_xxx)) {          
	mSystemServiceManager.startService(xxx_SERVICE_CLASS);
}
```

查看systemServer中的服务

```
service list
```

可以裁剪的服务

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

#### 7.4 延时启动presist app

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

### 8 Laucher

1 Laucher APK的编译方式改为 speed-profile

```
LOCAL_DEX_PREOPT_FLAGS += --compiler-filter=speed-profile

// debug
pm compile -m speed -f package
dumpsys package package
```

2 如果是内置的launch，可以将launch设置为presist apk

### 9 调试方法

#### 9.1 打开init log

1 init中的log会输出到kernel log中，打开init log查看耗时的时间点，不同平台方法可能不一样，需要将kernel log输出等级设置为7

2 rc中设置如下参数

vendor/etc/init/hw/init.xxx.rc

```
on early-init
    loglevel 7
```

#### 9.2 通过logcat -b events，过滤出启动阶段的主要事件

```
logcat -b events |grep "boot_p"
```

```
boot_progress_start: 6466
01-18 11:28:13.935   647   647 I boot_progress_system_run: 10467
01-18 11:28:15.368   647   647 I boot_progress_pms_start: 11900
01-18 11:28:15.748   647   647 I boot_progress_pms_system_scan_start: 12280
01-18 11:28:16.338   647   647 I boot_progress_pms_data_scan_start: 12870
01-18 11:28:16.350   647   647 I boot_progress_pms_scan_end: 12881
01-18 11:28:16.642   647   647 I boot_progress_pms_ready: 13174
01-18 11:28:18.973   647   647 I boot_progress_ams_ready: 15505
01-18 11:28:24.131   647   722 I boot_progress_enable_screen: 20663
01-18 11:28:24.143   392   767 I sf_stop_bootanim: 20675
01-18 11:28:24.144   647   722 I wm_boot_animation_done: 20676
```

#### 9.3 通过 logcat -b system查看系统中一些流程的耗时点, 结合trace一起分析

```
Zygote32Timing: ZygoteInit took to complete: 1454ms
SystemServerInitThreadPool: Creating instance with 4 threads
[20220127_12:10:10:054]01-18 12:04:51.352   667   667 D SystemServerTiming: InitBeforeStartServices took to complete: 745ms
**Zygote32Timing: ZygoteInit took to complete: 1454ms**
SystemServerTiming: InitBeforeStartServices took to complete: 745ms
SystemConfig: readAllPermissions took to complete: 160ms
SystemServerTimingAsync: InitThreadPoolExec:ReadingSystemConfig took to complete: 161ms
SystemServerTiming: StartActivityManager took to complete: 364ms
**PackageManagerTiming: read user settings took to complete: 313ms**
SystemServerTimingAsync: AppDataFixup took to complete: 177ms
PackageManagerTiming: GC took to complete: 106ms
**PackageManagerTiming: create package manager took to complete: 1411ms**
**SystemServerTiming: StartPackageManagerService took to complete: 1425ms**
SystemServerTiming: startBootstrapServices took to complete: 2129ms
SystemServerTimingAsync: InitThreadPoolExec:PersistentDataBlockService.onStart took to complete: 80ms
SystemServerTiming: OnBootPhase_500 took to complete: 204ms
SystemServerTiming: StartBootPhaseSystemServicesReady took to complete: 204ms
SystemServerTiming: MakePowerManagerServiceReady took to complete: 105ms
ZygoteInitTiming_lazy: PreloadClasses took to complete: 1737ms
ZygoteInitTiming_lazy: PreloadResources took to complete: 124ms
SystemServerTimingAsync: SecondaryZygotePreload took to complete: 1960ms
SystemServerTiming: OnBootPhase_550 took to complete: 212ms
SystemServerTiming: StartActivityManagerReadyPhase took to complete: 213ms
OnBootPhase_600 took to complete: 122ms
SystemServerTiming: PhaseThirdPartyAppsCanStart took to complete: 122ms
SystemServerTiming: ssm.StartUser-0 took to complete: 165ms
SystemServerTiming: showSystemReadyErrorDialogs took to complete: 119ms
SystemServerTiming: ActivityManagerStartApps took to complete: 537ms**
SystemServerTiming: PhaseActivityManagerReady took to complete: 1129ms
SystemServerTiming: startOtherServices took to complete: 3032ms
SystemServerTiming: StartServices took to complete: 5217ms
**SystemUIBootTiming: StartServicescom.android.systemui.volume.VolumeUI took to complete: 255ms**
**SystemUIBootTiming: StartServices took to complete: 429ms**
**WindowManager: Keyguard drawn timeout. Setting mKeyguardDrawComplete**
ActivityManagerTiming: OnBootPhase_1000 took to complete: 109ms
ActivityManagerTiming: TotalBootTime took to complete: 9939ms
**ActivityManagerTiming: FinishBooting took to complete: 134ms**
**SystemServerTiming: SystemUserUnlock took to complete: 611ms
```

#### 9.4 抓取trace

```
atrace -z -b 65536 gfx input view wm am sm audio video hal res dalvik rs power pm ss database aidl sched irq freq idle disk mmc sync workq memreclaim binder_driver binder_lock pagecache gfx -t 20 > /data/trace
```

#### 9.5 bootchart

启用 bootchart

```
adb shell 'touch /data/bootchart/enabled'
adb reboot
```

#### 9.6 perfboot

代码位置在system/core/init/perfboot.py，脚本执行结束后，可以在Excel里直接打开文件进行分析

```
./perfboot.py --iterations=2 --interval=30 -v --output=/tmp/J5D_UE.tsv
```

<https://source.android.google.cn/devices/tech/perf/boot-times?hl=zh-cn>\
<https://source.android.google.cn/security/verifiedboot/dm-verity?hl=zh-cn>
