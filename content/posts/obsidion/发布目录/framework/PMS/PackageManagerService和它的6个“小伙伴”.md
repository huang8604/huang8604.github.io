---
title: PackageManagerService和它的6个“小伙伴”
author: 牛晓伟
created: 2024-10-11
tags:
  - clippings
  - 转载
  - blog
collections: PMS
source: https://mp.weixin.qq.com/s/6o5wUN6H6x7ZHVuLoCHMPA
date: 2024-10-11T09:13:38.141Z
lastmod: 2024-10-11T09:40:11.614Z
---
**前言**

作为本系列文章的首篇文章，在开始之前我一直在思考，首篇文章应该写啥内容？才能让读者很容易明白**PackageManagerService**是啥呢，如何为后面文章起到承上启下的作用呢。

后来我决定以介绍**PackageManagerService**服务中的各种繁多复杂的数据类为开篇，理由是数据类是基础，故数据类先行。遂开始一顿猛的输出，当即将接近尾声的时候，我发现不对啊，假如我是一个对**PackageManagerService**完全没接触的人，刚一上来就看到这么多非常陌生的数据类，那完全就是一种懵逼的感觉啊。

于是我决定重新规划，以介绍**PackageManagerService**及它包含的模块为首篇，这样即使对**PackageManagerService**陌生的人，也能先对它有一个初步的认识。同时后面的文章中会逐步深入的介绍**PackageManagerService**的内容。

**本文摘要**

这里的**包管理**指的是**PackageManagerService**这个服务，本篇是**包管理**系列文章的第一篇，既然整个系列文章都在介绍**PackageManagerService**，那本篇就带大家先认识一下**PackageManagerService**，通过本文您将了解**PackageManagerService**是啥，它被划分为哪些主要模块，这些模块之间又有啥关系。(文中代码基于Android13)

*1*

我是一个服务

大家好，我的名字叫**PackageManagerService**，如果觉得这名字太长大家可以叫我的小名**PMS**，我运行于**systemserver**进程，**systemserver**进程中有很多很多的**服务**，比如大家熟知的ActivityManagerService、WindowManagerService。而我也是一个**服务**，一个非常非常重要的**服务**。

上面多次提到一个词**服务**，那它到底意味着啥呢？那我就来给大家解释下，在Android中进程之间的通信用的最多的是鼎鼎大名的**binder**，而binder是client/server模式，也就是binder分为client端和server端，server端可以提供各种服务供多个client端使用。因此**服务**这个词就是指binder的server端，也就是说我PMS提供了**包管理**相关的功能，其他进程如果要想使用的话可以通过binder通信来“呼我”。

既然提到了**包管理**，其中**包**指的就是一个**apk**，那**包管理**也可以理解为**对apk的管理**，那PMS对apk主要进行以下几项管理：

1. apk的安装/更新/卸载：这三件事情我全权交给**PackageInstallerService**，关于apk安装可以查看apk安装这篇文章。
2. apk信息查询，啥是apk信息查询呢？比如想要知道当前Android设备都安装了哪些apk，可以来我这取；再比如想要知道某个apk都包含了哪些四大组件，也可以来我这取。
3. apk权限管理，基本每个被安装的apk都是需要申请一些权限的，那这些权限是被用户授权了呢，还是被用户拒绝了呢，都可以来找我。我把这个事情交给**PermissionManagerService**来管理。

可别小看了我，我可不是只负责上面的这些事情，只不过上面这些事情经常用到，故只罗列了它们而已。。我就是一个运行于**systemserver**进程**对apk进行管理**的非常重要的**服务**。

介绍完我自己，我觉得非常有必要介绍一下我管理的对象**apk**，你们人类有句话是这样说的：对自己的管理对象没有深入了解的领导不是一个好领导。并且对apk有一个深入了解后，再来理解我就更容易了。(对apk熟悉的话该部分可以跳过)

*1*

我管理的对象

我管理的对象**apk**，是 Android Package 的缩写，它其实是一个zip格式的压缩文件，只不过为了能让大家从文件名上一眼认出来，故文件后缀是 **.apk**。

