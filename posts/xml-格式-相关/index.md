# Xml 格式 相关

#### 背景：

Android12高版本以后系统生成的很多data路径下的xml都变成了二进制类型

#### 具体原因分析

代码路径：\
frameworks/base/core/java/android/util/Xml.java

使用BinaryXmlSerializer最重要原因是：\
1、可以有更快的速度\
2、更小的体积

#### 如何把二进制变成正常可读xml

方法1：\
适合debug等版本上，可以随意进行恢复出厂删除相关的xml的场景\
具体操作：\
改变属性，删除原来的xml，或者恢复出厂，让系统重新生成xml，\
属性修改如下：\
adb shell setprop persist.sys.binary\_xml false\
修改后在把相关的xml要进行删除重启触发重新生成xml

方法2：\
这种属于已有了这种二进制格式的xml，也不想清除这个xml让重新生成，因为这样可能重新生成的数据就不是原来xml的数据。\
所以得考虑把原来的二进制格式的xml 转化成可读xml文件

Android系统中有一部分xml是二进制加密保存的，当我门用adb pull导出查看的时候发现是乱码，不用慌，下面我给大家介绍安卓系统自带的xml加解密工具，位于 /system/bin下，有abx2xml和xml2abx

* Android xml二进制解压

```
usage: abx2xml [-i] input [output]
adb shell abx2xml zipsrc.xml output.xml 
//注意：可进入到文件所在目录进行操作，也可以复制文件到sdcard进行操作
//例如我们要查看sdcard下的AndroidDemoZip.xml,手动不指定输出路径，默认覆盖当前XML
1、adb shell
2、cd sdcard
3、abx2xml AndroidDemoZip.xml output.xml 
```

* Android xml压缩二进制

```
 usage: xml2abx [-i] input [output]
 adb shell xml2abx inputsrc.xml zipsrc.xml 
 //用法跟解压同理
```

源码位置：

```shell
~/Downloads/Study/Code/LA.QSSI.12.0.r1/frameworks/base/cmds/abx % ll
total 56
drwxrwxr-x@  9 huangwentao  staff    288 12 31  2021 .
drwxrwxr-x  35 huangwentao  staff   1120  9 23 17:45 ..
-rw-rw-r--   1 huangwentao  staff    637 12 31  2021 Android.bp
-rw-rw-r--   1 huangwentao  staff      0 12 31  2021 MODULE_LICENSE_APACHE2
-rw-rw-r--   1 huangwentao  staff  10694 12 31  2021 NOTICE
-rwxrwxr-x   1 huangwentao  staff    128 12 31  2021 abx
-rwxrwxr-x   1 huangwentao  staff    128 12 31  2021 abx2xml
drwxrwxr-x   3 huangwentao  staff     96 12 31  2021 src
-rwxrwxr-x   1 huangwentao  staff    128 12 31  2021 xml2abx
```

```
xml2abx ./system/users/0/appwidgets-read.xml ./system/users/0/appwidgets-binary.xml
abx2xml ./system/users/0/appwidgets.xml ./system/users/0/appwidgets-read.xml
```


---

> 作者: [wentao](https://github.com/huang8604)  
> URL: https://huang8604.github.io/posts/xml-%E6%A0%BC%E5%BC%8F-%E7%9B%B8%E5%85%B3/  

