---
title: 自动背光算法-Android 8.1
author: []
created: 2025-08-21
tags:
  - clippings
  - 转载
  - blog
collections: 图形显示
source: https://segmentfault.com/a/1190000022874010?sort=newest
date: 2025-08-21T07:29:37.936Z
lastmod: 2025-10-23T07:56:56.596Z
---
本文基于工作中自动背光笔记扩充了下，记录下自动背光算法。\
基于Android 8.1, 代码可参看 [http://androidxref.com/8.1.0\_...](https://link.segmentfault.com/?enc=S%2FVnvVcbfiJWLu1xPNnWoQ%3D%3D.L2VvVVv78YDcKnviPVlOdjS3H7uSKqzygibjxQLrLdLb%2BAreqZCsVQ2EcdiTwPlq)\
Android 9加入了所谓的机器学习算法，根据用户调节时亮度和光感重新生成曲线，\
自动背光时的滑动条不再是调节adjustment值，暂时不想写了。。。\
Android 10简单看了下，加入了对foreground应用的微调支持，暂时不想写了。。。

## 一、简单说说

简单的说，其实背光调节就三大步，建曲线，采样去抖，计算亮度

### 1. 建曲线

```
Brightness
 1 ^
   |
   |
   |             。
   |            。
   |           。
   +       。
min+   。
   |
   0---+---+-------------------> lux 
          min
```

建曲线这个其实就是建立一个lux对应的brightness的曲线，这样就可以根据光感传来的lux值，计算出对应的亮度为多少。\
（安卓的算法需要Brightness比lux多一个值，所以上图曲线两者的min值不对应。另外需要严格单调递增，组数看需要，上图只是个示意）

### 2. 采样去抖

这个就是采集一段时间内的lux值，然后根据这段时间的lux,算出这段时间的lux值。\
当然考虑到抖动(如有闪电等突然变化很大，然后又恢复的情况;或者周期性的明暗明暗变化等情况），所以需要去抖，\
默认安卓是要求光照稳定在0.8~1.1倍之间，并且持续一段时间（由暗到亮是4秒，由亮到暗是8s）

### 3. 计算亮度

安卓的算法流程\
1\). 开自动背光时，下拉通知条为一系数，假设为x,其区间为\[-1.0, 1.0], 应用层为除以1024,浮点精度

2\). 计算背光时，先计算出lux对应的亮度，假设为y,其区间为\[0.0, 1.0], 考虑为用户会微调，所以然后再处理用户微调(1中的x)

3\). 最终计算屏幕亮度z向下取整

4\). 然后再处理z的极值区间，例如我们设置的11~255, 18~255

用公式表示即为

$$
\mathbf{z = y^{3^{-x}}*255, x\in[-1.0, 1.0], y\in[0.0, 1.0], z\in[min, 255]}
$$

公式中3表示为gamma3.0,后面是3的-x次方,即$3^{-x}$

举个例子，

```
/
出地库        光照稳定在0.8~1.1倍之间4S   /
  |         |                       |/
  +---------+-----------------------+--------------
10lux   5000lux         开始改变屏幕，假设屏幕亮度从55 -> 255, 
                        那么亮度调节完需要时间为((255-55)/rampRate)*16.6ms, 约3.3S
                     (16.6ms是一个vsync时间，rampRate默认为1，可以更改进行速率调节)
```

## 二、说点代码

代码分析其实基本也是按照前面的三点来说的，

### 1. 建曲线配置

```xml
frameworks/base/core/res/res/values/config.xml
<bool name="config_automatic_brightness_available">false</bool> <!-- 自动背光功能是否可用开关 -->
<integer-array name="config_autoBrightnessLevels"> <!-- lux 数组 -->
</integer-array>
<integer-array name="config_autoBrightnessLcdBacklightValues"> <!-- 对应lcd brightness 数组 -->
</integer-array>
```

曲线的配置为如上面的代码里，默认安卓是没有配置的，需要自己修改或者overlay,\
另外还有个是否使用自动背光的开关 config\_automatic\_brightness\_available，默认关的，需要打开。\
代码里都有详细的注释和说明可看看，我这儿就没贴出来了。\
config\_autoBrightnessLevels为lux数组\
config\_autoBrightnessLcdBacklightValues为对应的lcd亮度值，\
比lux数组多一个，组成和曲线需要严格单调递增。

