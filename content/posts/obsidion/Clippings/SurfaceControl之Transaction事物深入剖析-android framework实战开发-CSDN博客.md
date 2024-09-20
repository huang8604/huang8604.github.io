---
title: SurfaceControl之Transaction事物深入剖析-android framework实战开发-CSDN博客
source: https://blog.csdn.net/learnframework/article/details/135076600
author: 
published: 
created: 2024-09-20
description: 文章浏览阅读2.9k次，点赞22次，收藏36次。surfacecontrol
tags:
  - clippings
  - blog
  - Framework
collections:
  - SurfaceControl
date: 2024-09-20T07:45:02.050Z
lastmod: 2024-09-20T07:45:30.405Z
---
### 背景

前面已经讲解清楚了SurfaceControl整个创建过程，一般SurfaceControl都是一个静态图层的代表，但往往只有静态一个图层是没有意义的，即只是创建了一个图层其实啥也看不到，更多需要是SurfaceControl对应的Transaction,这个事务才是真正可以让SurfaceControl可以显示的关键所在，Transaction才是相当于一个个的动作让SurfaceControl静态东西可以动起来，接下来将详细分析一下Transaction。

### 常见Transaction使用案例

```cpp
  private final Transaction mTransaction = new Transaction();//构建或者获取一个事物
    mTransaction.setLayerStack(mSurfaceControl, mDisplayLayerStack); //开始用事物对象，对一个个SurfaceControl进行设置属性
    mTransaction.setWindowCrop(mSurfaceControl, mDisplayWidth, mDisplayHeight);//开始用事物对象，对一个个SurfaceControl进行设置属性
    mTransaction.apply();//针对事物进行apply操作
```

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/75c04dd83fe05883b95c342039f10d8f.png#pic_center)\
上面可以看出，Transaction是一个独立的事物对象，专门用于操作一个个SurfaceControl的属性，但并不是属于SurfaceControl对象的成员，即Transaction完全可以实现一对多个SurfaceControl情况。

### Transaction的构造

