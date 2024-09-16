---
tags:
  - 实战
  - 算法研究
  - blog
  - sensor
categories:
  - Android
title: adsp wristdown 流程与实现
date: 2024-03-31T02:00:33.830Z
lastmod: 2024-09-16T04:54:06.331Z
---
\#Peppermill #算法研究

目前 落腕灭屏算法大致在 framework 层实现。\
目前算法的思路：

1. [#运动检测](#%E8%BF%90%E5%8A%A8%E6%A3%80%E6%B5%8B)，通过一些特征值，来判断是否是一个运动段落。
2. 符合的运动检测后，我们来判断是否是落腕的动作。
3. 符合的落腕的动作后，sensor 上报 event。 framework 监听获取后进行灭屏的判断动作。

#### 背景介绍

早起的 Android 平台，Sensor 是放在 AP 侧实现的，Sensor 生成设备节点供上层使用，Sensor 的工作 CPU 就不能完好的休眠。\
后续的高版本 android 各个平台厂商都有了不同的方案，SensorHub，ADSP

###### SensorHub

个人理解啊，SensorHub 应该算一个独立的子系统，MCU，可以不依赖 CPU 架构独立运行，可以较大的程度省电。主要功能就是连接各类 sensor 并且去处理，MTK 采用的该方案，使用的是 SRAM 架构，通电就可以保存数据，相对内存需要不停的刷新，长期运行可以节省用电。

###### ADSP

#### ADSP 部分

之前把落腕的检测放在 framework 里面实现，在亮屏之后，监听 Asensor 和 GSensor，做为落腕其实只在亮屏后使用，整体上来说，是不影响耗流。\
将算法下沉到 ADSP，主要考虑后续方案的适配性，针对一些低耗流的需求，可以能够较好的满足。

##### 编译

记录我目前编译的一个问题，完整的 AMSS 编译 OK，但是单独编译 ADSP 的时候就会报错

```shell
RuntimeError: Cannot proceed with encryption/decryption: openssl v1.0.1 is unavailable.
```

应该是编译出 adsp 的 mbn 后，签名找不到 openssl 导致的，全编译和单编译的环境变量可能有个环节有问题。后来添加了编译的环境变量才能够正常的编译

```
 export PATH=/pkg/qct/software/python/2.7/bin:$PATH
```

##### 虚拟合成 Sensor 添加

这部分是驱动同事做的，目前我只是记录下来，具体代码我也同步在了 Media\&File -> WristDown 路径下面\
主要一个是在 adsp\_proc/ssc/build/ssc.scons 和 adsp\_proc/ssc\_api/build/ssc\_api.scons 下添加 sensor

```diff
diff --git a/adsp_proc/ssc/build/ssc.scons b/adsp_proc/ssc/build/ssc.scons
index 8b6ab05..28f98a9 100755 (executable)
--- a/adsp_proc/ssc/build/ssc.scons
+++ b/adsp_proc/ssc/build/ssc.scons
@@ -129,7 +129,7 @@ if env.IsKeyEnable(ssc_build_tags) is True:
   #-------------------------------------------------------------------------------
-  'sns_pedometer','sns_psmd','sns_rmd','sns_rotv','smd','sns_multishake','sns_threshold','sns_tilt','sns_tilt_to_wake','sdc',
+  'sns_pedometer','sns_psmd','sns_rmd','sns_rotv','smd','sns_multishake','sns_threshold','sns_tilt','sns_tilt_to_wake','sdc','sns_wrist_tilt_updown',
   'pah_813x_algo','pah_hr','sns_pah_hrd_algo_lib']

   #-------------------------------------------------------------------------------
```

```diff
diff --git a/adsp_proc/ssc_api/build/ssc_api.scons b/adsp_proc/ssc_api/build/ssc_api.scons
index b5ca9ee..d67bc30 100755 (executable)
--- a/adsp_proc/ssc_api/build/ssc_api.scons
+++ b/adsp_proc/ssc_api/build/ssc_api.scons
@@ -127,7 +127,8 @@ WHITELIST = [
   'sns_heart_rate.proto',
   'sns_offbody_detect.proto',
   'sns_ppg.proto',
-  'sns_philips_vso.proto']
+  'sns_philips_vso.proto',
+  'sns_wrist_tilt_updown.proto']

 if 'SENSORS_ALGO_DEV_FLAG' in env:
    WHITELIST += [
```

sensor hal 里面也要添加

###### 算法添加

主要是在 adsp\_proc/ssc/sensors/wrist\_tilt\_updown/src/sns\_wrist\_tilt\_updown\_sensor\_instance\_island.c 里面，\
根据获取到的 Asensor 和 Gsensor 数据来计算处理

#### 算法部分

##### 数据结构

定义了一个先进先出的队列，用来保存 A sensor 和 G sensor 的数据

```c
/*===========================================================================
  LinkedList Definitions
  ===========================================================================*/

/* init */
void LinkedList_init(LinkedList *list, int capacity) {
    list->elements = (SensorState *)malloc(sizeof(SensorState) * capacity);
    list->front = 0;
    list->rear = -1;
    list->capacity = capacity;
    list->count = 0;
}

/* destory */
void LinkedList_destroy(LinkedList *list) {
    free(list->elements);
}

/* add item to the tail */
void LinkedList_enqueue(LinkedList *list, SensorState item,sns_sensor_instance *const this) {
    list->rear = (list->rear + 1) % list->capacity;
    list->elements[list->rear] = item;

    SNS_INST_PRINTF(ERROR, this, "wentao LinkedList_enqueue  item speed = %d /1000", (int) (1000*item.mSpeed) );

    if (list->count < list->capacity) {
        list->count++;
    } else {
        list->front = (list->front + 1) % list->capacity;
    }
}

/* remove oldest item */
SensorState LinkedList_dequeue(LinkedList *list) {
    SensorState item = list->elements[list->front];
    list->front = (list->front + 1) % list->capacity;
    list->count--;

    return item;
}

typedef struct {
    double mSpeed;
    float x;
    float y;
    float z;
    uint64_t timestamp;
} SensorState;

typedef struct {
    SensorState *elements;
    int front;
    int rear;
    int capacity;
    int count;
} LinkedList;

```

##### 运动检测

采样数据

1. a Sensor 加速度
2. g Sensor 角速度

运动检测主要根据加速度来计算。\
目前的运动检测算法 根据经验值来判断，后续如果有优化机会，还是考虑采用更科学的计算方法。

```java

//这里面采样频率大概80ms一次，那么一个10个单位的采样数组，大概能记录到0.8秒前的数据。
            float deltaX = x - lastX;
            float deltaY = y - lastY;
            float deltaZ = z - lastZ;

//通过计算加速度差值的开方除以间隔时间，作为就检测运动强度的判断依据
            float sqrtV = sqrt(deltaX * deltaX + deltaY * deltaY + deltaZ * deltaZ);
            double speed =(double) ((sqrtV*1000.0f / (int)timeInterval));


            SensorState state = {speed,x,y,z,time};
//加入到队列里面
            if (accelLinkedList.count >= QUEE_LENGTH) {
                LinkedList_dequeue(&accelLinkedList);
                LinkedList_enqueue(&accelLinkedList, state,this);
/*
//一次有效的手势运动的特征：
手势动作的特征都是一个个的波形，有强度高峰，然后区域平静
1. 往前追溯，有初始运动强度的要求，数组开始阶段有一定的强度要求，高于某个阈值
2. 运动强度是趋于下降的
3. 后期趋于平静，后续运动强度要低于一定的阈值
这里的调整，如果强度阈值设置的太低，就会太灵敏。太高就会太迟缓
低强度阈值也不能设置的太低，除非放在桌子上，不然手腕一般都都有一定的运动值
TODO： 后续看看学习数字信号的处理，能不能通过滤波，过滤出运动来。
*/
int isEffectEvent(LinkedList *list,sns_sensor_instance *const this) {
    int result = 0;

    if (list->count >= QUEE_LENGTH) {
        SensorState state0 = list->elements[list->front];
        SensorState state1 = list->elements[(list->front + 1) % list->capacity];

        SensorState state7 = list->elements[(list->front + 7) % list->capacity];
        SensorState state8 = list->elements[(list->front + 8) % list->capacity];
        SensorState state9 = list->elements[(list->front + 9) % list->capacity];

        if (state0.mSpeed > state1.mSpeed &&
            (state0.mSpeed > 50.0 || state1.mSpeed > 50.0) &&
            state7.mSpeed < 25.0 &&
            state8.mSpeed < 25.0 &&
            state9.mSpeed < 25.0) {
            result = 1;
        }
    }
    //SNS_INST_PRINTF(ERROR, this, "wentao isEffectEvent result = %d",result);
    return result;
}
```

检测到有效运动之后，判断特征是否是落腕

```java
if (accelLinkedList.count >= QUEE_LENGTH) {
	LinkedList_dequeue(&accelLinkedList);
	LinkedList_enqueue(&accelLinkedList, state,this);

	if (isEffectEvent(&accelLinkedList,this) == 1) {

		int totaly = 0;
		int totalAccelz = 0;
		int index = 0;

		int totalx = 0;
		//累计陀螺仪的数值，角速度的累计，
		//这里算法有些问题。应该是角速度乘以与上次上报时间的间隔来累计，而不是固定乘率去累计
		//TODO:
		for (int i = 0; i < gyroLinkedList.count; i++) {
			totalx += (int)(gyroLinkedList.elements[i].x * 100.0f);
			totaly += (int)(gyroLinkedList.elements[i].y * 100.0f);
		}
		//平均Z方向的值，这里也有些问题,b不应该去计算完成的平均z方向。
		//应该计算运动静止下来后的平均方向，这里应该建议去后一半数组的平均值
		//TODO:
		for (; index < accelLinkedList.count; index++) {
			totalAccelz += accelLinkedList.elements[(accelLinkedList.front + index) % accelLinkedList.capacity].z;
		}

		int avgAccelZ = 15;
		if (index != 0) {
			avgAccelZ = totalAccelz / index;
		}

		//1.陀螺仪判断标准，目前是经验值，totalx 表示的向外翻转的程度 负数表达的是向外翻转
		//由于客户的要求，这里精度降低了，大部分情况可能下一个条件满足的情形下，这个条件都满足了。
		//加上了z 《 6的判断，表示表盘太正面的情况下也不算落腕
		if (totalx < -40000 && z < 6.0f ) {//gyro
			result = true;
		}

		//2 实时的z轴，也就是表盘正上方的姿态 9.8 对应的是地球重力，如果正上方，z应该是9，8
		//完全竖着z应该0， 4.9以上大概就是表盘比较正面的一个状态。
		//同时添加限制条件，y<5.0 ，过滤掉表面正对人脸侧过来的这种情况。
		if ((z < 4.9f || (avgAccelZ < 5.0f && z < 9.0f)) && y < 5.0f) {
			result = true;
		}

		if(result){
			LinkedList_init(&accelLinkedList, QUEE_LENGTH);
		}
	}
```

#### Framework 部分

由于虚拟的是一个一次性的 sensor\
注册 sensor 用的方法：

> mSensorManager.requestTriggerSensor(triggerEventListener, wristDownSensor);

取消 sensor 的方法：

> mSensorManager.cancelTriggerSensor(triggerEventListener, wristDownSensor);

```java
private void updateWristDownSensorListener() {
	SensorManager mSensorManager = (SensorManager) mContext.getSystemService(Context.SENSOR_SERVICE);
	Sensor wristDownSensor = mSensorManager.getDefaultSensor(Sensor.TYPE_WRIST_DOWN);

	if(wristDownSensor != null){
		//reset set listener
		mSensorManager.cancelTriggerSensor(triggerEventListener, wristDownSensor);
		triggerEventListener = null;

		//init triggerEventListener
		triggerEventListener = new TriggerEventListener() {
			@Override
			public void onTrigger(TriggerEvent event) {
				onWristDownTrigger();
			}
		};
		mSensorManager.requestTriggerSensor(triggerEventListener, wristDownSensor);
	}
}
```

亮屏注册，息屏取消

```java
@Override
public void onDisplayStateChange(int state) {
	// This method is only needed to support legacy display blanking behavior
	// where the display's power state is coupled to suspend or to the power HAL.
	// The order of operations matters here.
	synchronized (mLock) {
		if (mDisplayState != state) {
			mDisplayState = state;
			//... 省略

			if (state == Display.STATE_OFF) {
				Log.e(TAG,"onDisplayStateChange STATE_OFF");
				cancleWristDownSensor();
			}

			if (state == Display.STATE_ON) {
				Log.e(TAG,"onDisplayStateChange STATE_ON");
				updateWristDownSensorListener();
			}
		}
	}
}
```

触发函数：\
走 power key 的灭屏流程：

> goToSleepInternal(time,PowerManager.GO\_TO\_SLEEP\_REASON\_WRISTDOWN,0,Process.SYSTEM\_UID);

添加了一个息屏 reason: GO\_TO\_SLEEP\_REASON\_WRISTDOWN ,用于 log 和其余业务的判断依据

添加了 isSkipWristDown，增加了接口，用于过滤一些无需要手势息屏的业务。\
(mWakeLockSummary & WAKE\_LOCK\_STAY\_AWAKE) != 0 主要针对 申请 wakelock 的页面，不去执行息屏。

**PS：**\
由于 sensor 虚拟的是 TriggerSensor，在 onChange 方法后，sensor 会自动任务结束。\
所以如果判断到非息屏状态，要从新去注册 sensor

```java
private void onWristDownTrigger(){
	final long time = mClock.uptimeMillis();

	boolean isWristDownEnable = mSystemProperties.get(WRIST_DOWN_ENABLE, "1").equals("1");

	if(isWristDownEnable){
		if(!isSkipWristDown()){
			goToSleepInternal(time,PowerManager.GO_TO_SLEEP_REASON_WRISTDOWN,0,Process.SYSTEM_UID);
		}else{
			updateWristDownSensorListener();
		}
	}
}

private boolean isSkipWristDown(){
	return mStayOn
		|| mProximityPositive
		|| (mWakeLockSummary & WAKE_LOCK_STAY_AWAKE) != 0
		|| mScreenBrightnessBoostInProgress
		|| !mBootCompleted;
}
```

#### 参考部分

##### 参考到的资料

1.[基于加速度传感器的人体运动模式识别](http://www.c-s-a.org.cn/html/2020/6/7443.html)

2.[基于惯性传感器 MPU6050 的手势识别方法.pdf](%E5%9F%BA%E4%BA%8E%E6%83%AF%E6%80%A7%E4%BC%A0%E6%84%9F%E5%99%A8%20MPU6050%20%E7%9A%84%E6%89%8B%E5%8A%BF%E8%AF%86%E5%88%AB%E6%96%B9%E6%B3%95.pdf)

3.[基于三轴加速度传感器的手势识别-刘蓉.pdf](https://max.book118.com/html/2018/1013/6232040220001222.shtm)\
论文连接：[手势参考资料资源#基于三轴加速度传感器的手势识别-刘蓉](%E6%89%8B%E5%8A%BF%E5%8F%82%E8%80%83%E8%B5%84%E6%96%99%E8%B5%84%E6%BA%90#%E5%9F%BA%E4%BA%8E%E4%B8%89%E8%BD%B4%E5%8A%A0%E9%80%9F%E5%BA%A6%E4%BC%A0%E6%84%9F%E5%99%A8%E7%9A%84%E6%89%8B%E5%8A%BF%E8%AF%86%E5%88%AB-%E5%88%98%E8%93%89)

##### 搜索到的无关资料

[Site Unreachable](https://github.com/xioTechnologies/Fusion)