```
config_autoBrightnessLevels -> lux
config_autoBrightnessLcdBacklightValues -> brightness
       --> brightness[0]
lux[0] --> brightness[1]
lux[1] --> brightness[2]
lux[2] --> brightness[3]
.....
```

### 2. 曲线初始化

曲线设置好了，就应该扔代码里跑起来，模型建起来了，这个过程是咋样的呢？\
通过搜索配置关键词(\
config\_automatic\_brightness\_available\
config\_autoBrightnessLevels\
config\_autoBrightnessLcdBacklightValues\
) 或知道其在DisplayPowerController()构造的时候初始化的。

```java
frameworks/base/services/core/java/com/android/server/display/DisplayPowerController.java
public DisplayPowerController(......) {
......
    // 使能开关
    mUseSoftwareAutoBrightnessConfig = resources.getBoolean(
            com.android.internal.R.bool.config_automatic_brightness_available);
......
    if (mUseSoftwareAutoBrightnessConfig) {
        // 光感数组
        int[] lux = resources.getIntArray(
                com.android.internal.R.array.config_autoBrightnessLevels);
        // 背光亮度数组
        int[] screenBrightness = resources.getIntArray(
                com.android.internal.R.array.config_autoBrightnessLcdBacklightValues);
        // 其它一些配置
        int lightSensorWarmUpTimeConfig = resources.getInteger(
                com.android.internal.R.integer.config_lightSensorWarmupTime);
        final float dozeScaleFactor = resources.getFraction(
                com.android.internal.R.fraction.config_screenAutoBrightnessDozeScaleFactor,
                1, 1);
        // 创建自动背光样条
        Spline screenAutoBrightnessSpline = createAutoBrightnessSpline(lux, screenBrightness);
        if (screenAutoBrightnessSpline == null) {
            // 如果lux, brightness不合要求，禁用自动背光
            Slog.e(TAG, "Error in config.xml.  config_autoBrightnessLcdBacklightValues "
                    + "(size " + screenBrightness.length + ") "
                    + "must be monotic and have exactly one more entry than "
                    + "config_autoBrightnessLevels (size " + lux.length + ") "
                    + "which must be strictly increasing.  "
                    + "Auto-brightness will be disabled.");
            mUseSoftwareAutoBrightnessConfig = false;
        } else {
            ......
            mAutomaticBrightnessController = new AutomaticBrightnessController(this,
                   handler.getLooper(), sensorManager, screenAutoBrightnessSpline,
                   ...// gamma, 最大最小值等其他配置
        }
    }
```

如果打开了自动背光功能，会根据lux和brightness数组，创建样条曲线（以得到一条光滑曲线，安卓采用的是三次样条插值，有兴趣的可以搜索下学习学习），\
如果数组不满足要求，会禁用自动背光功能。\
创建样条曲线成功后赋值给 AutomaticBrightnessController(),后面背光控制的主体逻辑就转到 AutomaticBrightnessController类里了。\
AutomaticBrightnessController()构造就一些赋值，没啥可看的，重点看下 createAutoBrightnessSpline()

```java
private static Spline createAutoBrightnessSpline(int[] lux, int[] brightness) {
......
        // 归一化brightness[0]作为y[0]
        y[0] = normalizeAbsoluteBrightness(brightness[0]);
        // 处理对应的lux和brightness
        for (int i = 1; i < n; i++) {
            x[i] = lux[i - 1];
            y[i] = normalizeAbsoluteBrightness(brightness[i]);
        }
        Spline spline = Spline.createSpline(x, y);
......
}
```

创建样条时会做brightness归一化，即除以255,将背光值0~255限定在0~1之间，\
lux为横坐标x轴,\
归一化的亮度为y轴，\
之后再调用createSpline()创建样条。

