---
title: 案例分享：大家可能看到过这样的...-知识星球
source: https://wx.zsxq.com/group/48844241141518/topic/8858484285821152
author: 
published: 
created: 2024-12-06
description: 
tags:
  - clippings
  - blog
  - 转载
date: 2024-12-06T01:03:03.227Z
lastmod: 2024-12-06T01:05:43.975Z
---
案例分享：大家可能看到过这样的 Trace，App 的主线程和渲染线程，一个 Vsync 周期里面有两个 draw，对应的 RenderThread 就有两个 DrawFrame。 原因：这是因为这个 App 有多个 Window 同时更新内容导致的，每个 draw 对应一个 Window。这种情况下其实每一帧的负载会比较高，很容易出现 Buffer 阻塞的情况，RenderThread 阻塞导致 UI Thread 也阻塞，更容易出现卡顿问题。 对应的，两个 Window 就有两个出图源，对应的 SurfaceFlinger 这边就会有两个 BufferTX。这就导致在分析卡顿问题的时候，光看 SurfaceFlinger 主线程是否掉帧是不行的（因为两个 BufferTX 只要有一个是 Ready 的，那么 SF 主线程就不会出现掉帧），得看对应的每个 BufferTX 是否掉帧。这个是需要注意的。 [#案例分析](https://wx.zsxq.com/tags/%E6%A1%88%E4%BE%8B%E5%88%86%E6%9E%90/51122255482144) [#流畅性](https://wx.zsxq.com/tags/%E6%B5%81%E7%95%85%E6%80%A7/15558454111412)

展开全部

![eb50188e4b27114ffce473fa6fc3ae7b\_MD5](https://picgo.myjojo.fun:666/i/2024/12/06/67524de53fef6.jpg)
