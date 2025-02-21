# Android14 - WindowManagerService之客户端Activity布局-CSDN博客

## **A****n****d****r****oi****d****1\*\*\*\*4** **-** **WindowManagerService****之客户端****Activity\*\*\*\*布局**

## **一**\*\*、****主要****角色\*\*

WMS作为一个服务端，有多种客户端与其交互的场景。我们以常见的Activity为例：

![873897761998e36aeca2444101760422\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/6708eb606ad96.png)\
**Activity**：在ActivityThread构建一个Activity后，会调用其attach方法，在attach中构建了一个PhoneWindow。

**PhoneWindow**：一个Activity对应一个PhoneWindow。而每个PhoneWindow持有一个WindowManagerImpl对象。

**WindowManagerImpl**：是服务端WMS在客户端的代理。而WindowManagerImpl中持有WindowManagerGlobal。

**WindowManagerGlobal**：是进程内的单例。WindowManagerGlobal不仅持有WMS服务端的WindowManagerService、WindowSession的引用，并且集合了所有的ViewRootImpl、DecorView等

**ViewRootImpl**：每个Activity对应一个ViewRootImp对象。ViewRootImp是作为客户端与WMS的桥梁。从客户端到服务端的角度，ViewRootImp中持有的WindowSession对象是服务端Session的引用，一个Session对应一个客户端进程。从服务端到客户端的角度，ViewRootImp中有一个子类W，是binder通信的服务端，W对象的引用被WMS持有，用来服务端向客户端通信。另外，ViewRootImp中持有DecorView对象。从逻辑上，ViewRootImpl是View树的根，但其本身并不是一个View，只是实现了ViewParent接口。DecorView是ViewRootImp的子类（见setView()方法中的view.assignParent(this)）

）

**DecorView**：每个Activity对应一个DecorView。DecorView是真正的顶层View，是最外层的view。其inflate PhoneWidow传过来的layout后，将其addView作为自己的第0个类。DecorView维护了一个窗口的基本结构，包括主内容区域、titlebar区域等。

##

## **二**\*\*、\*\***窗口的构建过程**

窗口的结构大致如下：

![9cab045e506b7ba8efac0632869f6944\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/6708eaa644ff9.png)

其过程参考上图，在PhoneWindow的初始化过程中，调用了generateLayout(DecorView decor)方法，其中对一个window的整体画面进行了初始化。

