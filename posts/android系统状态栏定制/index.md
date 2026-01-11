# Android系统状态栏定制

### 背景

项目中为了适应产品形态需要对Android系统状态栏系统图标以及时钟和电池等做客制化，满足不同用户群体的视觉特性，那在定制过程中需要注意哪些事项？图标icon是否可以任意大小？状态栏多颜色模式下图标如何适配？复杂状态图标如何调整逻辑？

### 状态栏是什么？

首先来看下状态栏载体是什么？状态栏本质其实就是一个悬浮窗，在systemui初始化时创建显示。SystemUI/src/com/android/systemui/statusbar/phone/StatusBar.java

```java
protected void inflateStatusBarWindow(Context context) {
    mStatusBarWindow = (StatusBarWindowView) mInjectionInflater.injectable(
            LayoutInflater.from(context)).inflate(R.layout.super_status_bar, null);
}
```

由上可知状态栏就是使用super\_status\_bar.xml布局创建的一个悬浮窗。而这个布局包含了状态栏所有内容，应用通知，系统图标，时钟等。其主体内容如下

```xml
<com.android.systemui.statusbar.phone.StatusBarWindowView
    ...
    <FrameLayout
        android:id="@+id/status_bar_container"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />
    ...
</com.android.systemui.statusbar.phone.StatusBarWindowView>
```

其中包含status\_bar\_container 的framelayout的容器即为状态栏的view，在代码中通过fragmentmanager替换了了这个container。

```java
    protected void makeStatusBarView(@Nullable RegisterStatusBarResult result) {
        ...
        FragmentHostManager.get(mStatusBarWindow)
                .addTagListener(...).getFragmentManager()
                .beginTransaction()
                .replace(R.id.status_bar_container, new CollapsedStatusBarFragment(),
                        CollapsedStatusBarFragment.TAG)
                .commit();
```

而CollapsedStatusBarFragment的实现就是加载了status\_bar.xml 这个布局。

```java
    @Override
    public View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container,
            Bundle savedInstanceState) {
        return inflater.inflate(R.layout.status_bar, container, false);
    }
```

status\_bar.xml 布局内容就是显示出来的状态栏布局。这样状态栏整体布局就比较清晰，包含了应用通知，系统图标, 时钟，电池等。

```xml
<com.android.systemui.statusbar.phone.PhoneStatusBarView
    ...
    android:layout_height="@dimen/status_bar_height"
    android:id="@+id/status_bar"
    ...
    >
    ...
    <LinearLayout android:id="@+id/status_bar_contents"
        ...
        <!-- 左侧显示区域 整体权重只占了1-->
        <FrameLayout
            android:layout_height="match_parent"
            android:layout_width="0dp"
            android:layout_weight="1">
            ...
            <LinearLayout
                android:id="@+id/status_bar_left_side"
                ...             
                >
                <!-- 时钟 -->
                <com.android.systemui.statusbar.policy.Clock
                    android:id="@+id/clock"
                    ...
                    android:textAppearance="@style/TextAppearance.StatusBar.Clock"
                 />
                <!-- 应用通知icon区域 -->
                <com.android.systemui.statusbar.AlphaOptimizedFrameLayout
                    android:id="@+id/notification_icon_area"
                    android:layout_width="0dp"
                    android:layout_height="match_parent"
                    android:layout_weight="1"
                    android:orientation="horizontal"
                    android:clipChildren="false"/>

            </LinearLayout>
        </FrameLayout>
        ...
        <!-- 中间icon显示区域 -->
        <com.android.systemui.statusbar.AlphaOptimizedFrameLayout
            android:id="@+id/centered_icon_area"
            android:layout_width="wrap_content"
            android:layout_height="match_parent"
            android:orientation="horizontal"
            android:clipChildren="false"
            android:gravity="center_horizontal|center_vertical"/>

        <!-- 系统icon显示区域-->
        <com.android.keyguard.AlphaOptimizedLinearLayout android:id="@+id/system_icon_area"
            android:layout_width="0dp"
            android:layout_height="match_parent"
            android:layout_weight="1"
            android:orientation="horizontal"
            android:gravity="center_vertical|end"
            >
            <!-- 系统icon实际显示布局 -->
            <include layout="@layout/system_icons" />
        </com.android.keyguard.AlphaOptimizedLinearLayout>
    </LinearLayout>
...
</com.android.systemui.statusbar.phone.PhoneStatusBarView>
```