```java
frameworks/base/core/java/android/util/Spline.java
public static Spline createSpline(float[] x, float[] y) {
    // 是否严格单调递增
    if (!isStrictlyIncreasing(x)) {
        throw new IllegalArgumentException("The control points must all have strictly "
                + "increasing X values.");
    }

    // 单调的还是线性的
    if (isMonotonic(y)) {
        // 创建单调三次样条插值
        return createMonotoneCubicSpline(x, y);
    } else {
        return createLinearSpline(x, y);
    }
}
```

createSpline()先检查是否严格单调递增，然后看是单调的还是线性的，如果是线性的，那这好说，就一条直线，计算也方便，\
没啥可看的，所以可以继续看下 createMonotoneCubicSpline()。

```java
public static Spline createMonotoneCubicSpline(float[] x, float[] y) {
    return new MonotoneCubicSpline(x, y);
}

public static class MonotoneCubicSpline extends Spline {
    // 主要的就是这三个数组，横坐标x,纵坐标y,以及相邻点的斜率
    private float[] mX;
    private float[] mY;
    private float[] mM;

    public MonotoneCubicSpline(float[] x, float[] y) {
        ....
        // d存放相邻点的斜率，m存放相邻两斜率的平均值
        float[] d = new float[n - 1]; // could optimize this out
        float[] m = new float[n];
        // Compute slopes of secant lines between successive points.
        for (int i = 0; i < n - 1; i++) {
            float h = x[i + 1] - x[i];
            ...
            // 相邻点斜率
            d[i] = (y[i + 1] - y[i]) / h;
        }
        // Initialize the tangents as the average of the secants.
        m[0] = d[0];
        for (int i = 1; i < n - 1; i++) {
            // 相邻两斜率的平均值
            m[i] = (d[i - 1] + d[i]) * 0.5f;
        }
        m[n - 1] = d[n - 2];
        // Update the tangents to preserve monotonicity.
        // 再次更新m以保持单调性
        for (int i = 0; i < n - 1; i++) {
            if (d[i] == 0f) { // successive Y values are equal
                m[i] = 0f;
                m[i + 1] = 0f;
            } else {
                float a = m[i] / d[i];
                float b = m[i + 1] / d[i];
                if (a < 0f || b < 0f) {
                    throw new IllegalArgumentException("The control points must have "
                            + "monotonic Y values.");
                }
                float h = (float) Math.hypot(a, b);
                if (h > 3f) {
                    float t = 3f / h;
                    m[i] *= t;
                    m[i + 1] *= t;
                }
            }
        }
        mX = x;
        mY = y;
        mM = m;
    }
```

createMonotoneCubicSpline() 里new了个 MonotoneCubicSpline, 主要是为了得到横纵坐标及相邻斜率均值mX,mY,mM\
这个需要先研究下三次样条插值算法原理才能更好的理解。

至此呢，Spline就创建好了，那什么时候用到呢？\
MonotoneCubicSpline类里有个方法 interpolate(float x)，我们猜测最终根据光感值x计算出背光时就会用到该函数，\
但是讲这个函数之前，还是让我们来看看数据流向吧，\
即看下光传感器数据上来后数据处理流程，如何去抖，选出x,然后再计算亮度，之后再进一步的看看如何处理用户微调(adjustment),\
然后亮度调节就生效了...

### 3. 采样去抖

前面看了样条曲线的建立过程，这节我们先看下lux采集处理过程。

#### 3.1 采样

要有数据肯定先把sensor打通，

```java
frameworks/base/services/core/java/com/android/server/display/DisplayPowerController.java
private void updatePowerState() {
    int brightness = PowerManager.BRIGHTNESS_DEFAULT;
    ......
        // 检查是否使能自动背光
        autoBrightnessEnabled = mPowerRequest.useAutoBrightness
                && (state == Display.STATE_ON || autoBrightnessEnabledInDoze)
                // 这个brightness与Doze模式是否允许使用自动背光配置有关，不是表示屏幕亮度<0,默认是BRIGHTNESS_DEFAULT为-1
                && brightness < 0;
        mAutomaticBrightnessController.configure(autoBrightnessEnabled,
                mPowerRequest.screenAutoBrightnessAdjustment, state != Display.STATE_ON,
                userInitiatedChange);
```

其中updatePowerState()时会根据状态和条件看是否使能自动背光，\
这条条件是 电源请求是否使用自动背光，显示状态是否为开，Doze模式是否允许

