---
tags:
  - AMS
  - blog
  - Framework
categories:
  - AMS
  - Android
title: AMS   -- ActivityRecord、TaskRecord、ActivityStack、ActivityDisplay、ActivityStackSupervisor
date: 2024-09-16T04:32:47.976Z
lastmod: 2024-09-16T05:01:22.369Z
---
ActivityRecord、TaskRecord、ActivityStack、ActivityDisplay、ActivityStackSupervisor  的关系

# ActivityManagerService Activity栈管理

## ActivityRecord

记录Activity的信息，并通过成员变量task指向TaskRecord。

| 类型              | 名称            | 说明                 |
| --------------- | ------------- | ------------------ |
| ProcessRecord   | app           | 跑在哪个进程             |
| TaskRecord      | task          | 跑在哪个task           |
| ActivityInfo    | info          | Activity信息         |
| int             | mActivityType | Activity类型         |
| ActivityState   | state         | Activity状态         |
| ApplicationInfo | appInfo       | 跑在哪个app            |
| ComponentName   | realActivity  | 组件名                |
| String          | packageName   | 包名                 |
| String          | processName   | 进程名                |
| int             | launchMode    | 启动模式               |
| int             | userId        | 该Activity运行在哪个用户Id |

## TaskRecord

描述Activity的Affinity所属的栈。

| 类型                        | 名称              | 说明                                        |
| ------------------------- | --------------- | ----------------------------------------- |
| ActivityStack             | stack           | 当前所属的stack                                |
| ArrayList<ActivityRecord> | mActivities     | 当前task的所有Activity列表                       |
| int                       | taskId          | TaskRecord的Id                             |
| String                    | affinity        | root activity的affinity，即该Task中第一个Activity |
| int                       | mCallingUid     | 调用者的UserId                                |
| String                    | mCallingPackage | 调用者的包名                                    |

## ActivityStack

管理着TaskRecord，内部维护Activity所有状态、特殊状态的Activity和Activity相关的列表数据。

| 类型                       | 名称                   | 说明                                               |
| ------------------------ | -------------------- | ------------------------------------------------ |
| ArrayList<TaskRecord>    | mTaskHistory         | 保存所有的Task列表                                      |
| ArrayList<ActivityStack> | mStacks              | 所有的stack列表                                       |
| int                      | mStackId             | ActivityStackvisor的mActivityContainers的key值Id    |
| int                      | mDisplayId           | ActivityStackSupervisor的mActivityDisplays的key值Id |
| ActivityRecord           | mPauseingActivity    | 正在暂停的Activity                                    |
| ActivityRecord           | mLastPausedActivity  | 上一个已暂停的Activity                                  |
| ActivityRecord           | mResumedActivity     | 已经Resumed的Activity                               |
| ActivityRecord           | mLastStartedActivity | 最近一次启动的Activity                                  |

## ActivityStackSupervisor

管理所有的ActivityStack。

| 类型                             | 名称                  | 说明            |
| ------------------------------ | ------------------- | ------------- |
| ActivityStack                  | mHomeStack          | 桌面的stack      |
| ActivityStack                  | mFocusedStack       | 当前聚焦的stack    |
| ActivityStack                  | mLastFocusedStack   | 正在切换到聚焦的stack |
| SparseArray<ActivityDisplay>   | mActivityDisplays   | displayId为key |
| SparseArray<ActivityContainer> | mActivityContainers | mStackId为key  |

## ActivityDisplay

表示一个屏幕，Android支持三种屏幕：主屏幕，外接屏幕（HDMI等），虚拟屏幕（投屏）一般地，对于没有分屏功能以及虚拟屏的情况下，ActivityStackSupervisor与ActivityDisplay都是系统唯一；***ActivityDisplay主要有Home Stack和App Stack这两个栈。***

## 记忆关系链

每个ActivityStack中可以有若干个TaskRecord对象；每个TaskRecord中可以有若干个ActivityRecord对象；每个ActivityRecord记录一个Activity信息。 正向关系链表：

```java
java

 代码解读
复制代码ActivityStackSupervisor.mActivityDisplays
-> ActivityDisplay.mStack
-> ActivityStack.mTaskHistory
-> TaskRecord.mActivities
-> ActivityRecord
```

反向关系链

```java
java

 代码解读
复制代码ActivityRecord.task
-> TaskRecord.mStack
-> ActivityStack.mStackSupervisor
-> ActivityStackSupervisor
```

> ActivityStack.mDisplayId可以找到对应的ActivityDisplay，HOME\_STACK\_ID=0可以在ActivityStackSupervisor.mActivityDisplays找到桌面的ActivityStack。

![image.png](https://picgo.myjojo.fun:666/i/2024/09/16/66e7bb91a30eb.png)

![TaskRecord.webp](https://picgo.myjojo.fun:666/i/2024/09/16/66e7bb4e3cafe.webp)

引用：

作者：彭小铭\
链接：https://juejin.cn/post/7267554771540049957\
来源：稀土掘金