```java
protected ViewGroup generateLayout(DecorView decor) {
	// Apply data from current theme.
 
	// Style来自于Theme xml文件
	TypedArray a = getWindowStyle();
 
	mIsFloating = a.getBoolean(R.styleable.Window_windowIsFloating, false);
	int flagsToUpdate = (FLAG_LAYOUT_IN_SCREEN|FLAG_LAYOUT_INSET_DECOR)
			&amp; (~getForcedWindowFlags());
	if (mIsFloating) {
		setLayout(WRAP_CONTENT, WRAP_CONTENT);
		setFlags(0, flagsToUpdate);
	} else {
		setFlags(FLAG_LAYOUT_IN_SCREEN|FLAG_LAYOUT_INSET_DECOR, flagsToUpdate);
		getAttributes().setFitInsetsSides(0);
		getAttributes().setFitInsetsTypes(0);
	}
 
	if (a.getBoolean(R.styleable.Window_windowNoTitle, false)) {
		requestFeature(FEATURE_NO_TITLE);
	} else if (a.getBoolean(R.styleable.Window_windowActionBar, false)) {
		// Don&#39;t allow an action bar if there is no title.
		requestFeature(FEATURE_ACTION_BAR);
	}
 
	if (a.getBoolean(R.styleable.Window_windowActionBarOverlay, false)) {
		requestFeature(FEATURE_ACTION_BAR_OVERLAY);
	}
 
	if (a.getBoolean(R.styleable.Window_windowActionModeOverlay, false)) {
		requestFeature(FEATURE_ACTION_MODE_OVERLAY);
	}
 
	if (a.getBoolean(R.styleable.Window_windowFullscreen, false)) {
		setFlags(FLAG_FULLSCREEN, FLAG_FULLSCREEN &amp; (~getForcedWindowFlags()));
	}
 
	if (a.getBoolean(R.styleable.Window_windowTranslucentStatus,
			false)) {
		setFlags(FLAG_TRANSLUCENT_STATUS, FLAG_TRANSLUCENT_STATUS
				&amp; (~getForcedWindowFlags()));
	}
 
	if (a.getBoolean(R.styleable.Window_windowTranslucentNavigation,
			false)) {
		setFlags(FLAG_TRANSLUCENT_NAVIGATION, FLAG_TRANSLUCENT_NAVIGATION
				&amp; (~getForcedWindowFlags()));
	}
 
	if (a.getBoolean(R.styleable.Window_windowShowWallpaper, false)) {
		setFlags(FLAG_SHOW_WALLPAPER, FLAG_SHOW_WALLPAPER&amp;(~getForcedWindowFlags()));
	}
 
	if (a.getBoolean(R.styleable.Window_windowEnableSplitTouch,
			getContext().getApplicationInfo().targetSdkVersion
					&gt;= android.os.Build.VERSION_CODES.HONEYCOMB)) {
		setFlags(FLAG_SPLIT_TOUCH, FLAG_SPLIT_TOUCH&amp;(~getForcedWindowFlags()));
	}
 
	a.getValue(R.styleable.Window_windowMinWidthMajor, mMinWidthMajor);
	a.getValue(R.styleable.Window_windowMinWidthMinor, mMinWidthMinor);
	if (DEBUG) Log.d(TAG, &#34;Min width minor: &#34; &#43; mMinWidthMinor.coerceToString()
			&#43; &#34;, major: &#34; &#43; mMinWidthMajor.coerceToString());
	if (a.hasValue(R.styleable.Window_windowFixedWidthMajor)) {
		if (mFixedWidthMajor == null) mFixedWidthMajor = new TypedValue();
		a.getValue(R.styleable.Window_windowFixedWidthMajor,
				mFixedWidthMajor);
	}
	if (a.hasValue(R.styleable.Window_windowFixedWidthMinor)) {
		if (mFixedWidthMinor == null) mFixedWidthMinor = new TypedValue();
		a.getValue(R.styleable.Window_windowFixedWidthMinor,
				mFixedWidthMinor);
	}
	if (a.hasValue(R.styleable.Window_windowFixedHeightMajor)) {
		if (mFixedHeightMajor == null) mFixedHeightMajor = new TypedValue();
		a.getValue(R.styleable.Window_windowFixedHeightMajor,
				mFixedHeightMajor);
	}
	if (a.hasValue(R.styleable.Window_windowFixedHeightMinor)) {
		if (mFixedHeightMinor == null) mFixedHeightMinor = new TypedValue();
		a.getValue(R.styleable.Window_windowFixedHeightMinor,
				mFixedHeightMinor);
	}
	if (a.getBoolean(R.styleable.Window_windowContentTransitions, false)) {
		requestFeature(FEATURE_CONTENT_TRANSITIONS);
	}
	if (a.getBoolean(R.styleable.Window_windowActivityTransitions, false)) {
		requestFeature(FEATURE_ACTIVITY_TRANSITIONS);
	}
 
	mIsTranslucent = a.getBoolean(R.styleable.Window_windowIsTranslucent, false);
 
	final Context context = getContext();
	final int targetSdk = context.getApplicationInfo().targetSdkVersion;
	final boolean targetPreL = targetSdk &lt; android.os.Build.VERSION_CODES.LOLLIPOP;
	final boolean targetPreQ = targetSdk &lt; Build.VERSION_CODES.Q;
 
	if (!mForcedStatusBarColor) {
		mStatusBarColor = a.getColor(R.styleable.Window_statusBarColor, Color.BLACK);
	}
	if (!mForcedNavigationBarColor) {
		final int navBarCompatibleColor = context.getColor(R.color.navigation_bar_compatible);
		final int navBarDefaultColor = context.getColor(R.color.navigation_bar_default);
		final int navBarColor = a.getColor(R.styleable.Window_navigationBarColor,
				navBarDefaultColor);
 
		mNavigationBarColor =
				navBarColor == navBarDefaultColor
						&amp;&amp; !context.getResources().getBoolean(
								R.bool.config_navBarDefaultTransparent)
				? navBarCompatibleColor
				: navBarColor;
 
		mNavigationBarDividerColor = a.getColor(R.styleable.Window_navigationBarDividerColor,
				Color.TRANSPARENT);
	}
	if (!targetPreQ) {
		mEnsureStatusBarContrastWhenTransparent = a.getBoolean(
				R.styleable.Window_enforceStatusBarContrast, false);
		mEnsureNavigationBarContrastWhenTransparent = a.getBoolean(
				R.styleable.Window_enforceNavigationBarContrast, true);
	}
 
	WindowManager.LayoutParams params = getAttributes();
 
	// Non-floating windows on high end devices must put up decor beneath the system bars and
	// therefore must know about visibility changes of those.
	if (!mIsFloating) {
		if (!targetPreL &amp;&amp; a.getBoolean(
				R.styleable.Window_windowDrawsSystemBarBackgrounds,
				false)) {
			setFlags(FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS,
					FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS &amp; ~getForcedWindowFlags());
		}
		if (mDecor.mForceWindowDrawsBarBackgrounds) {
			params.privateFlags |= PRIVATE_FLAG_FORCE_DRAW_BAR_BACKGROUNDS;
		}
		params.privateFlags |= PRIVATE_FLAG_NO_MOVE_ANIMATION;
	}
	if (a.getBoolean(
			R.styleable.Window_windowNoMoveAnimation,
			false)) {
		params.privateFlags |= PRIVATE_FLAG_NO_MOVE_ANIMATION;
	}
	final int sysUiVis = decor.getSystemUiVisibility();
	final int statusLightFlag = View.SYSTEM_UI_FLAG_LIGHT_STATUS_BAR;
	final int statusFlag = a.getBoolean(R.styleable.Window_windowLightStatusBar, false)
			? statusLightFlag : 0;
	final int navLightFlag = View.SYSTEM_UI_FLAG_LIGHT_NAVIGATION_BAR;
	final int navFlag = a.getBoolean(R.styleable.Window_windowLightNavigationBar, false)
			? navLightFlag : 0;
	decor.setSystemUiVisibility(
			(sysUiVis &amp; ~(statusLightFlag | navLightFlag)) | (statusFlag | navFlag));
	if (a.hasValue(R.styleable.Window_windowLayoutInDisplayCutoutMode)) {
		int mode = a.getInt(R.styleable.Window_windowLayoutInDisplayCutoutMode, -1);
		if (mode &lt; LAYOUT_IN_DISPLAY_CUTOUT_MODE_DEFAULT
				|| mode &gt; LAYOUT_IN_DISPLAY_CUTOUT_MODE_ALWAYS) {
			throw new UnsupportedOperationException(&#34;Unknown windowLayoutInDisplayCutoutMode: &#34;
					&#43; a.getString(R.styleable.Window_windowLayoutInDisplayCutoutMode));
		}
		params.layoutInDisplayCutoutMode = mode;
	}
 
	if (mAlwaysReadCloseOnTouchAttr || getContext().getApplicationInfo().targetSdkVersion
			&gt;= android.os.Build.VERSION_CODES.HONEYCOMB) {
		if (a.getBoolean(
				R.styleable.Window_windowCloseOnTouchOutside,
				false)) {
			setCloseOnTouchOutsideIfNotSet(true);
		}
	}
 
	if (!hasSoftInputMode()) {
		params.softInputMode = a.getInt(
				R.styleable.Window_windowSoftInputMode,
				params.softInputMode);
	}
 
	if (a.getBoolean(R.styleable.Window_backgroundDimEnabled,
			mIsFloating)) {
		/* All dialogs should have the window dimmed */
		if ((getForcedWindowFlags()&amp;WindowManager.LayoutParams.FLAG_DIM_BEHIND) == 0) {
			params.flags |= WindowManager.LayoutParams.FLAG_DIM_BEHIND;
		}
		if (!haveDimAmount()) {
			params.dimAmount = a.getFloat(
					android.R.styleable.Window_backgroundDimAmount, 0.5f);
		}
	}
 
	if (a.getBoolean(R.styleable.Window_windowBlurBehindEnabled, false)) {
		if ((getForcedWindowFlags() &amp; WindowManager.LayoutParams.FLAG_BLUR_BEHIND) == 0) {
			params.flags |= WindowManager.LayoutParams.FLAG_BLUR_BEHIND;
		}
 
		params.setBlurBehindRadius(a.getDimensionPixelSize(
				android.R.styleable.Window_windowBlurBehindRadius, 0));
	}
 
	setBackgroundBlurRadius(a.getDimensionPixelSize(
			R.styleable.Window_windowBackgroundBlurRadius, 0));
 
 
	if (params.windowAnimations == 0) {
		params.windowAnimations = a.getResourceId(
				R.styleable.Window_windowAnimationStyle, 0);
	}
 
	// The rest are only done if this window is not embedded; otherwise,
	// the values are inherited from our container.
	if (getContainer() == null) {
		if (mBackgroundDrawable == null) {
 
			if (mFrameResource == 0) {
				mFrameResource = a.getResourceId(R.styleable.Window_windowFrame, 0);
			}
 
			if (a.hasValue(R.styleable.Window_windowBackground)) {
				mBackgroundDrawable = a.getDrawable(R.styleable.Window_windowBackground);
			}
		}
		if (a.hasValue(R.styleable.Window_windowBackgroundFallback)) {
			mBackgroundFallbackDrawable =
					a.getDrawable(R.styleable.Window_windowBackgroundFallback);
		}
		if (mLoadElevation) {
			mElevation = a.getDimension(R.styleable.Window_windowElevation, 0);
		}
		mClipToOutline = a.getBoolean(R.styleable.Window_windowClipToOutline, false);
		mTextColor = a.getColor(R.styleable.Window_textColor, Color.TRANSPARENT);
	}
 
	// Inflate the window decor.
    // 根据Feature的设置，选择不同的layout xml。具体的resource来源于platform/frameworks/base/core/res/res，根据不同平台选择具体的资源文件。
	// 如果什么都没有设置，最后默认的会选择R.layout.screen_simple;
	int layoutResource;
	int features = getLocalFeatures();
	// System.out.println(&#34;Features: 0x&#34; &#43; Integer.toHexString(features));
	if ((features &amp; ((1 &lt;&lt; FEATURE_LEFT_ICON) | (1 &lt;&lt; FEATURE_RIGHT_ICON))) != 0) {
		if (mIsFloating) {
			TypedValue res = new TypedValue();
			getContext().getTheme().resolveAttribute(
					R.attr.dialogTitleIconsDecorLayout, res, true);
			layoutResource = res.resourceId;
		} else {
			layoutResource = R.layout.screen_title_icons;
		}
		// XXX Remove this once action bar supports these features.
		removeFeature(FEATURE_ACTION_BAR);
		// System.out.println(&#34;Title Icons!&#34;);
	} else if ((features &amp; ((1 &lt;&lt; FEATURE_PROGRESS) | (1 &lt;&lt; FEATURE_INDETERMINATE_PROGRESS))) != 0
			&amp;&amp; (features &amp; (1 &lt;&lt; FEATURE_ACTION_BAR)) == 0) {
		// Special case for a window with only a progress bar (and title).
		// XXX Need to have a no-title version of embedded windows.
		layoutResource = R.layout.screen_progress;
		// System.out.println(&#34;Progress!&#34;);
	} else if ((features &amp; (1 &lt;&lt; FEATURE_CUSTOM_TITLE)) != 0) {
		// Special case for a window with a custom title.
		// If the window is floating, we need a dialog layout
		if (mIsFloating) {
			TypedValue res = new TypedValue();
			getContext().getTheme().resolveAttribute(
					R.attr.dialogCustomTitleDecorLayout, res, true);
			layoutResource = res.resourceId;
		} else {
			layoutResource = R.layout.screen_custom_title;
		}
		// XXX Remove this once action bar supports these features.
		removeFeature(FEATURE_ACTION_BAR);
	} else if ((features &amp; (1 &lt;&lt; FEATURE_NO_TITLE)) == 0) {
		// If no other features and not embedded, only need a title.
		// If the window is floating, we need a dialog layout
		if (mIsFloating) {
			TypedValue res = new TypedValue();
			getContext().getTheme().resolveAttribute(
					R.attr.dialogTitleDecorLayout, res, true);
			layoutResource = res.resourceId;
		} else if ((features &amp; (1 &lt;&lt; FEATURE_ACTION_BAR)) != 0) {
			layoutResource = a.getResourceId(
					R.styleable.Window_windowActionBarFullscreenDecorLayout,
					R.layout.screen_action_bar);
		} else {
			layoutResource = R.layout.screen_title;
		}
		// System.out.println(&#34;Title!&#34;);
	} else if ((features &amp; (1 &lt;&lt; FEATURE_ACTION_MODE_OVERLAY)) != 0) {
		layoutResource = R.layout.screen_simple_overlay_action_mode;
	} else {
		// Embedded, so no decoration is needed.
		layoutResource = R.layout.screen_simple;
		// System.out.println(&#34;Simple!&#34;);
	}
 
	mDecor.startChanging();
	// 将最终的layoutResource传给DecorView，在DecorView里面去inflate
	mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);
 
	ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
	if (contentParent == null) {
		throw new RuntimeException(&#34;Window couldn&#39;t find content container view&#34;);
	}
 
	if ((features &amp; (1 &lt;&lt; FEATURE_INDETERMINATE_PROGRESS)) != 0) {
		ProgressBar progress = getCircularProgressBar(false);
		if (progress != null) {
			progress.setIndeterminate(true);
		}
	}
 
	// Remaining setup -- of background and title -- that only applies
	// to top-level windows.
	if (getContainer() == null) {
		mDecor.setWindowBackground(mBackgroundDrawable);
 
		final Drawable frame;
		if (mFrameResource != 0) {
			frame = getContext().getDrawable(mFrameResource);
		} else {
			frame = null;
		}
		mDecor.setWindowFrame(frame);
 
		mDecor.setElevation(mElevation);
		mDecor.setClipToOutline(mClipToOutline);
 
		if (mTitle != null) {
			setTitle(mTitle);
		}
 
		if (mTitleColor == 0) {
			mTitleColor = mTextColor;
		}
		setTitleColor(mTitleColor);
	}
 
	mDecor.finishChanging();
 
	return contentParent;
}
```

