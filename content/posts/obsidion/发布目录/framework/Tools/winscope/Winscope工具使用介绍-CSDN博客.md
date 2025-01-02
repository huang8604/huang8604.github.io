---
title: android车机手机黑屏闪黑终结者-Winscope工具使用介绍-CSDN博客
author: 
created: 2024-10-23
tags:
  - clippings
  - 转载
  - blog
collections: 图形显示
source: https://blog.csdn.net/learnframework/article/details/129432374
date: 2024-11-06T06:57:54.639Z
lastmod: 2024-12-11T07:29:22.447Z
---
### 背景：

设想一下，假如我们又如下场景，一个闪黑一瞬间的问题，正常我们看到黑屏冻屏问题，是不是时刻想到是要来dumpsys SurfaceFlinger和dumpsys window windows相关的信息来辅助我们分析问题，但奈何这个是个瞬时问题。。。我们dumpsys很难抓住那一瞬间，而且即使抓到了黑一瞬间的，我们有时候分析也要又黑屏前一帧后一帧相关等才可以分析进一步原因。

所以在开发过程中，经常会遇到各种各样的窗口问题，比如动画异常、窗口异常、闪屏、闪黑、黑屏、错位显示…

对于这些问题，添加日志，调试分析代码等手段去解决，但这些 UI 问题往往出现在一瞬间，很难把握出现的时机，录制下来的日志往往也是巨大的，从海量的日志中提取有效的信息是一个枯燥且繁琐的事情，而且也根本没有办法把显示时间戳和日志时间戳完全对好。

Android 也意识到了这个问题，WinScope 的出现有效的帮助我们跟踪窗口和显示问题。它向开发者提供一个可视化的工具，让开发者能使用工具跟踪整个界面的变化过程。

### 怎么抓winscope相关文件：