系统icon区域 system\_icons.xml

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
              xmlns:systemui="http://schemas.android.com/apk/res-auto"
    android:id="@+id/system_icons"
    ...>

    <com.android.systemui.statusbar.phone.StatusIconContainer 
        android:id="@+id/statusIcons"
        android:layout_width="0dp"
        android:layout_weight="1"
        .../>

    <com.android.systemui.statusbar.phone.seewo.BatteryImageView
        android:id="@+id/battery"
        .../>
</LinearLayout>
```

整个状态栏整体布局示意如下：

![b60e15e8132947a9e8f5047f7c60d609\_MD5](https://picgo.myjojo.fun:666/i/2024/10/29/6720998beeb1c.webp)

其中我们需要定制的从UI设计稿中可以看出，是三个区域，时钟， 系统icon，电池， 应用通知在这个项目中不需要，可以直接去掉通知信息功能，就不会显示出来。clock和battery都是自定义控件，比较好处理。重点看下系统icon实现。

##### 系统ICON布局

由上客制系统图标区域包含一个statusIcons 的容器view，还有battery 显示view。

其布局也是自定义view, StatusIconContainer.java

```xml
    <com.android.systemui.statusbar.phone.StatusIconContainer android:id="@+id/statusIcons"
        android:layout_width="0dp"
        .../>
```

其实现是基于AlphaOptimizedLinearLayout布局实现的一个自定义布局。AlphaOptimizedLinearLayout是继承自LinearLayout只是覆盖了

```java
public boolean hasOverlappingRendering() {  
    return false;  
}
```

该方法用来标记当前view是否存在过度绘制，存在返回ture，不存在返回false，默认返回为true。 在android的View里有透明度的属性，当设置透明度setAlpha的时候，android里默认会把当前view绘制到offscreen buffer中，然后再显示出来。 这个offscreen buffer 可以理解为一个临时缓冲区，把当前View放进来并做透明度的转化，然后在显示到屏幕上。这个过程是消耗资源的，所以应该尽量避免这个过程。而当继承了hasOverlappingRendering()方法返回false后，android会自动进行合理的优化，避免使用offscreen buffer。

系统icon绘制流程会比较多。 先从顶层view StatusIconContainer的绘制来分析。view的绘制离不开三个步骤，onMeasure, onLayout, onDraw，现在来一一拆解查看。

###### StatusIconContainer -- onMeasure

```java
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        // 获取到所有需要展示的view，注意看不可见的，icon处于blocked状态的，
        // 还有需要忽略的都不会被加入mMeasureViews中
        for (int i = 0; i < count; i++) {
            StatusIconDisplayable icon = (StatusIconDisplayable) getChildAt(i);
            if (icon.isIconVisible() && !icon.isIconBlocked()
                    && !mIgnoredSlots.contains(icon.getSlot())) {
                mMeasureViews.add((View) icon);
            }
        }

        int visibleCount = mMeasureViews.size();
        //  计算最大可见的icon数量，默认为7
        int maxVisible = visibleCount <= MAX_ICONS ? MAX_ICONS : MAX_ICONS - 1;
        int totalWidth = mPaddingLeft + mPaddingRight;
        boolean trackWidth = true;

        int childWidthSpec = MeasureSpec.makeMeasureSpec(width, MeasureSpec.UNSPECIFIED);
        mNeedsUnderflow = mShouldRestrictIcons && visibleCount > MAX_ICONS;
        for (int i = 0; i < mMeasureViews.size(); i++) {
            // Walking backwards
            View child = mMeasureViews.get(visibleCount - i - 1);
            //测量每个childview的宽
            measureChild(child, childWidthSpec, heightMeasureSpec);
            if (mShouldRestrictIcons) {
                // 计算总的宽度
                if (i < maxVisible && trackWidth) {
                    totalWidth += getViewTotalMeasuredWidth(child);
                } else if (trackWidth) {
                    // 超过最大可见数量时 需要给省略点计算空间。
                    totalWidth += mUnderflowWidth;
                    trackWidth = false;
                }
            } else {
                totalWidth += getViewTotalMeasuredWidth(child);
            }
        }
        // 通过setMeasuredDimension设置view的宽高
        if (mode == MeasureSpec.EXACTLY) {
            ...
            setMeasuredDimension(width, MeasureSpec.getSize(heightMeasureSpec));
        } else {
            ...
            setMeasuredDimension(totalWidth, MeasureSpec.getSize(heightMeasureSpec));
        }
    }
