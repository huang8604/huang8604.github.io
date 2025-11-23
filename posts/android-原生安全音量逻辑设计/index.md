# Android 原生安全音量逻辑设计

\*\*\*\*## 前言

接到一个开发需求，需要定制化开发一个安全音量功能；此前有了解过为了符合欧盟等有关国家和地区的规定，原生Android是有自带一个安全音量功能的，想要定制则先要了解这个功能原先长什么样子，下面我们就从一个系统工程师的角度出发去探寻一下，原生Android的安全音量功能是如何实现的。

## 安全音量配置

安全音量的相关配置都在framework的config.xml里面，可以直接修改或者overlay配置修改其默认值。

```xml
<!-- Whether safe headphone volume is enabled or not (country specific). -->
<bool name="config_safe_media_volume_enabled">true</bool>
```

```xml
<!-- Safe headphone volume index. When music stream volume is below this index
the SPL on headphone output is compliant to EN 60950 requirements for portable music
players. -->
<integer name="config_safe_media_volume_index">10</integer>
```

config\_safe\_media\_volume\_enabled是安全音量功能的总开关，config\_safe\_media\_volume\_index则是表明触发安全音量弹框的音量大小值。

## 安全音量相关流程

安全音量的主要流程都在AudioService里面，其大致流程如下图所示：