```java
frameworks/base/services/core/java/com/android/server/display/AutomaticBrightnessController.java
public void configure(boolean enable, float adjustment, boolean dozing,
        boolean userInitiatedChange) {
......
    boolean changed = setLightSensorEnabled(enable && !dozing);
......
}
```

如果自动背光使能了且非dozing中，会将光感使能，这里面就是sensor listener的注册，之后有就可以得到lux数据了,\
setLightSensorEnabled() 注册listener就不贴代码了，贴下数据回调

```java
private final SensorEventListener mLightSensorListener = new SensorEventListener() {
    @Override
    public void onSensorChanged(SensorEvent event) {
        if (mLightSensorEnabled) {
            final long time = SystemClock.uptimeMillis();
            final float lux = event.values[0];
            handleLightSensorEvent(time, lux);
        }
    }
    ......
};
```

```java
private void handleLightSensorEvent(long time, float lux) {
    // 如果有 MSG_UPDATE_AMBIENT_LUX 消息，则移除
    mHandler.removeMessages(MSG_UPDATE_AMBIENT_LUX);
......
    // 更新缓冲区
    applyLightSensorMeasurement(time, lux);
    // 更新数据
    updateAmbientLux(time);
}

private void applyLightSensorMeasurement(long time, float lux) {
......
    // 将多少周期(默认10s)前的数据删掉
    mAmbientLightRingBuffer.prune(time - mAmbientLightHorizon);
    // 将当前数据放到缓冲中
    mAmbientLightRingBuffer.push(time, lux);
.....
}
```

handleLightSensorEvent()首先取消了 MSG\_UPDATE\_AMBIENT\_LUX 消息，\
该消息的作用是当传感器长时间没有数据上报时，周期性的更新一把缓冲区，\
把过期的数据删除(但需要保留最后个值，并把时间更新为当前时间)(这部分是不是可以优化下)\
然后调用 applyLightSensorMeasurement()将过期的数据删掉，并将当前的值加入到缓冲中。\
默认的周期为10s

```xml
<integer name="config_autoBrightnessAmbientLightHorizon">10000</integer>
```

即缓冲只最多保留10秒的数据，\
**注意这里还只是数据的一个简单记录**\
之后再调用updateAmbientLux() 更新值，\
**注意该函数已经进入到防抖过程了**

#### 3.1 防抖

防抖的流程都在updateAmbientLux()函数里，\
然后用HysteresisLevels类辅助计算明暗光感的阀值。

```java
private void updateAmbientLux(long time) {
    // If the light sensor was just turned on then immediately update our initial
    // estimate of the current ambient light level.
    if (!mAmbientLuxValid) {
        ......
        // 打开sensor后第一次更新数据情况，略过...
    }
    // 计算明暗变化的时间点
    long nextBrightenTransition = nextAmbientLightBrighteningTransition(time);
    long nextDarkenTransition = nextAmbientLightDarkeningTransition(time);
    // 计算10s和2s内的lux
    float slowAmbientLux = calculateAmbientLux(time, AMBIENT_LIGHT_LONG_HORIZON_MILLIS);
    float fastAmbientLux = calculateAmbientLux(time, AMBIENT_LIGHT_SHORT_HORIZON_MILLIS);
    // 如果满足去抖的要求，会进行lux和亮度的计算更新
    if (slowAmbientLux >= mBrighteningLuxThreshold &&
            fastAmbientLux >= mBrighteningLuxThreshold && nextBrightenTransition <= time
            || slowAmbientLux <= mDarkeningLuxThreshold
            && fastAmbientLux <= mDarkeningLuxThreshold && nextDarkenTransition <= time) {
        // 更新lux和明暗lux阀值
        setAmbientLux(fastAmbientLux);
        // 这个log应该放在setAmbientLux()前面，不然 fastAmbientLux > mAmbientLux永远为false
        if (DEBUG) {
            Slog.d(TAG, "updateAmbientLux: "
                    + ((fastAmbientLux > mAmbientLux) ? "Brightened" : "Darkened") + ": "
                    + "mBrighteningLuxThreshold=" + mBrighteningLuxThreshold
                    + ", mAmbientLightRingBuffer=" + mAmbientLightRingBuffer
                    + ", mAmbientLux=" + mAmbientLux);
        }
        // 更新亮度
        updateAutoBrightness(true);
    ......
    mHandler.sendEmptyMessageAtTime(MSG_UPDATE_AMBIENT_LUX, nextTransitionTime);
}
```