```

从上面可以看出来， onMeaure主要时计算每个子view的宽高，并计算出父view的整的宽度，其中会给超过最大数量的情况下 计算省略点的宽度，可以视项目情况来决定这个省略点的数量，其可在代码中通过常量来自定义。

###### StatusIconContainer -- onLayout

```java
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        float midY = getHeight() / 2.0f;

        for (int i = 0; i < getChildCount(); i++) {
            View child = getChildAt(i);
            int width = child.getMeasuredWidth();
            int height = child.getMeasuredHeight();
            int top = (int) (midY - height / 2.0f);
            child.layout(0, top, width, top + height);
        }
        // 重置每个view的状态。通过StatusIconState重置状态
        resetViewStates();
        // 重新依据实际情况计算每个icon的显示状态，下面单独拎出来讲。
        calculateIconTranslations();
        // 应用view的状态，包含icon显示的动画。
        applyIconStates();
    }
```

onLayou常规是计算每个view的宽高，并按预定的规则排放，然后计算每个view的位置。calculateIconTranslations显示逻辑会比较多，单独拎出来讲：

```java
    private void calculateIconTranslations() {
        mLayoutStates.clear();
        ...
        // 
        for (int i = childCount - 1; i >= 0; i--) {
            View child = getChildAt(i);
            StatusIconDisplayable iconView = (StatusIconDisplayable) child;
            StatusIconState childState = getViewStateFromChild(child);

            if (!iconView.isIconVisible() || iconView.isIconBlocked()
                    || mIgnoredSlots.contains(iconView.getSlot())) {
                childState.visibleState = STATE_HIDDEN;
                if (DEBUG) Log.d(TAG, "skipping child (" + iconView.getSlot() + ") not visible");
                continue;
            }

            childState.visibleState = STATE_ICON;
            // 位置显示的关键点， translationX 初始值是整个view的宽度，这样计算每个view
            // 的实际布局位置
            childState.xTranslation = translationX - getViewTotalWidth(child);
            mLayoutStates.add(0, childState);
            translationX -= getViewTotalWidth(child);
        }

        // Show either 1-MAX_ICONS icons, or (MAX_ICONS - 1) icons + overflow
        int totalVisible = mLayoutStates.size();
        int maxVisible = totalVisible <= MAX_ICONS ? MAX_ICONS : MAX_ICONS - 1;

        mUnderflowStart = 0;
        int visible = 0;
        int firstUnderflowIndex = -1;
        for (int i = totalVisible - 1; i >= 0; i--) {
            StatusIconState state = mLayoutStates.get(i);
            // Allow room for underflow if we found we need it in onMeasure
            // 这里比较关键 从列表中逆序获取到每个view的位置，如果view的xTranslation 下雨
            // 小于显示的内容就停止，后续就从这个index开始绘制
            if (mNeedsUnderflow && (state.xTranslation < (contentStart + mUnderflowWidth))||
                    (mShouldRestrictIcons && visible >= maxVisible)) {
                firstUnderflowIndex = i;
                break;
            }
            mUnderflowStart = (int) Math.max(contentStart, state.xTranslation - mUnderflowWidth);
            visible++;
        }

        //后续逻辑就是配置是否显示icon和显示多少个dot
        ...
    }
```

onLayout逻辑较多，简单来说就是通过每个子view的xTranslation和整体的view空间，计算需要显示多少icon，同时要给省略点预留空间。简单示意如下。可能超过空间的就用dot来显示。

![a28c56cd4d0f9b7451addeacade9ccad\_MD5](https://picgo.myjojo.fun:666/i/2024/10/29/6720998beeb1d.webp)

###### StatusIconContainer -- onDraw

这块没有定制处理，只是做了debug的一些信息绘制.

至此系统icon的顶层view分析完成，其主要是通过子view的状态以及父view的空间等情况来决定是否需要显示哪些icon，以及显示省略点符号。接下来再看每个子view的情况。

子view就是显示状态栏上icon，但是其封装了一层继承自AnimatedImageView，带动画效果的ImageView。 子view的具体实现 StatusBarIconView

SystemUI/src/com/android/systemui/statusbar/phone/StatusBarIconView.java

挑其中重点解析。图片的缩放，怎么把任意图片的大小适合在状态栏显示。

```java
    private void updateIconScaleForSystemIcons() {
        float iconHeight = getIconHeight();
        if (iconHeight != 0) {
            mIconScale = mSystemIconDesiredHeight / iconHeight;
        } else {
            mIconScale = mSystemIconDefaultScale;
        }
    }