<svg aria-roledescription="flowchart-v2" role="graphics-document document" viewBox="-8 -8 684.30859375 318.5" style="max-width: 684.30859375px;" xmlns:xlink="http://www.w3.org/1999/xlink" xmlns="http://www.w3.org/2000/svg" width="100%" id="bytemd-mermaid-1729844663611-0"><g><marker orient="auto" markerHeight="12" markerWidth="12" markerUnits="userSpaceOnUse" refY="5" refX="10" viewBox="0 0 12 20" class="marker flowchart" id="flowchart-pointEnd"><path style="stroke-width: 1; stroke-dasharray: 1, 0;" class="arrowMarkerPath" d="M 0 0 L 10 5 L 0 10 z"></path></marker><marker orient="auto" markerHeight="12" markerWidth="12" markerUnits="userSpaceOnUse" refY="5" refX="0" viewBox="0 0 10 10" class="marker flowchart" id="flowchart-pointStart"><path style="stroke-width: 1; stroke-dasharray: 1, 0;" class="arrowMarkerPath" d="M 0 5 L 10 10 L 10 0 z"></path></marker><marker orient="auto" markerHeight="11" markerWidth="11" markerUnits="userSpaceOnUse" refY="5" refX="11" viewBox="0 0 10 10" class="marker flowchart" id="flowchart-circleEnd"><circle style="stroke-width: 1; stroke-dasharray: 1, 0;" class="arrowMarkerPath" r="5" cy="5" cx="5"></circle></marker><marker orient="auto" markerHeight="11" markerWidth="11" markerUnits="userSpaceOnUse" refY="5" refX="-1" viewBox="0 0 10 10" class="marker flowchart" id="flowchart-circleStart"><circle style="stroke-width: 1; stroke-dasharray: 1, 0;" class="arrowMarkerPath" r="5" cy="5" cx="5"></circle></marker><marker orient="auto" markerHeight="11" markerWidth="11" markerUnits="userSpaceOnUse" refY="5.2" refX="12" viewBox="0 0 11 11" class="marker cross flowchart" id="flowchart-crossEnd"><path style="stroke-width: 2; stroke-dasharray: 1, 0;" class="arrowMarkerPath" d="M 1,1 l 9,9 M 10,1 l -9,9"></path></marker><marker orient="auto" markerHeight="11" markerWidth="11" markerUnits="userSpaceOnUse" refY="5.2" refX="-1" viewBox="0 0 11 11" class="marker cross flowchart" id="flowchart-crossStart"><path style="stroke-width: 2; stroke-dasharray: 1, 0;" class="arrowMarkerPath" d="M 1,1 l 9,9 M 10,1 l -9,9"></path></marker><g class="root"><g class="clusters"></g><g class="edgePaths"><path marker-end="url(#flowchart-pointEnd)" style="fill:none;" class="edge-thickness-normal edge-pattern-solid flowchart-link LS-A LE-B" id="L-A-B-0" d="M173.531,33.5L173.531,39.208C173.531,44.917,173.531,56.333,173.531,67.75C173.531,79.167,173.531,90.583,173.531,96.292L173.531,102"></path><path marker-end="url(#flowchart-pointEnd)" style="fill:none;stroke-width:2px;stroke-dasharray:3;" class="edge-thickness-normal edge-pattern-dotted flowchart-link LS-B LE-E" id="L-B-E-0" d="M173.531,135.5L173.531,139.667C173.531,143.833,173.531,152.167,195.858,160.5C218.185,168.833,262.839,177.167,285.165,181.333L307.492,185.5"></path><path marker-end="url(#flowchart-pointEnd)" style="fill:none;" class="edge-thickness-normal edge-pattern-solid flowchart-link LS-AudioManager LE-C" id="L-AudioManager-C-0" d="M464.82,33.5L453.558,39.208C442.295,44.917,419.771,56.333,408.508,67.75C397.246,79.167,397.246,90.583,397.246,96.292L397.246,102"></path><path marker-end="url(#flowchart-pointEnd)" style="fill:none;" class="edge-thickness-normal edge-pattern-solid flowchart-link LS-AudioManager LE-D" id="L-AudioManager-D-0" d="M530.914,33.5L542.177,39.208C553.439,44.917,575.964,56.333,587.226,67.75C598.488,79.167,598.488,90.583,598.488,96.292L598.488,102"></path><path marker-end="url(#flowchart-pointEnd)" style="fill:none;" class="edge-thickness-normal edge-pattern-solid flowchart-link LS-C LE-E" id="L-C-E-0" d="M397.246,135.5L397.246,139.667C397.246,143.833,397.246,152.167,397.246,160.5C397.246,168.833,397.246,177.167,397.246,181.333L397.246,185.5"></path><path marker-end="url(#flowchart-pointEnd)" style="fill:none;" class="edge-thickness-normal edge-pattern-solid flowchart-link LS-D LE-E" id="L-D-E-0" d="M598.488,135.5L598.488,139.667C598.488,143.833,598.488,152.167,578.404,160.5C558.32,168.833,518.152,177.167,498.068,181.333L477.984,185.5"></path><path marker-end="url(#flowchart-pointEnd)" style="fill:none;" class="edge-thickness-normal edge-pattern-solid flowchart-link LS-E LE-F" id="L-E-F-0" d="M397.246,219L397.246,223.167C397.246,227.333,397.246,235.667,397.246,244C397.246,252.333,397.246,260.667,397.246,264.833L397.246,269"></path></g><g class="edgeLabels"><g transform="translate(173.53125, 67.75)" class="edgeLabel"><g transform="translate(-173.53125, -9.25)" class="label"><foreignObject height="18.5" width="347.0625"><p><span class="edgeLabel">MSG\_CONFIGURE\_SAFE\_MEDIA\_VOLUME\_FORCED</span></p></foreignObject></g></g><g class="edgeLabel"><g transform="translate(0, 0)" class="label"><foreignObject height="0" width="0"></foreignObject></g></g><g class="edgeLabel"><g transform="translate(0, 0)" class="label"><foreignObject height="0" width="0"></foreignObject></g></g><g class="edgeLabel"><g transform="translate(0, 0)" class="label"><foreignObject height="0" width="0"></foreignObject></g></g><g class="edgeLabel"><g transform="translate(0, 0)" class="label"><foreignObject height="0" width="0"></foreignObject></g></g><g class="edgeLabel"><g transform="translate(0, 0)" class="label"><foreignObject height="0" width="0"></foreignObject></g></g><g class="edgeLabel"><g transform="translate(0, 0)" class="label"><foreignObject height="0" width="0"></foreignObject></g></g></g><g class="nodes"><g transform="translate(173.53125, 16.75)" id="flowchart-A-28" class="node default default"><rect height="33.5" width="125.34375" y="-16.75" x="-62.671875" ry="5" rx="5" style="" class="basic label-container"></rect><g transform="translate(-55.171875, -9.25)" style="" class="label"><foreignObject height="18.5" width="110.34375"><p><span class="nodeLabel">onSystemReady</span></p></foreignObject></g></g><g transform="translate(173.53125, 118.75)" id="flowchart-B-29" class="node default default"><rect height="33.5" width="184.5859375" y="-16.75" x="-92.29296875" ry="0" rx="0" style="" class="basic label-container"></rect><g transform="translate(-84.79296875, -9.25)" style="" class="label"><foreignObject height="18.5" width="169.5859375"><p><span class="nodeLabel">onConfigureSafeVolume</span></p></foreignObject></g></g><g transform="translate(397.24609375, 202.25)" id="flowchart-E-31" class="node default default"><rect height="33.5" width="181.5078125" y="-16.75" x="-90.75390625" ry="0" rx="0" style="" class="basic label-container"></rect><g transform="translate(-83.25390625, -9.25)" style="" class="label"><foreignObject height="18.5" width="166.5078125"><p><span class="nodeLabel">checkSafeMediaVolume</span></p></foreignObject></g></g><g transform="translate(497.8671875, 16.75)" id="flowchart-AudioManager-32" class="node default default"><rect height="33.5" width="115.125" y="-16.75" x="-57.5625" ry="0" rx="0" style="" class="basic label-container"></rect><g transform="translate(-50.0625, -9.25)" style="" class="label"><foreignObject height="18.5" width="100.125"><p><span class="nodeLabel">AudioManager</span></p></foreignObject></g></g><g transform="translate(397.24609375, 118.75)" id="flowchart-C-33" class="node default default"><rect height="33.5" width="162.84375" y="-16.75" x="-81.421875" ry="0" rx="0" style="" class="basic label-container"></rect><g transform="translate(-73.921875, -9.25)" style="" class="label"><foreignObject height="18.5" width="147.84375"><p><span class="nodeLabel">adjustStreamVolume</span></p></foreignObject></g></g><g transform="translate(598.48828125, 118.75)" id="flowchart-D-35" class="node default default"><rect height="33.5" width="139.640625" y="-16.75" x="-69.8203125" ry="0" rx="0" style="" class="basic label-container"></rect><g transform="translate(-62.3203125, -9.25)" style="" class="label"><foreignObject height="18.5" width="124.640625"><p><span class="nodeLabel">setStreamVolume</span></p></foreignObject></g></g><g transform="translate(397.24609375, 285.75)" id="flowchart-F-41" class="node default default"><rect height="33.5" width="163.6015625" y="-16.75" x="-81.80078125" ry="5" rx="5" style="" class="basic label-container"></rect><g transform="translate(-74.30078125, -9.25)" style="" class="label"><foreignObject height="18.5" width="148.6015625"><p><span class="nodeLabel">showSafetyWarningH</span></p></foreignObject></g></g></g></g></g></svg>

