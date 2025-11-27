---
title: FH20GMS-120    [VTS][MR1]vts_generic_boot_image_test - GMS - Confluence
author: []
created: 2025-10-29
tags:
  - clippings
  - 转载
  - blog
collections: 图形显示
source: 
date: 2025-11-21T08:01:29.000Z
lastmod: 2025-11-21T08:01:29.000Z
---
## FH20GMS-120 \[VTS]\[MR1]vts\_generic\_boot\_image\_test

![image.png](https://picgo.myjojo.fun:666/i/2025/10/29/690173b22ea59.png)

从报错信息来看ramdisk缺少 system/bin/e2fsck，但设备上是有这个bin文件的，如下：

![image.png](https://picgo.myjojo.fun:666/i/2025/10/29/690173ff685dc.png)

Ramdisk（内存磁盘）是Android启动过程中的一个关键组件，它是一个临时根文件系统，在启动初期被加载到内存中，用于初始化系统并挂载真正的根文件系统。

## Ramdisk的作用

1. **初始化系统环境** ：
   * 运行init进程，它是所有进程的父进程。
   * 解析init.rc脚本，设置系统属性、创建设备节点、挂载文件系统等。
2. **挂载必要的文件系统** ：
   * 挂载system、vendor、data等分区。
   * 设置文件系统的权限和SELinux安全上下文。
3. **启动关键系统服务** ：
   * 启动守护进程（如ueventd、logd等）。
   * 准备Android运行时环境。
4. **过渡到真正的根文件系统** ：
   * 在Android系统中，ramdisk通常用于启动到recovery模式或作为boot镜像的一部分，在正常启动时，它会将控制权交给system分区。

## Ramdisk在Android启动流程中的位置

1. **Bootloader加载内核和ramdisk** ：Bootloader将内核和ramdisk镜像加载到内存中。
2. **内核启动** ：内核解压ramdisk并将其作为根文件系统挂载。
3. **init进程启动** ：ramdisk中的init程序开始执行，解析init.rc文件。
4. **执行启动脚本** ：根据init.rc中的命令，挂载系统分区、启动服务等。
5. **切换根文件系统** ：在某些配置中，ramdisk可能会被切换为system作为根文件系统（例如在system-as-root配置中）。

## Ramdisk的内容

一个典型的ramdisk包含以下内容：

* `init` ：初始化程序。
* `init.rc` ：初始化脚本。
* `system/` 、 `vendor/` 、 `data/` 等目录的挂载点。
* 一些必要的工具和库（如 `e2fsck` 用于检查文件系统）。
* 安全策略文件（如SELinux策略）。

## 调试Ramdisk问题

当遇到ramdisk相关的问题时（如缺少文件、多余文件等），可以：

1. **检查构建配置** ：确保构建类型（user/userdebug/eng）正确，并且没有意外包含调试文件。
2. **检查设备配置** ：在设备Makefile中，确保只包含了必要的包和文件。
3. **检查GKI要求** ：如果使用GKI，确保ramdisk符合通用内核映像的要求。

## 总结

Ramdisk是Android启动过程中的一个临时根文件系统，它负责初始化系统并挂载真正的文件系统。确保ramdisk包含正确的文件和不包含多余的文件对于系统正常启动和通过测试（如GKI测试）至关重要。

![d74ff1170c203a7a03ad4e03fdf1830a.png](https://picgo.myjojo.fun:666/i/2025/10/29/6901744a5b4a4.png)

参考附件文档，可以移除e2fsck，修改如下\
![image.png](https://picgo.myjojo.fun:666/i/2025/10/29/6901741f53a4c.png)

这个问题就是google 利用手里gms的杠杆来撬动厂家按照指挥棒来运动. 通过修改VTS的条件,让厂家必须根据VTS的要求来做