```

先获取到mIconScale需要缩放的比例，mSystemIconDesiredHeight 是配置的全局的system icon的大小。

```java
        mSystemIconDesiredHeight = res.getDimension(
                com.android.internal.R.dimen.status_bar_system_icon_size);
```

存在icon的情况下，通过获取实际的icon的大小， 计算出 mIconScale.

在onDraw的时候 通过canvas.scale 把画布以icon的中心点根据mIconScale缩放到system\_icon\_size. 但是这样存在一个问题，icon的实际大小还是原大小，只是显示小了。其它部分包含动画就不再细讲。

```java
    @Override
    protected void onDraw(Canvas canvas) {
        if (mIconAppearAmount > 0.0f) {
            canvas.save();
            canvas.scale(mIconScale * mIconAppearAmount, mIconScale * mIconAppearAmount,
                    getWidth() / 2, getHeight() / 2);
            super.onDraw(canvas);
            canvas.restore();
        }
        ...
    }
```

到这里状态栏布局以及系统图标的view绘制大体分析完成。 接下来看icon是怎么控制添加，删除以及更新的。

### 状态栏图标显示逻辑控制

状态栏图标显示逻辑是通过 StatusBarIconControllerImpl 这个类来实现管理， 在对象构造的时候默认初始化

```java
    public StatusBarIconControllerImpl(Context context) {
        super(context.getResources().getStringArray(
                com.android.internal.R.array.config_statusBarIcons));
    ...
   }
```

config\_statusBarIcons 这个array中包含了所有支持的icon。如有需要定制图标顺序可在这个列表中对图标对应的item进行调整。

```java
   <string-array name="config_statusBarIcons">
        <item><xliff:g id="id">@string/status_bar_alarm_clock</xliff:g></item>
        ...
        <item><xliff:g id="id">@string/status_bar_battery</xliff:g></item>
        <item><xliff:g id="id">@string/status_bar_sensors_off</xliff:g></item>
   </string-array>
```

这些 icon字串信息当做一个title信息，保存在mSlots列表中。而Slot中包含StatusBarIconHolder:

```java
    public static class Slot {
        private final String mName;
        private StatusBarIconHolder mHolder;
        ...
    }
```

```java
public class StatusBarIconHolder {
    public static final int TYPE_ICON = 0;
    public static final int TYPE_WIFI = 1;
    public static final int TYPE_MOBILE = 2;

    private StatusBarIcon mIcon;
    private WifiIconState mWifiState;
    private MobileIconState mMobileState;
        ...
}
```

启动mIcon即为显示的图标资源保存类。其中包含了图标显示状态，标签信息以及状态信息等。将其都保存在mSlogs的列表中，方便管理显示。

总结数据保存链条 Slots--> StatusBarIconHolder --> StatusBarIcon;

##### 关键view管理

在状态栏初始化的时候 CollapsedStatusBarFragment 的view创建中 onViewCreated对系统icon管理进行初始化。

```java
        mDarkIconManager = new DarkIconManager(view.findViewById(R.id.statusIcons));
        mDarkIconManager.setShouldLog(true);
        Dependency.get(StatusBarIconController.class).addIconGroup(mDarkIconManager);
```

DarkIconManager 构造函数中传入了system\_icon的容器viewgroup, 负责view的增加和删除。StatusBarIconController管理DarkIconManager. 这样显示图标区域控制部分与显示部分关联起来。

##### 图标如何更新？

控制管理的实现策略类都在 PhoneStatusBarPolicy 这个里面实现。 具体实现通过StatusBarIconControllerImpl类实现，可以通过如下接口更新显示图标。

```java
mIconController.setIcon(mSlotVolume, volumeIconId, volumeDescription);
mIconController.setIconVisibility(mSlotVolume, volumeVisible);
```

以音量更新为例。setIcon流程分析。

```java
    @Override
    public void setIcon(String slot, int resourceId, CharSequence contentDescription) {        
//  检查是否在列表中存在holder，默认初始情况下是都没有holder的，需要新建
        int index = getSlotIndex(slot);
        StatusBarIconHolder holder = getIcon(index, 0);
        if (holder == null) {
            先通过resoureid和 contentDescription创建一个StatusBarIcon实例
            StatusBarIcon icon = new StatusBarIcon(UserHandle.SYSTEM, mContext.getPackageName(),
                    Icon.createWithResource(
                            mContext, resourceId), 0, 0, contentDescription);
            // 通过icon封装一个holder。
            holder = StatusBarIconHolder.fromIcon(icon);
            // 将holder赋值给mSlots
            setIcon(index, holder);
        } else {
            holder.getIcon().icon = Icon.createWithResource(mContext, resourceId);
            holder.getIcon().contentDescription = contentDescription;
            handleSet(index, holder);
        }
    }