#### onSystemReady 初始化

系统启动过程略去不表，在系统启动完成后会调用onSystemReady；在onSystemReady中，service会发送一个MSG\_CONFIGURE\_SAFE\_MEDIA\_VOLUME\_FORCED的msg，强制配置安全音量。

```csharp
public void onSystemReady() {
    ...
    sendMsg(mAudioHandler,
    MSG_CONFIGURE_SAFE_MEDIA_VOLUME_FORCED,
    SENDMSG_REPLACE,
    0,
    0,
    TAG,
    SystemProperties.getBoolean("audio.safemedia.bypass", false) ?
        0 : SAFE_VOLUME_CONFIGURE_TIMEOUT_MS);
    ...
}
```

发送的MSG\_CONFIGURE\_SAFE\_MEDIA\_VOLUME\_FORCED会调用onConfigureSafeVolume()来进行安全音量的配置

#### onConfigureSafeVolume() 安全音量配置

```scss
    private void onConfigureSafeVolume(boolean force, String caller) {
        synchronized (mSafeMediaVolumeStateLock) {
            //Mobile contry code，国家代码，主要用来区分不同国家，部分国家策略可能会不一致
            int mcc = mContext.getResources().getConfiguration().mcc;
            if ((mMcc != mcc) || ((mMcc == 0) && force)) {
                //从config_safe_media_volume_index中获取回来的安全音量触发阈值
                mSafeMediaVolumeIndex = mContext.getResources().getInteger(
                        com.android.internal.R.integer.config_safe_media_volume_index) * 10;

                mSafeUsbMediaVolumeIndex = getSafeUsbMediaVolumeIndex();

                //根据audio.safemedia.force属性值或者value配置的值来决定是否使能安全音量
                boolean safeMediaVolumeEnabled =
                        SystemProperties.getBoolean("audio.safemedia.force", false)
                        || mContext.getResources().getBoolean(
                                com.android.internal.R.bool.config_safe_media_volume_enabled);

                //确认是否需要bypass掉安全音量功能
                boolean safeMediaVolumeBypass =
                        SystemProperties.getBoolean("audio.safemedia.bypass", false);

                // The persisted state is either "disabled" or "active": this is the state applied
                // next time we boot and cannot be "inactive"
                int persistedState;
                if (safeMediaVolumeEnabled && !safeMediaVolumeBypass) {
                    persistedState = SAFE_MEDIA_VOLUME_ACTIVE; //这个值只能是disable或者active，不能是inactive，主要用于下次启动。
                    // The state can already be "inactive" here if the user has forced it before
                    // the 30 seconds timeout for forced configuration. In this case we don't reset
                    // it to "active".
                    if (mSafeMediaVolumeState != SAFE_MEDIA_VOLUME_INACTIVE) {
                        if (mMusicActiveMs == 0) { //mMusicActiveMs主要用于计数，当安全音量弹框弹出时，如果按了确定，这个值便开始递增，当其达到UNSAFE_VOLUME_MUSIC_ACTIVE_MS_MAX时，则重新使能安全音量
                            mSafeMediaVolumeState = SAFE_MEDIA_VOLUME_ACTIVE;
                            enforceSafeMediaVolume(caller);
                        } else {
                            //跑到这里则表示已经弹过安全音量警示了，并且按了确定，所以把值设置为inactive
                            // We have existing playback time recorded, already confirmed.
                            mSafeMediaVolumeState = SAFE_MEDIA_VOLUME_INACTIVE;
                        }
                    }
                } else {
                    persistedState = SAFE_MEDIA_VOLUME_DISABLED;
                    mSafeMediaVolumeState = SAFE_MEDIA_VOLUME_DISABLED;
                }
                mMcc = mcc;
                //持久化当前安全音量的状态
                sendMsg(mAudioHandler,
                        MSG_PERSIST_SAFE_VOLUME_STATE,
                        SENDMSG_QUEUE,
                        persistedState,
                        0,
                        null,
                        0);
            }
        }
    }
```

