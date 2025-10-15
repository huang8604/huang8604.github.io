---
title: 开发中常用到的一些调试命令 - tisheng zhang - Confluence
author:
  - "[[tisheng zhang]]"
created: 2025-07-16
tags:
  - clippings
  - 转载
  - blog
collections:
  - 图形显示
source: 
date: 2025-07-16T01:32:57.339Z
lastmod: 2025-07-16T01:48:27.000Z
---
## 单编模块命令：在SYSTEM/VENDOR目录下

source build/envsetup.sh\
lunch (选编号，也可以直接输入名称+"-"+user/userdebug，这个不知道可以看编译好的out/target/product下的目录名，或者直接看build\_android.sh脚本里定义的 )

make module\_name

一般都是在对应代码模块目录下的Android.bp就是 **`android_app java_library cc_library cc_binary apex等字段`**

**Android.bp中常见的module\_name:**

<table><colgroup><col> <col> <col> <col></colgroup><thead><tr><th><p><strong><code>android_app</code></strong></p></th><th><p>APK</p></th><th><p>Settings SystemUI <span>framework-res</span></p></th><th colspan="1"><p>system/app system/priv-app system_ext/app system_ext/priv-app</p></th></tr></thead><tbody><tr><td><strong><code>java_library</code></strong></td><td>Jar包</td><td>framework-minus-apex services</td><td colspan="1">system/framework</td></tr><tr><td><strong><code>cc_library</code></strong></td><td>so库</td><td>libaudiomanager libbattery</td><td colspan="1">system/lib system/lib64</td></tr><tr><td><strong><code>cc_binary</code></strong></td><td>bin</td><td>vold bugreport</td><td colspan="1">system/bin</td></tr><tr><td><strong><code>apex</code></strong></td><td>apex</td><td>com.android.wifi com.android.vndk.current</td><td colspan="1">system/apex</td></tr></tbody></table>

**adb root;adb remount;adb push xxx xxx;adb reboot**

