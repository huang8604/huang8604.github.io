---
title: BugReport中的App Processor wakeup字段意义-CSDN博客
source: https://blog.csdn.net/su749520/article/details/142696344?spm=1001.2014.3001.5502
author: 
published: 
created: 2024-12-06
description: 
tags:
  - clippings
  - blog
date: 2024-12-06T07:23:57.989Z
lastmod: 2024-12-06T07:24:05.994Z
---
xt\_idletimer 的 uevent 消息与 IdlertimerController 有关，主要用来监视网络设备的收发工作状态。当对应设备工作或空闲时间超过设置的监控时间后【[wifi](https://so.csdn.net/so/search?q=wifi\&spm=1001.2101.3001.7020)网络默认15秒，数据网络默认15秒超时监测】， Kernel将会发送携带其状态(idle或active)的UEvent消息。
