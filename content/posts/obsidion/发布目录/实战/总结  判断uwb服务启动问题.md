---
tags:
  - blog
  - 实战
title: 总结  判断uwb服务启动问题
date: 2025-10-23T04:06:12.595Z
lastmod: 2025-10-23T07:22:18.745Z
---
有一个问题,就是配置了uwb,如果在没有uwb的机器上,会不停的启动uwb的服务.

```shell
09-29 10:19:45.885 1 1 E init : Control message: Could not find 'aidl/android.hardware.uwb.IUwb/default' for ctl.interface_start from pid: 441 (/system/bin/servicemanager)
09-29 10:19:45.942 441 29301 W libc : Unable to set property "ctl.interface_start" to "aidl/android.hardware.uwb.IUwb/default": PROP_ERROR_HANDLE_CONTROL_MESSAGE (0x20)
09-29 10:19:45.948 1725 4002 W ServiceManagerCppClient: Waited one second for android.hardware.uwb.IUwb/default (is service started? Number of threads started in the threadpool: 32. Are binder threads started and available?)

```

网络上的一个类似的方案:

```java

static bool HandleControlMessage(std::string_view message, const std::string& name,pid_t from_pid) {
+  if("aidl/android.hardware.biometrics.fingerprint.IFingerprint/default" == name){
+    return true;
+  }

std::string cmdline_path = StringPrintf("proc/%d/cmdline", from_pid);
    std::string process_cmdline;
    if (ReadFileToString(cmdline_path, &process_cmdline)) {
        std::replace(process_cmdline.begin(), process_cmdline.end(), '\0', ' ');
        process_cmdline = Trim(process_cmdline);
    } else {
        process_cmdline = "unknown process";
    }
}


```

那么进过验证,对uwb没有作用.\
进过调试,如果把支持uwb 的feature删除, 确认服务是不会启动的.所以我们在pms里增加的打印堆栈

通过堆栈,我们发现是在SystemServer中

```java
    /**
     * Starts a miscellaneous grab bag of stuff that has yet to be refactored and organized.
     */
    private void startOtherServices(@NonNull TimingsTraceAndSlog t) {
        t.traceBegin("startOtherServices");

        if (context.getPackageManager().hasSystemFeature(PackageManager.FEATURE_UWB)) {
            t.traceBegin("UwbService");
            mSystemServiceManager.startServiceFromJar(UWB_SERVICE_CLASS, UWB_APEX_SERVICE_JAR_PATH);
            t.traceEnd();
        }
```

所以我们的修改方案就是在没有uwb的机器上,如果需要获取hasSystemFeature的地方,给返回false.

frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java

```java

    public boolean hasSystemFeature(String name, int version) {
        // allow instant applications

        if("android.hardware.uwb".equals(name)) {
            //Log.d("PackageManagerService wentao", "wentao hasSystemFeature uwb" +Log.getStackTraceString(new Throwable()));

            //getprop ro.hw.mcu.mac
            String mcuMac = SystemProperties.get("ro.hw.mcu.mac", "(unknown)");
            Log.d("PackageManagerService", "mcuMac: " + mcuMac);
            if("(unknown)".equals(mcuMac)) {
                return false;
            }else{
                return true;
            }
        }

```

**这个判断uwb的判断条件一定要在systemserver startOtherServices 之前,不然时序不对.**