贴心的我为了让大家更容易了解apk，故画了一幅图，下图展示了apk内包含的主要文件和目录。\
![image.png](https://picgo.myjojo.fun:666/i/2024/10/11/6708edfacac5a.png)

图解

**lib目录** 该目录下面包含了使用到的so库。

**res、assets目录** 这两个目录下面包含了各种资源比如图片、layout文件等。

**META-INF** 该目录下面包含了和签名证书有关的文件。

**dex** java/kotlin编译后的字节码文件。

**resources.arsc** 文件可以视为一个资源表，它存储了所有资源的ID和这些资源在APK文件或其他位置中的引用。

**AndroidManifest.xml**需要着重介绍下，因为它对于我**PMS**来说非常重要。

## **2.1 AndroidManifest.xml**

可以在AndroidManifest.xml文件中使用*activity*、*provider*、*service*、*receiver*标签来声明四大组件，也可以使用*uses-permission*标签来申请使用哪些权限，当然还有其他的标签。

那AndroidManifest.xml和使用标签声明或者申请的这些信息到底是给谁用的呢？

答案是**PMS**，AndroidManifest.xml就像饭馆的菜单，从菜单上可以看出这个饭馆到底有哪些饭、菜，菜单是给顾客使用的。而AndroidManifest.xml是给**PMS**使用的，**PMS**只有通过**AndroidManifest.xml才能知道一个apk内到底声明了哪些四大组件、申请了哪些权限、使用了哪些共享库等等**，因此对于**PMS**可以通过**解析AndroidManifest.xml内的各种标签**来获取apk内声明的各种信息。

下图展示了AndroidManifest.xml中常用的标签。\
![image.png](https://picgo.myjojo.fun:666/i/2024/10/11/6708ee45c91e9.png)

## **2.2 小结**

我把我的管理对象**apk**介绍给了大家，其实我主要目的是想把apk中的**AndroidManifest.xml**着重介绍给大家，因为它对于我**PMS**来说非常重要，我只有通过它**才能知道一个apk内到底声明了哪些四大组件、申请了哪些权限、使用了哪些共享库**等等。

那接下来大家把“视线”再次聚焦到我身上，继续把我自己介绍给大家。

*3*

我的“小伙伴”

你们可不要把我想象的很厉害，很多工作都是由我和我的“小伙伴们”共同完成的，单凭我一个“人”可不行。为了让大家能更清楚的认识我的“小伙伴”，我用一张图把它们展示给大家。\
![f8954c9c5e52200a9f115a716afe453b\_MD5](https://picgo.myjojo.fun:666/i/2024/10/11/6708ee9ad0467.png)

我的主要“小伙伴”有**apk管理模块**、**权限管理模块**、**共享库模块**、**记录存储模块**、**所有apk信息模块**、**四大组件模块**。那我就把舞台交给它们，让它们亲自把自己介绍给大家吧。

## **3.1 权限管理模块**

大家好，我是**权限管理模块**，为了让大家更了解我的工作，我先来介绍下**权限**，权限分为**声明权限**和**请求权限**。

声明权限

还记得在上面介绍apk的AndroidManifest.xml的时候，声明权限需要在AndroidManifest.xml文件中使用*permission*标签，如下例子：

```xml
<permission android:description="string resource"  
            android:icon="drawable resource"  
            android:label="string resource"  
            android:name="string"  
            android:permissionGroup="string"  
            android:protectionLevel=["normal" | "dangerous" |  
                                     "signature" | ...] />
```

每个apk都可以声明自己的权限，那当别的apk访问自己的一些关键信息时候就可以要求它具有某个声明的权限后才可以访问。

请求权限

每个安装在Android设备上的apk或多或少的都会用到一些权限，请求权限就是在AndroidManifest中通过*uses-permission*标签来使用权限，如下代码：

```
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>
```

而我**权限管理模块**所做的事情如下：

1. 把所有的apk**声明的权限**收集并集中管理起来
2. 而**每个apk请求的权限和权限对应的状态我也会保存起来**，权限对应的状态是指比如某个apk的权限*是否被允许*、*是否拒绝*、*是否只是允许一次*
3. apk在请求某个权限时，当用户不管是点击了*允许*、*拒绝*等，都需要经过我

以上就是我所做的事情，而我把这些事情全权交给了**PermissionManagerService**服务来处理，关于我会在后面的文章再次与大家见面。

## **3.2 共享库模块**

大家好，我是**共享库模块**，同样我也先来介绍下**共享库**，像*framework.jar*这个库包含了很多的类比如*Activity*、*Context*、*Service*等，而*framework.jar*在**zygote**进程中已经被预加载了，因此每个apk直接使用即可。而像一些库比如*com.google.android.maps*，它是没有被包含在*framework.jar*中的，而*com.google.android.maps*以共享库的方式提供给使用者来使用。而共享库也分为**声明共享库**和**使用共享库**。

声明共享库

首先只有系统apk才可以声明共享库，为啥有这样的规定？主要原因是既然是共享库就需要保证它的稳定性，如果是普通apk可以声明共享库，那设备上没有装该apk，那共享库就不存在。声明共享库非常简单在AndroidManifest.xml文件中使用*library*标签，如下例子：

```
<library android:name="android.ext.shared" />
```

#### **使用共享库**

使用共享库在AndroidManifest.xml文件中使用*uses-library* (使用java库)或者*uses-native-library* (使用native库) 标签，如下例子：

```
<uses-library android:name="com.google.android.maps"            android:required="false" /><uses-library android:name="com.here.android"            android:required="false" />
```

而我所做的事情如下：

1. 把所有**声明的共享库**收集起来，若共享库之间存在依赖，则把它们的依赖也初始化。
2. 若apk中使用了共享库，则会把共享库的信息 (如共享库文件路径) 交给该apk。

而我同样把这些事情交给了**SharedLibrariesImpl**类来处理，在后面的文章中我还会与大家见面。

## **3.3 记录存储模块**

大家好，我是**记录存储模块**，我的主要作用是记录信息并且把这些信息存储到文件中。

**PMS**：“兄弟，你这样介绍自己，我想大家肯定不知道你说的是啥，最好介绍的时候带一些例子。”

**记录存储模块**：“你说到即是，那我就重新介绍下我自己。”

首先我的名字主要是由**记录**和**存储**两个词组成的。**记录**主要是记录**apk的安装信息**、**shared userId**、**apk声明的权限**等。

**PMS**：“那我替大家问个问题，像你说的这些信息不记录行不行。”

**记录存储模块**：“答案是不记录肯定不行，为啥呢？如**apk的安装信息**指的是apk安装后是需要把它的关键信息用*PackageSetting*类记录下来的，就像你们人类入住酒店需要把身份证、房号等关键信息记录下来，而*PackageSetting*会把apk的包名、apk文件路径、apk的版本号及签名这些信息记录下来。记录下来后那其他使用者想要知道哪个apk是否安装了，就可以快速的从我这检索到。就像警察去某酒店查询某个人是否入住了酒店。”

那**存储**就是把记录的信息存储到文件中，存储到文件后在下次Android设备重新启动的时候就可以把存储的这些信息读取出来了。而我把我所做的这些事情交给了**Settings**类，在后面的文章中我会把我自己完完全全的介绍给大家，期待与大家见面。

## **3.4 所有apk信息模块**

大家好，我是**所有apk信息模块**，听着这个名字是不是很奇怪，主要的原因是我没有想到一个很好的名字，而我所做的事情是会把**所有已安装apk的信息收集起来**，还记得【2.我管理的对象】介绍apk中的**AndroidManifest**文件吗？这里的**apk的信息**指的就是**解析AndroidManifest文件后的数据**。通过这些信息可以知道apk的版本号、包名、application对应的class、声明了哪些四大组件、使用了哪些权限等等。

我的使用场景比较多，比如使用者可以从我这查询到*Application*的信息等，而我是存储在**PMS**的*mPackages*属性中，如下代码：

```
//key是包名，而AndroidPackage存储了解析出的AndroidManifest的信息final WatchedArrayMap<String, AndroidPackage> mPackages = new WatchedArrayMap<>();
```

## **3.5 四大组件模块**

大家好，我是**四大组件模块**，聪明的大家看了我的名字肯定能立马知道我是做啥的，我存储了**所有已安装apk的AndroidManifest中声明的四大组件**。

**PMS**对**四大组件模块**说到：“我有个问题，**所有apk信息模块**不是已经把所有的信息都包含进去了，那为啥还需要你呢？你们不是包含了重复的信息吗？”

“我存在的目的是为了快速的检索四大组件，假如从**所有apk信息模块**检索某个*Activity*信息的话，需要通过*包名*从*mPackages*中获取对应的*AndroidPackage*对象，再从*AndroidPackage*对象中检索*Activity*，这个流程是不是比较长，而从**四大组件模块**检索速度非常快。这也就是我存在的目的。”

我常用于以下场景：

1. *ActivityTaskManagerService*启动某个*Activity*时，需要从我这获取对应的*Activity*信息，获取到则返回；否则启动*Activity*失效
2. *ActivityManagerService*启动某个*Service*时，也需要从我这获取对应的*Service*信息。同理启动某个*BroadcastReceiver*、*ContentProvider*也需要从我哦这获取对应的信息

而我是存储在**PMS**的*mComponentResolver*属性中，如下代码：

```
final ComponentResolver mComponentResolver;
```

## **3.6 apk管理模块**

大家好，我是**apk管理模块**，从上图可以看出我在所有模块中的地位是多么重要，我把自己比作**根基模块**，毫不夸张的说没有我其他模块就没有存在的意义，我包含的功能有**扫描所有apk**、**apk安装/更新/卸载**、**解析apk**。那就来说下我为何有如此大的底气把自己说的如此重要。

### **3.6.1 apk安装**

安装一个apk后，如果apk中**声明了权限**或者**使用了权限**，则需要通知**权限管理模块**做相应的处理；如果apk是系统apk，使用了*library*标签**声明了共享库**，则需要通知**共享库模块**把声明的共享库收集起来；**记录存储模块**需要把该apk的关键信息记录并且存储下来；**所有apk信息模块**需要把该apk中AndroidManifest解析出来的信息存储下来；**四大组件模块**需要把该apk中声明的四大组件存储下来。

以上就是安装一个apk后，对其他模块产生的影响。

### **3.6.2 apk的删除**

删除一个apk后，如果apk中**声明了权限**或者**使用了权限**，则需要通知**权限管理模块**把相应的权限删除；如果apk是系统apk，使用了*library*标签**声明了共享库**，则需要通知**共享库模块**把声明的共享库删除；**记录存储模块**需要把该apk的关键信息删除；**所有apk信息模块**需要把该apk的信息删除；**四大组件模块**需要把该apk中声明的四大组件删除掉。

以上就是删除一个apk后，对其他模块产生的影响。

### **3.6.3 扫描所有apk**

当Android设备启动的时候，会启动一个任务就是把所有的**系统apk**和**非系统apk**都扫描一下，扫描apk其中最重要的事情是解析apk的AndroidManifest文件，扫描apk所做的事情简单总结下就是：如果apk存在升级的情况则把apk升级，如果存在apk删除的情况则把apk删除，若存在一个新的系统apk则安装apk。因此扫描所有apk也会对其他模块产生影响。

以上的原因是不是完全可以说明我为啥是**根基模块**了，不管是*apk的安装*还是*apk的卸载*都会引起其他模块的“震动”，我把安装/卸载/更新apk的功能交给了**PackageInstallerService**这个服务，对apk安装感兴趣可以查看apk安装之谜这篇文章。

## **3.7 小结**

我的“小伙伴”们都把它们自己介绍完毕了，那我简单总结下：

1. **权限管理模块**负责apk权限相关的事情，比如请求某个权限，apk权限状态存储，收集所有apk声明的权限
2. **共享库模块**负责apk使用到的所有共享库
3. **记录存储模块**会把apk相关的很多信息记录并且存储到文件中，比如apk安装后关于apk安装的信息会存储下来，这样就可以供其他使用者检索
4. **所有apk信息模块**会收集所有已安装apk的AndroidManifest解析出来的信息，以供其他使用者检索
5. **四大组件模块**为了加快检索四大组件的速度，会把所有已安装apk的四大组件信息收集起来
6. **apk管理模块**主要负责apk的安装/卸载/更新，它是**根基模块**，因为它的某个功能会对其他模块产生影响。

*4*

我的启动

介绍完我的“小伙伴”，那来介绍下我的启动过程，看下我的“小伙伴”在启动过程中做了什么？

## **4.1 共享库模块初始化**

**systemserver**进程有很多很多的**服务**，而在**systemserver**进程启动过程中把这些服务分为了**bootstrap services**、**core services**、**other services**，而我非常的“荣幸”作为bootstrap services最先被启动，当我**PMS**的构造方法被调用后，**共享库模块**会先进行初始化，共享库既有java共享库也有native共享库，下面是相应的代码：

```java
//下面代码位于PackageManagerService构造方法中  
    
  //从 SystemConfig.getInstance() 中把所有的lib读取出来  
  ArrayMap<String, SystemConfig.SharedLibraryEntry> libConfig = systemConfig.getSharedLibraries();  
    
  final int builtInLibCount = libConfig.size();  
  for (int i = 0; i < builtInLibCount; i++) {  
      //依次把存储在文件中的共享库添加到 mSharedLibraries 中  
      mSharedLibraries.addBuiltInSharedLibraryLPw(libConfig.valueAt(i));   
  }  
    
  long undefinedVersion = SharedLibraryInfo.VERSION_UNDEFINED;  
  //下面代码处理共享库之间的依赖  
  for (int i = 0; i < builtInLibCount; i++) {  
    String name = libConfig.keyAt(i);  
    SystemConfig.SharedLibraryEntry entry = libConfig.valueAt(i);  
    final int dependencyCount = entry.dependencies.length;  
    for (int j = 0; j < dependencyCount; j++) {  
      final SharedLibraryInfo dependency =  
      computer.getSharedLibraryInfo(entry.dependencies[j], undefinedVersion);  
      if (dependency != null) {  
         computer.getSharedLibraryInfo(name, undefinedVersion).addDependency(dependency);  
      }  
    }  
  }
```

## **4.2 记录存储模块初始化**

**记录存储模块**会从先前存储好的文件中把各种信息读取出来，并交给对应的记录类，比如存储的所有apk的安装信息会从 package.xml 文件中读取出来，交给*PackageSetting*类；在比如*声明的权限*信息也从package.xml 文件中读取出来，交给*LegacyPermissionSettings*类。该模块提前初始化，主要是为后面的模块服务的，其他模块都需要知道哪些apk安装了等信息。下面是相应代码：

```
//下面代码位于PackageManagerService构造方法中  
  
//从 /data/system/packages.xml及其他文件中把保存的数据读出来  
mFirstBoot = !mSettings.readLPw(computer,    
                    mInjector.getUserManagerInternal().getUsers(  
                    /* excludePartial= */ true,  
                    /* excludeDying= */ false,  
                    /* excludePreCreated= */ false));
```

## **4.3 权限管理模块初始化**

因为**记录存储模块**会把存储的权限信息读取出来，**权限管理模块**就可以使用这些信息来进行初始化了，下面是相应代码：

```
//下面代码位于PackageManagerService构造方法中  
  
//把所有apk声明的权限交给mPermissionManager  
mPermissionManager.readLegacyPermissionsTEMP(mSettings.mPermissions);   
//下面方法会使用每个apk的权限状态初始化自己  
mPermissionManager.readLegacyPermissionStateTEMP();
```

这样**权限管理模块**就可以接收权限管理方面的任务了。

## **4.4 扫描apk**

前面几个模块的初始化都是在为该步骤做准备，而扫描apk主要是把**系统apk**和**非系统apk**都扫描，扫描apk其实可以理解为是**一个简单的apk安装或更新过程**。扫描apk其中最重要的一步是需要把apk中AndroidManifest文件解析出来，在扫描过程是需要使用到**记录存储模块**来判断某个apk是否已经安装，同时还会用到**共享库模块**来判断是否有对应的共享库更新，同时也会用到**权限管理模块**来判断比如是否有新声明的权限加入。下面是扫描apk的对应代码：

```
//下面代码位于PackageManagerService构造方法中  
  
final int[] userIds = mUserManager.getUserIds();  
PackageParser2 packageParser = mInjector.getScanningCachingPackageParser();  
//扫描系统apk  
mOverlayConfig = mInitAppsHelper.initSystemApps(packageParser, packageSettings, userIds,startTime);   
//扫描非系统apk  
mInitAppsHelper.initNonSystemApps(packageParser, userIds, startTime);   
packageParser.close();
```

扫描apk后，**所有apk信息模块**和**四大组件模块**把所有已安装apk的信息收集起来，当然其他的模块也会进行相应的更新。

## **4.5 小结**

以上就是我**PMS**启动过程中，模块之间初始化和互相影响的一个过程，当然上面只是列了关键模块，我的启动过程可不是只有这么简单，还有其他过程就不赘述了。

*5*

总结

本文主要带大家认识了**PackageManagerService**，它是一个运行于**systemserver**进程的**管理apk的服务**，还带大家认识了它的“小伙伴”**权限管理模块**、**共享库模块**、**所有apk信息模块**、**四大组件模块**、**记录存储模块**、**apk管理模块**，在后面的文章这些模块还会与大家见面。

这是**包管理**的第一篇，敬请期待我的第二篇，谢谢！

***

最后推荐一下我做的网站，玩Android: *wanandroid.com* ，包含详尽的知识体系、好用的工具，还有本公众号文章合集，欢迎体验和收藏！