整个流程大体为

* 计算明暗变化的时间点 防止周期性的大幅度抖动
* 计算2s和10s内的lux 用于去抖比较，使光照稳定在0.8~1.1倍之间
* 如果满足去抖的要求，会进行lux、明暗阀值和亮度的计算更新

上面代码都有注释，下面我们看一些方法的具体代码

计算明暗变化(由暗到亮场景，由亮到暗场景)时间点

```java
private long nextAmbientLightBrighteningTransition(long time) {
    final int N = mAmbientLightRingBuffer.size();
    long earliestValidTime = time;
    // 从后往前找到离当前时间第一个大于阀值的点
    for (int i = N - 1; i >= 0; i--) {
        if (mAmbientLightRingBuffer.getLux(i) <= mBrighteningLuxThreshold) {
            break;
        }
        earliestValidTime = mAmbientLightRingBuffer.getTime(i);
    }
    // 找到的点时间加上去抖时间
    return earliestValidTime + mBrighteningLightDebounceConfig;
}

private long nextAmbientLightDarkeningTransition(long time) {
    final int N = mAmbientLightRingBuffer.size();
    long earliestValidTime = time;
    从后往前找到离当前时间第一个小于阀值的点
    for (int i = N - 1; i >= 0; i--) {
        if (mAmbientLightRingBuffer.getLux(i) >= mDarkeningLuxThreshold) {
            break;
        }
        earliestValidTime = mAmbientLightRingBuffer.getTime(i);
    }
    // 找到的点时间加上去抖时间
    return earliestValidTime + mDarkeningLightDebounceConfig;
}
```

由暗变亮 nextAmbientLightBrighteningTransition()\
从后往前找到离当前时间第一个大于阀值的点，然后加上去抖时间 mBrighteningLightDebounceConfig ，即为next Brightening的时间点，\
默认为4s

```xml
<integer name="config_autoBrightnessBrighteningLightDebounce">4000</integer>
```

由亮变暗 nextAmbientLightDarkeningTransition()\
从后往前找到离当前时间第一个小于阀值的点，然后加上去抖时间 mDarkeningLightDebounceConfig ，即为next Darkening的时间点，\
默认为8s

```xml
<integer name="config_autoBrightnessDarkeningLightDebounce">8000</integer>
```

**通过此方法，就可以防止数据周期性或者最近有大幅度波动的状况**

计算2s和10s内的lux\
用到的函数为calculateAmbientLux(),\
该函数用于计算当前时间，多少周期前内的lux值，采用的是加权平均算法。\
具体的代码可看下注释，我也没明白权值计算weightIntegral()为什么要用那个算法。

```java
private float calculateAmbientLux(long now, long horizon) {
......
    // Find the first measurement that is just outside of the horizon.
    int endIndex = 0;
    // 计算出指定时间的开始下标
    final long horizonStartTime = now - horizon;
    for (int i = 0; i < N-1; i++) {
        if (mAmbientLightRingBuffer.getTime(i + 1) <= horizonStartTime) {
            endIndex++;
        } else {
            break;
        }
    }
    ......
    float sum = 0;
    float totalWeight = 0;
    long endTime = AMBIENT_LIGHT_PREDICTION_TIME_MILLIS;
    for (int i = N - 1; i >= endIndex; i--) {
        long eventTime = mAmbientLightRingBuffer.getTime(i);
        if (i == endIndex && eventTime < horizonStartTime) {
            // If we're at the final value, make sure we only consider the part of the sample
            // within our desired horizon.
            eventTime = horizonStartTime;
        }
        final long startTime = eventTime - now;
        // 权值计算
        float weight = calculateWeight(startTime, endTime);
        float lux = mAmbientLightRingBuffer.getLux(i);
        ......
        totalWeight += weight;
        sum += mAmbientLightRingBuffer.getLux(i) * weight;
        endTime = startTime;
    }
    ......
    // 加权平均
    return sum / totalWeight;
}
```