该方法做了几件事：

1\. 获取主题属性。TypedArray a = getWindowStyle();

getWindowStyle最终调用的是

```java
platform/frameworks/base/core/java/android/content/Context.java
 
@NonNull
public final TypedArray obtainStyledAttributes(@NonNull @StyleableRes int[] attrs) {
    return getTheme().obtainStyledAttributes(attrs);
}
```

Theme获取根据不同平台、版本返回不同的文件：

```java
platform/frameworks/base/core/java/android/content/res/Resources.java
 
@UnsupportedAppUsage
public static int selectDefaultTheme(int curTheme, int targetSdkVersion) {
	return selectSystemTheme(curTheme, targetSdkVersion,
			com.android.internal.R.style.Theme,
			com.android.internal.R.style.Theme_Holo,
			com.android.internal.R.style.Theme_DeviceDefault,
			com.android.internal.R.style.Theme_DeviceDefault_Light_DarkActionBar);
}
 
/** @hide */
public static int selectSystemTheme(int curTheme, int targetSdkVersion, int orig, int holo,
		int dark, int deviceDefault) {
	if (curTheme != ID_NULL) {
		return curTheme;
	}
	if (targetSdkVersion &lt; Build.VERSION_CODES.HONEYCOMB) {
		return orig;
	}
	if (targetSdkVersion &lt; Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
		return holo;
	}
	if (targetSdkVersion &lt; Build.VERSION_CODES.N) {
		return dark;
	}
	return deviceDefault;
}
```

