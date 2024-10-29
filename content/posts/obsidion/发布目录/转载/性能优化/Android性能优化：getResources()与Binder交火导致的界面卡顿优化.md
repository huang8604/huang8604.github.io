---
title: Android性能优化：getResources()与Binder交火导致的界面卡顿优化
created: 2024-10-29
tags:
  - clippings
  - 转载
  - blog
  - 卡顿
  - Trace
  - Perfetto
collections:
  - 性能优化
source: https://juejin.cn/post/7198430801851531324
date: 2024-10-29T09:54:24.598Z
lastmod: 2024-10-29T10:00:12.255Z
---
\[TOC]

## 背景

某轮测试发现，我们的设备运行一个第三方的App时，卡顿感非常明显：

* 界面加载很慢，菊花转半天
* 滑屏极度不跟手，目测观感帧率低于15
* 对比机（竞品）也会稍微一点卡，但是好很多，基本不会有很大感觉的卡顿

可以初步判定我们的设备存在性能问题，亟需优化，拉平到竞品水准。

最后发现，这个问题实际上是应用自身奇怪的实现（getResources()的重载），加上Binder过度调用（沉重的Binder耗时）导致的。

本文做记录和分享。

其中对比机配置、Android版本均与本机不同，不做变量参考。

## 观测

由于这个可爱的App是第三方的App（应用市场下载的），我们没有源码，只能从系统端去干涉。先抓一份trace。

## 1. trace体现UI绘制操作严重耗时

trace一抓一看，显然App主线程已经陷入困境。可以看到：

1. CPU使用率并不高
2. 主线程几乎完全在执行Traversal工作（mersure和layout）
3. measure和layout极度耗时，显然达不到合理的帧率要求（甚至连PPT帧率都赶不上）