由上可知，onConfigureSafeVolume()主要用于配置和使能安全音量功能，并且通过发送MSG\_PERSIST\_SAFE\_VOLUME\_STATE来持久化安全音量配置的值，这个持久化的值只能是active或者disabled。

```arduino
case MSG_PERSIST_SAFE_VOLUME_STATE:
    onPersistSafeVolumeState(msg.arg1);
    break;
....
....
private void onPersistSafeVolumeState(int state) {
    Settings.Global.putInt(mContentResolver,
            Settings.Global.AUDIO_SAFE_VOLUME_STATE,
            state);
}
```

#### 安全音量触发

从实际操作可知，安全音量触发条件是：音量增大到指定值。 从调节音量的代码出发，在调用mAudioManager.adjustStreamVolume和mAudioManager.setStreamVolume时，最终会调用到AudioService中的同名方法，在执行该方法的内部：

```arduino
protected void adjustStreamVolume(int streamType, int direction, int flags,
        String callingPackage, String caller, int uid) {
    ...
    ...
    ...
    } else if ((direction == AudioManager.ADJUST_RAISE) &&
            !checkSafeMediaVolume(streamTypeAlias, aliasIndex + step, device)) {
        Log.e(TAG, "adjustStreamVolume() safe volume index = " + oldIndex);
        mVolumeController.postDisplaySafeVolumeWarning(flags);
    ....
    ...
    
```

```perl
private void setStreamVolume(int streamType, int index, int flags, String callingPackage,
            String caller, int uid) {
    ....
    ....
        if (!checkSafeMediaVolume(streamTypeAlias, index, device)) {
                mVolumeController.postDisplaySafeVolumeWarning(flags);
                mPendingVolumeCommand = new StreamVolumeCommand(
                                                    streamType, index, flags, device);
            } else {
                onSetStreamVolume(streamType, index, flags, device, caller);
                index = mStreamStates[streamType].getIndex(device);
            }
    ....
    ....
```

由以上代码可以看出，其安全音量弹框警告的触发地方就在checkSafeMediaVolume方法附近处，并且都是通过mVolumeController这个远程服务去调用UI显示安全音量弹框警告，但两种调节音量的方法，触发效果略有不同：

* adjustStreamVolume：当音量步进方向是上升并且checkSafeMediaVolume返回false时，直接弹出警告框；由于警告框占据了焦点，此时无法进行UI操作，并且再按音量+键时，会继续触发这个弹框，导致无法实质性地调整音量；
* setStreamVolume：当传入的音量形参大于安全音量阈值，会触发checkSafeMediaVolume返回false，弹出安全音量警告框；并且会通过mPendingVolumeCommand保存设置的音量值，待关掉安全音量后再赋回来。

```java
private boolean checkSafeMediaVolume(int streamType, int index, int device) {
        synchronized (mSafeMediaVolumeStateLock) {
            if ((mSafeMediaVolumeState == SAFE_MEDIA_VOLUME_ACTIVE) &&
                    (mStreamVolumeAlias[streamType] == AudioSystem.STREAM_MUSIC) &&
                    ((device & mSafeMediaVolumeDevices) != 0) &&
                    (index > safeMediaVolumeIndex(device))) {
                return false;
            }
            return true;
        }
    }
```