```

传入slot为icon的title， resourceId为资源文件，contentDescription为描述字串。如果判断为没有holder就会新建一个holder类，并传入mSlots的列表中。

```java
    @Override
    public void setIcon(int index, @NonNull StatusBarIconHolder holder) {
        boolean isNew = getIcon(index, holder.getTag()) == null;
        super.setIcon(index, holder);

        if (isNew) { 
            // 通过tag判断如果是新的就加到systemicon中
            addSystemIcon(index, holder);
        } else {
            //已经存在的直接设置
            handleSet(index, holder);
        }
    }
```

```java
    private void addSystemIcon(int index, StatusBarIconHolder holder) {
        String slot = getSlotName(index);
        int viewIndex = getViewIndex(index, holder.getTag());
        boolean blocked = mIconBlacklist.contains(slot);

        mIconGroups.forEach(l -> l.onIconAdded(viewIndex, slot, blocked, holder));
    }
```

onIconAdded是在DarkIconManager中实现。

```java
        protected void onIconAdded(int index, String slot, boolean blocked,
                StatusBarIconHolder holder) {
            addHolder(index, slot, blocked, holder);
        }
```

```java
        protected StatusIconDisplayable addHolder(int index, String slot, boolean blocked,
                StatusBarIconHolder holder) {
            switch (holder.getType()) {
                case TYPE_ICON:
                    return addIcon(index, slot, blocked, holder.getIcon());

                case TYPE_WIFI:
                    return addSignalIcon(index, slot, holder.getWifiState());

                case TYPE_MOBILE:
                    return addMobileIcon(index, slot, holder.getMobileState());
            }

            return null;
        }
```

```java
        protected StatusBarIconView addIcon(int index, String slot, boolean blocked,
                StatusBarIcon icon) {
            StatusBarIconView view = onCreateStatusBarIconView(slot, blocked);
            view.set(icon); //mGroup 即为状态栏系统图标的容器view。这里就完成了view的添加
            mGroup.addView(view, index, onCreateLayoutParams());
            return view;
        }
```

这样就通过setIcon把图标添加到了系统图标区，然后再通过setIconVisibility显示出图标。显示的逻辑和setIcon差不多，只是增加了visible状态，可以自行分析。更新过程中有两个特殊的图标，wifi和数据网络，其状态会包含多个，正常都是只有显示与否逻辑，所以这里逻辑会多一些，但是原理一样的。

至此就完成了整个系统图标显示控制分析。

### 如何定制？

1. 数量和顺序 通过配置 config\_statusBarIcons 增删自己需要的图标。
2. 状态栏大小定制 framework/base/core/res/res/values/dimens.xml

| 配置项                                        | 说明                                      |
| ------------------------------------------ | --------------------------------------- |
| status\_bar\_height\_portrait              | 状态栏高度                                   |
| status\_bar\_system\_icon\_intrinsic\_size | 系统图标期望大小， 用于icon的缩放，和icon\_size设置一样大小即可 |
| status\_bar\_system\_icon\_size            | 系统图标大小                                  |

SystemUI/res/values/dimens.xml

| 配置项                               | 说明          |
| --------------------------------- | ----------- |
| status\_bar\_padding\_start       | 状态栏离左侧空间    |
| status\_bar\_padding\_end         | 状态栏离右侧空间    |
| signal\_cluster\_battery\_padding | 系统图标离电池图标距离 |

3. 图标显示定制 通过上述分析代码 setIcon找到对应的icon进行替换自己项目的icon， StatusIconContainer中需要修改MAX\_DOTS为0，不显示省略点。再onLayout的时候需要根据项目设置icon的间距， child.layout中增加r值。
4. 显示逻辑策略都在PhoneStatusBarPhicy中实现，尤其是系统原生没有支持的图标逻辑会定制较多。

#### 注意事项：

1. 图标选择，使用新的svg图标时， 宽高最好和系统system\_icon\_size设置为一致，原生逻辑会把图标缩放到高度和system\_icon一致，倒是宽度却保持了原有图标宽，导致显示布局不对。


---

> 作者: SugarTurboS  
> URL: https://huang8604.github.io/posts/android%E7%B3%BB%E7%BB%9F%E7%8A%B6%E6%80%81%E6%A0%8F%E5%AE%9A%E5%88%B6/  

