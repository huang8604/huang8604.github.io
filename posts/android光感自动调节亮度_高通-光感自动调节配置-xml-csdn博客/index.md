# Android：光感自动调节亮度_高通 光感自动调节配置 Xml-CSDN博客

展锐光感自动调节亮度功能，光感数据上报对应的背光亮度设置，其实其他平台像 [MTK](https://so.csdn.net/so/search?q=MTK\&spm=1001.2101.3001.7020) ，高通也差不多，大同小异，主要是搜索关键字：config\_autoBrightnessLevels、config\_screenBrightnessBacklight，就会发现只是文件路径不太一样。

通过修改sensorhub下光感原始数据上报，或者根据实际情况去调节brightness level都能实现自动调节亮度功能的 [灵敏度](https://so.csdn.net/so/search?q=%E7%81%B5%E6%95%8F%E5%BA%A6\&spm=1001.2101.3001.7020) ，看个人喜欢。

展锐的代码路径如下：

device\sprd\roc1\common\overlay\frameworks\base\core\res\res\values\config.xml

```xml
<!-- Array of light sensor LUX values to define our levels for auto backlight brightness support.

         The N entries of this array define N  1 zones as follows:

         Zone 0:        0 <= LUX < array[0]

         Zone 1:        array[0] <= LUX < array[1]

         ...

         Zone N:        array[N - 1] <= LUX < array[N]

         Zone N + 1     array[N] <= LUX < infinity

         Must be overridden in platform specific overlays -->

    <integer-array name="config_autoBrightnessLevels">

        <item>16</item>

        <item>32</item>

        <item>50</item>

        <item>100</item>

        <item>140</item>

        <item>180</item>

        <item>240</item>

        <item>300</item>

        <item>600</item>

        <item>800</item>

        <item>1000</item>

        <item>2000</item>

        <item>3000</item>

        <item>4000</item>

        <item>5000</item>

        <item>6000</item>

        <item>8000</item>

        <item>10000</item>

        <item>20000</item>

        <item>30000</item>

    </integer-array>

    <!-- Array of desired screen brightness in nits corresponding to the lux values

         in the config_autoBrightnessLevels array. As with config_screenBrightnessMinimumNits and

         config_screenBrightnessMaximumNits, the display brightness is defined as the measured

         brightness of an all-white image.

         If this is defined then:

            - config_autoBrightnessLcdBacklightValues should not be defined

            - config_screenBrightnessNits must be defined

            - config_screenBrightnessBacklight must be defined

         This array should have size one greater than the size of the config_autoBrightnessLevels

         array. The brightness values must be non-negative and non-decreasing. This must be

         overridden in platform specific overlays -->

    <array name="config_autoBrightnessDisplayValuesNits">

        <item>10.45935</item>   <!-- 0-16 -->

        <item>29.25559</item>   <!-- 16-32 -->

        <item>34.240692</item>  <!-- 32-50 -->

        <item>37.514347</item>  <!-- 50-100 -->

        <item>40.018696</item>  <!-- 100-140 -->

        <item>46.885098</item>  <!-- 140-180 -->

        <item>51.626434</item>  <!-- 180-240 -->

        <item>58.610405</item>  <!-- 240-300 -->

        <item>66.890915</item>  <!-- 300-600 -->

        <item>77.61644</item>   <!-- 600-800 -->

        <item>90.221886</item>  <!-- 800-1000 -->

        <item>105.80314</item>  <!-- 1000-2000 -->

        <item>126.073845</item> <!-- 2000-3000 -->

        <item>154.16931</item>  <!-- 3000-4000 -->

        <item>191.83717</item>  <!-- 4000-5000 -->

        <item>240.74442</item>  <!-- 5000-6000 -->

        <item>294.84857</item>  <!-- 6000-8000 -->

        <item>348.05453</item>  <!-- 8000-10000 -->

        <item>394.98703</item>  <!-- 10000-20000 -->

        <item>405.2315</item>  <!-- 20000-30000 -->

        <item>410.3658</item>  <!-- 30000+ -->

    </array>

 

    <!-- An array describing the screen's backlight values corresponding to the brightness

         values in the config_screenBrightnessNits array.

         This array should be equal in size to config_screenBrightnessBacklight. -->

    <integer-array name="config_screenBrightnessBacklight">

        <item>0</item>

        <item>5</item>

        <item>10</item>

        <item>15</item>

        <item>20</item>

        <item>25</item>

        <item>30</item>

        <item>35</item>

        <item>40</item>

        <item>45</item>

        <item>60</item>

        <item>75</item>

        <item>90</item>

        <item>105</item>

        <item>120</item>

        <item>135</item>

        <item>150</item>

        <item>165</item>

        <item>180</item>

        <item>195</item>

        <item>210</item>

        <item>225</item>

        <item>240</item>

        <item>255</item>

    </integer-array>
```

查看光感数据上报原始值节点：

```cobol
cat /sys/devices/virtual/sprd_sensorhub/sensor_hub/raw_data_als
```

可以跟一下代码就会发现在 kernel 下：

kernel4.14\drivers\iio\sprd\_hub\shub\_core.c

```cobol
static ssize_t raw_data_als_show(struct device *dev,

                struct device_attribute *attr, char *buf)

{

    struct shub_data *sensor = dev_get_drvdata(dev);

    u8 data[2];

    u16 *ptr;

    int err;

 

    ptr = (u16 *)data;

    if (sensor->mcu_mode <= SHUB_OPDOWNLOAD) {

        dev_err(dev, "mcu_mode == SHUB_BOOT!\n");

        return -EINVAL;

    }

 

    err = shub_sipc_read(sensor,

                 SHUB_GET_LIGHT_RAWDATA_SUBTYPE, data, sizeof(data));

    if (err < 0) {

        dev_err(dev, "read RegMapR_GetLightRawData failed!\n");

        return err;

    }

    return sprintf(buf, "%d\n", ptr[0]);

}

static DEVICE_ATTR_RO(raw_data_als);
```

打赏作者

¥1 ¥2 ¥4 ¥6 ¥10 ¥20

扫码支付： ¥1

获取中

扫码支付

您的余额不足，请更换扫码支付或 [充值](https://i.csdn.net/#/wallet/balance/recharge?utm_source=RewardVip)

打赏作者

实付 元

[使用余额支付](https://blog.csdn.net/liu_xinglfz/article/details/)

点击重新获取

扫码支付

钱包余额 0

抵扣说明：

1.余额是钱包充值的虚拟货币，按照1:1的比例进行支付金额的抵扣。\
2.余额无法直接购买下载，可以购买VIP、付费专栏及课程。

[余额充值](https://i.csdn.net/#/wallet/balance/recharge)

举报

[![](https://csdnimg.cn/release/blogv2/dist/pc/img/toolbar/Group.png) 点击体验\
DeepSeekR1满血版](https://ai.csdn.net/?utm_source=cknow_pc_blogdetail\&spm=1001.2101.3001.10583) ![程序员都在用的中文IT技术交流社区](https://g.csdnimg.cn/side-toolbar/3.6/images/qr_app.png)

程序员都在用的中文IT技术交流社区

![专业的中文 IT 技术社区，与千万技术人共成长](https://g.csdnimg.cn/side-toolbar/3.6/images/qr_wechat.png)

专业的中文 IT 技术社区，与千万技术人共成长

![关注【CSDN】视频号，行业资讯、技术分享精彩不断，直播好礼送不停！](https://g.csdnimg.cn/side-toolbar/3.6/images/qr_video.png)

关注【CSDN】视频号，行业资讯、技术分享精彩不断，直播好礼送不停！

返回顶部


---

> 作者: [wentao](https://github.com/huang8604)  
> URL: https://huang8604.github.io/posts/android%E5%85%89%E6%84%9F%E8%87%AA%E5%8A%A8%E8%B0%83%E8%8A%82%E4%BA%AE%E5%BA%A6_%E9%AB%98%E9%80%9A-%E5%85%89%E6%84%9F%E8%87%AA%E5%8A%A8%E8%B0%83%E8%8A%82%E9%85%8D%E7%BD%AE-xml-csdn%E5%8D%9A%E5%AE%A2/  