以上是安全音量判断条件checkSafeMediaVolume，可以看出其判断主要根据以下条件：

* mSafeMediaVolumeState是否为active，这个是安全音量功能的开关变量；
* 音频流是否为STREAM\_MUSIC，只针对该音频流做安全音量；
* 设备类型，默认mSafeMediaVolumeDevices值如下：

```arduino
    /*package*/ final int mSafeMediaVolumeDevices = AudioSystem.DEVICE_OUT_WIRED_HEADSET
            | AudioSystem.DEVICE_OUT_WIRED_HEADPHONE
            | AudioSystem.DEVICE_OUT_USB_HEADSET;
```

由上可知，只针对耳机播放或者USB耳机才做安全音量功能，如有需要系统工程师可自行配置其他设备；

* 音量大小，只有音量index超过safeMediaVolumeIndex获取的值，才需要弹出安全音量警示框，而safeMediaVolumeIndex的值则是本文开头在config.xml中配置的config\_safe\_media\_volume\_index所得出的；

#### UI部分

上面有提到，当满足安全音量警示框的触发条件时，会通过mVolumeController这个远程服务去调用UI显示安全音量弹框警告，其调用链条有点长，中途略过不表，其最终会走到VolumeDialogImpl.java的showSafetyWarningH，如下：

```java
public class VolumeDialog {
    ...
    private void showSafetyWarningH(int flags) {
        if ((flags & (AudioManager.FLAG_SHOW_UI | AudioManager.FLAG_SHOW_UI_WARNINGS)) != 0
                || mShowing) {
            synchronized (mSafetyWarningLock) {
                if (mSafetyWarning != null) {
                    return;
                }
                mSafetyWarning = new SafetyWarningDialog(mContext, mController.getAudioManager()) {
                    @Override
                    protected void cleanUp() {
                        synchronized (mSafetyWarningLock) {
                            mSafetyWarning = null;
                        }
                        recheckH(null);
                    }
                };
                mSafetyWarning.show();
            }
            recheckH(null);
        }
        rescheduleTimeoutH();
    }
    ...
}
```

UI配置部分主要在SafetyWarningDialog.java，代码就不贴了，可自行查看，其本质是一个对话框，在弹出时会抢占UI焦点，如果不点击确定或取消，则无法操作其他UI；点击确定后，会调用mAudioManager.disableSafeMediaVolume()来暂时关闭安全音量警告功能，但上面有提到，当点击确定之后其实是启动了一个变量mMusicActiveMs的计数，当这个计数到达一定值（默认是20个小时），安全音量会重新启动；但如果点击了取消，再继续调大音量时，安全音量弹框还是会继续弹出；

#### disableSafeMediaVolume()

上面有提到，在安全音量弹框弹出后，点击确定可以暂时关闭安全音量警告功能，其实最终会调用到AudioService中的disableSafeMediaVolume()，代码如下：

```scss
public void disableSafeMediaVolume(String callingPackage) {
        enforceVolumeController("disable the safe media volume");
        synchronized (mSafeMediaVolumeStateLock) {
            setSafeMediaVolumeEnabled(false, callingPackage);
            if (mPendingVolumeCommand != null) {
                onSetStreamVolume(mPendingVolumeCommand.mStreamType,
                                  mPendingVolumeCommand.mIndex,
                                  mPendingVolumeCommand.mFlags,
                                  mPendingVolumeCommand.mDevice,
                                  callingPackage);
                mPendingVolumeCommand = null;
            }
        }
    }
```

一方面是调用setSafeMediaVolumeEnabled来暂时关闭安全音量功能，另一方面会把此前临时挂起的设置音量mPendingVolumeCommand重新设置回去。

## 小结

简单来讲，Android原生的安全音量功能默认强制打开，在插入耳机后，音量调节到指定阈值时，会触发音量警告弹框，该弹框会抢走焦点，不点击确定或取消无法进行其他操作；在点击确定后，默认操作者本人允许设备音量继续往上调，但此时系统会开始一个默认为20分钟的倒计时，在这20分钟内音量随意调节都不会触发安全音量弹框，但20分钟结束后，音量大于阈值时会继续触发安全音量弹框，提醒使用者注意。


---

> 作者: [wentao](https://github.com/huang8604)  
> URL: https://huang8604.github.io/posts/android-%E5%8E%9F%E7%94%9F%E5%AE%89%E5%85%A8%E9%9F%B3%E9%87%8F%E9%80%BB%E8%BE%91%E8%AE%BE%E8%AE%A1/  