接下来，如果2s和10s内都大于(或小于阀值)，且达到了防抖时间要求，则更新lux,阈值，亮度

```
if (slowAmbientLux >= mBrighteningLuxThreshold &&
            fastAmbientLux >= mBrighteningLuxThreshold && nextBrightenTransition <= time
            || slowAmbientLux <= mDarkeningLuxThreshold
            && fastAmbientLux <= mDarkeningLuxThreshold && nextDarkenTransition <= time)
```

更新lux和阀值 (**以2s的加权平均lux更新**)

```java
setAmbientLux(fastAmbientLux)
private void setAmbientLux(float lux) {
    ...
    // 更新lux
    mAmbientLux = lux;
    // 更新阀值
    mBrighteningLuxThreshold = mDynamicHysteresis.getBrighteningThreshold(lux);
    mDarkeningLuxThreshold = mDynamicHysteresis.getDarkeningThreshold(lux);
}
```

更新阀值getBrighteningThreshold() getDarkeningThreshold()的主要计算为

```java
float brightThreshold = lux * (1.0f + brightConstant);
float darkThreshold = lux * (1.0f - darkConstant);
```

Constant涉及到配置

```xml
<integer-array name="config_dynamicHysteresisBrightLevels">
    <item>100</item>
</integer-array>
<integer-array name="config_dynamicHysteresisDarkLevels">
    <item>200</item>
</integer-array>
<integer-array name="config_dynamicHysteresisLuxLevels">
</integer-array>
```

因为没有配置config\_dynamicHysteresisLuxLevels，所以最终\
即由暗到亮场景，需要<0.8\*lux\
即由暗到亮场景，需要>1.1\*lux\
**即可以简单认为数据在0.8~1.1倍之间波动都认为是OK的。这也是去抖一部分**

**至此，去抖功能就完成了，简单的说就是**\
**通过 nextAmbient...Transition 找到防抖完成时间 (稳定4/8秒)，**\
**然后在2～10s内加权平均lux在0.8~1.1倍阀值。**

### 4. 更新亮度 updateAutoBrightness()

setAmbientLux()完成后就该updateAutoBrightness(true)了，\
参数true表示最终会调用 DisplayPowerController 进行实际的亮度更新

```java
private void updateAutoBrightness(boolean sendUpdate) {
    // 如果自动光感没数据，直接返回了。
    if (!mAmbientLuxValid) {
        return;
    }

    // 从建模的曲线中算出对应的brightness，这只是初步的亮度值
    float value = mScreenAutoBrightnessSpline.interpolate(mAmbientLux);
    float gamma = 1.0f;

    // 如果用户有调整，处理用户的adjustment
    if (USE_SCREEN_AUTO_BRIGHTNESS_ADJUSTMENT
            && mScreenAutoBrightnessAdjustment != 0.0f) {
        // mScreenAutoBrightnessAdjustmentMaxGamma 配置里用的是300%
        // 即我们前面列的公式的 3^(-x) x为-1.0~1.0之间
        final float adjGamma = MathUtils.pow(mScreenAutoBrightnessAdjustmentMaxGamma,
                Math.min(1.0f, Math.max(-1.0f, -mScreenAutoBrightnessAdjustment)));
        gamma *= adjGamma;
        .....
    }

    // gamma不为1.0f表示用户有调整
    if (gamma != 1.0f) {
        final float in = value;
        // value^gamma
        value = MathUtils.pow(value, gamma);
        ....
    }
    
    // value*255向下取整并使值在mScreenBrightnessRangeMinimum～mScreenBrightnessRangeMaximum区间
    // 这样就得出了最终亮度
    int newScreenAutoBrightness =
            clampScreenBrightness(Math.round(value * PowerManager.BRIGHTNESS_ON));
    if (mScreenAutoBrightness != newScreenAutoBrightness) {
        ......
        if (sendUpdate) {
            // 通知更新
            mCallbacks.updateBrightness();
        }
    }
}
```

更新亮度的流程也挺简单的，就是

* 从建模的曲线中算出对应的初步亮度值
* 如果用户有调整，进一步处理用户调整
* 处理最终的亮度值，使其在极值区间里

