# 编译 Repo 调试 工程相关

## REPO

## 编译

##### 编译framework service

```shell
make services -j16
```

push

```
adb root; adb remount ; adb push out/target/product/qssi/system/framework/services.jar /system/framework/
```

##### 编译SystemUI

```
make SystemUI -j16
```

##### surfaceflinger

```
make surfaceflinger -j16

只需要把 system/bin/surfaceflinger push 进去，然后 kill surfaceflinger 进程就可以生效了

```

##### 编译framework下的jni

cpp文件需要编译：

```
#
make libservices.core -j16
make libandroid_servers -j16

make  services -j16

adb root; adb remount; 
adb push out/target/product/qssi/system/lib64/libandroid_servers.so /system/lib64/  ; 
adb push out/target/product/qssi/system/lib/libandroid_servers.so /system/lib/ ; 
adb push out/target/product/qssi/system/framework/services.jar /system/framework/ ; 
adb reboot

```

## 调试

设置电量温度 ,温度报警在BatteryService，但是低电量现在不在这里。

```shell

adb shell dumpsys battery set level 13
adb shell dumpsys battery set temp 551

```


---

> 作者: [wentao](https://github.com/huang8604)  
> URL: https://huang8604.github.io/posts/%E7%BC%96%E8%AF%91-repo-%E8%B0%83%E8%AF%95-%E5%B7%A5%E7%A8%8B%E7%9B%B8%E5%85%B3/  