对应的是platform/frameworks/base/core/res/res/values/themes.xml、platform/frameworks/base/core/res/res/values/themes\_holo.xml、platform/frameworks/base/core/res/res/values/themes\_device\_defaults.xml等等这些主题文件。

2\. 根据theme里的配置，设置feature、flags、背景色等。

3\. 根据设置后的feature，选择一个layout文件layoutResource。该layoutResource传给DecorView进行窗口View构建。

一般layout都分为上面的titlebar的区域，和下面的content主区域。titlebar部分的区域不是固定的，每个layout可能有不同的布局。content一般是固定的。就是说大部分layout都会有一个id为content的layout。

以默认的platform/frameworks/base/core/res/res/layout/screen\_simple.xml文件为例：

```xml
&lt;LinearLayout xmlns:android=&#34;http://schemas.android.com/apk/res/android&#34;
    android:layout_width=&#34;match_parent&#34;
    android:layout_height=&#34;match_parent&#34;
    android:fitsSystemWindows=&#34;true&#34;
    android:orientation=&#34;vertical&#34;&gt;
    &lt;ViewStub android:id=&#34;@&#43;id/action_mode_bar_stub&#34;
              android:inflatedId=&#34;@&#43;id/action_mode_bar&#34;
              android:layout=&#34;@layout/action_mode_bar&#34;
              android:layout_width=&#34;match_parent&#34;
              android:layout_height=&#34;wrap_content&#34;
              android:theme=&#34;?attr/actionBarTheme&#34; /&gt;
    &lt;FrameLayout
         android:id=&#34;@android:id/content&#34;
         android:layout_width=&#34;match_parent&#34;
         android:layout_height=&#34;match_parent&#34;
         android:foregroundInsidePadding=&#34;false&#34;
         android:foregroundGravity=&#34;fill_horizontal|top&#34;
         android:foreground=&#34;?android:attr/windowContentOverlay&#34; /&gt;
&lt;/LinearLayout&gt;
```

