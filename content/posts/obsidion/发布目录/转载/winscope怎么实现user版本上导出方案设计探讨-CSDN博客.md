---
title: winscope怎么实现user版本上导出方案设计探讨
author: 
created: 2024-10-24
tags:
  - clippings
  - 转载
  - blog
collections:
  - 图形显示
source: https://blog.csdn.net/learnframework/article/details/132823800?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522D742FA8A-B63E-4062-84B5-57BFDECFE060%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=D742FA8A-B63E-4062-84B5-57BFDECFE060&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_ecpm_v1~rank_v31_ecpm-2-132823800-null-null.nonecase&utm_term=Winscope&spm=1018.2226.3001.4450
date: 2024-10-24T09:44:28.692Z
lastmod: 2024-10-24T09:46:47.353Z
---
### 背景

发现有一个问题，那就发现在user版本的手机设备上发现无法抓取相关的winscope，哪怕可以抓取也发现没办法导出来分析。

### 1、user手机上网页获取winscope失败

winscope在user手机上的效果如下：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/b36f2d31f0981e6f06da67a0dd3a9f13.png#pic_center)可以看出一直是显示个error，但是具体是啥原因error呢？\
从服务端的python程序输出日日志看看：\
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/1e0d22c5f019507b9e27b872fc5c870a.png#pic_center)

明显看到其实服务端也只是去执行相关的[adb](https://so.csdn.net/so/search?q=adb\&spm=1001.2101.3001.7020) shell命令，只不过这个命令需要有su root这样高级别的权限。自然在user手机上是没有的

### 2、手机上可以抓取，但是无权限获取文件

快捷按钮可以去setting中放开\
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/059a7e25be5f19f24aa9dbb4f465fb07.png#pic_center)

抓取完成后有相关的trace文件：\
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/92295f9a3ef0ba3fa9038b4f2905efc4.png#pic_center)

可以看到trace文件被导出到了 data/misc/wmtrace文件夹，那么尝试取出\
发现有如下权限问题\
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/27cc3901cf095ba7fff71b59fbff7498.png#pic_center)

可以看出这个wmtrace文件夹压根没有权限哈，自然无法取出

### 3、尝试探索解决办法

使用bugreoport命令：

```cpp
test@test:~$ adb bugreport
/data/user_de/0/com.android.shell/files/bugreports/bugreport-crosshatch-SP1A.210812.016.C1-2022-06-28-12-21-13.zip: 1 file pulled. 27.7 MB/s (11790205 bytes in 0.406s)
test@test:~$ adb pull /data/user_de/0/com.android.shell/files/bugreports/bugreport-crosshatch-SP1A.210812.016.C1-2022-06-28-12-21-13.zip 
/data/user_de/0/com.android.shell/files/bugreports/bugreport-crosshatch-SP1A.210812.016.C1-2022-06-28-12-21-13.zip: 1 file pulled. 27.7 MB/s (11790205 bytes in 0.406s)
```

看看这个bugreport命令导出的相关文件：\
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/b1045aab12f7f7fd85d0ab543e470a14.png)

发现FS文件夹下面有个data的文件夹，还有misc，因为本身misc根本adb shell是没有权限可以查看的，看着是不是很有希望。。\
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/3f045d767d754e3498798bfefba8bb4a.png#pic_center)\
但是情况却如下：\
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/a32b9dceba25752d92383ad7972bfca9.png#pic_center)\
只有recovery相关，没有看到wmtrace相关文件夹

但是明明就是/data/misc/wmtrace路径，为啥没有导出呢？\
需要知道这个原因就必须要看源码了\
首先需要了解点bugreport其实本质上也最多只能有shell权限，因为也属于adb shell拉起的进程，\
但是为啥它可以导出data/misc/下面相关文件夹

为了解密这个我们可以来看看相关源码\
frameworks/native/cmds/bugreport/bugreport.cpp

```cpp
int main() {
    fprintf(stderr,
            "=============================================================================\n");
    fprintf(stderr, "WARNING: Flat (text file, non-zipped) bugreports are deprecated.\n");
    fprintf(stderr, "WARNING: Please generate zipped bugreports instead.\n");
    fprintf(stderr, "WARNING: On the host use: adb bugreport filename.zip\n");
    fprintf(stderr, "WARNING: On the device use: bugreportz\n");
    fprintf(stderr, "WARNING: bugreportz will output the filename to use with adb pull.\n");
    fprintf(stderr,
            "=============================================================================\n\n\n");

    return 0;
}
```

可以看出的这里其实bugreport已经是一个空壳了，真正还是bugreportz在起作用\
看看bugreportz的相关命令：\
frameworks/native/cmds/bugreportz/main.cpp

```cpp
int main(int argc, char* argv[]) {
  //省略

    // TODO: code below was copy-and-pasted from bugreport.cpp (except by the
    // timeout value);
    // should be reused instead.

    // Start the dumpstatez service.
    //启动相关的 dumpstate服务
    if (stream_data) {
        property_set("ctl.start", "dumpstate");
    } else {
        property_set("ctl.start", "dumpstatez");
    }

    // Socket will not be available until service starts.
    int s = -1;
    for (int i = 0; i < 20; i++) {
    //与dumpstate进行本地socket跨进程通讯
            s = socket_local_client("dumpstate", ANDROID_SOCKET_NAMESPACE_RESERVED, SOCK_STREAM);
        if (s >= 0) break;
        // Try again in 1 second.
        sleep(1);
    }

    int ret;
    //socket接受相关的数据进行处理
    if (stream_data) {
        ret = bugreportz_stream(s);
    } else {
        ret = bugreportz(s, show_progress);
    }
    return ret;
}
```

总结如下图所示：\
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/79278aeaa1c928cb86b62f572f196c94.png#pic_center)

验证方式：

在执行bugreport命令时候：

```bash
test@test:~/aosp/frameworks/native/cmds$ adb bugreport
[  5%] generating bugreport-crosshatch-SP1A.210812.016.C1-2022-06-28-13-02-19.zip
```

开另一个在终端进行adb shell查看一下阿dumpstate服务是啥权限：

```bash
crosshatch:/ $ ps -A | grep dump                                                                        
root         16137     1 10878140  5172 0                   0 S dumpstate
```

明显看到是一个root权限的进程

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