对应的就是我们前面列的公式

$$
\mathbf{z = y^{3^{-x}}*255, x\in[-1.0, 1.0], y\in[0.0, 1.0], z\in[min, 255]}
$$

具体的就不分步讲了，可看下我的注释，这里只贴下Spline interpolate()代码\
我们之前建模时讲了，创建spline可能是线性的，也可能是MonotoneCubicSpline，\
我们这只看看MonotoneCubicSpline的，所以对应代码为

```java
frameworks/base/core/java/android/util/Spline.java
public static class MonotoneCubicSpline extends Spline {
    ......
    @Override
    public float interpolate(float x) {
        ......
        // 小于最小lux，直接反回brightness[0]
        if (x <= mX[0]) {
            return mY[0];
        }
        // 大于最大的lux, 直接返回最大的
        if (x >= mX[n - 1]) {
            return mY[n - 1];
        }
        // Find the index 'i' of the last point with smaller X.
        // We know this will be within the spline due to the boundary tests.
        int i = 0;
        // 刚好落在采样点上
        while (x >= mX[i + 1]) {
            i += 1;
            if (x == mX[i]) {
                return mY[i];
            }
        }
        // Perform cubic Hermite spline interpolation.
        float h = mX[i + 1] - mX[i];
        float t = (x - mX[i]) / h;
        // 三次插值计算
        return (mY[i] * (1 + 2 * t) + h * mM[i] * t) * (1 - t) * (1 - t)
                + (mY[i + 1] * (3 - 2 * t) + h * mM[i + 1] * (t - 1)) * t * t;
}
```

插值时，大于最大小于最小或者落在点上的情况都好说，直接返回就行了\
注意 mX\[0] 并不是 config\_autoBrightnessLevels 中的第一个lux,\
实际上 createAutoBrightnessSpline() 时 float\[] x = new float\[n];\
其实mX\[0] = 0;

若点都不在极值和采样点上，那就得插值计算了，这个具体的算法原理我也没了解。

关于背光算法就说到这儿吧，大体上就这么多东西。\
之后的DisplayPowerController更新亮度也不说了，需要了解有个\
mBrightnessRampRateSlow //自动背光用的这个\
mBrightnessRampRateFast\
可以控制背光开始变化时的变化快慢就行了

```xml
<!-- Fast brightness animation ramp rate in brightness units per second-->
<integer translatable="false" name="config_brightness_ramp_rate_fast">180</integer>

<!-- Slow brightness animation ramp rate in brightness units per second-->
<integer translatable="false" name="config_brightness_ramp_rate_slow">60</integer>
```

### 5. 用户微调

当自动背光开时，背光亮度条变成了个调节系数，值在\[-1.0f, 1.0f], 默认在0.0f,\
当用户拖动这个条时，最终会写到 Settings.System.SCREEN\_AUTO\_BRIGHTNESS\_ADJ 数据库里，\
然后 PowerManagerService 监测到数据库变更，最终给了 mPowerRequest.screenAutoBrightnessAdjustment，\
DisplayPowerController.java updatePowerState() 时

```java
mAutomaticBrightnessController.configure(....
                    mPowerRequest.screenAutoBrightnessAdjustment....
```

最终赋值给了 mScreenAutoBrightnessAdjustment，计算亮度时就用到了该值

```java
AutomaticBrightnessController.java
public void configure(boolean enable, float adjustment, boolean dozing,
....
    changed |= setScreenAutoBrightnessAdjustment(adjustment);

private boolean setScreenAutoBrightnessAdjustment(float adjustment) {
    ....
        mScreenAutoBrightnessAdjustment = adjustment;
```

### 6. 总结:

* 涉及到的一些配置

