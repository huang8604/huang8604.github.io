---
tags:
  - blog
  - Framework
  - AMS
categories:
  - 学习总结
  - AMS
collections:
  - AMS
title: APP  启动过程
date: 2024-08-08T10:07:11.000Z
lastmod: 2024-09-18T10:23:06.442Z
---
### AMS中Activity栈管理

### WMS中app的添加过程

#### windows 层级树

![1725347707811.png](https://picgo.myjojo.fun:666/i/2024/09/03/66d6b7785aa4c.png)

#### 添加Task 下 windowstate的堆栈

添加DefaultTaskDisplayArea

```shell
02-23 14:36:19.116  565 I test2   : DefaultTaskDisplayArea@243571827 addChild child = Task{a13d730 #1 type=home ?? U=0 visible=false visibleRequested=false mode=undefined translucent=true sz=0}
02-23 14:36:19.116  565 I test2   : java.lang.Exception
02-23 14:36:19.116  565 I test2   : 	at com.android.server.wm.WindowContainer.addChild(WindowContainer.java:727)
02-23 14:36:19.116  565 I test2   : 	at com.android.server.wm.TaskDisplayArea.addChildTask(TaskDisplayArea.java:334)
02-23 14:36:19.116  565 I test2   : 	at com.android.server.wm.TaskDisplayArea.addChild(TaskDisplayArea.java:320)
02-23 14:36:19.116  565 I test2   : 	at com.android.server.wm.Task$Builder.build(Task.java:6551)
02-23 14:36:19.116  565 I test2   : 	at com.android.server.wm.TaskDisplayArea.createRootTask(TaskDisplayArea.java:1066)
02-23 14:36:19.116  565 I test2   : 	at com.android.server.wm.TaskDisplayArea.createRootTask(TaskDisplayArea.java:1040)
02-23 14:36:19.116  565 I test2   : 	at com.android.server.wm.TaskDisplayArea.getOrCreateRootHomeTask(TaskDisplayArea.java:1640)
02-23 14:36:19.116  565 I test2   : 	at com.android.server.wm.RootWindowContainer.setWindowManager(RootWindowContainer.java:1321)
02-23 14:36:19.116  565 I test2   : 	at com.android.server.wm.ActivityTaskManagerService.setWindowManager(ActivityTaskManagerService.java:1006)
02-23 14:36:19.116  565 I test2   : 	at com.android.server.am.ActivityManagerService.setWindowManager(ActivityManagerService.java:1923)
02-23 14:36:19.116  565 I test2   : 	at com.android.server.SystemServer.startOtherServices(SystemServer.java:1595)
02-23 14:36:19.116  565 I test2   : 	at com.android.server.SystemServer.run(SystemServer.java:939)
02-23 14:36:19.116  565 I test2   : 	at com.android.server.SystemServer.main(SystemServer.java:649)
02-23 14:36:19.116  565 I test2   : 	at java.lang.reflect.Method.invoke(Native Method)
```

添加Task

```shell
 14:36:22.676  Task{a13d730 #1 type=home ?? U=0 visible=true visibleRequested=false mode=fullscreen translucent=true sz=0} addChild Comparator child = Task{63f31d4 #2 type=undefined A=1000:com.android.settings.FallbackHome U=0 visible=false quested=false mode=undefined translucent=true sz=0}
 14:36:22.676  java.lang.Exception
 14:36:22.676  	at com.android.server.wm.WindowContainer.addChild(WindowContainer.java:694)
 14:36:22.676  	at com.android.server.wm.Task.addChild(Task.java:5935)
 14:36:22.676  	at com.android.server.wm.Task.-$$Nest$maddChild(Unknown Source:0)
 14:36:22.676  	at com.android.server.wm.Task$Builder.build(Task.java:6548)
 14:36:22.676  	at com.android.server.wm.Task.reuseOrCreateTask(Task.java:5819)
 14:36:22.676  	at com.android.server.wm.ActivityStarter.setNewTask(ActivityStarter.java:2872)
 14:36:22.676  	at com.android.server.wm.ActivityStarter.startActivityInner(ActivityStarter.java:1864)
 14:36:22.676  	at com.android.server.wm.ActivityStarter.startActivityUnchecked(ActivityStarter.java:1661)
 14:36:22.676  	at com.android.server.wm.ActivityStarter.executeRequest(ActivityStarter.java:1216)
 14:36:22.676  	at com.android.server.wm.ActivityStarter.execute(ActivityStarter.java:702)
 14:36:22.676  	at com.android.server.wm.ActivityStartController.startHomeActivity(ActivityStartController.java:179)
 14:36:22.676  	at com.android.server.wm.RootWindowContainer.startHomeOnTaskDisplayArea(RootWindowContainer.java:1493)
 14:36:22.676  	at com.android.server.wm.RootWindowContainer.lambda$startHomeOnDisplay$12$com-android-server-wm-RootWindowContainer(RootWindowContainer.java:1434)
 14:36:22.676  	at com.android.server.wm.RootWindowContainer$$ExternalSyntheticLambda7.apply(Unknown Source:16)
 14:36:22.676  	at com.android.server.wm.TaskDisplayArea.reduceOnAllTaskDisplayAreas(TaskDisplayArea.java:513)
 14:36:22.676  	at com.android.server.wm.DisplayArea.reduceOnAllTaskDisplayAreas(DisplayArea.java:404)
 14:36:22.676  	at com.android.server.wm.DisplayArea.reduceOnAllTaskDisplayAreas(DisplayArea.java:404)
 14:36:22.676  	at com.android.server.wm.DisplayArea.reduceOnAllTaskDisplayAreas(DisplayArea.java:404)
 14:36:22.676  	at com.android.server.wm.DisplayArea.reduceOnAllTaskDisplayAreas(DisplayArea.java:404)
 14:36:22.676  	at com.android.server.wm.DisplayArea.reduceOnAllTaskDisplayAreas(DisplayArea.java:404)
 14:36:22.676  	at com.android.server.wm.WindowContainer.reduceOnAllTaskDisplayAreas(WindowContainer.java:2283)
 14:36:22.676  	at com.android.server.wm.RootWindowContainer.startHomeOnDisplay(RootWindowContainer.java:1433)
 14:36:22.676  	at com.android.server.wm.RootWindowContainer.startHomeOnDisplay(RootWindowContainer.java:1420)
 14:36:22.676  	at com.android.server.wm.RootWindowContainer.startHomeOnAllDisplays(RootWindowContainer.java:1405)
 14:36:22.676  	at com.android.server.wm.ActivityTaskManagerService$LocalService.startHomeOnAllDisplays(ActivityTaskManagerService.java:5892)
 14:36:22.676  	at com.android.server.am.ActivityManagerService.systemReady(ActivityManagerService.java:8203)
 14:36:22.676  	at com.android.server.SystemServer.startOtherServices(SystemServer.java:2801)
 14:36:22.676  	at com.android.server.SystemServer.run(SystemServer.java:939)
 14:36:22.676  	at com.android.server.SystemServer.main(SystemServer.java:649)
 14:36:22.676  	at java.lang.reflect.Method.invoke(Native Method)
 14:36:22.676  	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:548)
 14:36:22.676  	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:914)

```

添加 ActivityRecord

```shell
 14:36:22.681  Task{63f31d4 #2 type=undefined A=1000:com.android.settings.FallbackHome U=0 rootTaskId=1 visible=true visibleRequested=false mode=fullscreen translucent=true sz=0} addChild child = ActivityRecord{983a135 u0 roid.settings/.FallbackHome}
 14:36:22.681  java.lang.Exception
 14:36:22.681  	at com.android.server.wm.WindowContainer.addChild(WindowContainer.java:727)
 14:36:22.681  	at com.android.server.wm.TaskFragment.addChild(TaskFragment.java:1835)
 14:36:22.681  	at com.android.server.wm.Task.addChild(Task.java:1429)
 14:36:22.681  	at com.android.server.wm.ActivityStarter.addOrReparentStartingActivity(ActivityStarter.java:2927)
 14:36:22.681  	at com.android.server.wm.ActivityStarter.setNewTask(ActivityStarter.java:2877)
 14:36:22.681  	at com.android.server.wm.ActivityStarter.startActivityInner(ActivityStarter.java:1864)
 14:36:22.681  	at com.android.server.wm.ActivityStarter.startActivityUnchecked(ActivityStarter.java:1661)
 14:36:22.681  	at com.android.server.wm.ActivityStarter.executeRequest(ActivityStarter.java:1216)
 14:36:22.681  	at com.android.server.wm.ActivityStarter.execute(ActivityStarter.java:702)
 14:36:22.681  	at com.android.server.wm.ActivityStartController.startHomeActivity(ActivityStartController.java:179)
 14:36:22.681  	at com.android.server.wm.RootWindowContainer.startHomeOnTaskDisplayArea(RootWindowContainer.java:1493)
 14:36:22.681  	at com.android.server.wm.RootWindowContainer.lambda$startHomeOnDisplay$12$com-android-server-wm-RootWindowContainer(RootWindowContainer.java:1434)
 14:36:22.681  	at com.android.server.wm.RootWindowContainer$$ExternalSyntheticLambda7.apply(Unknown Source:16)
 14:36:22.681  	at com.android.server.wm.TaskDisplayArea.reduceOnAllTaskDisplayAreas(TaskDisplayArea.java:513)
 14:36:22.681  	at com.android.server.wm.DisplayArea.reduceOnAllTaskDisplayAreas(DisplayArea.java:404)
 14:36:22.681  	at com.android.server.wm.DisplayArea.reduceOnAllTaskDisplayAreas(DisplayArea.java:404)
 14:36:22.681  	at com.android.server.wm.DisplayArea.reduceOnAllTaskDisplayAreas(DisplayArea.java:404)
 14:36:22.681  	at com.android.server.wm.DisplayArea.reduceOnAllTaskDisplayAreas(DisplayArea.java:404)
14:36:22.681  	at com.android.server.wm.DisplayArea.reduceOnAllTaskDisplayAreas(DisplayArea.java:404)
 14:36:22.681  	at com.android.server.wm.WindowContainer.reduceOnAllTaskDisplayAreas(WindowContainer.java:2283)
 14:36:22.681  	at com.android.server.wm.RootWindowContainer.startHomeOnDisplay(RootWindowContainer.java:1433)
 14:36:22.681  	at com.android.server.wm.RootWindowContainer.startHomeOnDisplay(RootWindowContainer.java:1420)
 14:36:22.681  	at com.android.server.wm.RootWindowContainer.startHomeOnAllDisplays(RootWindowContainer.java:1405)
 14:36:22.681  	at com.android.server.wm.ActivityTaskManagerService$LocalService.startHomeOnAllDisplays(ActivityTaskManagerService.java:5892)
 14:36:22.681  	at com.android.server.am.ActivityManagerService.systemReady(ActivityManagerService.java:8203)
 14:36:22.681  	at com.android.server.SystemServer.startOtherServices(SystemServer.java:2801)
 14:36:22.681  	at com.android.server.SystemServer.run(SystemServer.java:939)
 14:36:22.681  	at com.android.server.SystemServer.main(SystemServer.java:649)
 14:36:22.681  	at java.lang.reflect.Method.invoke(Native Method)
 14:36:22.681  	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:548)
 14:36:22.681  	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:914)

```

添加WindowState

```shell
 14:36:23.276   565   627 I test2   : ActivityRecord{983a135 u0 com.android.settings/.FallbackHome} t2} addChild Comparator child = Window{ae9b359 u0 com.android.settings/com.android.settings.FallbackHome}
 14:36:23.276   565   627 I test2   : java.lang.Exception
 14:36:23.276   565   627 I test2   : 	at com.android.server.wm.WindowContainer.addChild(WindowContainer.java:694)
 14:36:23.276   565   627 I test2   : 	at com.android.server.wm.WindowToken.addWindow(WindowToken.java:302)
 14:36:23.276   565   627 I test2   : 	at com.android.server.wm.ActivityRecord.addWindow(ActivityRecord.java:4212)
 14:36:23.276   565   627 I test2   : 	at com.android.server.wm.WindowManagerService.addWindow(WindowManagerService.java:1773)
 14:36:23.276   565   627 I test2   : 	at com.android.server.wm.Session.addToDisplayAsUser(Session.java:209)
 14:36:23.276   565   627 I test2   : 	at android.view.IWindowSession$Stub.onTransact(IWindowSession.java:652)
 14:36:23.276   565   627 I test2   : 	at com.android.server.wm.Session.onTransact(Session.java:175)
 14:36:23.276   565   627 I test2   : 	at android.os.Binder.execTransactInternal(Binder.java:1280)
 14:36:23.276   565   627 I test2   : 	at android.os.Binder.execTransact(Binder.java:1244)

```

#### 最终的WindowState 挂载数据

每addChild时候其实都会触发WindowContainer.onParentChanged，从而触发DisplayContent中创建对应的SurfaceControl，而且大家注意mWmService.makeSurfaceBuilder(s).setContainerLayer();创建出来的都是容器，而不是真正可以绘制的SurfaceControl

app调用到wms的relayout才可以获取可以绘制画面的surfacecontrol\
frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java

```java
   public int relayoutWindow(Session session, IWindow client, LayoutParams attrs,
            int requestedWidth, int requestedHeight, int viewVisibility, int flags,
            ClientWindowFrames outFrames, MergedConfiguration mergedConfiguration,
            SurfaceControl outSurfaceControl, InsetsState outInsetsState,
            InsetsSourceControl[] outActiveControls, Bundle outSyncIdBundle) {


```

可以看到outSurfaceControl参数，这个就是wms端写入数据，然后app端的ViewRootImpl来进行读取的。

```java
  // Create surfaceControl before surface placement otherwise layout will be skipped
            // (because WS.isGoneForLayout() is true when there is no surface.
            if (shouldRelayout) {
                try {
                    result = createSurfaceControl(outSurfaceControl, result, win, winAnimator);
                } catch (Exception e) {
                    displayContent.getInputMonitor().updateInputWindowsLw(true /*force*/);

                    ProtoLog.w(WM_ERROR,
                            "Exception thrown when creating surface for client %s (%s). %s",
                            client, win.mAttrs.getTitle(), e);
                    Binder.restoreCallingIdentity(origId);
                    return 0;
                }
            }
```

```java
private int createSurfaceControl(SurfaceControl outSurfaceControl, int result,
            WindowState win, WindowStateAnimator winAnimator) {
        if (!win.mHasSurface) {
            result |= RELAYOUT_RES_SURFACE_CHANGED;
        }

        WindowSurfaceController surfaceController;
        try {
            Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, "createSurfaceControl");
            surfaceController = winAnimator.createSurfaceLocked();
        } finally {
            Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
        }
        if (surfaceController != null) {
            surfaceController.getSurfaceControl(outSurfaceControl);
            ProtoLog.i(WM_SHOW_TRANSACTIONS, "OUT SURFACE %s: copied", outSurfaceControl);

        } else {
            // For some reason there isn't a surface.  Clear the
            // caller's object so they see the same state.
            ProtoLog.w(WM_ERROR, "Failed to create surface control for %s", win);
            outSurfaceControl.release();
        }

        return result;
    }

```

//这里首先又调用到了winAnimator.createSurfaceLocked();

```java
 WindowSurfaceController createSurfaceLocked() {
//ignore
  mSurfaceController = new WindowSurfaceController(attrs.getTitle().toString(), format,
                    flags, this, attrs.type);
//ignore
}
```

调用到了WindowSurfaceController构造：

```java
WindowSurfaceController(String name, int format, int flags, WindowStateAnimator animator,
            int windowType) {
        mAnimator = animator;

        title = name;

        mService = animator.mService;
        final WindowState win = animator.mWin;
        mWindowType = windowType;
        mWindowSession = win.mSession;

        Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, "new SurfaceControl");
        final SurfaceControl.Builder b = win.makeSurface()
                .setParent(win.getSurfaceControl())
                .setName(name)
                .setFormat(format)
                .setFlags(flags)
                .setMetadata(METADATA_WINDOW_TYPE, windowType)
                .setMetadata(METADATA_OWNER_UID, mWindowSession.mUid)
                .setMetadata(METADATA_OWNER_PID, mWindowSession.mPid)
                .setCallsite("WindowSurfaceController");//此时其实还是属于Container，因为DisplayerContent创建

        final boolean useBLAST = mService.mUseBLAST && ((win.getAttrs().privateFlags
                & WindowManager.LayoutParams.PRIVATE_FLAG_USE_BLAST) != 0);

        if (useBLAST) {
            b.setBLASTLayer();//非常关键，变成非Container
        }

        mSurfaceControl = b.build();

        Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
    }

```

主要通过 BLAST 属性的设置，变为实际的layer。

```java

public int relayoutWindow(Session session, IWindow client, LayoutParams attrs,
            int requestedWidth, int requestedHeight, int viewVisibility, int flags,
            ClientWindowFrames outFrames, MergedConfiguration mergedConfiguration,
            SurfaceControl outSurfaceControl, InsetsState outInsetsState,
            InsetsSourceControl[] outActiveControls, Bundle outSyncIdBundle) {
   //ignore
   //根据client找到对应的WindowState 
            final WindowState win = windowForClientLocked(session, client, false);
           //ignore

            // Create surfaceControl before surface placement otherwise layout will be skipped
            // (because WS.isGoneForLayout() is true when there is no surface.
            if (shouldRelayout) {
                try {
                    //创建对应的window需要的SurfaceControl，传递回应用，应用用他进行绘制
                    result = createSurfaceControl(outSurfaceControl, result, win, winAnimator);
                } catch (Exception e) {
                  //ignore
                }
            }
			//调用最核心的performSurfacePlacement进行相关的layout操作
            // We may be deferring layout passes at the moment, but since the client is interested
            // in the new out values right now we need to force a layout.
            mWindowPlacerLocked.performSurfacePlacement(true /* force */);
			//ignore
                
            if (!win.isGoneForLayout()) {
                win.mResizedWhileGone = false;
            }
 			//把相关config数据填回去
            win.fillClientWindowFramesAndConfiguration(outFrames, mergedConfiguration,
                    false /* useLatestConfig */, shouldRelayout);
            //把相关inset数据设置回去
            outInsetsState.set(win.getCompatInsetsState(), win.isClientLocal());
           //ignore
            getInsetsSourceControls(win, outActiveControls);
        }

        Binder.restoreCallingIdentity(origId);
        return result;
    }

```

主要就有2个最关键步骤：\
1、创建对应Window的SurfaceControl\
2、计算出对应的window区域等，把inset和config传递回去 ， 这里面应该针对跨进程的回复传递。

计算出对应的window区域等，把inset和config传递回去

```java
final void performSurfacePlacement(boolean force) {
	  if (mDeferDepth > 0 && !force) {
		  mDeferredRequests++;
		  return;
	  }
	  int loopCount = 6;
	  do {
		  mTraversalScheduled = false;
		  performSurfacePlacementLoop();//调用performSurfacePlacementLoop
		  mService.mAnimationHandler.removeCallbacks(mPerformSurfacePlacement);
		  loopCount--;
	  } while (mTraversalScheduled && loopCount > 0);
	  mService.mRoot.mWallpaperActionPending = false;
  }
```

performSurfacePlacementLoop

```java
private void performSurfacePlacementLoop() {
//ignore 
	mService.mRoot.performSurfacePlacement();
//ignore
}
```

这里会调用到RootWindowContainer的performSurfacePlacement

```java
void performSurfacePlacement() {
  Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, "performSurfacePlacement");
  try {
	  performSurfacePlacementNoTrace();
  } finally {
	  Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
  }
}

```

接下来又到performSurfacePlacementNoTrace，接下来又会调用到最关键的applySurfaceChangesTransaction方法：

```java
private void applySurfaceChangesTransaction() {
   //ignore

	final int count = mChildren.size();
	for (int j = 0; j < count; ++j) {
		final DisplayContent dc = mChildren.get(j);
		dc.applySurfaceChangesTransaction();
	}

	// Give the display manager a chance to adjust properties like display rotation if it needs
	// to.
	mWmService.mDisplayManagerInternal.performTraversal(mDisplayTransaction);
	SurfaceControl.mergeToGlobalTransaction(mDisplayTransaction);
}
```

这里又会调用到DisplayContent 的applySurfaceChangesTransaction

```java
void applySurfaceChangesTransaction() {
   //执行布局相关
	// Perform a layout, if needed.
	performLayout(true /* initial */, false /* updateInputWindows */);
	pendingLayoutChanges = 0;

   //判断布局策略
	mDisplayPolicy.beginPostLayoutPolicyLw();
	forAllWindows(mApplyPostLayoutPolicy, true /* traverseTopToBottom */);
  
	//
	Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, "applyWindowSurfaceChanges");
	try {
	//挨个迭代每个窗口执行mApplySurfaceChangesTransaction
		forAllWindows(mApplySurfaceChangesTransaction, true /* traverseTopToBottom */);
	} finally {
		Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
	}
	 //做绘制前最后一些处理
	prepareSurfaces();

   //ignore
}
```

```java
void performLayout(boolean initial, boolean updateInputWindows) {
	Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, "performLayout");
	try {
		performLayoutNoTrace(initial, updateInputWindows);
	} finally {
		Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
	}
}
private void performLayoutNoTrace(boolean initial, boolean updateInputWindows) {
	if (!isLayoutNeeded()) {
		return;
	}
	clearLayoutNeeded();

	if (DEBUG_LAYOUT) {
		Slog.v(TAG, "-------------------------------------");
		Slog.v(TAG, "performLayout: dw=" + mDisplayInfo.logicalWidth
				+ " dh=" + mDisplayInfo.logicalHeight);
	}

	int seq = mLayoutSeq + 1;
	if (seq < 0) seq = 0;
	mLayoutSeq = seq;

	mTmpInitial = initial;


	// First perform layout of any root windows (not attached to another window).
	forAllWindows(mPerformLayout, true /* traverseTopToBottom */);

	// Now perform layout of attached windows, which usually depend on the position of the
	// window they are attached to. XXX does not deal with windows that are attached to windows
	// that are themselves attached.
	forAllWindows(mPerformLayoutAttached, true /* traverseTopToBottom */);

	// Window frames may have changed. Tell the input dispatcher about it.
	mInputMonitor.setUpdateInputWindowsNeededLw();
	if (updateInputWindows) {
		mInputMonitor.updateInputWindowsLw(false /*force*/);
	}
}
```

最后也是循环遍历层级结构树的每个windowstate进行mPerformLayout的执行

```java
private final Consumer<WindowState> mPerformLayout = w -> {
 //ignore
	getDisplayPolicy().layoutWindowLw(w, null, mDisplayFrames);
//ignore
};
```

调用是DisplayPolicy.java的layoutWindowLw

```java
 public void layoutWindowLw(WindowState win, WindowState attached, DisplayFrames displayFrames) {
        //ignore
        mWindowLayout.computeFrames(attrs, win.getInsetsState(), displayFrames.mDisplayCutoutSafe,
                win.getBounds(), win.getWindowingMode(), requestedWidth, requestedHeight,
                win.getRequestedVisibilities(), attachedWindowFrame, win.mGlobalScale,
                sTmpClientFrames);

        win.setFrames(sTmpClientFrames, win.mRequestedWidth, win.mRequestedHeight);
    }
```

这里调用是WindowLayout.computeFrames\
具体computeFrames的计算代码较长，主要就是会根据系统的inset情况来决定，比如statusbar，和navigationbar等，然后给app一个合适的frame

\---end
