---
title: 动态篡改 so 函数返回值：一篇带你玩转 Android Hook 技术！
author:
  - "[[CYRUS STUDIO]]"
created: 2025-08-26
tags:
  - clippings
  - 转载
  - blog
collections: 图形显示
source: https://mp.weixin.qq.com/s/3bK7o01fgT_P0jOgSrviVg
date: 2025-08-26T00:48:00.203Z
lastmod: 2025-08-26T12:05:04.141Z
---
*2025年08月26日 08:35*

以下文章来源于CYRUS STUDIO ，作者CYRUS STUDIO

\[

**CYRUS STUDIO**.

Android 逆向｜聚焦移动安全、攻防对抗 | 分享 Frida / Native 层分析 / 加壳脱壳 / 加密算法还原等方向内容｜欢迎关注、交流、合作

]\(https://mp.weixin.qq.com/s/#)

> 版权归作者所有，如有转发，请注明文章出处：https://cyrus-studio.github.io/blog/

## 前言

第三方 SDK 中的 so ，VMP壳 + OLLVM 混淆。

使用 frida 获取到目标 native 函数信息如下：

```
------------ [ #1 Native Method ] ------------
Method Name     : public static final native boolean ****************************************.ca(android.content.Context)
ArtMethod Ptr   : 0x7cc748a128
Native Addr     : 0x7cb77c5834
Module Name     : libhexymsb.so
Module Offset   : 0x09C8
Module Base     : 0x7cb7788000
Module Size     : 335872 bytes
Module Path     : /data/app/com.demotestapp.test-B17GBv-5lpTxH5GABufXfw==/lib/arm64/libhexymsb.so
------------------------------------------------
```

ca 函数在 libhexymsb.so 偏移 0x09C8 的位置。

需求：修改 so 中 ca 函数的返回值，固定为 true。

## 获取 so 模块基址

由于 ca 函数是动态注册的，所有没有符号信息，只能通过函数地址去 hook，那么就需要先得到目标 so 的基址。

可以通过遍历所有已加载的动态库（使用 dl\_iterate\_phdr），匹配库名后返回其基址。

dl\_iterate\_phdr 是 Linux（包括 Android）系统中提供的一个函数，用于 遍历当前进程加载的所有 ELF 动态模块（共享库 / 可执行文件），从而获取它们的加载地址、段信息等。

它的定义（在 \<link.h> 中）如下：

```
int dl_iterate_phdr(int (*callback)(struct dl_phdr_info *info, size_t size, void *data), void *data);
```

**参数说明：**

* • callback：每加载一个模块（比如某个.so 文件），就会调用一次这个回调函数，传入当前模块的信息。
* • size：dl\_phdr\_info 结构的大小。
* • data：你传入给 dl\_iterate\_phdr 的用户数据指针，会在回调时传给你，通常用于传参或传出信息。

struct dl\_phdr\_info 结构常用字段：

```
struct dl_phdr_info {
    const char *dlpi_name;      // 模块路径名（可能为空）
    ElfW(Addr) dlpi_addr;       // 加载地址（基址 base address）
    const ElfW(Phdr) *dlpi_phdr;// Program Header 表的起始地址
    ElfW(Half) dlpi_phnum;      // Program Header 表的数量
    // ... 还有其他字段
};
```

相关源码链接：

* • https://cs.android.com/android/platform/superproject/+/android-latest-release:external/musl/include/link.h;l=21
* • https://cs.android.com/android/platform/superproject/+/android-latest-release:external/musl/include/link.h;l=47

具体代码实现如下：

```
/**
 * @brief 获取指定共享库（.so 文件）的加载基址。
 *
 * @param libname 要查找的共享库名称的子串，例如 "libtarget.so"。
 *                支持模糊匹配（通过 strstr），无需提供完整路径。
 *
 * @return void* 返回找到的库的加载基地址（dlpi_addr），
 *               如果未找到匹配的库，则返回 nullptr。
 *
 * @example
 *     void* base = get_library_base("libexample.so");
 *     if (base) {
 *         printf("libexample.so 加载基址为: %p\n", base);
 *     }
 */
void *get_library_base(const char *libname) {
    // 定义一个回调数据结构，用于在遍历时传递库名和保存找到的基地址
    struct callback_data {
        const char *libname;
        void *base;
    } data = {libname, nullptr};

    // 使用 dl_iterate_phdr 遍历所有已加载的动态库，传递一个匿名函数（lambda）处理 item
    dl_iterate_phdr( {

        // 将传入的 data 指针强制转换为 callback_data 类型
        auto *cb = (callback_data *) data;

        // 判断当前库名中是否包含目标库名
        if (info->dlpi_name && strstr(info->dlpi_name, cb->libname)) {
            // 找到匹配库，保存其基地址
            cb->base = (void *) info->dlpi_addr;

            // 返回 1 表示停止迭代（即已经找到了目标库）
            return 1;
        }

        // 继续查找其他库
        return 0;
    }, &data);

    // 返回找到的基址
    return data.base;
}
```

## 编写 Hook 代码

## 1. CMakeLists.txt

编辑 CMakeLists.txt 增加一个自定义的 hook 库

```
find_package(shadowhook REQUIRED CONFIG)

add_library(
        # 设置库的名称
        sohooker

        # 设置库的类型
        SHARED

        # 设置源文件路径
        sohooker.cpp
)

target_link_libraries(
        sohooker
        # 链接 log 库
        ${log-lib}
        # 链接 shadowhook
        shadowhook::shadowhook
)
```

这里使用的是 ShadowHook，具体参考： 搞懂 Android Hook 的两大核心：PLT Hook 与 Inline Hook 全解析 <sup><span>\[1]</span></sup>

## 2. 实现函数劫持与返回值替换

通过 get\_library\_base 函数先获取到目标 so 的基址，基址 + 函数偏移 得到函数内存中的地址。

再通过 ShadowHook 的 shadowhook\_hook\_func\_addr 函数对函数地址进行 Hook，将其替换为一个总是返回 true 的 fake\_ca 函数，从而实现篡改原函数逻辑的效果；

它在库加载时（通过构造函数 **attribute** ((constructor))）自动执行，并支持不同架构（如 arm64 和 armeabi-v7a）的地址适配。

具体代码实现如下：

```
#include <jni.h>
#include <android/log.h>
#include "shadowhook.h"
#include <link.h>
#include <string.h>

#define TAG "sohooker"
#define LOGI(...) __android_log_print(ANDROID_LOG_INFO, TAG, __VA_ARGS__)

// 原始函数指针类型
typedef bool (*orig_ca_func_t)(JNIEnv *, jclass, jobject);

static orig_ca_func_t orig_ca_func = nullptr;

// 替换函数
static bool fake_ca(JNIEnv *env, jclass clazz, jobject context) {
    LOGI("🚀 fake_ca called, always return true");
    return true;
}

void print_arch_info() {
#if defined(__aarch64__)
    LOGI("Current architecture: arm64-v8a");
#elif defined(__arm__)
    LOGI("Current architecture: armeabi-v7a");
#elif defined(__i386__)
    LOGI("Current architecture: x86");
#elif defined(__x86_64__)
    LOGI("Current architecture: x86_64");
#else
    LOGI("Unknown architecture");
#endif
}

// 在库加载时自动执行
__attribute__((constructor)) static void init_hook() {
    // 初始化 ShadowHook
    shadowhook_init(SHADOWHOOK_MODE_UNIQUE, true);  // debug 开启日志

    // 获取 so 基址
    void *base = get_library_base("libhexymsb.so");
    if (!base) {
        LOGI("❌ libhexymsb.so not loaded yet");
        return;
    }

    uintptr_t target_addr = 0;

    // 适配不同的架构
#ifdef __aarch64__
    // arm64-v8a
    target_addr = (uintptr_t) base + 0x09C8; // 函数内存地址 = so 基址 + 函数偏移
#else
    // armeabi-v7a
    target_addr = (uintptr_t) base + 0x0610;
#endif

    // 打印当前设备架构信息
    print_arch_info();

    LOGI("🎯 hooking address: %p", (void *) target_addr);

    // Hook
    shadowhook_hook_func_addr(
            (void *) target_addr,             // target
            (void *) fake_ca,                 // replacement
            (void **) &orig_ca_func           // backup
    );
}
```

## 注入自定义 Hook 库

只需要在 java 层添加如下代码，加载 sohooker 动态库就能实现篡改目标 so 函数的返回值。

```
System.loadLibrary("sohooker")
```

## 运行测试与验证

当没有加载 sohooker 时，调用 native 方法返回结果为 false。

![word/media/image1.png](https://mp.weixin.qq.com/s/www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

word/media/image1.png

当加载 sohooker 后，再调用 native 方法结果返回 true

![word/media/image2.png](https://mp.weixin.qq.com/s/www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

word/media/image2.png

从日志可以看到 ca 函数已经成功被替换为 fake\_ca 函数

![word/media/image3.png](https://mp.weixin.qq.com/s/www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

word/media/image3.png

## 完整源码

示例代码地址：https://github.com/CYRUS-STUDIO/AndroidExample

#### 引用链接

`[1]` 搞懂 Android Hook 的两大核心：PLT Hook 与 Inline Hook 全解析: *https://cyrus-studio.github.io/blog/posts/%E6%90%9E%E6%87%82-android-hook-%E7%9A%84%E4%B8%A4%E5%A4%A7%E6%A0%B8%E5%BF%83plt-hook-%E4%B8%8E-inline-hook-%E5%85%A8%E8%A7%A3%E6%9E%90/*

![图片](https://mp.weixin.qq.com/s/www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**扫一扫** 关注我的公众号

如果你想要跟大家分享你的文章，欢迎投稿~

┏(＾0＾)┛明天见！

继续滑动看下一个

鸿洋

向上滑动看下一个