```
frameworks/base/core/res/res/values/config.xml
// 自动背光功能是否可用开关, 默认关，需要打开
config_automatic_brightness_available
// lux 数组 
config_autoBrightnessLevels
// 对应lcd brightness 数组
config_autoBrightnessLcdBacklightValues
// 光感数据缓冲区周期，即采集多少时间内的数据
config_autoBrightnessAmbientLightHorizon
// 由暗到亮场景防抖时间，注意不要超过 config_autoBrightnessAmbientLightHorizon
config_autoBrightnessBrighteningLightDebounce
// 由亮到暗场景防抖时间，注意不要超过 config_autoBrightnessAmbientLightHorizon
config_autoBrightnessDarkeningLightDebounce
// 计算变亮变暗场景阀值时的Constant
config_dynamicHysteresisBrightLevels
config_dynamicHysteresisDarkLevels
config_dynamicHysteresisLuxLevels
// 屏幕亮度变化时的ramp rate
config_brightness_ramp_rate_slow
```

* 创建 Spline

```
DisplayPowerController.java
public DisplayPowerController()
    + Spline screenAutoBrightnessSpline = createAutoBrightnessSpline(lux, screenBrightness);
    |                                        + normalize
    |                                        + createSpline()
    |                                            + createMonotoneCubicSpline()
    |                                                + new MonotoneCubicSpline(x, y)
    |                                                    + mX = x;
    |                                                    + mY = y;
    |                                                    + mM = m;
    + mAutomaticBrightnessController = new AutomaticBrightnessController(...screenAutoBrightnessSpline,...)
```

* 使能自动背光功能并注册光感listener

```
updatePowerState() DisplayPowerController.java
  + mAutomaticBrightnessController.configure(autoBrightnessEnabled,
      + configure() AutomaticBrightnessController.java
          + setLightSensorEnabled()
              + registerListener(mLightSensorListener
```

* 当有光感数据上来时, 处理数据，删掉过期数据，并将数据加入到缓冲中，然后去抖，更新lux, 更新亮度\
  当然，如果没新数据还上，还会周期性的 MSG\_UPDATE\_AMBIENT\_LUX

```java
mLightSensorListener
onSensorChanged()
  + handleLightSensorEvent()
  |   + applyLightSensorMeasurement()
  |   |   + mAmbientLightRingBuffer.prune(time - mAmbientLightHorizon);
  |   |   + mAmbientLightRingBuffer.push(time, lux);
  |   + updateAmbientLux(time)
  |   |   + long nextBrightenTransition = nextAmbientLightBrighteningTransition(time);
  |   |   + long nextDarkenTransition = nextAmbientLightDarkeningTransition(time);
  |   |   + float slowAmbientLux = calculateAmbientLux(time, AMBIENT_LIGHT_LONG_HORIZON_MILLIS);
  |   |   + float fastAmbientLux = calculateAmbientLux(time, AMBIENT_LIGHT_SHORT_HORIZON_MILLIS);
  |   |   + if (slowAmbientLux >= mBrighteningLuxThreshold &&...
  |   |   |     + setAmbientLux(fastAmbientLux);
  |   |   |     |   + mAmbientLux = lux;
  |   |   |     |   + mBrighteningLuxThreshold = mDynamicHysteresis.getBrighteningThreshold(lux);
  |   |   |     |   |                              + lux * (1.0f + brightConstant); // 1.1lux
  |   |   |     |   + mDarkeningLuxThreshold = mDynamicHysteresis.getDarkeningThreshold(lux);
  |   |   |     |                                  + lux * (1.0f - brightConstant); // 0.8lux
  |   |   |     + updateAutoBrightness(true);
  |   |   |     |   + mScreenAutoBrightnessSpline.interpolate(mAmbientLux) // 计算出y
  |   |   |     |   + MathUtils.pow(mScreenAutoBrightnessAdjustmentMaxGamma... -mScreenAutoBrightnessAdjustment) // 3^(-x)
  |   |   |     |   + MathUtils.pow(value, gamma); // y^{3^{-x}}
  |   |   |     |   + clampScreenBrightness(Math.round(value * PowerManager.BRIGHTNESS_ON)); // y^{3^{-x}} * 255
  +   +   +     +   + mCallbacks.updateBrightness()
```

## 三、一些调试命令

* 打开自动背光

```shell
adb shell settings put system screen_brightness_mode 1
```

* 背光系数调整

```shell
adb shell settings put system screen_auto_brightness_adj 0.0
```

* dump display, 可查看dump信息，包括建模的一些点值等...

```shell
adb shell dumpsys display
```