![ae7d5c5cd646867161b11074d1f80cc2\_MD5](https://picgo.myjojo.fun:666/i/2024/10/29/6720b21a122e0.png)

可以看到，这份trace表明App的整个measure和layout工作存在整体性的不合理耗时。但并不能准确提示细节，也不能看出问题部分。可以肯定，耗时工作位于App层（不是指耗时原因也来自App）。

## 2. 排查measure和layout慢的原因：可疑的多次binder

上面可以确认绘制缓慢造成耗时。但是一来App不是自己的，二来这么复杂的调用，通过分析调用、跟代码来定位慢方法、慢路径显然足够低效。

定位到Traversal，统计一下Traversal各部分的耗时占比，可以大致定位出耗时部分可能是什么业务的：

![8016512afd5d6f28ecaf2b55efd08492\_MD5](https://picgo.myjojo.fun:666/i/2024/10/29/6720b21b08c65.png)

可以看到，traversal意外地包含了数量巨大的binder调用，它占据总耗时的80%+，使得应用层绘图超出生命线10倍以上：

1. 这次doFrame->travesal耗时接近200ms，属于"无法使用的垃圾"级别，不是性能问题而是故障
2. binder调用（binder transaction）次数很多，在几毫秒的时间里（预期的一次应用层绘图时间）进行了194次IPC
3. binder耗时占比很高：83%左右
4. 还有一个ioctl调用次数也很多、很耗时；由于binder驱动调用talkWithDriver()需要使用ioctl，因此这里初步判断ioctl是binder IPC的伴生，无碍

> 生命线：对于60Hz的屏幕，生命线为16ms左右。但是16ms为图形栈全链路的极限时间，留给应用层的时间更低

可以确认，过多的binder调用导致了这个恼火的性能问题。

## 3. binder：在哪、谁为、为何频繁调用

通常应用（和应用集成的库），出于一定的目的，会通过IBinder、AIDL、封装组件（如startService）、直接调用驱动节点（talkWithDriver）等方式来进行一次Binder IPC。

性能问题中，与Binder IPC相关的，最常见的主要如下：

1. 频繁调用Binder
2. 关键、敏感、紧张的位置调用Binder
3. Binder对端响应太慢，对端繁忙
4. Binder传递的数据太大
5. Binder客户端线程数超限（发起请求的线程满）
6. Binder服务端线程数超限（处理请求的线程满）

对于Binder传递数据太大、线程数导致的性能问题，由于应用不是自己的（不好干涉、不关注），且对比机卡顿不那么明显（可以粗略排除），因此不太值得去看。（另外我们是在滑屏的时候卡的，主线程UIHandler也做不到并发发出Binder IPC）

这里还是展示一下怎么分析。下列命令可以提供一些关于binder状态、traction状态、传递数据大小等内容：

```bash
cat /sys/kernel/debug/binder/failed_transaction_log
cat /sys/kernel/debug/binder/transaction_log
cat /sys/kernel/debug/binder/transactions
```

同样的，我们不好关注应用为何调用binder（因为没有App的代码，最近也忙的不想逆向它；但实际上最后我们知道了为何调用），也很显然是在哪调用的（在App UI线程 performTraversal时调用的），因此先来看看这群IPC的对端是谁。

trace一看，binder调用确实很多（画蓝紫色线部分都是binder；本是细线，溢满则刚）：

![bf5681f8e0b6d0f5c4e44ae713989bba\_MD5](https://picgo.myjojo.fun:666/i/2024/10/29/6720b21a1f80e.png)

上图binder调用很多，其实很多是同一种类，各IPC都最终归属于一类Binder。分类看，数量巨大、占比最高的两类binder（称为第一部分binder和第二部分binder）是值得探讨的主要耗时部分。

首先，分析第一部分binder的对端。跟踪发现第一部分binder“飞”往SurafceFlinger，耗时较短，次数合理，评估正常，不再跟进，不贴图展示。

第二部分binder，从次数、耗时来看，确实可疑。它从App进程“飞”往System\_server（Framework服务层）：

![8da49406f41f97481ba87bbf09baf0a7\_MD5](https://picgo.myjojo.fun:666/i/2024/10/29/6720b21a31e97.png)

## 4. binder：频繁调用的具体定位

性能分析的其中一个关键方向是找到慢方法、慢路径。上面一步已经体现了，慢是因为App在敏感且关键的位置调用了Binder，这个binder的对端是Framework。

从系统侧分析这个binder的性能，难以像App那样轻松定位——因为App里面有多少个调用、系统里面暴露了多少个binder，在哪里触发的，都不好搞。

因此直接来粗暴的方法，把所有binder调用抓堆栈下来。

多次复现、多次抓取，阅读堆栈、总结分类，可以抓到蛛丝马迹。由于最长的堆栈高达33万行（包含合理的正常的binder和造成性能问题的binder），且抓了好几份，这里只能将问题的关键点做个展示输出。

```c
ls
20230209.fk.trace  binder.20230209.2.fk.trace.log  binder.20230209.4.fk.trace.log  binder.20230209.fk.trace.log  binder.20230210.1.fk.trace.log
20230209.ok.trace  binder.20230209.3.fk.trace.log  binder.20230209.5.fk.trace.log  binder.20230209.ok.trace.log
```

> fk表示fuck，即不正常情况下的binder堆栈；ok表示正常。

其中性能故障对应的堆栈如下（几类有性能问题的binder调用；仅截取关键位置）：

第一个堆栈放全一些，可以看出，在正常的traversal过程中，View体系正常调用getResources()，binder发生在getResources()内部：它调用了IWindowManager.getInitialDisplayDensity()，通过binder“飞”到system\_server：

```php
Count: 15
Trace: java.lang.Throwable
	at android.os.BinderProxy.transact(BinderProxy.java:547)
	at android.view.IWindowManager$Stub$Proxy.getInitialDisplayDensity(IWindowManager.java:3025)
	at java.lang.reflect.Method.invoke(Native Method)
	at refactor.common.base.FActivity.e5(FActivity.java:7)
	at refactor.common.base.FActivity.getResources(FActivity.java:7)
	at androidx.appcompat.widget.ContentFrameLayout.onMeasure(ContentFrameLayout.java:1)
	at android.view.View.measure(View.java:25597)
	at android.view.ViewGroup.measureChildWithMargins(ViewGroup.java:7114)
	at android.widget.LinearLayout.measureChildBeforeLayout(LinearLayout.java:1632)
	at android.widget.LinearLayout.measureVertical(LinearLayout.java:922)
	at android.widget.LinearLayout.onMeasure(LinearLayout.java:801)
	at android.view.View.measure(View.java:25597)
	at android.view.ViewGroup.measureChildWithMargins(ViewGroup.java:7114)
	at android.widget.FrameLayout.onMeasure(FrameLayout.java:331)
	at android.view.View.measure(View.java:25597)
	at android.view.ViewGroup.measureChildWithMargins(ViewGroup.java:7114)
	at android.widget.LinearLayout.measureChildBeforeLayout(LinearLayout.java:1632)
	at android.widget.LinearLayout.measureVertical(LinearLayout.java:922)
	at android.widget.LinearLayout.onMeasure(LinearLayout.java:801)
	at android.view.View.measure(View.java:25597)
	at android.view.ViewGroup.measureChildWithMargins(ViewGroup.java:7114)
	at android.widget.FrameLayout.onMeasure(FrameLayout.java:331)
	at com.android.internal.policy.DecorView.onMeasure(DecorView.java:763)
	at android.view.View.measure(View.java:25597)
	at android.view.ViewRootImpl.performMeasure(ViewRootImpl.java:3665)
	at android.view.ViewRootImpl.measureHierarchy(ViewRootImpl.java:2302)
	at android.view.ViewRootImpl.performTraversals(ViewRootImpl.java:2564)
	at android.view.ViewRootImpl.doTraversal(ViewRootImpl.java:2026)
	at android.view.ViewRootImpl$TraversalRunnable.run(ViewRootImpl.java:8469)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:972)
	at android.view.Choreographer.doCallbacks(Choreographer.java:796)
	at android.view.Choreographer.doFrame(Choreographer.java:731)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:957)
	at android.os.Handler.handleCallback(Handler.java:938)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loop(Looper.java:223)
	at android.app.ActivityThread.main(ActivityThread.java:8024)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:605)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:947)
```

其实这里已经能看出问题，并且看清问题的严重性了。可以说，tarversal阶段是一个App最紧张、最重要的阶段之一，在这个关键时间窗口内，还调用了binder通信这一不可靠的方法（IPC是不可预期的），对性能影响很大。

该应用的View实现喜欢在traversal阶段调用上述Binder，包括但不限于如下几个：

```php
Count: 5
Trace: java.lang.Throwable
	at android.os.BinderProxy.transact(BinderProxy.java:547)
	at android.view.IWindowManager$Stub$Proxy.getInitialDisplayDensity(IWindowManager.java:3025)
	at java.lang.reflect.Method.invoke(Native Method)
	at refactor.common.base.FActivity.e5(FActivity.java:7)
	at refactor.common.base.FActivity.getResources(FActivity.java:7)
	at android.widget.FrameLayout.onMeasure(FrameLayout.java:221)
	at android.view.View.measure(View.java:25597)
	at android.view.ViewGroup.measureChildWithMargins(ViewGroup.java:7114)
	at android.widget.LinearLayout.measureChildBeforeLayout(LinearLayout.java:1632)
	at android.widget.LinearLayout.measureVertical(LinearLayout.java:922)
	at android.widget.LinearLayout.onMeasure(LinearLayout.java:801)
	at android.view.View.measure(View.java:25597)
	at android.widget.LinearLayout.measureHorizontal(LinearLayout.java:1463)
	at android.widget.LinearLayout.onMeasure(LinearLayout.java:803)
	at android.view.View.measure(View.java:25597)
	...
```

```php
Count: 20
Trace: java.lang.Throwable
	at android.os.BinderProxy.transact(BinderProxy.java:547)
	at android.view.IWindowManager$Stub$Proxy.getInitialDisplayDensity(IWindowManager.java:3025)
	at java.lang.reflect.Method.invoke(Native Method)
	at refactor.common.base.FActivity.e5(FActivity.java:7)
	at refactor.common.base.FActivity.getResources(FActivity.java:7)
	at android.widget.LinearLayout.onMeasure(LinearLayout.java:762)
	at android.view.View.measure(View.java:25597)
	at android.widget.RelativeLayout.measureChild(RelativeLayout.java:849)
	at android.widget.RelativeLayout.onMeasure(RelativeLayout.java:652)
	at android.view.View.measure(View.java:25597)
	at android.view.ViewGroup.measureChildWithMargins(ViewGroup.java:7114)
	at android.widget.FrameLayout.onMeasure(FrameLayout.java:331)
	at androidx.appcompat.widget.ContentFrameLayout.onMeasure(ContentFrameLayout.java:21)
	at android.view.View.measure(View.java:25597)
    ...
```

到此，已经定位到了慢路径了：

* App喜欢在View的关键回调里面调用IPC，产生巨大的性能问题
* 在measure和traversal阶段不厌其烦地调用一个通常情况很少调用的接口getInitialDisplayDensity()
* View的加载阶段、traversal阶段，在measure、layout阶段因Binder IPC过度频繁触发了性能问题

## 结论

App在多个不同的View（及其子类ViewGroup和ViewGroup的子类们），在不合适的时机频繁调用了Binder，以很低的CPU占用，领先性地实现了很卡的效果。

虽然卡顿的贡献来自不同的View调用的同名Binder，这个binder却是同一个接口（不易变的getInitialDisplayDensity()，这意味着返回值可以被缓存下来并确保有效），而且触发的直接原因是同一个——App在Context.getResources()方法内部调用了这个binder，getResources()在App运行时会被频繁调用（尤其是View创建、绘制阶段）。

> Context.getResources()默认实现是直接返回mResources，但是会有可爱的人会override它（或通过优美的kotlin扩展函数），往里面塞入耗时的慢方法。

清晰的定位到了慢方法、卡顿根因后，还有一个残酷的问题：对比机不卡。

回答这个问题感觉像是对自己写出来的卡顿型代码有点欲盖弥彰的感觉。不过经过分析，排除掉竞品的优化、App在竞品的Android版本（Android版本和我们不一样）上业务逻辑不同、竞品的系统原生逻辑就不一样（Android版本原生逻辑差异）这三个变量因素后，结合代码阅读，发现我们的View Tree在Measure和Layout阶段，我们自己添加的功能会比原生要调用更多次的getResources()方法。

这在大多数情况下非常正常（逻辑上也正常，因为这个方法只有一行直接返回Resources对象实例的代码），碰到一个在超高频方法里面加慢调用、不可靠IPC的App后只能傻眼认栽。

## 方案

从App角度看，它错的很离谱。优化方案也很简单，去掉一个多余的、过度的Binder调用，一般是将调用集中在关键位置（临界区）以外、缓存返回值（确保返回值没有失效的前提下重用cache）、不在高频方法里面加东西、尽量不override sdk方法等等。

在系统侧，本着拉平甚至超越竞品的愿景，同样有减少binder调用的优化目标。不论是对应用自身问题的解决，还是对竞品的竞争性跟进，无所谓，都出手。主要有如下一些方案：

1. Framework可以实现缓存，在Binder IPC发出前检查有效性，仅在失效后真正发出IPC
2. 把我们加进去的额外的getResources()去掉、重构
3. 在严酷的竞争、高标准的要求下，会将考虑一些非标准的操作（魔改）

> 有效性，是指IPC对端返回的内容没有发生改变（本质上是软件维护的状态并未发生改变）。比如，从未旋转过屏幕，那么我们上一次获取的屏幕宽高就仍然有效，不需要再次获取，而应复用缓存

最终方案2采用并取得良好效果：帧率提升几倍、跟手了，还有一点卡卡的（App自己的+原生的getResources()调用），达到和竞品一致的水准了。

## 参考

1. [Context.getResources()源码参考](https://link.juejin.cn/?target=https%3A%2F%2Fcs.android.com%2Fandroid%2Fplatform%2Fsuperproject%2F%2B%2Fmaster%3Aframeworks%2Fbase%2Fcore%2Fjava%2Fandroid%2Fapp%2FContextImpl.java%3Fq%3DContextImpl.getResources "https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/app/ContextImpl.java?q=ContextImpl.getResources")