![ada79e4c71f451a379a6402b877351aa\_MD5](https://picgo.myjojo.fun:666/i/2024/10/23/6718c038bc3e5.png)

![0c0fb505c0cacc95b434e4c25cbf2feb\_MD5](https://picgo.myjojo.fun:666/i/2024/10/23/6718c0397a409.png)

### 抓取winscope

把这里面的Winscope Trace开关打开\
这时候下拉状态栏多了它的图标\
![a15fcb85c816f2c7de34f661a7295f48\_MD5](https://picgo.myjojo.fun:666/i/2024/10/23/6718c03896f8e.png)

当我们需要开始抓取时候点击图标既可以\
![0b58bc48737067887a58811aed844cdb\_MD5](https://picgo.myjojo.fun:666/i/2024/10/23/6718c0381e3f1.png)

然后开始操作手机复现对应的bug现象，复现完毕则再点击图标关闭\
最后会再系统的如下路径生成对应的winsco[pe文件](https://so.csdn.net/so/search?q=pe%E6%96%87%E4%BB%B6\&spm=1001.2101.3001.7020)\
![f6ba2cc7bd71fcacb664e9d0a9dbd27a\_MD5](https://picgo.myjojo.fun:666/i/2024/10/23/6718c038c1880.png)

### 使用chrome浏览器加载观看winscope

打开使用配套源码[aosp](https://so.csdn.net/so/search?q=aosp\&spm=1001.2101.3001.7020)中的winscope的html文件\
文件路径如下：

```bash
/home/test/aosp/prebuilts/misc/common/winscope/winscope.html
```

![5c5fa61ec216c0d5ac583885915fade7\_MD5](https://picgo.myjojo.fun:666/i/2024/10/23/6718c03859f29.png)\
把这个winscope.html打开\
![7b704002fc25c9507bad82b1490dbbe3\_MD5](https://picgo.myjojo.fun:666/i/2024/10/23/6718c03863865.png)

然后把手机上的winscope抓取的文件pull到本地\
adb pull /data/misc/wmtrace\
再点击如下区域：\
![7effa5c5f55a5bccadb10d6ef9326d7f\_MD5](https://picgo.myjojo.fun:666/i/2024/10/23/6718c0381efee.png)\
选择对应文件，这里一般常用是SurfaceFlinger和Window的相关：\
![382fc9292276bc56f6d80906f43a0cb6\_MD5](https://picgo.myjojo.fun:666/i/2024/10/23/6718c038b2376.png)\
这里我们最常见的就是SurfaceFlinger和Window的分析\
选择后点击Submit

![97c33e018d89713d5ba4402d0e779a01\_MD5](https://picgo.myjojo.fun:666/i/2024/10/23/6718c0397639c.png)

然后就可以相当于对着录屏的每一帧图像看对应的surfaceflinger中各个layer的信息，相当于每一帧我们都可以又对应的dumpsys数据分析

### 4、原因寻找及解决办法

上面已经分析了bugreport的原理，实际是借助dumpstate来实现获取高权限root的，那么问题来了，为啥wmtrace相关文件夹呢？这个问题就得看dumpstate相关源码了：

frameworks/native/cmds/dumpstate/dumpstate.cpp\
看到了如下的代码：

```cpp
#define PSTORE_LAST_KMSG "/sys/fs/pstore/console-ramoops"
#define ALT_PSTORE_LAST_KMSG "/sys/fs/pstore/console-ramoops-0"
#define BLK_DEV_SYS_DIR "/sys/block"

#define RECOVERY_DIR "/cache/recovery"
#define RECOVERY_DATA_DIR "/data/misc/recovery"
#define UPDATE_ENGINE_LOG_DIR "/data/misc/update_engine_log"
#define LOGPERSIST_DATA_DIR "/data/misc/logd"
#define PREREBOOT_DATA_DIR "/data/misc/prereboot"
#define PROFILE_DATA_DIR_CUR "/data/misc/profiles/cur"
#define PROFILE_DATA_DIR_REF "/data/misc/profiles/ref"
#define XFRM_STAT_PROC_FILE "/proc/net/xfrm_stat"
#define WLUTIL "/vendor/xbin/wlutil"
#define WMTRACE_DATA_DIR "/data/misc/wmtrace"
#define OTA_METADATA_DIR "/metadata/ota"
#define SNAPSHOTCTL_LOG_DIR "/data/misc/snapshotctl_log"
#define LINKERCONFIG_DIR "/linkerconfig"
#define PACKAGE_DEX_USE_LIST "/data/system/package-dex-usage.list"
#define SYSTEM_TRACE_SNAPSHOT "/data/misc/perfetto-traces/bugreport/systrace.pftrace"
#define CGROUPFS_DIR "/sys/fs/cgroup"
```

可以看到有列出一个个的data相关目录，有RECOVERY\_DATA\_DIR和WMTRACE\_DATA\_DIR，这里重点看看WMTRACE\_DATA\_DIR为啥没有被导出看看是否有相关的限制条件：

看到了如下的代码，这里有一个条件就是!PropertiesHelper::IsUserBuild()\
即只有在非user手机才可以导出WMTRACE\_DATA\_DIR目录

```cpp
    /* Add window and surface trace files. */
    if (!PropertiesHelper::IsUserBuild()) {
        ds.AddDir(WMTRACE_DATA_DIR, false);
    }
```

修改方案探索：\
1、直接删除 if (!PropertiesHelper::IsUserBuild()) 条件（比较暴力不安全）

```cpp
   //if (!PropertiesHelper::IsUserBuild()) {
        ds.AddDir(WMTRACE_DATA_DIR, false);
  //  }
```

2、可以加一个或条件，加入自己的暗门（建议这种），比如自己也搞一个prop，可以通过adb shell改变的prop

```cpp
   if (!PropertiesHelper::IsUserBuild() || isEnableProp（）) {
        ds.AddDir(WMTRACE_DATA_DIR, false);
    }
```