**有些是 [Android.mk](http://android.mk/) 文件中定义的LOCAL\_MODULE字段，比如：**

**selinux**

**make selinux\_policy**

**system/etc/selinux**

**vendor/etc/selinux**

## 预编译的 SELinux 政策：sepolicy\_and\_mapping.sha256也需要push

<https://source.android.google.cn/docs/security/features/selinux/build?hl=zh-cn#precompiled-policy>

## adb settings命令：

`settings` 命令通过 **Binder IPC** 与 Android 的 **Settings Provider** （位于 `com.android.providers.settings` ）交互，操作三个核心命名空间的配置数据库：

* **`system`**\
  存储设备系统级配置（如屏幕超时、亮度），路径为 `/data/system/users/0/settings_system.xml`
  * **权限** ：普通应用可读，修改需 `WRITE_SETTINGS` 权限。
* **`secure`**\
  存储敏感配置（如密码、默认输入法），路径为 `/data/system/users/0/settings_secure.xml`
  * **权限** ：仅系统应用或特权进程可修改。
* **`global`**\
  存储全局配置（如飞行模式、ADB 状态），路径为 `/data/system/users/0/settings_global.xml`

  * **权限** ：需系统签名或 `WRITE_SECURE_SETTINGS` 权限。

> ⚠️ 注意：不同 Android 版本可能对配置项有定制，部分命令需 **Root 权限** 。

#### 基础操作

| **命令**   | **用途**    | **示例**                                                       |
| -------- | --------- | ------------------------------------------------------------ |
| `list`   | 列出命名空间所有键 | `adb shell settings list global`                             |
| `get`    | 获取键值      | `adb shell settings get system screen_brightness` （返回亮度值）    |
| `put`    | 设置键值      | `adb shell settings put system screen_brightness 150` （设置亮度） |
| `delete` | 删除键值      | `adb shell settings delete secure test_ke`                   |
| `reset`  | 重置配置      | `adb shell settings reset global com.example.app` （重置配置）     |

跳过开机向导：adb shell settings put global device\_provisioned 1;adb shell settings put secure user\_setup\_complete 1

设置不检查互联网连接 adb shell settings put global captive\_portal\_mode 0/1

修改长亮时间：adb shell settings put system screen\_off\_timeout 60000 # 60 秒

开发过程中可能遇到需要临时性的关闭某个功能进行验证或测试的情况，修改代码时可以通过添加这个开关的方式动态的控制：

```java
privatevoidsetCpuPerformance(booleanenable) {
    booleanneedPerformance = Settings.Global.getInt(mContext.getContentResolver(), "wifi_cpu_performance_enable", 1) == 1;
    if(!needPerformance) {
        return;
    }
    android.os.SystemProperties.set("sys.cpu.performance",enable?"1":"0");
}
```

setprop getprop

## 截图和录屏命令

adb shell screencap /sdcard/screenshot.png

adb shell screenrecord /sdcard/screenrecord.mp4

adb pull /sdcard/xxx

Window 命令行支持直接保存截图 adb exec-out screencap -p > screenshot.png

## bugreport命令

**抓取系统日志，打包成bugreport-xxxx-xxxx.zip，包含的数据有** ：

* **系统属性** （ `getprop` 输出）
* **内核日志** （ `dmesg` 、 `/proc` 文件）
* **服务状态** （ `dumpsys` 输出，通过 Binder 调用服务的 `dump()` 方法）
* **日志文件** （logcat、ANR、tombstones）
* **硬件信息** （ `lshal` 输出）

## dumpsys 命令

`dumpsys` 通过 Binder IPC 与 Android 的核心组件 **ServiceManager** 通信，获取所有已注册系统服务的列表。每个服务（如 `activity` 、 `window` 、 `power` ）均实现 `dump()` 接口，用于返回自身状态信息

#### 基础命令

| **命令**                       | **作用**              | **示例输出内容**                                                     |
| ---------------------------- | ------------------- | -------------------------------------------------------------- |
| `adb shell dumpsys -l`       | 列出所有支持的服务（约 100+ 项） | `activity`, `window`, `power`, battery 等                       |
| `adb shell dumpsys <服务名>`    | 获取特定服务的详细信息         | 如 `dumpsys activity` 输出任务栈、Broadcast 队列等                       |
| `adb shell dumpsys <服务名> -h` | 查看服务的帮助文档及子命令       | `dumpsys activity -h` 显示 `a[ctivities]` 、 `b[roadcasts]` 等子命令选 |

#### 2. 高频服务详解

<table><colgroup><col> <col> <col></colgroup><thead><tr><th><p><strong>服务名</strong></p></th><th><p><strong>关键信息</strong></p></th><th><p><strong>实用场景</strong></p></th></tr></thead><tbody><tr><td><strong><code>activity</code></strong></td><td>任务栈、Activity 生命周期、Broadcast 队列</td><td>分析 ANR、排查页面跳转异常</td></tr><tr><td><strong><code>settings</code></strong></td><td>settings 设置的数据，修改历史等</td><td>分析用户操作行为，机器开关状态</td></tr><tr><td><strong><code>battery</code></strong></td><td>电量百分比、充电状态、温度</td><td>监控电池健康状态</td></tr><tr><td><strong><code>window</code></strong></td><td>窗口焦点、Surface 状态、Display 信息</td><td>调试悬浮窗、分屏异常</td></tr><tr><td colspan="1">input</td><td colspan="1">输入设备信息，按键事件记录</td><td colspan="1">分析input输入事件</td></tr><tr><td>package<strong></strong></td><td>应用安装信息，权限等信息</td><td>分析已安装应用行为</td></tr><tr><td><strong><code>wifi</code></strong></td><td>wifi信息</td><td>分析wifi运行情况</td></tr><tr><td colspan="1">overlay</td><td colspan="1">系统的overlay信息</td><td colspan="1">分析系统overlay配置情况</td></tr></tbody></table>

比如：

打印systemui应用的信息：adb shell dumpsys activity service com.android.systemui

查看当前页面报名：adb shell 'dumpsys window | grep mFocus'

## cmd

`cmd` 是 Android 系统提供的一个 **高级系统管理命令** ，用于直接调用设备内置的系统服务（System Services），实现对硬件、软件配置的深度控制。其核心逻辑是通过 `cmd` 命令的子命令（如 `battery` 、 `wifi` 等）访问 Android 的 `Binder` IPC 接口，与系统服务（如 `BatteryService` 、 `WifiService` ）交互

adb shell cmd <服务名> <子命令> \[参数]

***

### 🔋

### 一、电池管理 (battery)

用于获取或模拟电池状态（ **需 Android 6.0+** ）：

| **命令**             | **功能**             | **示例**                               |
| ------------------ | ------------------ | ------------------------------------ |
| `set level <百分比>`  | 设置电池电量             | `cmd battery set level 20` (电量设为20%) |
| `set status <状态码>` | 修改充电状态（1:放电, 2:充电） | `cmd battery set status 2` (模拟充电)    |
| `set usb <0/1>`    | 禁用/启用USB充电         | `cmd battery set usb 0` (禁用USB充电)    |
| `unplug`           | 模拟断开充电（仅限软件层面）     | `cmd battery unplug`                 |
| `reset`            | 恢复实际电池状态           | `cmd battery reset`                  |

> **注意** ：修改后需通过 `adb shell dumpsys battery` 验证状态

***

### ⚡ 二、电源管理 (power)

控制设备休眠、唤醒等电源策略：

| **命令**                              | **功能**      | **示例**                                  |
| ----------------------------------- | ----------- | --------------------------------------- |
| `set-mode <模式>`                     | 设置电源模式（0-3） | `cmd power set-mode 2` (低功耗模式)          |
| `set-adaptive-power-saver <on/off>` | 启用/禁用自适应省电  | `cmd power set-adaptive-power-saver on` |
| `go-to-sleep`                       | 立即休眠屏幕      | `cmd power go-to-sleep`                 |
| `wakeup`                            | 唤醒设备        | `cmd power wakeup`                      |

***

### 📱 三、Activity管理 (activity)

启动/停止应用组件：

| **命令**                     | **功能** | **示例**                                                           |
| -------------------------- | ------ | ---------------------------------------------------------------- |
| `start-foreground-service` | 启动前台服务 | `cmd activity start-foreground-service com.example/.MyService`   |
| `broadcast`                | 发送广播   | `cmd activity broadcast -a android.intent.action.BOOT_COMPLETED` |
| `force-stop`               | 强制停止应用 | `cmd activity force-stop com.example.app`                        |

> 启动Activity更常用 **`adb shell am start`** （非 `cmd` 子命令）

***

### 📦 四、包管理 (package)

应用安装、权限控制：

| **命令**                   | **功能**  | **示例**                                                        |
| ------------------------ | ------- | ------------------------------------------------------------- |
| `install <apk路径>`        | 安装应用    | `cmd package install /sdcard/test.apk`                        |
| `uninstall <包名>`         | 卸载应用    | `cmd package uninstall com.example.app`                       |
| `grant/revoke <包名> <权限>` | 授予/取消权限 | `cmd package grant com.example.app android.permission.CAMERA` |
| `clear <包名>`             | 清楚应用缓存  | `cmd package clear com.example.app`                           |

> 查询应用列表建议用 **`adb shell pm list packages`**

***

### 📶 五、Wi-Fi管理 (wifi)

控制网络扫描、热点等：

<table><colgroup><col> <col> <col></colgroup><thead><tr><th><p><strong>命令</strong></p></th><th><p><strong>功能</strong></p></th><th><p><strong>示例</strong></p></th></tr></thead><tbody><tr><td>get-country-code</td><td>获取当前国家码</td><td><code>cmd wifi get-country-code</code></td></tr><tr><td colspan="1">force-country-code enabled &lt;two-letter code&gt; | disabled</td><td colspan="1">设置国家码</td><td colspan="1">cmd wifi force-country-code enabled US</td></tr><tr><td>set-wifi-enabled enabled|disabled</td><td>启用/禁用WIFI</td><td>cmd wifi set-wifi-enabled disabled</td></tr><tr><td>start-scan</td><td>进行一次扫描</td><td><code>cmd wifi start-scan</code></td></tr></tbody></table>

大部分子命令都可以通过-h获取帮助

其他一些常用的命令am、pm、wm

恢复出厂设置设置：adb root;adb shell am broadcast -a android.intent.action.FACTORY\_RESET -p android --receiver-foreground --es android.intent.extra.REASON MainClearConfirm --ez android.intent.extra.WIPE\_EXTERNAL\_STORAGE true --ez com.android.internal.intent.extra.WIPE\_ESIMS true

快速调起工模：adb shell am start -a android.intent.action.FACTORY\_TEST

快速调起设置：adb shell am start -a android.settings.SETTINGS

显示当前Activity堆栈：adb shell am stack list

列出已安装应用报名：adb shell pm list packages （-f 显示路径 -s只显示系统应用 -3只显示三方应用）

参看已知报名的路径：adb shell pm -p xxx.xxx.xxx

打印应用的相关信息：adb shell pm dump-package xxx.xxx.xxx

修改屏幕尺寸：adb shell wm size

修改屏幕分辨率：adb shell wm density

nohup

### ⚙️ 核心用法

#### 1. 基础后台执行

将命令通过 `nohup` 在设备后台运行：

`adb shell "nohup <command> &"  # & 表示后台执行`

**示例** ：后台持续收集日志

`adb shell "nohup logcat -f /sdcard/log.txt &"  `

* **效果** ：断开 USB 或关闭终端后，日志仍持续写入 `/sdcard/log.txt`

#### 2. 重定向输出

默认输出到 `nohup.out` ，可通过重定向保存到自定义文件：

`adb shell "nohup <command> > /sdcard/output.log 2>&1 &"  `

* **`2>&1`** ：将错误输出合并到标准输出
* **示例** ：

  `adb shell "nohup top -b > /sdcard/top.log 2>&1 &" `

***

### ⚡ 进阶场景

#### 1. 执行复杂脚本

若需运行多行脚本（如循环或条件判断）：

`adb shell "nohup sh -c 'while true; do dumpsys battery; sleep 10; done > /sdcard/battery.log 2>&1' &"`

* 使用 `sh -c` 包裹多行命令

  apk签名命令（需要代码编译过一次）：java -jar out/host/linux-x86/framework/apksigner.jar sign -key build/make/target/product/security/platform.pk8 -cert build/make/target/product/security/platform.x509.pem --in unsigned.apk --out signed.apk

使用Android Studio 直接编译签名

openssl pkcs8 -inform DER -nocrypt -in platform.pk8 -out private.pem;openssl pkcs12 -export -in platform.x509.pem -inkey private.pem -out platform.p12 -password pass:android -name platform

```java
android {
    signingConfigs {
        create("platform") {
            storeFile =
                file("C:\\Users\\zhangtisheng\\AndroidStudioProjects\\Demo\\app\\platform.p12")
            storePassword = "android"
            keyAlias = "platform"
            keyPassword = "android"
        }
    }
    buildTypes {
        create("platform") {
            signingConfig = signingConfigs.getByName("platform")
        } 
    }
}
```

比如我们需要对某个界面布局做出调整：

1.使用adb shell 'dumpsys window | grep mFocus'获取此界面的包名和Activity名

2.使用 adb shell pm -p xxx.xxx.xxx通过包名获取应用的模块名

3.根据模块名到相应的代码目录下，一般原生应用在packages/apps或者modules或者services下，高通的定制应用可能在vendor/codeaurora或者vendor/qcom下

4.根据Activity名，找到对应的界面代码，再定位到对应的xml进行调整

5.此时还可以使用monitor（Sdk\tools\lib\monitor-x86\_64）工具，进一步获取界面的布局的详细信息，根据这些id来进一步确认对应的控件