action\_mode\_bar\_stub是title的部分。

@android:id/content是主内容区域。对应的是PhoneWindow的mContentParent成员。

而整个布局对应的是DecorView的mContentRoot成员变量，在mContentRoot构建后，也被add到DecorView的第0个Child。具体

```java
platform/frameworks/base/core/java/com/android/internal/policy/DecorView.java
 
void onResourcesLoaded(LayoutInflater inflater, int layoutResource) {
	if (mBackdropFrameRenderer != null) {
		loadBackgroundDrawablesIfNeeded();
		mBackdropFrameRenderer.onResourcesLoaded(
				this, mResizingBackgroundDrawable, mCaptionBackgroundDrawable,
				mUserCaptionBackgroundDrawable, getCurrentColor(mStatusColorViewState),
				getCurrentColor(mNavigationColorViewState));
	}
 
	mDecorCaptionView = createDecorCaptionView(inflater);
	final View root = inflater.inflate(layoutResource, null);
	if (mDecorCaptionView != null) {
		if (mDecorCaptionView.getParent() == null) {
			addView(mDecorCaptionView,
					new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
		}
		mDecorCaptionView.addView(root,
				new ViewGroup.MarginLayoutParams(MATCH_PARENT, MATCH_PARENT));
	} else {
 
		// Put it below the color views.
		addView(root, 0, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
	}
	mContentRoot = (ViewGroup) root;
	initializeElevation();
}
```


---

> 作者: [wentao](https://github.com/huang8604)  
> URL: https://huang8604.github.io/posts/android14-windowmanagerservice%E4%B9%8B%E5%AE%A2%E6%88%B7%E7%AB%AFactivity%E5%B8%83%E5%B1%80-csdn%E5%8D%9A%E5%AE%A2/  

