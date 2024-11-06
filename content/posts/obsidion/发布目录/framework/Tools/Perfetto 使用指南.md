---
title: Perfetto 使用指南
author: 阿豪
created: 2024-10-18
tags:
  - clippings
  - 转载
  - 工具
  - blog
collections: 工具
source: https://mp.weixin.qq.com/s/ad-exk3ZCWPME6JbustR_w
date: 2024-11-06T06:57:56.792Z
lastmod: 2024-11-05T01:34:01.917Z
---
## 1. Perfetto 是什么？

Perfetto 是 google 从 Android10 开始引入的一个全新的平台级跟踪分析工具。它可以记录 Android 系统运行过程中的关键数据，并通过图形化的形式展示这些数据。Perfetto 不仅可用于系统级的性能分析，也是我们学习系统源码流程的好帮手。

Perfetto 算是Systrace的升级版本，可以直观的看到跨进程的调用方式。

## 2. 如何抓取 Trace

使用 Perfetto 一般分两步进行：

* 收集手机运行过程中的信息，这些信息通常称之为 Trace，收集的过程称之为抓取 Trace。
* 使用 Perfetto 打开 Trace，分析 Trace

本节介绍如何抓取 Trace 。

### 2.1 使用命令行抓取 Trace

#### 2.1.1 使用 perfetto 命令抓取

首先使用 usb 线将电脑和手机连接，确保 `adb shell` 命令能正常工作。

接着执行下面的命令：

```shell
adb shell perfetto -o /data/misc/perfetto-traces/trace_file.perfetto-trace t 20s \ 
sched freq idle am wm gfx view binder_driver hal dalvik camera input res memory
```

这个命令会启动一个 20 秒钟的跟踪，收集指定的数据源信息，并将跟踪文件保存到 `/data/misc/perfetto-traces/trace_file.perfetto-trace`  ,执行完会在后台执行。

最后，把 trace 文件 pull 出来：

```
adb pull /data/misc/perfetto-traces/trace_file.perfetto-trace
```

整个抓取过程就完成了。

也可以做成手机的内置的功能，执行shell命令来离线抓去trace。

#### 2.1.2 使用 record\_android\_trace 命令抓取

record\_android\_trace 是 Perfetto 团队提供的一个简化脚本，使得我们的抓取工作更加简单。record\_android\_trace 在源码路径下面也有。

```shell
curl -O https://raw.githubusercontent.com/google/perfetto/master/tools/record_android_tracechmod u+x record_android_trace
./record_android_trace -o trace_file.perfetto-trace -t 10s -b 64mb \ 
sched freq idle am wm gfx view binder_driver hal dalvik camera input res memory
```

record\_android\_trace 命令能自动处理好路径，抓取完成后自动打开 Trace 文件。

### 2.2 使用 UI 工具抓取

#### 2.2.1 Perfetto UI 抓取

Perfetto 也提供了图形化的工具来抓取 Trace。

该工具以网站的方式提供：*https://ui.perfetto.dev/#!/record*

打开后，第一步，完成基本的设置：