frameworks/base/core/java/[android](https://so.csdn.net/so/search?q=android\&spm=1001.2101.3001.7020)/view/SurfaceControl.java

```cpp
  public Transaction() {
            this(nativeCreateTransaction());
        }
```

来看看nativeCreateTransaction方法\
frameworks/base/core/[jni](https://so.csdn.net/so/search?q=jni\&spm=1001.2101.3001.7020)/android\_view\_SurfaceControl.cpp

```cpp
static jlong nativeCreateTransaction(JNIEnv* env, jclass clazz) {
    return reinterpret_cast<jlong>(new SurfaceComposerClient::Transaction);
}
```

就是简单的构造了一个SurfaceComposerClient::Transaction对象\
frameworks/native/libs/[gui](https://so.csdn.net/so/search?q=gui\&spm=1001.2101.3001.7020)/SurfaceComposerClient.cpp

```cpp
SurfaceComposerClient::Transaction::Transaction() {
    mId = generateId();
}
```

可以看到如果默认的构造的Transaction其实啥也没有干，就是生成了个Id然后赋值给了mId成员变量。

### Transaction相关介绍

#### 主要成员

**layer\_state\_t结构体**\
用来代表Layer图层的的相关信息，SurfaceControl与sf的Layer共用这个layer\_state\_t结构体，layer\_state\_t包括layer所有属性\
主要成员如下：\
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/b341b3a278a1f8a4b62e5849a28ba4f5.png#pic_center)\
可以看到常见的主要属性：坐标，长宽，变换矩阵，变化值what，flags，mask等，一般是一个图层就有一个layer\_state\_t结构体。\
ComposerStates结构体

```cpp
struct ComposerState {
    layer_state_t state;
    status_t write(Parcel& output) const;
    status_t read(const Parcel& input);
};
```

可以看出就是layer\_state\_t进行了一个包装而已

**mComposerStates**\
定义如下：\
std::unordered\_map\<sp, ComposerState, IBinderHash> mComposerStates;\
可以看出来其实就是一个装载ComposerState的map容器，map的key是每个SurfaceControl的handle,具体可以看一下这个mComposerStates的容器添加方法：

```cpp
layer_state_t* SurfaceComposerClient::Transaction::getLayerState(const sp<SurfaceControl>& sc) {
    auto handle = sc->getLayerStateHandle();//获取SurfaceControl的handle，
	//判断是否mComposerStates容器是否存在这个sc的相关信息，如果没有则进入添加
    if (mComposerStates.count(handle) == 0) {
        // we don't have it, add an initialized layer_state to our list
        ComposerState s;//初始化一个ComposerState

        s.state.surface = handle;
        s.state.layerId = sc->getLayerId();

        mComposerStates[handle] = s;//把初始化的ComposerState放到map集合mComposerStates中
    }
//如果存在，则直接通过handle从mComposerStates获取state返回
    return &(mComposerStates[handle].state);
}
```

上面其实可以得出如下结论：\
1、每个Transaction都有自己的一个mComposerStates集合\
2、mComposerStates集合会放入Transaction中会操作的SurfaceControl对应的layer\_state\_t\
补充一下 sc->getLayerStateHandle方法：

```cpp
sp<IBinder> SurfaceControl::getLayerStateHandle() const
{
    return mHandle;
}
```

这个mHandle其实就是sf端创建一个Handle

```cpp
sp<IBinder> Layer::getHandle() {
    Mutex::Autolock _l(mLock);
    if (mGetHandleCalled) {
        ALOGE("Get handle called twice" );
        return nullptr;
    }
    mGetHandleCalled = true;
    return new Handle(mFlinger, this);
}
```

即mHandle是sf端代表Layer的BpBinder对象。

#### 主要方法

##### 一些属性常见设置方法

```cpp
//可以看到一般属性操作方法第一参数是SurfaceControl，后面属性需要参数
SurfaceComposerClient::Transaction& SurfaceComposerClient::Transaction::setPosition(
        const sp<SurfaceControl>& sc, float x, float y) {
    layer_state_t* s = getLayerState(sc); //获取sc对应的layer_state_t
    s->what |= layer_state_t::ePositionChanged;//改变what，即标记layer_state_t哪个属性是有变化的，方便sf进行识别获取
    s->x = x; //接下来才是改变具体的值
    s->y = y;

    registerSurfaceControlForCallback(sc);
    return *this;
}
```

上面就是一个经典的Transaction改变属性的方法，常规就是以下几步：\
1、通过传递来的sc，获取sc的layer\_state\_t\
2、标记layer\_state\_t的what属性，主要为了明显表达出哪个属性变化了\
3、进行具体属性改变

其他的属性方法套路都和上面基本一样\
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/10a848d82fd3779ad24c754feada9a30.png#pic_center)

##### merge方法

主要目的是把多个Transaction的内容合并到一个，即把other的这个Transaction内容都拷贝到当前Transaction

```cpp
SurfaceComposerClient::Transaction& SurfaceComposerClient::Transaction::merge(Transaction&& other) {
    for (auto const& [handle, composerState] : other.mComposerStates) {
        if (mComposerStates.count(handle) == 0) {
            mComposerStates[handle] = composerState;
        } else {
            if (composerState.state.what & layer_state_t::eBufferChanged) {
                releaseBufferIfOverwriting(mComposerStates[handle].state);
            }
            mComposerStates[handle].state.merge(composerState.state);
        }
    }

    for (auto const& state : other.mDisplayStates) {
        ssize_t index = mDisplayStates.indexOf(state);
        if (index < 0) {
            mDisplayStates.add(state);
        } else {
            mDisplayStates.editItemAt(static_cast<size_t>(index)).merge(state);
        }
    }

    for (const auto& [listener, callbackInfo] : other.mListenerCallbacks) {
        auto& [callbackIds, surfaceControls] = callbackInfo;
        mListenerCallbacks[listener].callbackIds.insert(std::make_move_iterator(
                                                                callbackIds.begin()),
                                                        std::make_move_iterator(callbackIds.end()));

        mListenerCallbacks[listener].surfaceControls.insert(surfaceControls.begin(),
                                                            surfaceControls.end());

        auto& currentProcessCallbackInfo =
                mListenerCallbacks[TransactionCompletedListener::getIInstance()];
        currentProcessCallbackInfo.surfaceControls
                .insert(std::make_move_iterator(surfaceControls.begin()),
                        std::make_move_iterator(surfaceControls.end()));

        // register all surface controls for all callbackIds for this listener that is merging
        for (const auto& surfaceControl : currentProcessCallbackInfo.surfaceControls) {
            TransactionCompletedListener::getInstance()
                    ->addSurfaceControlToCallbacks(surfaceControl,
                                                   currentProcessCallbackInfo.callbackIds);
        }
    }

    mInputWindowCommands.merge(other.mInputWindowCommands);

    mContainsBuffer |= other.mContainsBuffer;
    mEarlyWakeupStart = mEarlyWakeupStart || other.mEarlyWakeupStart;
    mEarlyWakeupEnd = mEarlyWakeupEnd || other.mEarlyWakeupEnd;
    mApplyToken = other.mApplyToken;

    mFrameTimelineInfo.merge(other.mFrameTimelineInfo);

    other.clear();
    return *this;
}
```

##### apply方法

```cpp

status_t SurfaceComposerClient::Transaction::apply(bool synchronous, bool oneWay) {
  //省略非关键
    for (auto const& kv : mComposerStates){ //收集各个图层sc的ComposerState信息
        composerStates.add(kv.second);
    }

    displayStates = std::move(mDisplayStates);
//省略部分
//最后把上面收集的Transaction相关信息，调用sf的setTransactionState进行跨进程传递到sf进程
    sf->setTransactionState(mFrameTimelineInfo, composerStates, displayStates, flags, applyToken,
                            mInputWindowCommands, mDesiredPresentTime, mIsAutoTimestamp,
                            {} /*uncacheBuffer - only set in doUncacheBufferTransaction*/,
                            hasListenerCallbacks, listenerCallbacks, mId);
    mId = generateId();

    // Clear the current states and flags
    clear();//apply后就需要把Transaction进行clear
    return NO_ERROR;
}

```

apply方法主要就是收集之前通过transaction属性设置方法设置所有信息都需要收集起来，比如最重要的composerStates，然后调用sf的跨进程方法setTransactionState传递到sf中。

##### 补充sf端的处理：

frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp

```cpp
uint32_t SurfaceFlinger::setClientStateLocked(const FrameTimelineInfo& frameTimelineInfo,
                                              ComposerState& composerState,
                                              int64_t desiredPresentTime, bool isAutoTimestamp,
                                              int64_t postTime, uint32_t permissions) {
    layer_state_t& s = composerState.state;
    s.sanitize(permissions);
//省略
    const uint64_t what = s.what;
     sp<Layer> layer = nullptr;
     //这里会使用layer_state_t的surface即代表Layer的binder对象，把Layer找出
    if (s.surface) {
        layer = fromHandle(s.surface).promote();
    } 
//可以看出这里会获取layer_state_t的what看看是否是否哪个部分有变化，有变化就直接把变化数据设置到Layer中
    if (what & layer_state_t::eSizeChanged) {
        if (layer->setSize(s.w, s.h)) {
            flags |= eTraversalNeeded;
        }
    }
    if (what & layer_state_t::eAlphaChanged) {
        if (layer->setAlpha(s.alpha))
            flags |= eTraversalNeeded;
    }
    if (what & layer_state_t::eColorChanged) {
        if (layer->setColor(s.color))
            flags |= eTraversalNeeded;
    }
    if (what & layer_state_t::eColorTransformChanged) {
        if (layer->setColorTransform(s.colorTransform)) {
            flags |= eTraversalNeeded;
        }
    }
    if (what & layer_state_t::eBackgroundColorChanged) {
        if (layer->setBackgroundColor(s.color, s.bgColorAlpha, s.bgColorDataspace)) {
            flags |= eTraversalNeeded;
        }
    }
    if (what & layer_state_t::eMatrixChanged) {
        if (layer->setMatrix(s.matrix)) flags |= eTraversalNeeded;
    }
    if (what & layer_state_t::eTransparentRegionChanged) {
        if (layer->setTransparentRegionHint(s.transparentRegion))
            flags |= eTraversalNeeded;
    }
    if (what & layer_state_t::eFlagsChanged) {
        if (layer->setFlags(s.flags, s.mask)) flags |= eTraversalNeeded;
    }
    if (what & layer_state_t::eCornerRadiusChanged) {
        if (layer->setCornerRadius(s.cornerRadius))
            flags |= eTraversalNeeded;
    }
//省略
    return flags;
}

```

本文章对应视频手把手教你学framework：\
hal+perfetto+surfaceflinger\
<https://mp.weixin.qq.com/s/LbVLnu1udqExHVKxd74ILg>\
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/4fe7ed2eaa2349de18d5d39ef9a1d04b.png#pic_center)

私聊作者+v(androidframework007)

七件套专题：![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/44ae39edb2ef03be6f2aeff206317b23.png#pic_center)\
点击这里 <https://mp.weixin.qq.com/s/Qv8zjgQ0CkalKmvi8tMGaw>

视频：<https://www.bilibili.com/video/BV1wc41117L4/>
