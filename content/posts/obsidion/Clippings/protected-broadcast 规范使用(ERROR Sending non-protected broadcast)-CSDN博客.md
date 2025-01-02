---
title: "protected-broadcast 规范使用(ERROR: Sending non-protected broadcast)-CSDN博客"
source: https://blog.csdn.net/shift_wwx/article/details/82350455
author: 
published: 
created: 2024-12-26
description: 
tags:
  - clippings
  - blog
date: 2024-12-26T06:21:41.359Z
lastmod: 2024-12-26T06:21:56.245Z
---
文章出处：[https://blog.csdn.net/shift\_wwx/article/details/82350455](https://blog.csdn.net/shift_wwx/article/details/82350455 "https://blog.csdn.net/shift_wwx/article/details/82350455")

请转载的朋友标明出处，请支持原创~~

相关博文：

[Android基础总结之五：BroadcastReceiver](https://blog.csdn.net/shift_wwx/article/details/9283179 "Android基础总结之五：BroadcastReceiver")

[Android 中broadcast 注册过程解析](https://blog.csdn.net/shift_wwx/article/details/81223021 "Android 中broadcast 注册过程解析")

[Android 中broadcast 发送过程解析](https://blog.csdn.net/shift_wwx/article/details/81227435 "Android 中broadcast 发送过程解析")

[protected-broadcast 规范使用(ERROR: Sending non-protected broadcast)](https://blog.csdn.net/shift_wwx/article/details/82350455?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522162815029116780366591111%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D\&request_id=162815029116780366591111\&biz_id=0\&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_v2~rank_v29-1-82350455.pc_v2_rank_blog_default\&utm_term=Broadcast\&spm=1018.2226.3001.4450 "protected-broadcast 规范使用(ERROR: Sending non-protected broadcast)")

先来看下log：

```cobol
11-05 00:00:09.609   688  3933 W ContextImpl: Calling a method in the system process without a qualified user: android.app.ContextImpl.sendBroadcast:966 com.android.server.pm.PackageManagerService.sendPermissionAccessBroadcast:5429 com.android.server.pm.PackageManagerService.checkUidPermission:5436 android.app.ActivityManager.checkUidPermission:3981 com.android.server.am.UidRecord.updateHasInternetPermission:113 11-05 00:00:09.610   688  3933 E ActivityManager: Sending non-protected broadcast action_shift_permission_access_log from system 688:system/1000 pkg android11-05 00:00:09.610   688  3933 E ActivityManager: java.lang.Throwable11-05 00:00:09.610   688  3933 E ActivityManager: 	at com.android.server.am.ActivityManagerService.checkBroadcastFromSystem(ActivityManagerService.java:19280)11-05 00:00:09.610   688  3933 E ActivityManager: 	at com.android.server.am.ActivityManagerService.broadcastIntentLocked(ActivityManagerService.java:19885)11-05 00:00:09.610   688  3933 E ActivityManager: 	at com.android.server.am.ActivityManagerService.broadcastIntent(ActivityManagerService.java:20027)11-05 00:00:09.610   688  3933 E ActivityManager: 	at android.app.ContextImpl.sendBroadcast(ContextImpl.java:970)11-05 00:00:09.610   688  3933 E ActivityManager: 	at com.android.server.pm.PackageManagerService.sendPermissionAccessBroadcast(PackageManagerService.java:5429)11-05 00:00:09.610   688  3933 E ActivityManager: 	at com.android.server.pm.PackageManagerService.checkUidPermission(PackageManagerService.java:5436)11-05 00:00:09.610   688  3933 E ActivityManager: 	at android.app.ActivityManager.checkUidPermission(ActivityManager.java:3981)11-05 00:00:09.610   688  3933 E ActivityManager: 	at com.android.server.am.UidRecord.updateHasInternetPermission(UidRecord.java:113)11-05 00:00:09.610   688  3933 E ActivityManager: 	at com.android.server.am.ActivityManagerService.addProcessNameLocked(ActivityManagerService.java:6910)11-05 00:00:09.610   688  3933 E ActivityManager: 	at com.android.server.am.ActivityManagerService.newProcessRecordLocked(ActivityManagerService.java:12517)11-05 00:00:09.610   688  3933 E ActivityManager: 	at com.android.server.am.ActivityManagerService.startProcessLocked(ActivityManagerService.java:3791)11-05 00:00:09.610   688  3933 E ActivityManager: 	at com.android.server.am.ActivityManagerService.startProcessLocked(ActivityManagerService.java:3706)11-05 00:00:09.610   688  3933 E ActivityManager: 	at com.android.server.am.BroadcastQueue.processNextBroadcast(BroadcastQueue.java:1359)11-05 00:00:09.610   688  3933 E ActivityManager: 	at com.android.server.am.ActivityManagerService.finishReceiver(ActivityManagerService.java:20131)11-05 00:00:09.610   688  3933 E ActivityManager: 	at android.app.IActivityManager$Stub.onTransact(IActivityManager.java:283)11-05 00:00:09.610   688  3933 E ActivityManager: 	at com.android.server.am.ActivityManagerService.onTransact(ActivityManagerService.java:2971)11-05 00:00:09.610   688  3933 E ActivityManager: 	at android.os.Binder.execTransact(Binder.java:697)
```

这是我在一次开发中出现的，系统中需要发送一个应用自定义的广播，send 之后会报出Sending non-protected [broadcast](https://so.csdn.net/so/search?q=broadcast\&spm=1001.2101.3001.7020) 的异常。

借此机会来解析protected broadcast 的使用，我们在 [Android 中broadcast 发送过程解析](https://blog.csdn.net/shift_wwx/article/details/81227435 "Android 中broadcast 发送过程解析") 中了解了broadcast 发送的整个过程，通过Context 的接口最终会调用到[AMS](https://so.csdn.net/so/search?q=AMS\&spm=1001.2101.3001.7020) 中broadcastIntent()。

## 1. protected broadcast 必须要特殊uid

```java
if (!isCallerSystem) {if (isProtectedBroadcast) {String msg = "Permission Denial: not allowed to send broadcast "                        + action + " from pid="                        + callingPid + ", uid=" + callingUid;                Slog.w(TAG, msg);throw new SecurityException(msg);            }
```

这段code 比较简单，如果isProtectedBroadcast 为true，即该广播为protected broadcast，那么变量isCallerSystem 不能为false。即，要求如果是protected broadcast 必须要特殊的uid，不然会丢出SecurityException，这特殊uid 指的是：

```java
switch (UserHandle.getAppId(callingUid)) {case ROOT_UID:case SYSTEM_UID:case PHONE_UID:case BLUETOOTH_UID:case NFC_UID:                isCallerSystem = true;break;default:                isCallerSystem = (callerApp != null) && callerApp.persistent;break;        }
```

ROOT\_UID、SYSTEM\_UID、PHONE\_UID、BLUETOOTH\_UID、NFC\_UID 或者是persistent 的app都可以让变量isCallerSystem 为true，躲开这个SecurityException。

##

## 2. 如果通过系统发送广播，广播必须是protected broadcast

```java
if (isCallerSystem) {                checkBroadcastFromSystem(intent, callerApp, callerPackage, callingUid,                        isProtectedBroadcast, registeredReceivers);            }
```

```java
private void checkBroadcastFromSystem(Intent intent, ProcessRecord callerApp,            String callerPackage, int callingUid, boolean isProtectedBroadcast, List receivers) {if ((intent.getFlags() & Intent.FLAG_RECEIVER_FROM_SHELL) != 0) {return;        }final String action = intent.getAction();if (isProtectedBroadcast                || Intent.ACTION_CLOSE_SYSTEM_DIALOGS.equals(action)                || Intent.ACTION_DISMISS_KEYBOARD_SHORTCUTS.equals(action)                || Intent.ACTION_MEDIA_BUTTON.equals(action)                || Intent.ACTION_MEDIA_SCANNER_SCAN_FILE.equals(action)                || Intent.ACTION_SHOW_KEYBOARD_SHORTCUTS.equals(action)                || Intent.ACTION_MASTER_CLEAR.equals(action)                || Intent.ACTION_FACTORY_RESET.equals(action)                || AppWidgetManager.ACTION_APPWIDGET_CONFIGURE.equals(action)                || AppWidgetManager.ACTION_APPWIDGET_UPDATE.equals(action)                || LocationManager.HIGH_POWER_REQUEST_CHANGE_ACTION.equals(action)                || TelephonyIntents.ACTION_REQUEST_OMADM_CONFIGURATION_UPDATE.equals(action)                || SuggestionSpan.ACTION_SUGGESTION_PICKED.equals(action)                || AudioEffect.ACTION_OPEN_AUDIO_EFFECT_CONTROL_SESSION.equals(action)                || AudioEffect.ACTION_CLOSE_AUDIO_EFFECT_CONTROL_SESSION.equals(action)) {return;        }if (receivers != null && receivers.size() > 0                && (intent.getPackage() != null || intent.getComponent() != null)) {boolean allProtected = true;for (int i = receivers.size()-1; i >= 0; i--) {Object target = receivers.get(i);if (target instanceof ResolveInfo) {ResolveInfo ri = (ResolveInfo)target;if (ri.activityInfo.exported && ri.activityInfo.permission == null) {                        allProtected = false;break;                    }                } else {BroadcastFilter bf = (BroadcastFilter)target;if (bf.requiredPermission == null) {                        allProtected = false;break;                    }                }            }if (allProtected) {return;            }        }if (callerApp != null) {            Log.wtf(TAG, "Sending non-protected broadcast " + action                            + " from system " + callerApp.toShortString() + " pkg " + callerPackage,new Throwable());        } else {            Log.wtf(TAG, "Sending non-protected broadcast " + action                            + " from system uid " + UserHandle.formatUid(callingUid)                            + " pkg " + callerPackage,new Throwable());        }    }
```

通过code 可以看到这个标题不是很正确，条件要求该广播除了是protected broadcast，也可以是：

```java
                || Intent.ACTION_CLOSE_SYSTEM_DIALOGS.equals(action)                || Intent.ACTION_DISMISS_KEYBOARD_SHORTCUTS.equals(action)                || Intent.ACTION_MEDIA_BUTTON.equals(action)                || Intent.ACTION_MEDIA_SCANNER_SCAN_FILE.equals(action)                || Intent.ACTION_SHOW_KEYBOARD_SHORTCUTS.equals(action)                || Intent.ACTION_MASTER_CLEAR.equals(action)                || Intent.ACTION_FACTORY_RESET.equals(action)                || AppWidgetManager.ACTION_APPWIDGET_CONFIGURE.equals(action)                || AppWidgetManager.ACTION_APPWIDGET_UPDATE.equals(action)                || LocationManager.HIGH_POWER_REQUEST_CHANGE_ACTION.equals(action)                || TelephonyIntents.ACTION_REQUEST_OMADM_CONFIGURATION_UPDATE.equals(action)                || SuggestionSpan.ACTION_SUGGESTION_PICKED.equals(action)                || AudioEffect.ACTION_OPEN_AUDIO_EFFECT_CONTROL_SESSION.equals(action)                || AudioEffect.ACTION_CLOSE_AUDIO_EFFECT_CONTROL_SESSION.equals(action)
```

这里标题之所以这样标注，主要是为了突出protected（保护），指的是对于普通的广播进行的保护。

上面code 看到除了code 中指定的这些特殊的action，以及系统AndroidManifest.xml 中特殊规定的protected broadcast（详见framewors/base/core/res/AndroidManifest.xml）：

```cobol
<protected-broadcast android:name="android.intent.action.SCREEN_OFF" />                             <protected-broadcast android:name="android.intent.action.SCREEN_ON" /> 
```

通过code 知道如果想不抛出 exception，那变量allProtected 必须为true 并返回，那对于普通的广播要求：

* **必须指明package**

```java
if (receivers != null && receivers.size() > 0                && (intent.getPackage() != null || intent.getComponent() != null)) {
```

要求广播发送的package 必须指出来，或者直接将package 、class 做成component 。

* **静态广播的exported为true的时候必须同时加上permission限制**

```java
if (ri.activityInfo.exported && ri.activityInfo.permission == null) {            allProtected = false;break;        }
```

要求exported 为true的时候，必须有permission 进行限制。

如果exported 为false，那当然不会有影响。

* **动态广播的filter 中必须指定permission**

```cobol
        BroadcastFilter bf = (BroadcastFilter)target;if (bf.requiredPermission == null) {            allProtected = false;            break;        }
```

综合，对于不是系统中指定的广播，需要要满足上面3个条件，这就是标题中说到的保护。

## **3. 总结**

* protected 广播必须是特殊uid 才能发送
* 系统发送的广播如果是protected 广播，可以正常发送
* 系统code 中的特殊广播，可以正常发送
* 对于静态注册的普通广播，要求exported为true 的时候必须指定广播要求的permission进行限制
* 对于动态注册的普通广播，要求广播必须指定permission
* 对于普通广播，在满足上面3、4点要求的同时，该广播的package 或者component 必须指定
