---
title: 【Android 12】认识窗口_android 窗口模式 窗口层级-CSDN博客
source: 
tags:
  - clippings
  - blog
  - 转载
collections:
  - WMS
  - 图形显示
  - 总结文章
date: 2024-09-24T06:59:16.867Z
lastmod: 2024-09-29T06:49:17.787Z
---
该文章为窗口层级结构系列文章的总结，重新回看这方面内容的时候我自己也有了一些新的感悟，希望通过本次总结能让大家再次对窗口有一个全面的认识。

一般来说，屏幕上最起码包含三个窗口，StatusBar窗口、Activity窗口以及NavigationBar窗口：

![4f668f131ab8d9e7aa48e7e95f1cb582\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f26539af544.png)

我们想要了解窗口，可以按照从小到大，从表到里的顺序进行：

1）、微观角度，探究单个窗口内部的组成。

2）、宏观角度，探究[WMS](https://so.csdn.net/so/search?q=WMS\&spm=1001.2101.3001.7020)如何管理屏幕上显示的诸多窗口。

3）、底层角度，探究SurfaceFlinger如何管理窗口，和WMS在这方面有何联系。

至于窗口，我们先理解它为一块在屏幕上可以显示图像的区域。

## 1 微观下的Window —— [View层](https://so.csdn.net/so/search?q=View%E5%B1%82\&spm=1001.2101.3001.7020)级结构

窗口本身作为View对象的容器只是一个抽象的概念，真正可见的则是View对象，那么我们探究单个窗口的表现形式，其实就是探究组成窗口的View层级结构。

这里我们写一个单选按钮组：

```xml
    <RadioGroup
        android:id="@+id/radio_group"
        android:layout_width="300dp"
        android:layout_height="wrap_content"
        android:layout_marginTop="50dp"
        android:layout_marginStart="20dp"
        android:background="@color/material_dynamic_neutral70">
        <RadioButton
            android:id="@+id/radio_button1"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="A" />
        <RadioButton
            android:id="@+id/radio_button2"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="B" />
        <RadioButton
            android:id="@+id/radio_button3"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="C" />
    </RadioGroup>
```

看起来是这样子：

![04da1b0ea8aa16695d8c98940d8c763c\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653975b0f.png)

这已经是一个简单的View层级结构了，父View为RadioGroup，其中有3个RadioButtion的子View，那么它的层级结构就是：

![8e6613b820567ff382a14ff8a6cb3980\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f26539a6c73.png)

我之前习惯用树状图的形式：

![2cea5bf3e35f9a66e0f0b6e045ca9d2b\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f265397a51b.png)

不过似乎还是第一种表现形式更好，能够更加直观的反映上下级的关系，后面描述层级结构的时候我仍然用树状图的一些叫法。

Android对此层级结构的建立提供了哪些支持呢？

### 1.1 每一个UI元素都是一个View

比如这里的单选按钮RadioButton类，它的继承关系为：

```
RadioButtion -> CompoundButton -> Button -> TextView -> View
```

RadioGroup类的继承关系为：

```
RadioGroup -> LinearLayout -> ViewGroup -> View
```

就像每一个类的基类是Object一样，每一个用来组成Activity界面的UI元素，其基类都是View类，因此我个人喜欢称这些由UI元素搭建的组织架构为View层级结构，层级结构里的每一个节点都是一个View。

### 1.2 父View和子View

以刚刚的单选按钮组为例，这里是一个RadioGroup包含了3个RadioButtion，父View为RadioGroup，子View为3个RadioButtion。

回看RadioGroup，它是一种特殊的View，继承自ViewGroup，ViewGroup顾名思义，就是一种能够包含其它View的特殊View，它有一个View数组类型的成员变量mChildren：

```java
// frameworks\base\core\java\android\view\ViewGroup.java

	// Child views of this ViewGroup
    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.P)
    private View[] mChildren;
```

用来保存它容纳的子View。

既然父View有一个mChildren用来拿到子View对象，那么子View也应该有一个成员变量用来保存父View的引用，实际上也的确如此，View中也定义了一个ViewParent类型的成员变量mParent用来指向当前View的父View：

```java
// frameworks\base\core\java\android\view\View.java

	/**
     * The parent this view is attached to.
     * {@hide}
     *
     * @see #getParent()
     */
    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.P)
    protected ViewParent mParent;
```

ViewParent是一个接口类，定义一个能够作为父View的类的通用接口，这里看到ViewGroup就实现了这个接口类：

```java
// frameworks\base\core\java\android\view\ViewGroup.java

public abstract class ViewGroup extends View implements ViewParent, ViewManager
```

那么为什么不让View来实现ViewParent接口，而让ViewGroup来实现呢？不难想到，对于参与建立View层级结构的View来说，其实可以分为两类，一类就像RadioButtion一样，它们是View层级结构中的最小单位，或者叫做View层级结构中的叶节点，无法作为父View再去容纳其它子View。而对于RadioGroup，它们作为父View可以容纳其它子View，在View层级结构中充当中间节点（当然也可以作为叶节点）。

最后说下如何向ViewGroup中添加子View，用的是ViewGroup.addChild方法：

```java
    /**
     * Adds a child view with the specified layout parameters.
     *
     * <p><strong>Note:</strong> do not invoke this method from
     * {@link #draw(android.graphics.Canvas)}, {@link #onDraw(android.graphics.Canvas)},
     * {@link #dispatchDraw(android.graphics.Canvas)} or any related method.</p>
     *
     * @param child the child view to add
     * @param index the position at which to add the child or -1 to add last
     * @param params the layout parameters to set on the child
     */
    public void addView(View child, int index, LayoutParams params)
```

最终会将child参数代表的子View加入到当前VIewGroup.mChildren中，同时将子View的mParent指向当前ViewGroup。

LayoutInflater解析xml文件，其实也是实例化xml中定义的View，然后通过ViewGroup.addChild方法将子View添加到父View中，以这种方式将xml文件中定义的View层级结构建立起来。

### 1.3 根View

对于所有组成同一个Activity界面的UI元素，它们都在同一个层级结构中，自然它们都有一个共同的根View，那这个根View是谁？

我们新建一个Activity，一般来说都在其onCreate方法中去加载其布局，比如像这样：

```java
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        setContentView(R.layout.activity_pip);
    }
```

那通过解析这个 **R.layout.activity\_pip** 对应的xml文件所生成的View就是根View吗，仍然拿这里的demo进行验证，我自己定义的actiivty\_pip.xml是一个LInearLayout，里面包含了两个单选按钮组。需要注意的是这个LinearLayout有一个不为“match\_parent”的自定义高度，并且设置了一个背景色，其层级结构为：

![c78614c049b115bf4a90e9443146e9ab\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653d29bea.png)

看下实际情况：

![6f8f12a4065cb1ebb77ea8c70e06233b\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f265397ac31.png)

对于它没有覆盖到的区域，仍有一个白色背景，说明这个LinearLayout并不是根View。

如果想要找到根View，就需要去跟踪Activity.setContentView流程，这个流程我之前有总结过文档，这里就不再具体叙述，直接放结论，是DecorView。

### 1.4 View层级结构的生成

看下View层级结构生成的流程，即Activity.setContentView方法流程，只说关键部分。

1）、创建DecorView实例。

![287600476edcb0f3518efa7629173829\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653abb68c.png)

2）、根据Activity请求的feature（比如是否需要标题、图标）来决定DecorView的直接子View的布局类型，我这里的是 **R.layout.screen\_simple**：

```xml
// frameworks/base/core/res/res/layout/screen_simple.xml

<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true"
    android:orientation="vertical">
    <ViewStub android:id="@+id/action_mode_bar_stub"
              android:inflatedId="@+id/action_mode_bar"
              android:layout="@layout/action_mode_bar"
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:theme="?attr/actionBarTheme" />
    <FrameLayout
         android:id="@android:id/content"
         android:layout_width="match_parent"
         android:layout_height="match_parent"
         android:foregroundInsidePadding="false"
         android:foregroundGravity="fill_horizontal|top"
         android:foreground="?android:attr/windowContentOverlay" />
</LinearLayout>
```

具体为一个LinearLayout包含一个ViewStub和FrameLayout。

接着将这个xml实例化为一个View层级结构，将该层级结构的根节点作为子View加入到DecorView中：

![c4b0c5bb05eba50cd36ef4558d5081f6\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653a343cb.png)

3）、接着通过LayoutInflater.inflate解析我们通过Activity.setContentView传入的xml文件，指定其父View为上一步解析的screen\_simple.xml中定义的ID为：

```java
    /**
     * The ID that the main layout in the XML layout file should have.
     */
    public static final int ID_ANDROID_CONTENT = com.android.internal.R.id.content;
```

的View，将我们自定义的布局作为子View添加到该View上，生成最终的层级结构：

![c79d19f54478e415157c2c9d22007e7c\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f265396f4a6.png)

用”Legacy Layout Inspector“插件看是：

![b3e6cd49666c99f75ac8c59838b378d5\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653cae4d0.png)

4）、回看我们之前的Activity界面：

![6f8f12a4065cb1ebb77ea8c70e06233b\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f265397ac31.png)

能看到顶部和底部分别还有一个区域，如果我们的Activity不要求隐藏状态栏和导航栏，那么会创建两个View：

* 状态栏背景区域，com.android.internal.R.id.statusBarBackground。
* 导航栏背景区域，com.android.internal.R.id.navigationBarBackground。

当状态栏窗口和导航栏窗口盖在我们的Activity窗口上时，这两个区域就是它们的背景。

DecorView将这两个View作为子View加入其中，和存放我们自定义布局的LinearLayout（内部层级被折叠）同级，如图所示：

![b868efa2a70f10754ddd5c52ee887261\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653a35c9f.png)

并且对LinearLayout设置内外边距，从而避免我们的自定义区域伸延到这些为系统窗口预留的背景区域中，以防我们期望显示的内容被这些系统窗口覆盖。

### 1.5 ViewRootImpl —— 实际的根节点？

ViewRootImpl，不管是开发App还是做Framework，都少不了跟它打交道，从名字上来看，它似乎是作为根View的角色所存在，那么事实上如何呢？

不管是Activity窗口还是非Activity窗口，最终都要通过ViewRootImpl.setView向WMS注册窗口：

```java
    /**
     * We have one child
     */
    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView,
            int userId) {
        synchronized (this) {
            if (mView == null) {
                mView = view;

                // ......
                
                try {
                    // ......
                    res = mWindowSession.addToDisplayAsUser(mWindow, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(), userId,
                            mInsetsController.getRequestedVisibilities(), inputChannel, mTempInsets,
                            mTempControls);
                    // ......
                }

                // ......

                view.assignParent(this);
                
                // ......
            }
        }
    }
```

值得注意的是以下几点：

1）、将View类型的传参view赋值给View类型的成员变量mView，这里的传参view是一个窗口的View层级结构的顶级View，对于Activity窗口来说，就是DecorView，而对于非Activity窗口，如状态栏和导航栏，这些窗口的View层级结构完全由它们自定义，不会像Activity窗口那样再由Framework给它们外面再套几个layout，这里的传参view就是这些窗口的自定义View层级结构的顶级View。

后续ViewRootImpl就可以通过其成员变量mView拿到顶级View。

2）、接着通过IWindowSession.addToDisplayAsUser向WMS添加窗口。

3）、最后调用View.assignParent方法，将传参View的mParent指向当前ViewRootImpl：

```java
    @UnsupportedAppUsage
    void assignParent(ViewParent parent) {
        if (mParent == null) {
            mParent = parent;
        }
        // .......
    }
```

那么对于Activity窗口来说，其DecorView还有一个父View，ViewRootImpl？

但是再看ViewRootImpl的定义：

```java
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks,
        AttachedSurfaceControl {
```

ViewRootImpl并不是一个View对象，因此它并没有一个可视的内容。

ViewRootImpl也不是一个ViewGroup，因此没有mChildren来保存其子View，所以它才新建了一个mView成员变量来保存其唯一的子View，对于Activity窗口来说，ViewRootImpl的唯一子View就是DecorView。

那么把ViewRootImpl视为真正的根View，虽也未尝不可，但是需要注意，我们之前说的以DecorView为根节点的View层级结构，它的每一个节点都是一个可视的View，而ViewRootImpl只是一个抽象的概念，相比于DecorView这种“实体”根节点，它是一个虚拟的根节点。这不禁让人思考，ViewRootImpl为啥要这么取名？

那就要看看ViewRootImpl扮演了怎样的角色，上面分析过，ViewRootImpl保存了View层级结构的顶级View的引用（为数不多的保存顶级View引用的地方之一），那么很多需要遍历View层级结构的流程都会从ViewRootImpl发起。比如，经典的measure流程，便是通过ViewRootImpl.performMeasure方法发起：

```java
    private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
        if (mView == null) {
            return;
        }
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
        try {
            mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
```

ViewRootImpl先调用其直接子View的measure方法，再由该子View调用子子View的measure方法，依次类推，最终整个View层级结构中的所有View的measure方法都能够得到调用。

除此之外，layout流程和draw流程，Input事件的传递以及Insets的更新，大部分需要遍历View层级结构的流程，起点都是在ViewRootImpl。

尽管如此，对于Activity窗口，我个人仍然倾向于称DecorView为的View层级结构的根View。

### 1.6 Freeform的实现

这里根据Freeform的实现方式，来验证一下我们上面的分析内容。

这里只关心Freeform是如何为Activity添加包含最大化按钮和退出按钮的标题栏。

如果让我们自己写一个这个形式的界面，也非常容易，只需要一个LInearLayout包含两个Button即可：

![8e71deb82fc1422616a9fb2acccf1e7a\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653971ad9.png)

但是如果想要为每一个进入Freeform的Activity都自动设置这样一个标题栏，就得靠Framework去统一处理。

google的做法如下：

1）、实例化一个DecorCaptionView，它通过xml文件定义：

```xml
 <com.android.internal.widget.DecorCaptionView xmlns:android="http://schemas.android.com/apk/res/android"
         android:orientation="vertical"
         android:layout_width="match_parent"
         android:layout_height="match_parent"
         android:descendantFocusability="beforeDescendants" >
     <LinearLayout
             android:id="@+id/caption"
             android:layout_width="match_parent"
             android:layout_height="wrap_content"
             android:gravity="end"
             android:background="@drawable/decor_caption_title"
             android:focusable="false"
             android:descendantFocusability="blocksDescendants" >
         <Button
                 android:id="@+id/maximize_window"
                 android:layout_width="32dp"
                 android:layout_height="32dp"
                 android:layout_margin="5dp"
                 android:padding="4dp"
                 android:layout_gravity="center_vertical|end"
                 android:contentDescription="@string/maximize_button_text"
                 android:background="@drawable/decor_maximize_button_dark" />
         <Button
                 android:id="@+id/close_window"
                 android:layout_width="32dp"
                 android:layout_height="32dp"
                 android:layout_margin="5dp"
                 android:padding="4dp"
                 android:layout_gravity="center_vertical|end"
                 android:contentDescription="@string/close_button_text"
                 android:background="@drawable/decor_close_button_dark" />
     </LinearLayout>
 </com.android.internal.widget.DecorCaptionView>
```

有一个LInearLayout，包含了两个按钮：

![066c0f0e154dcbe12642782a9fbebe49\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653ccf3a2.png)

2）、实例化一个DecorCaptionView后，我们原先的Activity窗口层级结构是（忽略具体的细节部分）：

![1d69839e33935224c327a4cb8b58e2cd\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653b09dd5.png)

接着DecorView将DecorCaptionView作为子View添加进来，接着将原先的唯一子View移除，并且将其作为子View添加到DecorCaptionView中，即：

![523cf1bcc657d70b3d7c2652634a35e1\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653c00bee.png)

## 2 宏观下的Window —— WindowContainer层级结构

上一节以微观的角度去探究一个窗口的组成，即View层级结构。

这一节以宏观的角度，把窗口当作WMS窗口体系下的最小单位，去探究WMS是如何管理窗口的。

### 2.1 窗口 —— WindowState类

```java
/** A window in the window manager. */
public class WindowState extends WindowContainer<WindowState> implements
    WindowManagerPolicy.WindowState, InsetsControlTarget {
```

在WMS窗口体系中，一个WindowState对象就代表了一个窗口。

这里看到，WindowState继承自WindowContainer。

### 2.2 窗口容器类 —— WindowContainer类

```java
/**
 * Defines common functionality for classes that can hold windows directly or through their
 * children in a hierarchy form.
 * The test class is {@link WindowContainerTests} which must be kept up-to-date and ran anytime
 * changes are made to this class.
 */
class WindowContainer<E extends WindowContainer> extends ConfigurationContainer<E>
        implements Comparable<WindowContainer>, Animatable, SurfaceFreezer.Freezable {
```

WindowContainer定义了能够直接或者间接以层级结构的形式持有窗口的类的通用功能。

从类的定义和名称，可以看到WindowContainer是一个容器类，可以容纳WindowContainer及其子类对象。如果有一个容器类想作为WindowState的容器，那么这个容器类需要继承WindowContainer或其子类。

继续看下，WindowContainer为了能够作为一个容器类，提供了哪些支持：

```java
    /**
     * The parent of this window container.
     * For removing or setting new parent {@link #setParent} should be used, because it also
     * performs configuration updates based on new parent's settings.
     */
    private WindowContainer<WindowContainer> mParent = null;

	// ......

    // List of children for this window container. List is in z-order as the children appear on
    // screen with the top-most window container at the tail of the list.
    protected final WindowList<E> mChildren = new WindowList<E>();
```

* mParent，保存的是当前WindowContainer的父WindowContainer的引用，可以类比于View.mParent。
* mChidlren，保存的是当前WindowContainer作为一个容器，所包含的子WindowContainer，可以类比于ViewGroup.mChildren。它是一个列表，列表的顺序反映了子容器在Z轴上的顺序，越靠近队尾层级越高。

这两个成员变量为生成WindowContainer层级结构提供了支持，就像根据View.mParent和ViewGroup.mChildren创建View层级结构那样。

容器这个概念非常重要，理解了它才能理解WindowContainer层级结构的组织形式。为了理解容器这个概念，可以把WindowContainer比作一个堆栈，这里举几个例子方便理解：

1）、当前WindowContainer已经连续添加了两个子容器，WindowContainer\_A以及WindowContainer\_B，继续添加WindowContainer\_C，就可以看作是像当前堆栈栈顶入栈一个WindowContainer\_C：

![e63d2ecb8dc1ee079b1a1124da832bfb\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653cae827.png)

这里用堆栈来表示一个WindowContainer也是因为堆栈比较能够直观的反映WindowContainer内部的Z轴顺序，WindowContainer层级结构和View层级结构是有相似之处的，ViewGroup也可以看作是View的容器，但是我个人感觉View层级结构中的Z轴这个概念稍微弱了一点，所以之前没有用堆栈的表现形式。

堆栈的表现形式虽好，但是当层级结构稍微复杂一点的话，画起来就非常麻烦，而且呈现出来的形式也不够直观，这里转为我个人比较习惯的层级结构图：

![3c4054cb6237f1160042185355eab86f\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653990a2c.png)

这个图其实也能反映层级顺序，这里是WindowContainer\_C高于WindowContainer\_B高于WindowContainer\_A。

2）、稍微扩展一点，当前WindowContainer有两个子WindowContainer，同时这两个子WindowContainer也分别有各自的子WindowContainer。

![c59730e96b83553e6d90223008383d22\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f26539905cc.png)

转为层级结构图：

![f0ea55f99dc3113b3cc5c25c5b8b6941\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653e09430.png)

有两点需要注意：

* 位于同一个父容器中的WindowContainer可以直接进行比较。在WindowContainer\_B中，很明显的能够看到WindowContainer\_G高于WindowContainer\_F高于WindowContainer\_E。
* 位于不同父容器的两个WindowContainer不能直接进行比较。这里如果我们想比较WindowContainer\_G和WindowContainer\_D谁的层级高，就需要比较它们两个的父容器谁的层级高。这里看到WindowContainer\_A和WindowContainer\_B都位于同一个父容器中，所以可以得出WindowContainer\_B高于WindowContainer\_A，进而WindowContainer\_G高于WindowContainer\_D。如果WindowContainer\_A和WindowContainer\_B也没有处于同一个父容器中，那么就得继续沿着层级结构图往上找，直到找出处于同一个节点下的两个父容器。

此时回看WindowState类的定义：

```java
public class WindowState extends WindowContainer<WindowState>
```

有了新的发现，即WindowState本身也是一个容器，不过这里指明了WindowState作为容器类可以持有的容器类型限定为WindowState类型，即WindowState作为父容器，其中只能存放WindowState类型的WindowContainer。

事实上也的确如此，Android是有父窗口和子窗口的概念的，比如打开一个Message的弹窗：

![610c256e7d8fc33bb404dd0b0edd8142\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653b2bfd2.jpg)

查看PopupWindow的信息：

```
 Window #6 Window{193e28f u0 PopupWindow:b98815a}:
    // ......
    mParentWindow=Window{bfc77c9 u0 com.google.android.apps.messaging/com.google.android.apps.messaging.ui.ConversationListActivity} mLayoutAttached=true
```

父窗口为Activity对应的窗口。

再查看层级信息：

```
          #0 6b8748a com.google.android.apps.messaging/com.google.android.apps.messaging.ui.ConversationListActivity type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
           #0 3c3dfc1 PopupWindow:a760a0c type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
```

转为WindowContainer层级结构图为：

![da23df46dee05b9efd998174878c86c8\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f26539865a2.png)

这里表示WindowState名字的形式一般是哈希值加上包名或类名或窗口名称。

### 2.3 WindowState的容器 —— WindowToken、ActivityRecord和WallpaperWindowToken

WindowContainer是能够直接或者间接持有WindowState的容器类的通用基类，间接持有WindowState的容器类暂且不提，肯定很多，那么可以直接持有WindowState的类有哪些呢？

答案是WindowToken（除了上面提到的子窗口的概念），并且WindowToken衍生了两个子类，ActivityRecord和WallpaperWindowToken，用来对窗口进行更精细的分类。

#### 2.3.1 App窗口和非App窗口

首先窗口可以分为App窗口和非App窗口：

* App窗口，当App创建Activity、Dialog或者PopupWindow的时候，系统会为自动为其创建一个对应的窗口，是普通的App窗口。
* 非App窗口，是系统为了特定目的而使用的特殊类型的窗口，可以理解为系统窗口，如状态栏、导航栏以及壁纸窗口那些。

具体如何划分，则是看窗口的类型，即定义在WindowManager.LayoutParams中的成员变量type：

```java
        @WindowType
        public int type;
```

窗口类型从FIRST\_APPLICATION\_WINDOW（1）到LAST\_APPLICATION\_WINDOW（99）的窗口为App窗口。

窗口类型从FIRST\_SYSTEM\_WINDOW（2000）到LAST\_SYSTEM\_WINDOW（2999）的窗口为系统窗口。

其实除此之外只有一种特殊类型的窗口，子窗口，窗口类型从FIRST\_SUB\_WINDOW（1000）到LAST\_SUB\_WINDOW（1999）。这个类型的窗口必须依附于一个父窗口，比如刚刚提到的PopupWindow。这类窗口属于App窗口还是非App窗口，完全看其父窗口归于哪类。

那么以此分类，非App窗口的容器为WindowToken，App窗口的容器为ActivityRecord（子窗口的容器为父窗口，WindowState）。

#### 2.3.2 进一步划分

非App窗口中一种特殊的窗口，壁纸窗口，由于其是作为Activity的背景存在的，因此它的层级应该始终低于App窗口。而除壁纸窗口之外的非App窗口，层级则高于App窗口。

那么我们结合窗口类型以及层级关系，可以对窗口进一步划分：

1）、App之上的窗口，父容器为WindowToken，如StatusBar：

```
       #0 WindowToken{ef15750 android.os.BinderProxy@4562c02} type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
        #0 d3cbd49 StatusBar type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
```

![b11cd9981323f0202467deccc92e64e6\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f265397c965.png)

2）、App窗口，父容器为ActivityRecord，如Message相关Activity的窗口：

```
         #0 ActivityRecord{bc0999c u0 com.google.android.apps.messaging/.main.MainActivity} t14} type=standard mode=fullscreen override-mode=undefined requested-bou
nds=[0,0][0,0] bounds=[0,0][720,1612]
          #0 adb45c7 com.google.android.apps.messaging/com.google.android.apps.messaging.main.MainActivity type=standard mode=fullscreen override-mode=undefined req
uested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
```

![8cf73ec89fb19817577700a5430e8378\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653c7e776.png)

3）、App之下的窗口，父容器为WallpaperWindowToken，如ImageWallpaper窗口：

```
        #0 WallpaperWindowToken{145ca49 token=android.os.Binder@a727850} type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][0,0] bounds=
[0,0][720,1612]
         #0 ae9c947 com.android.systemui.ImageWallpaper type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
```

![08dfc70c6cd9c4d4527be5fbf0f60186\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653ccf44a.png)

### 2.4 ActivityRecord的容器 —— Task

按理说这一步应该继续分析WindowToken的父容器了，但是ActivityRecord作为一种特殊的WindowToken，它自己有一个特殊的父容器，Task。

Task类的定义为：

```java
class Task extends WindowContainer<WindowContainer> {
```

再次温习一下google关于Task的说明：

> 任务（Task）是用户在执行某项工作时与之互动的一系列 Activity 的集合。这些 Activity 按照每个 Activity 打开的顺序排列在一个返回堆栈中。例如，电子邮件应用可能有一个 Activity 来显示新邮件列表。当用户选择一封邮件时，系统会打开一个新的 Activity 来显示该邮件。这个新的 Activity 会添加到返回堆栈中。如果用户按**返回**按钮，这个新的 Activity 即会完成并从堆栈中退出。
>
> 大多数Task都从Launcher上启动。当用户轻触Launcher中的图标（或主屏幕上的快捷方式）时，该应用的Task就会转到前台运行。如果该应用没有Task存在（应用最近没有使用过），则会创建一个新的Task，并且该应用的“主”Activity 将会作为堆栈的根 Activity 打开。
>
> 在当前 Activity 启动另一个 Activity 时，新的 Activity 将被推送到堆栈顶部并获得焦点。上一个 Activity 仍保留在堆栈中，但会停止。当 Activity 停止时，系统会保留其界面的当前状态。当用户按**返回**按 钮时，当前 Activity 会从堆栈顶部退出（该 Activity 销毁），上一个 Activity 会恢复（界面会恢复到上一个状态）。堆栈中的 Activity 永远不会重新排列，只会被送入和退出，在当前 Activity 启动时被送入堆栈，在用户使用**返回**按钮离开时从堆栈中退出。因此，返回堆栈按照“后进先出”的对象结构运作。图 1 借助一个时间轴直观地显示了这种行为。该时间轴显示了 Activity 之间的进展以及每个时间点的当前返回堆栈。
>
> ![98edd96b24568f2f0c4c2b618efee845\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f26539dd33e.png)
>
> **图 1.** 有关Task中的每个新 Activity 如何添加到返回堆栈的图示。当用户按**返回**按钮时，当前 Activity 会销毁，上一个 Activity 将恢复。
>
> 如果用户继续按**返回**，则堆栈中的 Activity 会逐个退出，以显示前一个 Activity，直到用户返回到主屏幕（或Task开始时运行的 Activity）。移除堆栈中的所有 Activity 后，该Task将不复存在。

WMS中的Task类的概念和以上说明基本一致，用来管理ActivityRecord。

还是上面的Message，情况是：

```
        #3 Task=14 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
         #0 ActivityRecord{bc0999c u0 com.google.android.apps.messaging/.main.MainActivity} t14} type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
          #0 adb45c7 com.google.android.apps.messaging/com.google.android.apps.messaging.main.MainActivity type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
```

![04054062e552625b4a6c1f8800ca0c82\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653d2a8d4.png)

接着再启动Message的另外一个界面：

```
        #3 Task=14 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
         #1 ActivityRecord{357651e u0 com.google.android.apps.messaging/.ui.appsettings.ApplicationSettingsActivity} t14} type=standard mode=fullscreen override-mod
e=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
          #0 e611691 com.google.android.apps.messaging/com.google.android.apps.messaging.ui.appsettings.ApplicationSettingsActivity type=standard mode=fullscreen ov
erride-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
         #0 ActivityRecord{bc0999c u0 com.google.android.apps.messaging/.main.MainActivity} t14} type=standard mode=fullscreen override-mode=undefined requested-bou
nds=[0,0][0,0] bounds=[0,0][720,1612]
          #0 adb45c7 com.google.android.apps.messaging/com.google.android.apps.messaging.main.MainActivity type=standard mode=fullscreen override-mode=undefined req
uested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
```

![2bdc29c8b3379416efacbf8b0e373bf8\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653c7abf6.png)

新的ActivityRecord被置于栈顶，盖在之前的ActivityRecord上。

#### Task的嵌套

Task是支持嵌套的，从Task类的定义也可以看出来（Task是WindowContainer的容器，而非只能容纳ActivityRecord）。

1）、看下当只有一个Launcher的情况：

```
        #2 Task=1 type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         #0 Task=50 type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
          #0 ActivityRecord{b864ccc u0 com.tcl.android.launcher/.uioverrides.TclQuickstepLauncher} t50} type=home mode=fullscreen override-mode=undefined requested-
bounds=[0,0][0,0] bounds=[0,0][1080,2400] 
```

![aba0074f9a56db9ffb379eeca4bbcf15\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f26539942a4.png)

这里的示意图忽略了ActivityRecord的下一级，WindowState。

只看到这里可能无法理解Task嵌套的意义何在。

2）、接着启动Message：

```
       #1 DefaultTaskDisplayArea type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
        #3 Task=61 type=standard mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         #0 ActivityRecord{fac1d1c u0 com.google.android.apps.messaging/.ui.ConversationListActivity} t61} type=standard mode=fullscreen override-mode=undefined req
uested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
        #2 Task=1 type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
         #0 Task=50 type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
          #0 ActivityRecord{b864ccc u0 com.tcl.android.launcher/.uioverrides.TclQuickstepLauncher} t50} type=home mode=fullscreen override-mode=undefined requested-
bounds=[0,0][0,0] bounds=[0,0][1080,2400]
```

![fc4a1450756b656409e9aedbaa2f31e9\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653bab3d3.png)

此时并没有发生Task的嵌套，新启动的Message对应的Task#61并没有启动到Task#1中，而是并入了DefaultTaskDisplayArea中。

3）、如果我们此时再切换到NexusLauncher：

```
        #5 Task=1 type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
         #1 Task=47 type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
          #0 ActivityRecord{31ce9e6 u0 com.google.android.apps.nexuslauncher/.NexusLauncherActivity t47} type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1440,2960]
         #0 Task=50 type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1080,2400]
          #0 ActivityRecord{b864ccc u0 com.tcl.android.launcher/.uioverrides.TclQuickstepLauncher} t50} type=home mode=fullscreen override-mode=undefined requested-
bounds=[0,0][0,0] bounds=[0,0][1080,2400]    
```

![e4d0fe72b7b5ef3977a98290fbedc63a\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f26539e1b68.png)

看到新启动的Task#47，嵌套到了Task#1中，也就是作为子节点，添加到了容器Task#1中。

为什么刚刚启动的Message的Task，Task#61，没有加入到Task#1中，而NexusLauncher对应的Task#47，却加入到了Task#1中呢？这里要说下ActivityRecord的几种类型：

```java
    /** The current activity type of the configuration. */
    private @ActivityType int mActivityType;

    /** Activity type is currently not defined. */
    public static final int ACTIVITY_TYPE_UNDEFINED = 0;
    /** Standard activity type. Nothing special about the activity... */
    public static final int ACTIVITY_TYPE_STANDARD = 1;
    /** Home/Launcher activity type. */
    public static final int ACTIVITY_TYPE_HOME = 2;
    /** Recents/Overview activity type. There is only one activity with this type in the system. */
    public static final int ACTIVITY_TYPE_RECENTS = 3;
    /** Assistant activity type. */
    public static final int ACTIVITY_TYPE_ASSISTANT = 4;
    /** Dream activity type. */
    public static final int ACTIVITY_TYPE_DREAM = 5;
```

大部分普通App的Activity，都是ACTIVITY\_TYPE\_STANDARD，如注释所说，标准的Activity类型， 没有什么特别的。

而对于Home/Launcher类型的App，它们的Activity都是ACTIVITY\_TYPE\_HOME类型，对于这类App对应的Task，系统会在开机的时候创建一个该类型的Task，作为该类型的RootTask，用来统一管理该类型的Task。因此，如果我们又启动了第三个Launcher，该Launcher对应的Task，也会作为子节点添加到Task#1中。

但是不会出现一个Task的子容器既有Task，又有ActivityRecord的情况（至少现在不会）。

### 2.5 Task的容器 —— TaskDisplayArea

```java
/**
 * {@link DisplayArea} that represents a section of a screen that contains app window containers.
 *
 * The children can be either {@link Task} or {@link TaskDisplayArea}.
 */
final class TaskDisplayArea extends DisplayArea<WindowContainer>
```

TaskDisplayArea代表了屏幕上的一个包含App类型的WindowContainer的区域。它的子节点可以是Task，或者是TaskDisplayArea。但是目前在代码中，我看到创建TaskDisplayArea的地方只有一处，该处创建了一个名为“DefaultTaskDisplayArea”的TaskDisplayArea对象，除此之外并没有发现其他地方有创建TaskDisplayArea对象的地方，自然也没有找到有关TaskDisplayArea嵌套的痕迹。

因此目前可以说，TaskDisplayArea存放Task的容器。

在手机的近期任务列表界面将所有App都清掉后，查看一下此时的TaskDisplayArea情况：

```
       #0 DefaultTaskDisplayArea type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
        #5 Task=1 type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
         #0 Task=8 type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
          #0 ActivityRecord{5e73ff4 u0 com.tcl.android.launcher/.Launcher t8} type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
           #1 ca4f5c4 com.tcl.android.launcher/com.tcl.android.launcher.Launcher type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
           #0 5784a50 com.tcl.android.launcher/com.tcl.android.launcher.Launcher type=home mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
        #4 Task=2 type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
        #3 Task=3 type=undefined mode=multi-window override-mode=multi-window requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
        #2 Task=4 type=undefined mode=multi-window override-mode=multi-window requested-bounds=[0,0][0,0] bounds=[0,0][1920,1200]
        #1 Task=5 type=undefined mode=split-screen-primary override-mode=split-screen-primary requested-bounds=[0,0][950,1200] bounds=[0,0][950,1200]
        #0 Task=6 type=undefined mode=split-screen-secondary override-mode=split-screen-secondary requested-bounds=[970,0][1920,1200] bounds=[970,0][1920,1200]
```

![06dc420ea311f582c5b66e1b7bd56916\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653b46d83.png)

预期应该是DefaultTaskDisplayArea的子容器，只有Launcher应用对应的Task#1，但是却多Task#2、Task#3、Task#4、Task#5和Task#6这几个Task，并且这几个Task并没有持有任何一个ActivityRecord对象。正常来说，App端调用startActivity后，WMS会为新创建的ActivityRecord对象创建Task，用来存放这个ActivityRecord。当Task中的所有ActivityRecord都被移除后，这个Task就会被移除，比如Message对应的Task。

但是对于Task#1之类的Task来说并不是这样。这些Task是WMS启动后就由TaskOrganizerController创建的，这些Task并没有和某一具体的App联系起来，因此当它里面的子WindowContainer被移除后，这个Task也不会被销毁。比如分屏Task#5和Task#6：

```
        #1 Task=5 type=undefined mode=split-screen-primary override-mode=split-screen-primary requested-bounds=[0,0][950,1200] bounds=[0,0][950,1200]
        #0 Task=6 type=undefined mode=split-screen-secondary override-mode=split-screen-secondary requested-bounds=[970,0][1920,1200] bounds=[970,0][1920,1200]
```

这两个Task由TaskOrganizerController启动，用来管理系统进入分屏后，需要跟随系统进入分屏模式的那些App对应的Task，也就是说这些App对应的Task的父容器会从DefaultTaskDisplayArea变为Task#3和Task#2。

#### Task的调度

Task的调度其实也非常简单，这里拿一个实际的例子说明。

1）、我先从Launcher启动Message，接着通过adb启动我自己写的DemoApp，那么此时这三个Task的顺序从高到低分别是：DemoApp -> Message -> Launcher，如图所示：

![10ab48a267449af19ef9601fb2c11f99\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653bc28f7.png)

2）、接着我在DemoApp界面下点击Back键，此时的行为是：

![3f2c242d9b3c904fd30d023a38a11dfb\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653bc225f.png)

DemoApp所在的Task#64，跑到了TaskDisplayArea的栈底，其它Task顺序不变，那么最终是Message出现在了前台。

这是因为我们重写了Activity的onBackPressed方法：

```java
    @Override
    public void onBackPressed() {
        moveTaskToBack(true);
    }
```

当接收到Back事件的时候，将Activity所在的Task移动到最后，具体的行为就是移动到父容器的栈底，也就是TaskDisplayArea的栈底。

3）、这里我们更改onBackPressed方法的内容：

```java
    @Override
    public void onBackPressed() {
        Intent intent = new Intent(Intent.ACTION_MAIN);
        intent.addCategory(Intent.CATEGORY_HOME);
        intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        startActivity(intent);
    }
```

在接收到Back键时启动Launcher，那么当我们再次点击Back键时，行为是：

![f875ece50a198036a5db3d263f8bccd3\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653ca3402.png)

看到Launche所在的Task#50，以及RootTask，Task#1，一起移动到了前台，而其它Task的顺序不变。

4）、最后能看到，同样的操作，并且肉眼看到了同样的行为，但是TaskDisplayArea内部的Task的顺序却不一样：一个是将前台App移动到栈底，一个是将后台App移动到前台，所以具体问题还是要具体分析。

### 2.6 DisplayArea层级结构

能够容纳非App窗口的父容器为DisplayArea.Tokens：

```java
    /**
     * DisplayArea that contains WindowTokens, and orders them according to their type.
     */
    public static class Tokens extends DisplayArea<WindowToken> {
```

容纳App窗口的父容器则是Task，容纳Task的则是TaskDisplayArea，TaskDisplayArea也是DisplayArea的一个子类：

```java
final class TaskDisplayArea extends DisplayArea<WindowContainer>
```

那么我们接下来研究的对象就是DisplayArea。

DisplayArea本身也非常复杂，有很多子类：

![2b9c57e788a028723ef9bb55a097c289\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653ccccb3.png)

这里只说一下比较重要的几个类：

1）、能直接容纳WindowToken的容器，除了刚刚说的DisplayArea.Tokens类和TaskDisplayArea（严谨点说TaskDisplayArea是Task的容器，Task才是ActivityRecord的容器）之外，还有一个DisplayArea.Tokens类的子类，定义在DisplayContent中的内部类ImeContainer，专门用来容纳输入法窗口。这三个类由于能够容纳WindowToken，所以在DisplayArea层级结构中作为叶节点存在（最小单位，无法再容纳其它DisplayArea对象），就像WindowState是作为WindowContainer层级结构的最小单位一样。

2）、DisplayContent，代表一个屏幕，作为DisplayArea层级结构的根节点所存在。

#### 2.6.1 窗口层级值

我们都知道状态栏窗口的层级是比普通的App窗口层级高的，但是为什么是这样的，不知道大家有没有想过。

看一个方法，WindowManagerPolicy.getWindowLayerFromTypeLw：

```java
    /**
     * Returns the layer assignment for the window type. Allows you to control how different
     * kinds of windows are ordered on-screen.
     *
     * @param type The type of window being assigned.
     * @param canAddInternalSystemWindow If the owner window associated with the type we are
     *        evaluating can add internal system windows. I.e they have
     *        {@link Manifest.permission#INTERNAL_SYSTEM_WINDOW}. If true, alert window
     *        types {@link android.view.WindowManager.LayoutParams#isSystemAlertWindowType(int)}
     *        can be assigned layers greater than the layer for
     *        {@link android.view.WindowManager.LayoutParams#TYPE_APPLICATION_OVERLAY} Else, their
     *        layers would be lesser.
     * @param roundedCornerOverlay {#code true} to indicate that the owner window is rounded corner
     *                             overlay.
     * @return int An arbitrary integer used to order windows, with lower numbers below higher ones.
     */
    default int getWindowLayerFromTypeLw(int type, boolean canAddInternalSystemWindow,
            boolean roundedCornerOverlay) {
        // Always put the rounded corner layer to the top most.
        if (roundedCornerOverlay && canAddInternalSystemWindow) {
            return getMaxWindowLayer();
        }
        if (type >= FIRST_APPLICATION_WINDOW && type <= LAST_APPLICATION_WINDOW) {
            return APPLICATION_LAYER;
        }

        switch (type) {
            case TYPE_WALLPAPER:
                // wallpaper is at the bottom, though the window manager may move it.
                return  1;
            case TYPE_PRESENTATION:
            case TYPE_PRIVATE_PRESENTATION:
            case TYPE_DOCK_DIVIDER:
            case TYPE_QS_DIALOG:
            case TYPE_PHONE:
                return  3;
            case TYPE_SEARCH_BAR:
            case TYPE_VOICE_INTERACTION_STARTING:
                return  4;
            case TYPE_VOICE_INTERACTION:
                // voice interaction layer is almost immediately above apps.
                return  5;
            case TYPE_INPUT_CONSUMER:
                return  6;
            case TYPE_SYSTEM_DIALOG:
                return  7;
            case TYPE_TOAST:
                // toasts and the plugged-in battery thing
                return  8;
            case TYPE_PRIORITY_PHONE:
                // SIM errors and unlock.  Not sure if this really should be in a high layer.
                return  9;
            case TYPE_SYSTEM_ALERT:
                // like the ANR / app crashed dialogs
                // Type is deprecated for non-system apps. For system apps, this type should be
                // in a higher layer than TYPE_APPLICATION_OVERLAY.
                return  canAddInternalSystemWindow ? 13 : 10;
            case TYPE_APPLICATION_OVERLAY:
                return  12;
            case TYPE_INPUT_METHOD:
                // on-screen keyboards and other such input method user interfaces go here.
                return  15;
            case TYPE_INPUT_METHOD_DIALOG:
                // on-screen keyboards and other such input method user interfaces go here.
                return  16;
            case TYPE_STATUS_BAR:
                return  17;
            case TYPE_STATUS_BAR_ADDITIONAL:
                return  18;
            case TYPE_NOTIFICATION_SHADE:
                return  19;
            case TYPE_STATUS_BAR_SUB_PANEL:
                return  20;
            case TYPE_KEYGUARD_DIALOG:
                return  21;
            case TYPE_VOLUME_OVERLAY:
                // the on-screen volume indicator and controller shown when the user
                // changes the device volume
                return  22;
            case TYPE_SYSTEM_OVERLAY:
                // the on-screen volume indicator and controller shown when the user
                // changes the device volume
                return  canAddInternalSystemWindow ? 23 : 11;
            case TYPE_NAVIGATION_BAR:
                // the navigation bar, if available, shows atop most things
                return  24;
            case TYPE_NAVIGATION_BAR_PANEL:
                // some panels (e.g. search) need to show on top of the navigation bar
                return  25;
            case TYPE_SCREENSHOT:
                // screenshot selection layer shouldn't go above system error, but it should cover
                // navigation bars at the very least.
                return  26;
            case TYPE_SYSTEM_ERROR:
                // system-level error dialogs
                return  canAddInternalSystemWindow ? 27 : 10;
            case TYPE_MAGNIFICATION_OVERLAY:
                // used to highlight the magnified portion of a display
                return  28;
            case TYPE_DISPLAY_OVERLAY:
                // used to simulate secondary display devices
                return  29;
            case TYPE_DRAG:
                // the drag layer: input for drag-and-drop is associated with this window,
                // which sits above all other focusable windows
                return  30;
            case TYPE_ACCESSIBILITY_OVERLAY:
                // overlay put by accessibility services to intercept user interaction
                return  31;
            case TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY:
                return 32;
            case TYPE_SECURE_SYSTEM_OVERLAY:
                return  33;
            case TYPE_BOOT_PROGRESS:
                return  34;
            case TYPE_POINTER:
                // the (mouse) pointer layer
                return  35;
            default:
                Slog.e("WindowManager", "Unknown window type: " + type);
                return 3;
        }
    }
```

这个方法返回给定窗口类型对应的层级值。允许你控制不同类型的窗口在屏幕上的排序方式。

看方法内容很简单，对每一种类型的窗口，都规定了一个层级值。层级值反映了这些窗口在Z轴上的排序方式，层级值越高在Z轴上的位置也就越高。但是层级值的大小和窗口类型对应的那个值的大小并没有关系，窗口类型值大的窗口，对应的层级值不一定大。

另外，大部分层级值都唯一对应一种窗口类型，比较特殊的是：

* 层级值2，所有App窗口会被归类到这一个层级。
* 层级值3，该方法中没有提到的窗口类型会被归类到这一个层级。

这里看到状态栏窗口对应的窗口层级值是17，而普通的App窗口的窗口层级值是2，自然状态栏窗口就会比App窗口层级值高。

另外这里也能看到，窗口类型的数值和窗口层级值不是一个正相关的关系，即窗口类型值大的窗口，它在Z轴上的层级并不一定就高，窗口在Z轴的排序具体要看该窗口的类型值转换后得到的窗口层级值。

#### 2.6.2 DisplayArea层级结构的生成

接下来继续分析窗口层级值是如何作用的，但是仍有一些内容需要提前了解，即DisplayArea按照具体作用，又可以分为以下几种类型：

```java
   /**
     * The value in display area indicating that no value has been set.
     */
    public static final int FEATURE_UNDEFINED = -1;

    /**
     * The Root display area on a display
     */
    public static final int FEATURE_SYSTEM_FIRST = 0;

    /**
     * The Root display area on a display
     */
    public static final int FEATURE_ROOT = FEATURE_SYSTEM_FIRST;

    /**
     * Display area hosting the default task container.
     */
    public static final int FEATURE_DEFAULT_TASK_CONTAINER = FEATURE_SYSTEM_FIRST + 1;

    /**
     * Display area hosting non-activity window tokens.
     */
    public static final int FEATURE_WINDOW_TOKENS = FEATURE_SYSTEM_FIRST + 2;

    /**
     * Display area for one handed feature
     */
    public static final int FEATURE_ONE_HANDED = FEATURE_SYSTEM_FIRST + 3;

    /**
     * Display area that can be magnified in
     * {@link Settings.Secure.ACCESSIBILITY_MAGNIFICATION_MODE_WINDOW}. It contains all windows
     * below {@link WindowManager.LayoutParams#TYPE_ACCESSIBILITY_MAGNIFICATION_OVERLAY}.
     */
    public static final int FEATURE_WINDOWED_MAGNIFICATION = FEATURE_SYSTEM_FIRST + 4;

    /**
     * Display area that can be magnified in
     * {@link Settings.Secure.ACCESSIBILITY_MAGNIFICATION_MODE_FULLSCREEN}. This is different from
     * {@link #FEATURE_WINDOWED_MAGNIFICATION} that the whole display will be magnified.
     * @hide
     */
    public static final int FEATURE_FULLSCREEN_MAGNIFICATION = FEATURE_SYSTEM_FIRST + 5;

    /**
     * Display area for hiding display cutout feature
     * @hide
     */
    public static final int FEATURE_HIDE_DISPLAY_CUTOUT = FEATURE_SYSTEM_FIRST + 6;

    /**
     * Display area that the IME container can be placed in. Should be enabled on every root
     * hierarchy if IME container may be reparented to that hierarchy when the IME target changed.
     * @hide
     */
    public static final int FEATURE_IME_PLACEHOLDER = FEATURE_SYSTEM_FIRST + 7;

    /**
     * Display area for one handed background layer, which preventing when user
     * turning the Dark theme on, they can not clearly identify the screen has entered
     * one handed mode.
     * @hide
     */
    public static final int FEATURE_ONE_HANDED_BACKGROUND_PANEL = FEATURE_SYSTEM_FIRST + 8;

    /**
     * The last boundary of display area for system features
     */
    public static final int FEATURE_SYSTEM_LAST = 10_000;
```

看一下这些Feature的含义都是什么：

* FEATURE\_ROOT，一个屏幕上的根DisplayArea，对应DisplayContent。
* FEATURE\_DEFAULT\_TASK\_CONTAINER，容纳默认Task容器的DisplayArea，对应TaskDisplayArea。
* FEATURE\_WINDOW\_TOKENS，容纳非activity窗口的DisplayArea，对应DisplayArea.Tokens。
* FEATURE\_ONE\_HANDED，用于单手模式的DisplayArea，对应名为“OneHanded”的DisplayArea。
* FEATURE\_WINDOWED\_MAGNIFICATION，在ACCESSIBILITY\_MAGNIFICATION\_MODE\_WINDOW模式下可以对窗口的某些区域进行放大的DisplayArea，对应名为“WindowedMagnification”的DisplayArea。
* FEATURE\_FULLSCREEN\_MAGNIFICATION，在ACCESSIBILITY\_MAGNIFICATION\_MODE\_FULLSCREEN模式下可以对整个屏幕进行放大的DisplayArea，对应名为“FullscreenMagnification”的DisplayArea。
* FEATURE\_HIDE\_DISPLAY\_CUTOUT，隐藏DisplayCutout的DisplayArea，对应名为“HideDisplayCutout”的DisplayArea。
* FEATURE\_IME\_PLACEHOLDER，存放输入法窗口的DisplayArea，对应名为“ImePlaceholder”的DisplayArea。
* FEATURE\_ONE\_HANDED\_BACKGROUND\_PANEL，容纳单手背景图层的DisplayArea，避免用户开启暗黑模式后，无法分辨屏幕是否已经进入了单手模式，对应名为“OneHandedBackgroundPanel”的DisplayArea。

其中DisplayContent作为DisplayArea根节点，TaskDisplayArea和DisplayArea.Tokens（它还有一个ImeContainer子类）作为叶节点，其它类型的DisplayArea作为中间节点。

举个例子来说明一下DisplayArea层级结构的生成规则。

1）、假如我们现在有一个能够容纳窗口层级值等于17的DisplayArea.Tokens，那么可以将它命名为“Leaf:17:17”，代表它是一个DisplayArea叶节点，并且能够容纳的窗口层级值的最大值和最小值都是17，即它只能容纳窗口层级值为17的窗口：

![f677c269210a4858c4b3088f4fa3d78c\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653cbbffe.png)

2）、由于它是作为叶节点存在的，本身只作为一个容器存在，如果我们想要让它具备一些其它的功能，如能够容纳输入法，那么就需要为它增加一个FEATURE\_IME\_PLACEHOLDER类型的父容器，子容器会继承父容器的特性：

![9c192649751aeeb87f102ad049b40676\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653975cbf.png)

3）、同理，如果我们想要让它拥有定义的所有功能，那么就需要为它添加每一个feature对应的节点。最后也别忘了根节点，这样才能形成一个完成的DisplayArea层级结构，即从根节点到叶节点：

![3dddc9a4c2199fdad6fc09cdb72663c0\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653977c63.png)

4）、再来考虑这样一种情况，假如有两个相邻的Leaf，“Leaf:16:16”和“Leaf:17:17”，它们本来分别是有一个ImePlaceholder类型的父容器：

![f6198fe6471319a613c43d62ccb19137\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653e526cf.png)

但是看到它们都的父容器都是ImePlaceholder类型的，代表它们都希望拥有同一种功能，那么这里可以把它们进行合并，就像这样：

![15b875a954d8e6682c5e94f2c93a7f46\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f265397b811.png)

毕竟既然可以重复利用，那就没有必要创建额外的节点。

注意这里必须要是相邻的节点才行，如果是这样：

![ae77f35db539aa2714fed30dbf6f0909\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653bc454d.png)

这里的“Leaf:15:15”和“Leaf:17:17”的父节点是无法合并的，就只能为它们分别创建一个FullscreenMagnification类型的父容器。

5）、了解了以上内容，现在我们可以来分析一下DisplayArea的生成规则。

由于总共有37个窗口层级值，那么最初可能就会有37个独立并且完整的DisplayArea层级结构：

![0aa33a9853a8250cef181173cfdc21f1\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653e23b49.png)

这里的每一列都代表了一个独立并且完整的DisplayArea层级结构，每一个有颜色的格子都代表一个DisplayArea节点，从根节点DisplayContent一直到叶节点Leaf。

Leaf所在的层级结构代表了它所容纳的窗口所具有的一些功能。由于窗口都是有特定的用途，所以肯定不是所有的窗口都能作为输入法窗口，那么对于大部分层级结构，需要将删去ImePlaceholder类型的节点。同样，有一些窗口也不希望具备放大功能，那么它们的层级结构就可能需要删去WindowMagnification或者FullscreenMagnification类型的节点。

根据Android对各种窗口层级值可以具备的功能定义：

| Feature.mName            | Feature.mID                             | Feature.mWindowLayers                          |
| ------------------------ | --------------------------------------- | ---------------------------------------------- |
| WindowedMagnification    | FEATURE\_WINDOWED\_MAGNIFICATION        | 0~31                                           |
| HideDisplayCutout        | FEATURE\_HIDE\_DISPLAY\_CUTOUT          | 0<sub>16,18,20</sub>23,26~35                   |
| OneHandedBackgroundPanel | FEATURE\_ONE\_HANDED\_BACKGROUND\_PANEL | 0~1                                            |
| OneHanded                | FEATURE\_ONE\_HANDED                    | 0<sub>23,26</sub>35                            |
| FullscreenMagnification  | FEATURE\_FULLSCREEN\_MAGNIFICATION      | 0<sub>14,17</sub>23,26<sub>27,29</sub>31,33~35 |
| ImePlaceholder           | FEATURE\_IME\_PLACEHOLDER               | 15~16                                          |

我们需要删除一些节点，删除后的情况是（这里把ImeContainer和TaskDisplayArea也标记了出来）：

![0f42d5fb133827ede55607e974ddbb66\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f26539757f2.png)

空出的格子代表了该层级结构不需要的节点，由于该节点被删除，所以需要将该节点的父节点与该节点的子节点进行连接，即将有颜色的格子上移来填补表里的空白格子，连接后的情况是：

![fe78f0b7e1b2ddf377ce91e5c35dcac1\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653aeb28d.png)

最后，如我们上面所说的，合并那些可以合并的相邻节点的父节点，最终得到：

![8bdfce2b6b51a7f56d6d5d6baacffd51\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653c2e1f7.png)

仍然是一个格子代表一个节点，转为树状图是：

![6eee986ba8bf972e4d1abba7c58f123f\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653c8e2d1.png)

最后再在每一个叶节点上，添加它们对应的窗口，得到最终的WindowContainer层级结构图：

![72dd0040ff5a30c81b468f4aeaf79962\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653a2fc5f.png)

最后再提一下，我们将窗口分为App之上的窗口、App窗口以及App之下的窗口，基准便是TaskDisplayArea。

### 2.7 WindowContainer层级结构根节点 —— RootWindowContainer

回看上面的WindowContainer层级结构图，能看到根节点为RootWindowContainer，而非DisplayArea层级结构的根节点DisplayContent，毕竟DisplayContent只代表一块屏幕，而Android是支持多屏幕的，展开来说就是包括一个内部屏幕（内置于手机或平板电脑中的屏幕）、一个外部屏幕（如通过 HDMI 连接的电视）以及一个或多个虚拟屏幕。

那么就需要一个DisplayContent的容器出现，这个角色自然也就是RootWindowContainer了，从名字也能看出来，它是作为WindowContainer层级结构的根节点所存在的：

```java
/** Root {@link WindowContainer} for the device. */
class RootWindowContainer extends WindowContainer<DisplayContent>
        implements DisplayManager.DisplayListener
```

那么如果我们在开发者选项里通过“Simulate secondary displays”开启另一个虚拟屏幕，此时的情况是：

```
ROOT type=undefined mode=fullscreen override-mode=undefined requested-bounds=[0,0][0,0] bounds=[0,0][720,1612]
  #1 Display 0 name="Built-in screen" type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][720,1612] bounds=[0,0][720,1612]
  #0 Display 2 name="Overlay #1" type=undefined mode=fullscreen override-mode=fullscreen requested-bounds=[0,0][720,480] bounds=[0,0][720,480]
```

转为层级结构图：

![de6dbdb6360e583c3266fed0a88a0a40\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f26539a5545.png)

“Root”即RootWindowContainer，“Display 0”代表默认屏幕，“Display 2”是我们开启的虚拟屏幕。

### 2.8 小结

在代码搜索WindowContainer的继承关系，可以得到：

![ee4f48954a6b2467495c38675b41c4a3\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653c8d308.png)

除了不参与WindowContainer层级结构的WindowingLayerContainer之外，其他类都在本文有提及，用类图总结为：

![447e19319f2af7c7037554ea61b5eb53\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653db4acc.png)

除了WindowState可以显示图像外，大部分的WindowContainer，如WindowToken、TaskDisplayArea是不会有内容显示的，都只是一个抽象的容器概念，说白了就是一个收纳用的容器。极端点说，WMS如果只为了管理窗口，WMS也可以不创建这些个WindowContainer类，直接用一个WindowState的列表管理屏幕上的所有窗口。但是为了更有逻辑地管理屏幕上显示的窗口，比如哪些窗口属于同一个Activity？哪些Activity属于同一个App？如何将App窗口和非App窗口清楚得分隔开？因此还是需要创建各种各样的窗口容器类，即WindowContainer及其子类，来对WindowState进行分类，并且每一种WindowContainer通常都由另外一种专门的WindowContainer来管理，从而对窗口进行层次化的管理。这种层次化管理保证了每一个WiindowContainer都会在其对应的父容器中进行调整而不会越界。这个一点很重要，比如我点击HOME键回到Launcher，此时TaskDisplayArea就会把Launcher对应的Task，移动到它所对应的堆栈顶部，而这个调整仅限于TaskDisplayArea内部（因为TaskDisplayArea是Task的容器），这就保证了再怎么调整Launcher，Launcher的窗口也永远不可能高于StatusBar窗口，也不会低于Wallpaper窗口。

另外这个层级结构建立起来后，对窗口的遍历也是基于这个层级结构。比如发生转屏后，DisplayContent会基于新的屏幕方向重新计算Configuration，然后按照这个层级结构的顺序，依次把这个新计算的Configuration发给每一个WindowContainer，最终每一个WindowContainer都能基于新的Configuration更新自己的Configuration，而非每一个WindowContainer都需要重新去计算一次Configuration，并且大部分的更新其实都是直接拿从父容器传过来的Configuration赋值给自己的Configuration（相当于完全继承了父容器的Configuration，当然会有一些WindowContainer会显式的声明自己有独特的策略，不会完全继承父容器的Configuration，但这只是少数）。这体现容器的一个特性，即继承特性，比如你把一个盒子翻转，那么盒子里的小盒子也会跟着翻转，而不是每一个小盒子再去考虑要不要翻转。再举一个例子就是，分屏模式下，我们只需要计算好分屏Task的bounds，Task里的ActivityRecord以及ActivityRecord里的WindowState，会自动继承Task的Configuration，变成和Task同样的大小（大盒子缩小后，里面的小盒子会跟随大盒子缩小）。

## 3 Window的底层实现 —— Layer层级结构

### 3.1 Layer基础概念

之前我们可能会认为窗口就是屏幕上一块显示图像的区域，这个当然不错，但是反之不一样成立，即屏幕上一块显示图像的区域不一定就是一个窗口，但是它一定是一个Layer。至于Layer，则一个比窗口更底层的实现，代表屏幕上一块显示内容的区域：

![1bc03234face4f8861899be7a78f387c\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653976771.png)

SurfaceFlinger如果要将所有Layer进行合成，那么需要知道每一个Layer的两个基本信息：

* 显示边框/边界，即该Layer在屏幕上的显示位置（x和y）和大小（w和h）。
* 显示内容，即该Layer所绘制图像的图形数据，保存在Layer拥有的图形缓冲区中。

这两个信息需要WMS和App这两方分别提供：

* WMS作为窗口管理服务，每一个窗口要显示多大，显示在哪儿，自然是由它来决定。
* App决定它定义的每一个界面的具体绘制内容，这个也不用多说。

SurfaceFlinger拿到从WMS和App传过来的数据，将所有Layer合成送显。

那么还有一个问题，处于不同的进程的App、WMS以及SurfaceFlinger，通过什么方式来进行信息传递？

![52c13556ef8e4386c3b042edb666b4be\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653adc10d.png)

1）、当App进入前台时，WMS会向SurfaceFlinger请求为App创建Layer。

2）、SurfaceFlinger创建Layer后，返回给WMS一个该Layer对应的SurfaceControl对象。WMS持有这些SurfaceControl，并且后续可以通过SurfaceControl向SurfaceFlinger提供窗口元数据，用来控制App在屏幕上的外观。

3）、WMS将SurfaceControl发送给App，App会根据SurfaceControl创建BufferQueue和Surface。Surface持有一组图形缓冲区，并且链接了BufferQueue中的图像生产者BufferQueueProducer，后续由它们来向SurfaceFlinger提供绘制了内容的图形缓冲区。

总的来说，SurfaceFlinger消费Surface生产的图像缓冲区，并使用WMS通过SurfaceControl提供的窗口元数据，来对Layer进行合成，因此Layer是Surface（包含 BufferQueue）和SurfaceControl（包含显示边框等窗口元数据）的组合。

### 3.2 Layer层级结构

再来看一段SurfaceFlinger的信息：

```
+ EffectLayer (Task=16#0) uid=1000
  Region TransparentRegion (this=0 count=0)
  Region VisibleRegion (this=0 count=0)
  Region SurfaceDamageRegion (this=0 count=0)
      layerStack=   0, z=       11, pos=(0,0), size=(  -1,  -1), crop=[  0,   0,   0,   0], cornerRadius=0.000000, isProtected=0, isTrustedOverlay=0, isOpaque=0, invalidate=0, dataspace=De
fault, defaultPixelFormat=Unknown/None, backgroundBlurRadius=0, color=(-1.000,-1.000,-1.000,1.000), flags=0x00000000, tr=[1.00, 0.00][0.00, 1.00], mBounds=[0.00, 0.00, 1200.00, 1920.00], l
ayerId=0
      parent=DefaultTaskDisplayArea#0
      zOrderRelativeOf=none
      activeBuffer=[   0x   0:   0,Unknown/None], tr=[0.00, 0.00][0.00, 0.00] queued-frames=0, mRefreshPending=0, metadata={taskId:16}, cornerRadiusCrop=[0.00, 0.00, 0.00, 0.00],  shadowRa
dius=0.000,

+ ContainerLayer (1c7dd1d ActivityRecordInputSink com.android.settings/.Settings#0) uid=1000
  Region TransparentRegion (this=0 count=0)
  Region VisibleRegion (this=0 count=0)
  Region SurfaceDamageRegion (this=0 count=0)
      layerStack=   0, z=-2147483648, pos=(0,0), size=(  -1,  -1), crop=[  0,   0,  -1,  -1], cornerRadius=0.000000, isProtected=0, isTrustedOverlay=0, isOpaque=0, invalidate=1, dataspace=
Default, defaultPixelFormat=Unknown/None, backgroundBlurRadius=0, color=(0.000,0.000,0.000,1.000), flags=0x00000000, tr=[1.00, 0.00][0.00, 1.00], mBounds=[0.00, 0.00, 1200.00, 1920.00], la
yerId=0
      parent=ActivityRecord{8e62b92 u0 com.android.settings/.Settings t16}#0
      zOrderRelativeOf=none
      activeBuffer=[   0x   0:   0,Unknown/None], tr=[0.00, 0.00][0.00, 0.00] queued-frames=0, mRefreshPending=0, metadata={}, cornerRadiusCrop=[0.00, 0.00, 0.00, 0.00],  shadowRadius=0.00
0,

+ ContainerLayer (ActivityRecord{8e62b92 u0 com.android.settings/.Settings t16}#0) uid=1000
  Region TransparentRegion (this=0 count=0)
  Region VisibleRegion (this=0 count=0)
  Region SurfaceDamageRegion (this=0 count=0)
      layerStack=   0, z=        0, pos=(0,0), size=(  -1,  -1), crop=[  0,   0,  -1,  -1], cornerRadius=0.000000, isProtected=0, isTrustedOverlay=0, isOpaque=0, invalidate=1, dataspace=De
fault, defaultPixelFormat=Unknown/None, backgroundBlurRadius=0, color=(0.000,0.000,0.000,1.000), flags=0x00000000, tr=[1.00, 0.00][0.00, 1.00], mBounds=[0.00, 0.00, 1200.00, 1920.00], laye
rId=0
      parent=Task=16#0
      zOrderRelativeOf=none
      activeBuffer=[   0x   0:   0,Unknown/None], tr=[0.00, 0.00][0.00, 0.00] queued-frames=0, mRefreshPending=0, metadata={}, cornerRadiusCrop=[0.00, 0.00, 0.00, 0.00],  shadowRadius=0.00
0,

+ ContainerLayer (5cadaf2 com.android.settings/com.android.settings.Settings#0) uid=1000
  Region TransparentRegion (this=0 count=0)
  Region VisibleRegion (this=0 count=0)
  Region SurfaceDamageRegion (this=0 count=0)
      layerStack=   0, z=        0, pos=(0,0), size=(  -1,  -1), crop=[  0,   0,  -1,  -1], cornerRadius=0.000000, isProtected=0, isTrustedOverlay=0, isOpaque=0, invalidate=1, dataspace=De
fault, defaultPixelFormat=Unknown/None, backgroundBlurRadius=0, color=(0.000,0.000,0.000,1.000), flags=0x00000000, tr=[1.00, 0.00][0.00, 1.00], mBounds=[0.00, 0.00, 1200.00, 1920.00], laye
rId=0
      parent=ActivityRecord{8e62b92 u0 com.android.settings/.Settings t16}#0
      zOrderRelativeOf=none
      activeBuffer=[   0x   0:   0,Unknown/None], tr=[0.00, 0.00][0.00, 0.00] queued-frames=0, mRefreshPending=0, metadata={}, cornerRadiusCrop=[0.00, 0.00, 0.00, 0.00],  shadowRadius=0.00
0,

+ BufferStateLayer (com.android.settings/com.android.settings.Settings#0) uid=1000
  Region TransparentRegion (this=0 count=0)
  Region VisibleRegion (this=0 count=1)
    [  0,   0, 1200, 1920]
  Region SurfaceDamageRegion (this=0 count=1)
    [  0, 286, 1200, 1824]
      layerStack=   0, z=        0, pos=(0,0), size=(1200,1920), crop=[  0,   0,  -1,  -1], cornerRadius=0.000000, isProtected=0, isTrustedOverlay=0, isOpaque=1, invalidate=0, dataspace=De
fault, defaultPixelFormat=RGBA_8888, backgroundBlurRadius=0, color=(0.000,0.000,0.000,1.000), flags=0x00000102, tr=[1.00, 0.00][0.00, 1.00], mBounds=[0.00, 0.00, 1200.00, 1920.00], layerId
=0
      parent=5cadaf2 com.android.settings/com.android.settings.Settings#0
      zOrderRelativeOf=none
      activeBuffer=[1200x1920:1200,RGBA_8888], tr=[1.00, 0.00][0.00, 1.00] queued-frames=0, mRefreshPending=0, metadata={dequeueTime:505108725337, windowType:1, ownerPID:6640, ownerUID:100
0}, cornerRadiusCrop=[0.00, 0.00, 0.00, 0.00],  shadowRadius=0.000,
      *BufferQueueDump mIsBackupBufInited=0, mAcquiredBufs(size=1)
       [00] handle=0xb400007e3a327d80, fence=0xb400007e3a281c80, time=0x759ae697c5, xform=0x00
       FPS ring buffer:
       (0) 13:19:15.488 fps=1.94  dur=1544.49       max=1494.64       min=16.54
```

这是当从Launcher启动Settings应用后，Settings中所有Layer的信息，转为SurfaceFlinger层级结构图：

![cff97659d89f32db215f927d81817d69\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f26539efb5e.png)

对比Java层WindowContainer层级结构图：

![6f60afafb9e70fe565e1a388641ba4c0\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f26539b9625.png)

能看到：

1）、WMS侧中每一级的WindowContainer，在SurfaceFlinger侧中都有一个对应的Layer，这是因为每创建一个WindowContainer，都会为其创建一个SurfaceControl对象。那么换句话说，在SurfaceFlinger侧的每一个Layer，大部分情况下在WMS侧都能找到一个对应的的surfaceControl，这也就是为什么说WMS能够通过SurfaceControl来控制SurfaceFlinger中Layer的显示。

2）、这里有一个名为”ContainerLayer (1c7dd1d ActivityRecordInputSink com.android.settings/.Settings#0)“的Layer，这个类是用来拦截输入事件向Activity发送的，可以看下它创建的地方：

```java
// frameworks\base\services\core\java\com\android\server\wm\ActivityRecordInputSink.java

	private SurfaceControl createSurface(SurfaceControl.Transaction t) {
        SurfaceControl surfaceControl = mActivityRecord.makeChildSurface(null)
                .setName(mName)
                .setHidden(false)
                .setCallsite("ActivityRecordInputSink.createSurface")
                .build();
        // Put layer below all siblings (and the parent surface too)
        t.setLayer(surfaceControl, Integer.MIN_VALUE);
        return surfaceControl;
    }
```

* 通过ActivityRecord.makeChildSurface，将该SurfaceControl对应的Layer的父Layer设置为了ActivityRecord。
* 通过Transaction.setLayer，将该SurfaceControl对应的Layer的层级设置为最低 —— Integer.MIN\_VALUE（-2147483648）。

那么结果就是该Layer的信息所显示的那样；

```
      z=-2147483648
      parent=ActivityRecord{8e62b92 u0 com.android.settings/.Settings t16}#0
```

所以ActivityRecord这一级WindowContainer对应的Layer，除了有一个WindowState对应的子Layer以外，还有一个ActivityRecordInputSink对应的子Layer。

另外由于该Layer的层级值是负数，所以我这边在画层级结构图的时候，将它放在了父Layer上面，即父Layer上面的子Layer，层级值为负数，父Layer下面的子Layer，层级值为正数（SurfaceFlinger的信息，越往下的Layer层级值越高，而之前贴的WindowContainer的信息则是越靠上层级值越高）。

3）、SurfaceFlinger中，WindowState这一级下面，还有一个Layer：”BufferStateLayer (com.android.settings/com.android.settings.Settings#0)“。那么这里就需要介绍一下Layer类的继承关系：

![c99351f63f40d2795eb6fe7cd7116d3b\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653e71413.png)

* Layer，所有Layer的父类。
* EffectLayer，代表了一个有纯色或者阴影效果的Layer，比如实现Dim效果的时候，添加的就是这个Layer，另外一提，Task对应的WindowContainer也是这种Layer，说明Task也是可以显示一些内容的（ShadowRadius）。
* ContainerLayer，代表了一个容器类Layer，这种Layer没有缓冲区，只是用来作为其他Layer的容器，比如ActivityRecord和WindowState对应的Layer，都是这种类型。
* BufferLayer，是携带图形缓冲区的Layer，即真正显示App绘制内容的Layer。其中又分BufferQueueLayer和BufferStateLayer，由于BufferQueueLayer使用旧版的API，支持的功能有限，如今已经弃用，现在用的是BufferStateLayer。BufferStateLayer对应Java层WindowSurfaceController对象持有的SurfaceControl对象，会在relayoutWindow流程中创建，当窗口绘制完成后进行显示。

将Layer层级结构和WindowContainer层级结构进行比较后可以知道，Layer层级结构的建立是以WindowContainer层级结构为基础，向下又延申了一层BufferStateLayer，如果你已经理解了WindowContainer层级结构，那么理解Layer层级结构起来就非常容易。

另外注意这只是一般规则，实际上系统运行过程中可能会创建许多Layer穿插其中，如动画的Leash、实现阴影效果的DimLayer以及旋转用到的RotationLayer等等，对于这些特殊的Layer则需要具体场景具体分析。

接下来介绍几种能够层级结构变化的情况。

### 3.3 Leash

以一个从Launcher启动Settings的动画过程为例来说明一下Leash的用法，由于是App从后台移动到前台，因此动画的主体是Task。

#### 3.3.1 动画前

动画前Task#16相关的层级结构是：

![a22a17387f36cf6d673b703516e44d2f\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f26539eeb4a.png)

Task#16的Layer层级结构，我们上面也说明过了，这里额外加入了它的父Layer，DefaultTaskDisplayArea。

此时没有BufferStateLayer，因为此时Settings还在后台，窗口并没有绘制。

#### 3.3.2 动画中

动画过程中Task#16相关的层级结构是：

![d1bdc885e21ebe4a412602467f0c5157\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653cbe2cd.png)

这里多了两个Layer：

* BufferStateLayer，窗口绘制完成后才开始的动画，所以动画过程中BufferStateLayer已经在了。
* 一个名为”animation-leash of app\_transition“的Layer，且该Layer插在了DefaultTaskDisplayArea和Task中间。

Leash的这种行为有点像我们之前分析Freeform的时候DecorCaptionView的做法：

1）、先创建一个Leash，然后指定该Leash对应的Layer的父Layer为动画的主体（这个例子是Task）的父Layer（DefaultTaskDisplayArea）。

2）、接着将Task对应的Layer从原父Layer（DefaultTaskDisplayArea）上剥离，变为Leash的子Layer（将一个子Layer从原父Layer重定向到一个新的父Layer上，称为reparent操作）。

从而实现在DefaultTaskDisplayArea和Task中插进一个Leash的操作。

注意此时执行动画的对象是Leash，而非Task。但由于Task的父Layer是Leash，所以Leash移动，Task会跟着移动，我们能够看到的BufferStateLayer也会跟着移动。

#### 3.3.3 动画后

动画结束后Task#16相关的层级结构是：

![c255a9fc514d8be2860decdf11180b6d\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653a07c6f.png)

动画结束，Leash已经完成了它的任务，可以撤了，但是在撤之前需要将绑定在Leash上的Task重新reparent回原父Layer上（DefaultTaskDisplayArea）。

#### 3.3.4 小结

上面的例子大致说明了Leash的运行机制，这其中有3个Layer参与，一个是动画主体（Task），一个是动画主体的父Layer（DefaultTaskDisplayArea），还有一个是Leash。

Leash的作用即是代替动画的主体去执行动画：

1）、创建Leash，将Leash绑定到动画主体的父Layer上，动画主体则reparent到Leash上。

2）、对Leash进行动画，而非动画主体。

3）、动画结束后，将动画主体reparent回原父Layer上，移除Leash。

至于为什么要搞Leash这么一个东西出来，我个人认为是，如果直接对动画主体执行动画，那么就会涉及一个恢复的操作。比如说对Activity间的切换，如果不用Leash直接对Activity进行操作，那么情况是：旧的Activity切走的时候，可能会有一个向左平移的动画，后续如果我们想要这个旧的Activity重新出现，那么我们不仅需要将它显示出来，还需要将它从之前的动画结束位置向右平移，这样它才能出现在屏幕上。并且我们的动画越复杂，恢复需要的步骤也就越多。

如果我们把动画主体reparent到Leash上，然后对Leash进行相同的动画，不仅能达到同样的效果，而且不管我们对Leash如何进行动画，只要在动画结束后将动画主体reparent回原来的父Layer上，就可以恢复原状，而不用像上面一样考虑要做多少相反的操作才能恢复原状。

最后，Leash理论上可以被应用到任何希望进行动画的Surface上，比如这里举的例子是应用到了Task上，因为这个动画发生于App间的切换过程中。如果是Activity间的切换，那么Leash就可以应用到ActivityRecord，同样WindowState也是可以的，主要看动画的主体是整个App？还是单个Activity？还是某个窗口？

### 3.4 DimLayer和相对层级

DimLayer的本质是一个EffectLayer，默认是一个纯黑Layer，我们可以通过设置Layer的透明度来施加阴影效果，或者设置Layer模糊半径来施加模糊效果。

来看一个实际应用的例子，Files：

![bc02b622ee0136390b9be96d147f73a4\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f26539daefe.png)

此时Files有两个窗口，子窗口盖在主窗口上，并且此时子窗口声明了一个特殊窗口标志位：

```java
        /** Window flag: everything behind this window will be dimmed.
         *  Use {@link #dimAmount} to control the amount of dim. */
        public static final int FLAG_DIM_BEHIND        = 0x00000002;
```

该窗口标志位的作用是，将声明了该标志位的窗口的后面所有内容都变暗，因此最终的效果就是如上图所示，即主窗口被蒙上了一层阴影效果。

此时的窗口层级结构图是：

![7a3c7ed04a460026bdbe531510209ea6\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f26539dba4e.png)

在这个层级结构图中，只有可见的Layer才被标记了颜色，此时可见Layer的Z轴顺序是：子窗口 > DImLayer > 主窗口。

主窗口和子窗口对应的两个BufferStateLayer不用多说，这里主要看下DimLayer，相关信息如下：

```
+ EffectLayer (Dim Layer for - Task=36#0) uid=1000
      z=       -1
      color=(0.000,0.000,0.000,0.600)
      parent=Task=36#0
      zOrderRelativeOf=f19c0b8 com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0
```

* alpha值为0.6，上面我们也提到过，EffectLayer默认是一个纯黑的Layer，我们需要设置它的透明度来达到变暗的效果（透明度为1则会完全遮住后面的内容）。
* DimLayer的父Layer是Task#36，这个跟上层的实现有关系，目前是Task和DisplayArea支持创建这么一个DimLayer。
* 这里设置了DimLayer的相对层级为子窗口WindowState这一级对应的ContainerLayer，**ContainerLayer (f19c0b8 com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0)**。设置一个Layer的相对层级，相当于是将该Layer从其原来的父Layer上剥离，这里是 **EffectLayer (Task=36#0)**，然后将这个相对层级的Layer作为父Layer，这里是 **ContainerLayer (f19c0b8 com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0)**，重新绑定到这个父Layer上面。所以这里能够看到虽然DimLayer的父Layer是 **EffectLayer (Task=36#0)**，但是在实际上的层级结构中，DimLayer是作为了 **ContainerLayer (f19c0b8 com.google.android.apps.nbu.files/com.google.android.apps.nbu.files.home.HomeActivity#0)** 的子Layer。
* 当然只设置相对层级还不够，还需要将DimLayer的层级值设置为-1，这样它才不会把子窗口也盖住，并且能够遮住层级比子窗口低的Layer。

#### 3.4.1 DimLayer层级值从-1修改为100

这里我们手动修改DimLayer的实现，将其层级值从-1修改为100，那么显示效果为：

![7b4877d45fe263fe70b09471a8aa29fb\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653deeca4.png)

此时阴影效果不仅覆盖了主窗口，连子窗口也覆盖了，层级结构图为：

![7c533778b6dcf572ef116951de4922cf\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653bc2114.png)

很显然，当设置EffectLayer的层级值为100后，它就变成了子窗口中层级值最高的Layer，既然比子窗口对应的BufferStateLayer层级值还高，自然会把整个Activity都遮住，符合预期。

#### 3.4.2 DimLayer相对层级从子窗口修改为主窗口

这里我们手动修改DimLayer的实现，将其相对层级的对象从子窗口修改为主窗口，那么显示效果为：

![0cc2f779ba438a6caa9e2bfa86484a08\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653e24f48.png)

此时阴影效果消失，层级结构图为：

![0f3e27c2da76027dd80d044270c3273c\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f2653c995df.png)

将EffectLayer的相对层级的对象从子窗口修改为主窗口，它就变成了子窗口中层级值最低的Layer，自然无法再遮挡任何Layer，并且被全屏的主窗口完全遮挡从而无法显示，符合预期。

#### 3.4.3 小结

相对层级的出现，使得Layer的层级顺序调整更加灵活，并且后续如果不需要相对层级了，也不需要执行繁琐的”先将当前Layer从相对层级Layer上移除，再重新绑定回原父Layer“的操作，直接设置一下层级值就可以将相对层级Layer移除了。

另外你也可以根据需要，为DimLayer设置不同的相对层级对象，从而实现将某个子窗口变暗，或者将某个Activity变暗，或者将整个App变暗。

## 4 总结

这里我按照自己的理解画了一下Android架构图，主要是对看下Framework这块：

![01e745577c88b795fd309d8b8d131129\_MD5](https://picgo.myjojo.fun:666/i/2024/09/24/66f26539e2eaf.png)

* 顶层是基于ApplicationFramework层的Apps层，ApplicationFramework是运行在App进程的Framework代码，如四大组件，各种Manager。在这一层，屏幕上的一块显示区域，典型代表是Activity，但是Activity毕竟是一个综合性比较强的概念，具体到内容显示这块还是由Window类负责，Window则是容纳View对象的容器。我们要了解屏幕的内容是如何组成的，首先需要知道的就是组成屏幕的最小单位 —— Window，的内部构造，即我们介绍的第一部分，View层级结构。

* 接着是系统服务，其中系统服务既有基于Java的WMS之类（我习惯叫做JavaFramework），也有基于C++的SurfaceFlinger之类（我习惯叫做NativeFramework）：

* JavaFramework，在这一层，屏幕上的一块显示区域，典型代表是WindowState，当然更加通用的结构是WindowContainer。WMS作为管理窗口的服务，它需要把屏幕上的所有窗口按照一定的规则系统化的组织起来，即我们介绍的第二部分，WindowContainer层级结构。这样SurfaceFlinger就只需要操心合成Layer的相关事务就行了，不需要去关心如何去建立如此庞大的一个层级结构。

* NativeFramework，在这一层，屏幕上的一块显示区域，典型代表是BufferStateLayer，当前更加通用的结构是Layer。SurfaceFlinger对Layer进行合成需要基于Layer层级结构，而Layer层级结构的基本框架就是WindowContainer层级结构，另外WMS也可以通过SurfaceControl来在Layer层级结构里进行一些局部的调整。

转载：https://blog.csdn.net/ukynho/article/details/132327216