![14be9842a24628a0f3f6a6580e1aba4a\_MD5](https://picgo.myjojo.fun:666/i/2024/10/18/671213b7ae95e.png)

* 左侧 tab 栏，选择 Record new trace 选项。

* 使用 usb 线连接好手机和电脑后，选项好目标平台。

* 选择抓取的模式。

* Stop when full，抓取的 trace 达到设置的容量后就停止。

* Ring buffer，环形缓存，设置的容量满了后会被覆盖。

* Long Trace，任性，一直记录。

第二步，配置我们要抓取的内容：

![4168204c70695e02a742eb4785e754b3\_MD5](https://picgo.myjojo.fun:666/i/2024/10/18/671213fc924af.png)

箭头指向的内容全部选中，这部分主要是 App 相关的内容。

![82e01b3a7264c8ac5038fdd9d7bab939\_MD5](https://picgo.myjojo.fun:666/i/2024/10/18/6712145dc5501.png)

把调用栈选上，这对我们分析代码很有帮助。

最后点击，右上角的 `Start Recording` 就开始抓取了。

### 2.3 使用配置文件简化命令行抓取

在使用 Perfetto UI 抓取时，当我们配置好以后。

![d4dffdfeb318a54a093ccbb6b8135f5c\_MD5](https://picgo.myjojo.fun:666/i/2024/10/18/6712147cedee9.png)

进入到 `Recording command`，然后把右侧两个 `EOF` 之间的内容复制下来，保存在 config.pbtx 配置文件中。

接着就可以用这个配置文件来抓取 Trace 了：

```
./record_android_trace -c config.pbtx -o trace_file.perfetto-trace -t 10s -b 64mb
```

## 3. Perfetto 使用基础

### 3.1 进入 Perfetto Trace 界面

使用 record\_android\_trace 命令或者 Perfetto UI 抓取 Trace 后，会自动打开 Perfetto Trace 界面。

使用 perfetto 命令抓取 Trace 后，需要手动打开 Perfetto Trace 界面。打开的方法如下：

使用浏览器打开 https://ui.perfetto.dev/，界面如下：

![f8eeeade087c9011666e787e05601681\_MD5](https://picgo.myjojo.fun:666/i/2024/10/18/6712154a372e3.png)

可以把 trace 文件直接拖到浏览器中，也可以通过左上角的 `Open trace file` 打开 trace 文件。

### 3.2 Perfetto Trace 界面基本内容

Perfetto Trace 界面大致可分为 4 个区域：

![f431e40b92dd06021a9e5056464e17bc\_MD5](https://picgo.myjojo.fun:666/i/2024/10/18/67121568a29b5.png)

* 操作区，主要用到 Current Trace 下的几个选项：

* Show timeline ：显示当前 Trace，切到了别的界面之后，再点这个就会回到 Trace View 界面。

* Query：写 SQL 查询语句和查看执行结果的地方。

* Metrics：官方默认帮你写好的一些分析结果。

* Info and stats ：当前 Trace 和手机 App 的一些信息。

* 信息区：时间与搜索。

* Trace 内容区：图形化展示 Trace 的区域。

* 信息区：展示Trace 内容区中选择中的元素的详细信息。

在 Trace 内容区中，可以通过 w/s 按键缩小放大界面， a/d 移动界面。

Trace 内容区中主要有以下一些元素：

### 3.2.1 slice，片段

![8334ab145a6a4ccb13646715fe1a7fe6\_MD5](https://picgo.myjojo.fun:666/i/2024/10/18/67121645cbafe.png)

鼠标单击后会有一个黑框包围住，信息区会显示相关信息：

![531a7006d369ce8deba539bc8bca19c3\_MD5](https://picgo.myjojo.fun:666/i/2024/10/18/67121653b7038.png)

slice 代表了一段代码的执行过程，起于 `Trace.traceBegin \  `，终于 `Trace.traceEnd \ ATRACE_END`

```java
  Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");  
  AppBindData data = (AppBindData)msg.obj;  
  handleBindApplication(data);  
  Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
```

#### 线程状态查看

##### 深绿色 : 运行中（Running）

在Running状态就代表着处于[cpu](https://so.csdn.net/so/search?q=cpu\&spm=1001.2101.3001.7020)上的运行中\
状态作用：看某个方法是否耗时，可以通过测量Running时间长短判断，也可以进行竞品对比看看cpu能力如何，或者前后对比各个大小核cpu影响方法的耗时

![26035b00bc9bc4bb3e9658df3a3f8f42\_MD5](https://picgo.myjojo.fun:666/i/2024/10/29/6720a936d4558.png)

##### 浅绿色 : 可运行（Runnable）

代表线程可以运行但当前没有真正运行中，需要等待 cpu 调度，这个时间长短代表着cpu调度快慢\
**重要作用：点击Runnable这个块，下面信息会显示当前线程唤醒者是谁，即可以清楚知道整个线程之间唤醒逻辑。**

![b16cdfc5d92701ed3ad9e618f68a816e\_MD5](https://picgo.myjojo.fun:666/i/2024/10/29/6720a9383d0f3.png)

##### 白色/无色: 睡眠中（Sleeping）

代表当前线程没有工作可以做，等待[事件驱动](https://so.csdn.net/so/search?q=事件驱动\&spm=1001.2101.3001.7020)干货，比如looper就是大部分时间睡眠，小部分时间有消息后处理消息

![520824c7d42e49c3c06c60ae7aebd66b\_MD5](https://picgo.myjojo.fun:666/i/2024/10/29/6720a93ba652f.png)

##### 橙色Uninterruptible Sleep (IO)

代表不可以中断的休眠状态，一般线程在[IO操作](https://so.csdn.net/so/search?q=IO操作\&spm=1001.2101.3001.7020)上阻塞了\
不可中断状态实际上是系统对进程和硬件设备的一种保护机制。比如，当一个进程向磁盘读写数据时，为了保证数据的一致性，在得到磁盘回复前，它是不能被其他进程或者中断打断的。

![545576891abf020bad7bd16d051c95c8\_MD5](https://picgo.myjojo.fun:666/i/2024/10/29/6720a93cd6442.png)

##### 紫红色Uninterruptible Sleep (Non IO)

不可中断的休眠状态，非IO导致，在等内核锁。通常是低内存导致等待、各种各样的内核锁。

![7b2def33d51543a87ab926f359e0443a\_MD5](https://picgo.myjojo.fun:666/i/2024/10/29/6720a93c2c532.png)

Uninterruptible情况都可以点击后看到blocked方法是哪个

```
Blocked function   jbd2_log_wait_commit
```

### 3.2.2 counter，计数器

用于记录一些关键数据。

![96a000924935d095ee6aaf7efd29576c\_MD5](https://picgo.myjojo.fun:666/i/2024/10/18/67121780e45eb.png)

![fecfd5f864712f49a4e1b12904dc6c63\_MD5](https://picgo.myjojo.fun:666/i/2024/10/18/6712185cba4e3.png)

对应于代码中的 `Trace.traceCounter/ATRACE_INT`：

```
    ATRACE_INT(ftl::Concat("HW_VSYNC_", displayIdOpt->value).c_str(),    displayData.vsyncTraceToggle);
```

### 3.2.3 CPU Sched Slice， cpu 调度片段

用于展示 cpu 的调度情况。

![9a7921b120e22cd98620b4a8bbfec902\_MD5](https://picgo.myjojo.fun:666/i/2024/10/18/6712188122854.png)

### 3.2.4 thread\_state，线程状态

点击片段上方线程调度信息片段(Running)，可以看到线程当前运行在哪个CPU上。

![71cb37357dd944ef0727f3b304d56339\_MD5](https://picgo.myjojo.fun:666/i/2024/10/18/6712189c122fb.png)

![1855f4b4fc05bda7df0b4eca44eb2a40\_MD5](https://picgo.myjojo.fun:666/i/2024/10/18/671218b219c23.png)

点击信息区中的 `Running on CPU 7` 旁边的斜箭头：

![85f359462dfb6ed688bad374e631aca9\_MD5](https://picgo.myjojo.fun:666/i/2024/10/18/67121b18e7ebc.png)

可以在 CPU 调度中看到该运行片段。

![164ea040fd0a573aca7e17d486dc9d70\_MD5](https://picgo.myjojo.fun:666/i/2024/10/18/67121b248d8ec.png)

再次点击斜箭头，可以回到原来位置。

这里的 thread\_state 实际是我们的主线程，由用户点击屏幕唤醒运行，实际很多线程都是由其他线程/进程唤醒的，比如在 CPU 调度中选择一个 Slice：

![6d28a7173a13532d4c5bcf438682d7a8\_MD5](https://picgo.myjojo.fun:666/i/2024/10/18/67121b57a58f3.png)

这里的意思是当前 thread 由 P（Process）`/system/bin/surfaceflinger [584]` 中的 T（Thread）`app [689]` 唤醒。

线程从就绪到运行延迟了 48us 381ns

### 3.3 Perfetto Trace 界面基本操作

W : 放大 perfetto , 放大可以更好地看清局部细节\
S : 缩小 perfetto, 缩小以查看整体\
A : 左移\
D : 右移\
M : 高亮选中当前鼠标点击的段

F：快速放大到选择位置

SHIFT + 鼠标 可以拖动

用得最多的其实就是添加标记。

当我们选中一个片段时，点击 m，就可以做一个临时的标记，当标记另一个片段以后，前一个临时标记就会取消。

选中一个片段以后，如果点击 `shift + m`，就会添加一个普通标记。

标记以后，会出现两个小三角：

![a39994dcc7880ce89c75dc8f5b1f1ff8\_MD5](https://picgo.myjojo.fun:666/i/2024/10/18/67121d3198f40.png)

点击小三角后，信息区会出现一个 remove 按键：

![0d5aa3ecaff5b4ae5586642e819cfe2d\_MD5](https://picgo.myjojo.fun:666/i/2024/10/18/67121d4397b76.png)

点击即可取消。

##### 3.4 跨进程通讯的发起端与接受端跳转

![b18ce5a621a2d8b948b44b536a0e428c\_MD5](https://picgo.myjojo.fun:666/i/2024/10/29/6720a93d5ae91.png)

发起端如下，也可以直接点击跳转到接收端

![e63b2f2ff4e66ff2b51414f4572f6ff9\_MD5](https://picgo.myjojo.fun:666/i/2024/10/29/6720a93c11c4d.png)

## 4. 如何添加trace Log

### 4.1 使用atrace相关类方法介绍

这里还需要看对应的trace源码是最全面的\
路径：system/core/libcutils/include/cutils/trace.h

```java
/*
 * Copyright (C) 2012 The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

#ifndef _LIBS_CUTILS_TRACE_H
#define _LIBS_CUTILS_TRACE_H

#include <inttypes.h>
#include <stdatomic.h>
#include <stdbool.h>
#include <stdint.h>
#include <stdio.h>
#include <sys/cdefs.h>
#include <sys/types.h>
#include <unistd.h>
#include <cutils/compiler.h>

__BEGIN_DECLS

/**
 * The ATRACE_TAG macro can be defined before including this header to trace
 * using one of the tags defined below.  It must be defined to one of the
 * following ATRACE_TAG_* macros.  The trace tag is used to filter tracing in
 * userland to avoid some of the runtime cost of tracing when it is not desired.
 *
 * Defining ATRACE_TAG to be ATRACE_TAG_ALWAYS will result in the tracing always
 * being enabled - this should ONLY be done for debug code, as userland tracing
 * has a performance cost even when the trace is not being recorded.  Defining
 * ATRACE_TAG to be ATRACE_TAG_NEVER or leaving ATRACE_TAG undefined will result
 * in the tracing always being disabled.
 *
 * ATRACE_TAG_HAL should be bitwise ORed with the relevant tags for tracing
 * within a hardware module.  For example a camera hardware module would set:
 * #define ATRACE_TAG  (ATRACE_TAG_CAMERA | ATRACE_TAG_HAL)
 *
 * Keep these in sync with frameworks/base/core/java/android/os/Trace.java.
 */
#define ATRACE_TAG_NEVER            0       // This tag is never enabled.
#define ATRACE_TAG_ALWAYS           (1<<0)  // This tag is always enabled.
#define ATRACE_TAG_GRAPHICS         (1<<1)
#define ATRACE_TAG_INPUT            (1<<2)
#define ATRACE_TAG_VIEW             (1<<3)
#define ATRACE_TAG_WEBVIEW          (1<<4)
#define ATRACE_TAG_WINDOW_MANAGER   (1<<5)
#define ATRACE_TAG_ACTIVITY_MANAGER (1<<6)
#define ATRACE_TAG_SYNC_MANAGER     (1<<7)
#define ATRACE_TAG_AUDIO            (1<<8)
#define ATRACE_TAG_VIDEO            (1<<9)
#define ATRACE_TAG_CAMERA           (1<<10)
#define ATRACE_TAG_HAL              (1<<11)
#define ATRACE_TAG_APP              (1<<12)
#define ATRACE_TAG_RESOURCES        (1<<13)
#define ATRACE_TAG_DALVIK           (1<<14)
#define ATRACE_TAG_RS               (1<<15)
#define ATRACE_TAG_BIONIC           (1<<16)
#define ATRACE_TAG_POWER            (1<<17)
#define ATRACE_TAG_PACKAGE_MANAGER  (1<<18)
#define ATRACE_TAG_SYSTEM_SERVER    (1<<19)
#define ATRACE_TAG_DATABASE         (1<<20)
#define ATRACE_TAG_NETWORK          (1<<21)
#define ATRACE_TAG_ADB              (1<<22)
#define ATRACE_TAG_VIBRATOR         (1<<23)
#define ATRACE_TAG_AIDL             (1<<24)
#define ATRACE_TAG_NNAPI            (1<<25)
#define ATRACE_TAG_RRO              (1<<26)
#define ATRACE_TAG_THERMAL          (1 << 27)
#define ATRACE_TAG_LAST             ATRACE_TAG_THERMAL

// Reserved for initialization.
#define ATRACE_TAG_NOT_READY        (1ULL<<63)

#define ATRACE_TAG_VALID_MASK ((ATRACE_TAG_LAST - 1) | ATRACE_TAG_LAST)

#ifndef ATRACE_TAG
#define ATRACE_TAG ATRACE_TAG_NEVER
#elif ATRACE_TAG > ATRACE_TAG_VALID_MASK
#error ATRACE_TAG must be defined to be one of the tags defined in cutils/trace.h
#endif

/**
 * Opens the trace file for writing and reads the property for initial tags.
 * The atrace.tags.enableflags property sets the tags to trace.
 * This function should not be explicitly called, the first call to any normal
 * trace function will cause it to be run safely.
 */
void atrace_setup();

/**
 * If tracing is ready, set atrace_enabled_tags to the system property
 * debug.atrace.tags.enableflags. Can be used as a sysprop change callback.
 */
void atrace_update_tags();

/**
 * Set whether tracing is enabled for the current process.  This is used to
 * prevent tracing within the Zygote process.
 */
void atrace_set_tracing_enabled(bool enabled);

/**
 * This is always set to false. This forces code that uses an old version
 * of this header to always call into atrace_setup, in which we call
 * atrace_init unconditionally.
 */
extern atomic_bool atrace_is_ready;

/**
 * Set of ATRACE_TAG flags to trace for, initialized to ATRACE_TAG_NOT_READY.
 * A value of zero indicates setup has failed.
 * Any other nonzero value indicates setup has succeeded, and tracing is on.
 */
extern uint64_t atrace_enabled_tags;

/**
 * Handle to the kernel's trace buffer, initialized to -1.
 * Any other value indicates setup has succeeded, and is a valid fd for tracing.
 */
extern int atrace_marker_fd;

/**
 * atrace_init readies the process for tracing by opening the trace_marker file.
 * Calling any trace function causes this to be run, so calling it is optional.
 * This can be explicitly run to avoid setup delay on first trace function.
 */
#define ATRACE_INIT() atrace_init()
#define ATRACE_GET_ENABLED_TAGS() atrace_get_enabled_tags()

void atrace_init();
uint64_t atrace_get_enabled_tags();

/**
 * Test if a given tag is currently enabled.
 * Returns nonzero if the tag is enabled, otherwise zero.
 * It can be used as a guard condition around more expensive trace calculations.
 */
#define ATRACE_ENABLED() atrace_is_tag_enabled(ATRACE_TAG)
static inline uint64_t atrace_is_tag_enabled(uint64_t tag)
{
    return atrace_get_enabled_tags() & tag;
}

/**
 * Trace the beginning of a context.  name is used to identify the context.
 * This is often used to time function execution.
 */
#define ATRACE_BEGIN(name) atrace_begin(ATRACE_TAG, name)
static inline void atrace_begin(uint64_t tag, const char* name)
{
    if (CC_UNLIKELY(atrace_is_tag_enabled(tag))) {
        void atrace_begin_body(const char*);
        atrace_begin_body(name);
    }
}

/**
 * Trace the end of a context.
 * This should match up (and occur after) a corresponding ATRACE_BEGIN.
 */
#define ATRACE_END() atrace_end(ATRACE_TAG)
static inline void atrace_end(uint64_t tag)
{
    if (CC_UNLIKELY(atrace_is_tag_enabled(tag))) {
        void atrace_end_body();
        atrace_end_body();
    }
}

/**
 * Trace the beginning of an asynchronous event. Unlike ATRACE_BEGIN/ATRACE_END
 * contexts, asynchronous events do not need to be nested. The name describes
 * the event, and the cookie provides a unique identifier for distinguishing
 * simultaneous events. The name and cookie used to begin an event must be
 * used to end it.
 */
#define ATRACE_ASYNC_BEGIN(name, cookie) \
    atrace_async_begin(ATRACE_TAG, name, cookie)
static inline void atrace_async_begin(uint64_t tag, const char* name,
        int32_t cookie)
{
    if (CC_UNLIKELY(atrace_is_tag_enabled(tag))) {
        void atrace_async_begin_body(const char*, int32_t);
        atrace_async_begin_body(name, cookie);
    }
}

/**
 * Trace the end of an asynchronous event.
 * This should have a corresponding ATRACE_ASYNC_BEGIN.
 */
#define ATRACE_ASYNC_END(name, cookie) atrace_async_end(ATRACE_TAG, name, cookie)
static inline void atrace_async_end(uint64_t tag, const char* name, int32_t cookie)
{
    if (CC_UNLIKELY(atrace_is_tag_enabled(tag))) {
        void atrace_async_end_body(const char*, int32_t);
        atrace_async_end_body(name, cookie);
    }
}

/**
 * Trace the beginning of an asynchronous event. In addition to the name and a
 * cookie as in ATRACE_ASYNC_BEGIN/ATRACE_ASYNC_END, a track name argument is
 * provided, which is the name of the row where this async event should be
 * recorded. The track name, name, and cookie used to begin an event must be
 * used to end it.
 */
#define ATRACE_ASYNC_FOR_TRACK_BEGIN(track_name, name, cookie) \
    atrace_async_for_track_begin(ATRACE_TAG, track_name, name, cookie)
static inline void atrace_async_for_track_begin(uint64_t tag, const char* track_name,
                                                const char* name, int32_t cookie) {
    if (CC_UNLIKELY(atrace_is_tag_enabled(tag))) {
        void atrace_async_for_track_begin_body(const char*, const char*, int32_t);
        atrace_async_for_track_begin_body(track_name, name, cookie);
    }
}

/**
 * Trace the end of an asynchronous event.
 * This should correspond to a previous ATRACE_ASYNC_FOR_TRACK_BEGIN.
 */
#define ATRACE_ASYNC_FOR_TRACK_END(track_name, name, cookie) \
    atrace_async_for_track_end(ATRACE_TAG, track_name, name, cookie)
static inline void atrace_async_for_track_end(uint64_t tag, const char* track_name,
                                              const char* name, int32_t cookie) {
    if (CC_UNLIKELY(atrace_is_tag_enabled(tag))) {
        void atrace_async_for_track_end_body(const char*, const char*, int32_t);
        atrace_async_for_track_end_body(track_name, name, cookie);
    }
}

/**
 * Trace an instantaneous context. name is used to identify the context.
 *
 * An "instant" is an event with no defined duration. Visually is displayed like a single marker
 * in the timeline (rather than a span, in the case of begin/end events).
 *
 * By default, instant events are added into a dedicated track that has the same name of the event.
 * Use atrace_instant_for_track to put different instant events into the same timeline track/row.
 */
#define ATRACE_INSTANT(name) atrace_instant(ATRACE_TAG, name)
static inline void atrace_instant(uint64_t tag, const char* name) {
    if (CC_UNLIKELY(atrace_is_tag_enabled(tag))) {
        void atrace_instant_body(const char*);
        atrace_instant_body(name);
    }
}

/**
 * Trace an instantaneous context. name is used to identify the context.
 * track_name is the name of the row where the event should be recorded.
 *
 * An "instant" is an event with no defined duration. Visually is displayed like a single marker
 * in the timeline (rather than a span, in the case of begin/end events).
 */
#define ATRACE_INSTANT_FOR_TRACK(trackName, name) \
    atrace_instant_for_track(ATRACE_TAG, trackName, name)
static inline void atrace_instant_for_track(uint64_t tag, const char* track_name,
                                            const char* name) {
    if (CC_UNLIKELY(atrace_is_tag_enabled(tag))) {
        void atrace_instant_for_track_body(const char*, const char*);
        atrace_instant_for_track_body(track_name, name);
    }
}

/**
 * Traces an integer counter value.  name is used to identify the counter.
 * This can be used to track how a value changes over time.
 */
#define ATRACE_INT(name, value) atrace_int(ATRACE_TAG, name, value)
static inline void atrace_int(uint64_t tag, const char* name, int32_t value)
{
    if (CC_UNLIKELY(atrace_is_tag_enabled(tag))) {
        void atrace_int_body(const char*, int32_t);
        atrace_int_body(name, value);
    }
}

/**
 * Traces a 64-bit integer counter value.  name is used to identify the
 * counter. This can be used to track how a value changes over time.
 */
#define ATRACE_INT64(name, value) atrace_int64(ATRACE_TAG, name, value)
static inline void atrace_int64(uint64_t tag, const char* name, int64_t value)
{
    if (CC_UNLIKELY(atrace_is_tag_enabled(tag))) {
        void atrace_int64_body(const char*, int64_t);
        atrace_int64_body(name, value);
    }
}

__END_DECLS

#endif // _LIBS_CUTILS_TRACE_H


```

### 4.2 比如常见的几个方法：

#### 4.2.1 ATRACE\_CALL()

这个方法，代表对一个function的开头和结尾进行tag，它的源码如下\
\#define ATRACE\_CALL() ATRACE\_NAME(**FUNCTION**)\
即本质调用的是ATRACE\_NAME

#### 4.2.2 ATRACE\_NAME方法

调用如下：

![f41d5bab5ad75815f82682d93be190be\_MD5](https://picgo.myjojo.fun:666/i/2024/10/29/6720a93b6639d.png)

效果如下：

![b762eb06df9418207b5208ae07ebf7b0\_MD5](https://picgo.myjojo.fun:666/i/2024/10/29/6720a93adec33.png)

#### 4.2.3 trace相关实战使用：

1、引入相关的头文件及相关库\
\#include \<utils/Trace.h>\
编译时候还得考率相关utils的lib是否引入\
如这里以截图代码为例就有引入libutils\
frameworks/base/cmds/screencap/Android.bp

```java
cc_binary {
    name: "screencap",

    srcs: ["screencap.cpp"],

    shared_libs: [
        "libcutils",
        "libutils",
        "libbinder",
        "libjnigraphics",
        "libui",
        "libgui",
    ],

    cflags: [
        "-Wall",
        "-Werror",
        "-Wunused",
        "-Wunreachable-code",
    ],
}


```

2、定义好相关的TAG

```
 #undef ATRACE_TAG
#define ATRACE_TAG ATRACE_TAG_GRAPHICS

```

这里就选了一个ATRACE\_TAG\_GRAPHICS即graphics这个组

3、调用相关actrace方法\
ATRACE\_CALL\
ATRACE\_NAME\
一般针对整个方法或者单独作用域，ATRACE\_CALL和ATRACE\_NAME本质没有区别，只是把名字变成了function的name

![2fd48ec7a226024f796c4c848f61b39f\_MD5](https://picgo.myjojo.fun:666/i/2024/10/29/6720a93b8bc0a.png)

具体效果如下：

![e8251e04016f6f68000afbd66587e411\_MD5](https://picgo.myjojo.fun:666/i/2024/10/29/6720a93a5a949.png)

优点：只需要一个调用既可以实现tag打印，自动跟随作用域进行tag的结束\
缺点：需要自己熟练掌握好作用域，没有可以手动控制的tag结束的点

4、相关灵活控制开始与结束的trace方法\
直接使用atrace\_begin和atrace\_end方法，使用如下：

```java
GraphicBuffer::GraphicBuffer()
    : BASE(), mOwner(ownData), mBufferMapper(GraphicBufferMapper::get()),
      mInitCheck(NO_ERROR), mId(getUniqueId()), mGenerationNumber(0)
{
    ATRACE_NAME("GraphicBuffer::GraphicBuffer()1");
    atrace_begin(ATRACE_TAG, "GraphicBuffer 1");
    width  =
    height =
    stride =
    format =
    usage_deprecated = 0;
    atrace_end(ATRACE_TAG);
    usage  = 0;
    layerCount = 0;
    handle = nullptr;
}

```

同时也可以使用对应[宏定义](https://so.csdn.net/so/search?q=宏定义\&spm=1001.2101.3001.7020)：\
ATRACE\_BEGIN(name);\
ATRACE\_END();

![a888b7da3d14319a6175179e79f8cab9\_MD5](https://picgo.myjojo.fun:666/i/2024/10/29/6720a93ad7d83.png)![197ff0830a077cc075b59917cf894bb4\_MD5](https://picgo.myjojo.fun:666/i/2024/10/29/6720a93c82d24.png)

执行后效果如下：

![a947d73a381cd5c0c3a4a81f214b54d8\_MD5](https://picgo.myjojo.fun:666/i/2024/10/29/6720a93a3c766.png)

### 4.3 java端的trace打印

app层面时候也有trace打印方式

```java
  Trace.traceBegin(Trace.TRACE_TAG_ALWAYS,"aaaaa");//开始trace 字符为aaaaa
  Trace.traceEnd(Trace.TRACE_TAG_ALWAYS);//结束trace

```

源码如下：\
frameworks/base/core/java/android/os/Trace.java

```java


    /**
     * Writes a trace message to indicate that a given section of code has
     * begun. Must be followed by a call to {@link #traceEnd} using the same
     * tag.
     *
     * @param traceTag The trace tag.
     * @param methodName The method name to appear in the trace.
     *
     * @hide
     */
    @UnsupportedAppUsage
    @SystemApi(client = MODULE_LIBRARIES)
    public static void traceBegin(long traceTag, @NonNull String methodName) {
        if (isTagEnabled(traceTag)) {
            nativeTraceBegin(traceTag, methodName);
        }
    }

    /**
     * Writes a trace message to indicate that the current method has ended.
     * Must be called exactly once for each call to {@link #traceBegin} using the same tag.
     *
     * @param traceTag The trace tag.
     *
     * @hide
     */
    @UnsupportedAppUsage
    @SystemApi(client = MODULE_LIBRARIES)
    public static void traceEnd(long traceTag) {
        if (isTagEnabled(traceTag)) {
            nativeTraceEnd(traceTag);
        }
    }

```

具体使用方法：

```java

  protected void onStart() {
        Trace.traceBegin(Trace.TRACE_TAG_ALWAYS,"Settings Home onStart()");
        ((SettingsApplication) getApplication()).setHomeActivity(this);
        super.onStart();
        doAidlHalCall();
        Trace.traceEnd(Trace.TRACE_TAG_ALWAYS);
    }

```

## 参考资料

* perfetto 官方文档
* Android Perfetto 系列 1：Perfetto 工具简介
* Android Perfetto 系列 3：熟悉 Perfetto View
* Perfetto分析进阶
* perfetto/systrace基础知识讲解
* [使用 Perfetto 分析你的 Android 应用性能](https://mp.weixin.qq.com/s?__biz=Mzg5MzYxNTI5Mg==\&mid=2247494899\&idx=1\&sn=7fe08dadfddc5298216070e3e8d31a3a\&poc_token=HOaO2maj5PTkR2nB_uZVUnXaUUib2E6IlSfsU7Q_\&scene=21#wechat_redirect)
