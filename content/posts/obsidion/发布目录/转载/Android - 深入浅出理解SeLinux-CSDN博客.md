---
title: Android - 深入浅出理解SeLinux-CSDN博客
author: 
created: 2024-10-12
tags:
  - clippings
  - 转载
  - blog
collections: 图形显示
source: https://blog.csdn.net/temp7695/article/details/136945974?spm=1001.2014.3001.5502
date: 2024-11-06T06:57:56.422Z
lastmod: 2024-11-05T01:34:01.449Z
---
官方文档：

[https://source.android.com/docs/security/features/selinux](https://source.android.com/docs/security/features/selinux "https://source.android.com/docs/security/features/selinux")

[https://source.android.com/docs/security/features/selinux/images/SELinux\_Treble.pdf](https://source.android.com/docs/security/features/selinux/images/SELinux_Treble.pdf "https://source.android.com/docs/security/features/selinux/images/SELinux_Treble.pdf")

[Your visual how-to guide for SELinux policy enforcement | Opensource.com](https://opensource.com/business/13/11/selinux-policy-guide "Your visual how-to guide for SELinux policy enforcement | Opensource.com")

SeLinux（Security-Enhanced Linux）是一个标签系统（labeling system）。每个进程都有一个label（称为process label），每个文件系统所涵盖的文件/目录、网络端口、设备等对象也有一个lable（称为Object label）。SeLinux通过编写规则来控制一个process label对一个Object label的访问，这个规则称之为策略（Policy）。SeLinux的这种安全机制称为Mandatory Access Control (MAC)，是在内核层实现的。

在标准的Linux上，也就是未实施SeLinux的Linux上，传统的访问控制是通过owner/group + permission flags（比如rwx）实现的，称之为Discretionary Access Control (DAC).

SeLinux和传统的DAC不是替代的关系，而是并行的关系。两者同时在起作用。之所以出现SeLinux，是因为传统DAC的安全机制过于粗粝，而Selinux提供了更为细致和安全的访问控制。简言之，传统DAC机制下，一旦你获得了root权限，将无所不能，但在SeLinux的机制下，即使获得了root权限，也仍然需要遵循已经设置好的访问策略，只有指定的进程才可以访问指定的文件。

SeLinux有两种运行模式：

* Permissive mode：当访问并未授权时，不会阻止访问，但会打印log
* Enforcing mode：当访问未被授权时，会阻止访问，并且会打印log

log将出现在dmesg和logcat。

除了可以对整体进行模式设置，也可以针对某个进程单独设置某个模式，除此之外的进程使用全局模式。

### **设计**

SeLinux在Android 4~7和Android 8以后采用了不同的设计

Android 4~7上，SeLinux的策略是作为一个整体来编译和更新的；

Android 8及以后，SeLinux采用了模块化、动态化设计，Platform（可以理解为AOSP的部分）、Non-Platform（vendor、odm、oem的部分，这里总体称为vendor部分）的SeLinux策略分别独立编译、升级。

附一张Android设备的架构图：

![f463c287b1ece4ba4e7f06d0e102abbf\_MD5](https://picgo.myjojo.fun:666/i/2024/10/12/670a460123295.png)

编译后会生成对应的img文件

● system.img. Contains mainly Android framework.

● boot.img. (kernel/ramdisk) Contains Linux kernel + Android patches.

● vendor.img. Contains SoC-specific code and configurations.

● odm.img. Contains device-specific code and configurations.

● oem.img. Contains OEM/carrier-related configurations and customizations.

● bootloader. Brings up the kernel (vendor-proprietary).

● radio. Modem (proprietary).

Android 8以后，SeLinux的策略文件可以伴随相应的img独立编译或者OTA。

## **2. 概念**

什么是标签（Label）？怎么基于Label对访问进行控制？

先抛开Label这个概念不说。所谓SeLinux里的访问控制，就是判定一个Source有没有权限去访问Target。这里的Source一般就是进程，Target最长见的就是文件系统（比如文件、目录、socket、设备等等），当然还有其他类型的Target。换句话说，SeLinux的机制就是通过读取一个“规则”，来控制一个进程有没有权限去访问一个文件（或其他类型）。

上面说的“规则”，在SeLinux里的术语叫做Policy（策略）或叫Access Vector Rule。是可以由AOSP和厂商、供应商来编写的。

上面的Source，还叫做Subject（主体）

上面的Target，还叫做Object（对象或客体）

Label、Source、Target、Subject、Object，这些都不重要， 在实际语法中并没有相关关键词，只要各种资料里出现这些词的时候知道其所指就可以。

### **2.1 规则Policy Rule（或叫Access Vector Rule）**

策略规则的语法为：

```kotlin
allow source target:class permissions;
```

* Source - The type (or attribute) of the subject of the rule. Who is requesting the access?（是谁在请求访问一个资源）
* Target - The type (or attribute) of the object. To what is the access requested?（被访问的对象）
* Class - The kind of object (e.g. file, socket) being accessed.（被访问者的类型）
* Permissions - The operation (or set of operations) (e.g. read, write) being performed.（对Target具体要做什么操作？比如对被访问者是文件来说，是要读、写它还是其他操作？）

具体例子如下：

```cobol
allow sample_app app_data_file:file { read write };
```

这个例子是说，允许sample\_app这个进程去访问app\_data\_file（它是一个file类型，也就是文件），允许的操作是read和write。

而其实这里的sample\_app并不是一个真正的具体的进程名，而是在系统编译阶段就定义好的一个标签（Label），一些真正的进程被映射到sample\_app这个标签上，那么在执行上面规则的时候，其实生效的、有权限访问app\_data\_file的是所有映射了sample\_app标签的那些进程。同样的，app\_data\_file也不是一个具体的文件名。它也是一个提前声明了的标签，一些真正的文件被映射到这个标签上，sample\_app有权访问的是所映射的这些文件。

![bf156f6a023d402861339106a2ca1f65\_MD5](https://picgo.myjojo.fun:666/i/2024/10/12/670a460122661.png)

从这里看出来，有别于传统DAC的Owner、Group、Permissions 的控制方式，所谓的“基于标签系统”的SeLinux，就是这种通过声明标签的方式来表述访问规则的。

标签只是一种概念性的东西，具体体现在策略文件里，则是抽象成了Type、Attribute、Class、Permissions这些具体关键字。

### **2.2 规则里的关键字说明**

以下面这个规则举例

```cobol
allow sample_app app_data_file:file { read write };
```

#### **Rule Name**

上面规则示例中，allow就是Rule Name的一种。SeLinux有多种RuleName：

* allow：表示允许Source对Target执行许可的操作
* neverallow：表示不允许Source对Target执行制定的操作
* auditallow：表示允许操作并记录访问决策信息
* dontaudit：表示不记录违反规则的决策信息，切违反规则不影响运行。

其中allow是用的最多的。

#### **Type**

上面的示例中，sample\_app、app\_data\_file都是一个Type。简单理解就是将一个或几个进程声明为Type A， 将一个或多个文件声明为Type B。那么在控制这个进程有没有权限访问这个文件的时候，只用A和B来表示就可以了。

这样做有什么好处？“一类进程”总比具体的“一个进程”要灵活的多。把多个进程声明为同一个Type，那么在写规则的时候只要描述这个Type，那么这个Type对应的所有进程都会生效。文件或其他对象也是同样的。

也就是说，在规则中描述Source能不能访问Target，是通过Type来表述的。

Type是厂商或供应商可以自定义的

##### **Attribute**

将多个Type归为一组，就是一个Attribute。

通俗的说，把一些进程声明为Type，但是多个Type有某种共通的特性，就可以把这些Type声明为同一个Attribute。在描述规则的时候，可以将Source或者Target指定为一个Attribute而不是Type，这样所有属于这个Attribute的Type都生效。

AOSP本身内置了一些Attribute，而这些Attribute很多都是约定俗称的固定含义。比如：

##### **domain**

所有进程的type必须归属于domain。domain因此成了进程type的常见表述。

##### **file\_type**

所有文件type都归属于file\_type

...

AOSP内置的Attribute见

> platform/system/sepolicy/public/attributes
>
> platform/system/sepolicy/private/attributes
>
> platform/system/sepolicy/prebuilts/api/\[version]/public/attributes
>
> platform/system/sepolicy/prebuilts/api/\[version]/private/attributes

#### **Class**

上面示例中，“:file”就是一个Class。简单说就是将某些被访问对象归为一种Class，比如说被访问的是文件，Class一般就是file，如果是目录，Class就是dir，如果是socket，Class就是socket等等。Class是AOSP内定义好的，一般不需要自定义

具体有那些Class，可见源码platform/system/sepolicy/private/security\_classes

Class是用来做什么的？其实是与Permissions相关的。

#### **Permissions**

即示例中的 { read write }。表示具体可以做什么操作。不同的Class有不同的Permissions集合。罗列一些Class对应的Permission（非完整）：

<table><tbody><tr><td colspan="1" rowspan="1"><p><span>Class</span></p></td><td colspan="1" rowspan="1"><p><span>Permission</span></p></td></tr><tr><td colspan="1" rowspan="1"><p><span>file</span></p></td><td colspan="1" rowspan="1"><p><span>ioctl&nbsp;read&nbsp;write&nbsp;create&nbsp;getattr&nbsp;setattr&nbsp;lock&nbsp;relabelfrom&nbsp;relabelto&nbsp;append&nbsp;unlink&nbsp;link&nbsp;rename&nbsp;execute&nbsp;swapon&nbsp;quotaon&nbsp;mounton</span></p></td></tr><tr><td colspan="1" rowspan="1"><p><span>directory</span></p></td><td colspan="1" rowspan="1"><p><span>add_name&nbsp;remove_name&nbsp;reparent&nbsp;search&nbsp;rmdir&nbsp;open&nbsp;audit_access&nbsp;execmod</span></p></td></tr><tr><td colspan="1" rowspan="1"><p><span>socket</span></p></td><td colspan="1" rowspan="1"><p><span>ioctl&nbsp;read&nbsp;write&nbsp;create&nbsp;getattr&nbsp;setattr&nbsp;lock&nbsp;relabelfrom&nbsp;relabelto&nbsp;append&nbsp;bind&nbsp;connect&nbsp;listen&nbsp;accept&nbsp;getopt&nbsp;setopt&nbsp;shutdown&nbsp;recvfrom&nbsp;sendto&nbsp;recv_msg&nbsp;send_msg&nbsp;name_bind</span></p></td></tr><tr><td colspan="1" rowspan="1"><p><span>filesystem</span></p></td><td colspan="1" rowspan="1"><p><span>mount&nbsp;remount&nbsp;unmount&nbsp;getattr&nbsp;relabelfrom&nbsp;relabelto&nbsp;transition&nbsp;associate&nbsp;quotamod&nbsp;quotaget</span></p></td></tr><tr><td colspan="1" rowspan="1"><p><span>process</span></p></td><td colspan="1" rowspan="1"><p><span>fork&nbsp;transition&nbsp;sigchld&nbsp;sigkill&nbsp;sigstop&nbsp;signull&nbsp;signal&nbsp;ptrace&nbsp;getsched&nbsp;setsched&nbsp;getsession&nbsp;getpgid&nbsp;setpgid&nbsp;getcap&nbsp;setcap&nbsp;share&nbsp;getattr&nbsp;setexec&nbsp;setfscreate&nbsp;noatsecure&nbsp;siginh&nbsp;setrlimit&nbsp;rlimitinh&nbsp;dyntransition&nbsp;setcurrent&nbsp;execmem&nbsp;execstack&nbsp;execheap&nbsp;setkeycreate&nbsp;setsockcreate</span></p></td></tr><tr><td colspan="1" rowspan="1"><p><span>security</span></p></td><td colspan="1" rowspan="1"><p><span>compute_av&nbsp;compute_create&nbsp;compute_member&nbsp;check_context&nbsp;load_policy&nbsp;compute_relabel&nbsp;compute_user&nbsp;setenforce&nbsp;setbool&nbsp;setsecparam&nbsp;setcheckreqprot&nbsp;read_policy</span></p></td></tr><tr><td colspan="1" rowspan="1"><p><span>capability</span></p></td><td colspan="1" rowspan="1"><p><span>chown&nbsp;dac_override&nbsp;dac_read_search&nbsp;fowner&nbsp;fsetid&nbsp;kill&nbsp;setgid&nbsp;setuid&nbsp;setpcap&nbsp;linux_immutable&nbsp;net_bind_service&nbsp;net_broadcast&nbsp;net_admin&nbsp;net_raw&nbsp;ipc_lock&nbsp;ipc_owner&nbsp;sys_module&nbsp;sys_rawio&nbsp;sys_chroot&nbsp;sys_ptrace&nbsp;sys_pacct&nbsp;sys_admin&nbsp;sys_boot&nbsp;sys_nice&nbsp;sys_resource&nbsp;sys_time&nbsp;sys_tty_config&nbsp;mknod&nbsp;lease&nbsp;audit_write&nbsp;audit_control&nbsp;setfcap</span></p></td></tr><tr><td colspan="1" rowspan="1"><p><strong><span>MORE</span></strong></p></td><td colspan="1" rowspan="1"><p><strong><span>AND&nbsp;MORE</span></strong></p></td></tr></tbody></table>

可以从对应的Class中选取任意个Permission

Class与Perimssion的完整映射具体见源码：kernel/arm64/security/selinux/include/classmap.h

#### **示例**

![161e79e977c6d226e5c3d195a2e75e58\_MD5](https://picgo.myjojo.fun:666/i/2024/10/12/670a460121437.png)

以控制狗是否有权吃猫粮狗粮为例。

有一个狗叫小黄， 一种狗粮叫“狗粮A”。

现在声明一个Type叫dog，一个Type叫dog\_chow，系统本身内置了Class叫food，food对应的permissions里包含了eat这个权限。

随后将小黄映射到dog这个type，将狗粮A映射到dog\_chow这个type

这样，在添加了这样一条规则后，小黄就有权限去吃狗粮A了：

```css
allow dog dog_chow:food eat;
```

此时如果还有一只狗小白，那么可以将小白映射到dog这个type，这样小白也可以有权限吃狗粮A。

### **2.3 Context**

上面所说的“映射”，即将一个进程关联到一个Type，或者将一个文件关联到一个Type，是通过context来完成的。

#### **进程Context（seapp\_contexts）**

一个进程的Context条目可能如下：

```cobol
user=_app isPrivApp=true name=com.android.vzwomatrigger domain=vzwomatrigger_app type=privapp_data_file levelFrom=all
```

user=\_app代表这是一个常规的app；

isPrivApp=true代表这是一个预置app；

name=com.android.vzwomatrigger 指定了进程名

domain=vzwomatrigger\_app 将进程名关联到一个type

type=privapp\_data\_file 这是一个文件type，指定了app的data directory目录所属的type

levelFrom=all MLS/MCS的level相关

AOSP内置进程Context见platform/system/sepolicy/private/seapp\_contexts

供应商或厂商要定义自己的seapp\_context，将在/vender下或\<root>/device/manufacturer/device-name/sepolicy下相应目录添加

#### **文件Context**

一个文件的Context可能这样：

```haskell
/system/bin/bcc                 u:object_r:rs_exec:s0
```

/system/bin/bcc 指的是具体文件

u:object\_r:rs\_exec:s0是一个security context。其中的rs\_exec为type。这样文件和type就进行了关联

#### **Security Context**

其格式为：

```haskell
user:role:type:sensitivity[:categories]
```

在Android中，

user是固定的，永远为u

role是固定的，访问者（称subject或source）永远为r；被访问者（称object或target）永远为object\_r。一般情况下，进程一方就是subject，文件一方就是object，所以一般进程的role为r，文件的role为object\_r。

type为与文件关联的type

sensitivity是固定的，永远为s0

categories是Multi-Level Security (MLS) 协议，用来隔离一个app的data，防止被另一个app访问，或者隔离不同用户间对同一个app data的访问。

> AOSP内置的文件Context见platform/system/sepolicy/private/file\_contexts

#### **seinfo**

seapp\_context里除了可以将一个具体应用映射到一个domain，也可以将seinf映射到domain。mac\_permissions.xml里定义了seinfo。其会根据app所属的signature来为其分配一个seinfo。

比如：

platform/system/sepolicy/private/mac\_permissions.xml

```xml
<!-- Platform dev key in AOSP -->
    <signer signature="@PLATFORM" >
      <seinfo value="platform" />
    </signer>
 
    <!-- Sdk Sandbox key -->
    <signer signature="@SDK_SANDBOX" >
      <seinfo value="sdk_sandbox" />
    </signer>
 
    <!-- Bluetooth key in AOSP -->
    <signer signature="@BLUETOOTH" >
      <seinfo value="bluetooth" />
    </signer>
 
    <!-- Media key in AOSP -->
    <signer signature="@MEDIA" >
      <seinfo value="media" />
    </signer>
 
    <signer signature="@NETWORK_STACK" >
      <seinfo value="network_stack" />
    </signer>
```

也就是说所有拥有PLATFORM signature的app，将拥有platform这个seinfo。在seapp\_context中可以如下配置：

```cpp
user=radio seinfo=platform domain=radio type=radio_data_file
```

所有拥有platform签名的，并且未配置具体context的app，将遵循其seinfo platform的规则。

#### **untrusted**\*\*\_****a****pp\*\*

既为配置seinfo，又未配置seapp\_context的app，将默认为untrusted\_app。各级seapp\_context中也有对untrusted\_app的权限配置。

#### **查看当前Context**

要看进程的Context，使用ps -Z

```haskell
emu64xa:/ $ ps -AZ | grep google
u:r:hal_camera_default:s0      system         362     1   10891800   3272 0                   0 S android.hardware.camera.provider@2.7-service-google
u:r:permissioncontroller_app:s0:c179,c256,c512,c768 u0_a179 976 354 13918328 83660 0          0 S com.google.android.permissioncontroller
u:r:bluetooth:s0               bluetooth     1033   354   14050488  73628 0                   0 S com.google.android.bluetooth
u:r:priv_app:s0:c512,c768      u0_a167       1090   354   14016372 119796 0                   0 S com.google.android.apps.nexuslauncher
u:r:priv_app:s0:c512,c768      u0_a170       1157   354   13873728  68724 0                   0 S com.google.android.ext.services
u:r:untrusted_app_32:s0:c142,c256,c512,c768 u0_a142 1393 354 14070168 104520 0                0 S com.google.android.inputmethod.latin
```

查看文件的Context，使用

```haskell
emu64xa:/ $ ls -Z
u:object_r:cgroup:s0                acct         u:object_r:tmpfs:s0                 debug_ramdisk    u:object_r:vendor_file:s0           odm                     u:object_r:sysfs:s0                 sys
u:object_r:apex_mnt_dir:s0          apex         u:object_r:device:s0                dev              u:object_r:vendor_file:s0           odm_dlkm                u:object_r:system_file:s0           system
u:object_r:rootfs:s0                bin          u:object_r:rootfs:s0                etc              u:object_r:oemfs:s0                 oem                     ?                                  
...
```

## **3. SeLinux的配置**

参见[https://android.googlesource.com/platform/system/sepolicy/](https://android.googlesource.com/platform/system/sepolicy/ "https://android.googlesource.com/platform/system/sepolicy/")

### **3**\*\*.\*\***1** **SeLinux****文件****体系**

之前提到Android架构中大致包含AOSP、厂商、Vendor等部分。在Android 8以上的系统中，AOSP和厂商、供应商的部分是独立配置的，方便OTA更新。

针对这种架构，SeLinux的Policy文件体系包含以下目录：

**system/sepolicy/public**：这部分是AOSP公开给vender使用的，作为其基础api。比如声明了domain的attribute就在这下面：platform/system/sepolicy/public/attributes。这部分Policy需要做兼容处理，因为Vender引用了这里policy，如果OTA单独升级了AOSP，需要做向后兼容处理。详见[https://source.android.com/docs/security/features/selinux/compatibility](https://source.android.com/docs/security/features/selinux/compatibility "https://source.android.com/docs/security/features/selinux/compatibility")

**system/sepolicy/private**：AOSP内部使用的，即system image内部用的policy。vender不应该感知到这部分policy

**system/sepolicy/vendor**：vender部分的policy

**device/manufacturer/device-name/sepolicy**：特定设备的policy，比如三星设备：device/samsung/tuna/sepolicy

##### **BoardConfig.mk makefile**

AOSP的SeLinux Policy文件一般不需要更改，或少量更改。厂商侧主要修改和定制的Policy在/device/manufacturer/device-name/下，/device/manufacturer/device-name/BoardConfig.mk用于指定Policy文件的具体路径，通过BOARD\_VENDOR\_SEPOLICY\_DIRS，比如：

device/samsung/tuna/BoardConfig.mk

```haskell
BOARD_VENDOR_SEPOLICY_DIRS += device/samsung/tuna/sepolicy
```

##### **system\_ext** **和** **product分区**

在Android 11及以上系统中，system\_ext 和 product分区也独立出单独的policy，并且区分了public和private。public部分同样共vender引用。system\_ext和product的版本兼容处理由各partner自己负责，AOSP不负责。

SYSTEM\_EXT\_PUBLIC\_SEPOLICY\_DIRS：指定system\_ext的public的目录，安装到system\_ext分区。

SYSTEM\_EXT\_PRIVATE\_SEPOLICY\_DIRS：指定system\_ext的private目录。system\_ext内部使用。安装到system\_ext分区

PRODUCT\_PUBLIC\_SEPOLICY\_DIRS：指定product分区的public目录。安装到product分区

PRODUCT\_PRIVATE\_SEPOLICY\_DIRS：指定product的private目录。安装到product分区。

### **3**\*\*.2\*\* **Policy文件**

把SeLinux策略配置相关的文件统称为Policy文件

Policy文件一般包含：

##### **TE\*\*\*\*文件**

就是一堆.te文件。TE里定义了Type、访问规则等。对进程和文件Type的定义、访问规则的定制就只在TE里完成的。一般每个进程有一个独立的te文件，而所有文件的te统一声明在一个file.te文件中。详见3.3节

##### **Context files**

**file\_contexts**：前面说过，file\_contexts里面定义了文件的context，用于将文件目录和文件的type关联起来。

**genfs\_contexts**：用于将文件系统（比如/proc和vfat）与type关联起来。

**property\_contexts**：用于将Android系统properties与type关联起来

**service\_contexts**：用于将service进程与type关联

**seapp\_contexts**：用于将app与type关联

**mac\_permissions.xml**：根据app的signature或包名，为app分配一个seinfo。seinfo用于在app没有明确关联一个type时归属一个默认type。

**keystore2\_key\_contexts** ：为Keystore 2.0 namespaces分配一个label。

##### **Attribute**

用来声明Attribute

##### **security\_classes**

用来声明Class

### \*\*3.****3** **TE****（Type Enforcement）\*\***文件**

所有的规则配置，或称Policy，都写在.te文件中。编写完TE文件后，将TE文件放在对应目录下，Android系统编译后.te文件将被编译成.cil文件。在init进程启动阶段，会将.cil文件汇总统一编译成一个命名为policy的文件。cil文件是可读的，policy文件是二进制的。在系统运行时最终加载使用的是policy文件。

policy文件一般在/sys/fs/selinux/policy

一般一个进程单独声明一个TE文件。比如一个dhcp进程单独有一个te文件叫dhcp.te。而文件的te统一整合在file.te（比如platform/system/sepolicy/public/file.te）中。

针对Platform（AOSP的部分）、Non-Platform（厂商、供应商的部分），TE会有不同的放置目录。

下面是TE文件的一个例子：

platform/system/sepolicy/public/dhcp.te

```haskell
type dhcp, domain;permissive dhcp;type dhcp_exec, exec_type, file_type;type dhcp_data_file, file_type, data_file_type;init_daemon_domain(dhcp)net_domain(dhcp)allow dhcp self:capability { setgid setuid net_admin net_raw net_bind_service};allow dhcp self:packet_socket create_socket_perms;allow dhcp self:netlink_route_socket { create_socket_perms nlmsg_write };allow dhcp shell_exec:file rx_file_perms;allow dhcp system_file:file rx_file_perms;# For /proc/sys/net/ipv4/conf/*/promote_secondariesallow dhcp proc_net:file write;allow dhcp system_prop:property_service set ;unix_socket_connect(dhcp, property, init)type_transition dhcp system_data_file:{ dir file } dhcp_data_file;allow dhcp dhcp_data_file:dir create_dir_perms;allow dhcp dhcp_data_file:file create_file_perms;allow dhcp netd:fd use;allow dhcp netd:fifo_file rw_file_perms;allow dhcp netd:{ dgram_socket_class_set unix_stream_socket } { read write };allow dhcp netd:{ netlink_kobject_uevent_socket netlink_route_socketnetlink_nflog_socket } { read write };
```

#### **TE文件语法解析**

type dhcp, domain; -- 声明一个type名为dhcp，其继承domain这个Attribute，也就是说在声明时便继承了domain所拥有的权限。domain属性是进程专用的，很显然这是一个进程的te文件。

permissive dhcp; -- 将dhcp标识为permissive模式。之前说到，SeLinux开启模式可以全局设置，也可以针对进程单独设置。这里是单独设置该进程的模式。

type dhcp\_exec, exec\_type, file\_type; -- 声明一个type名为dhcp\_exec， 从属于exec\_type和file\_type两个Attribute。也就是说同时继承两个Attribute所代表的文件。exec\_type用于代表可执行文件，也就是进程的可执行文件入口；file\_type代表通用文件。从这里我们知道dhcp\_exec代表dhcp可执行文件的入口。

init\_daemon\_domain(dhcp) -- 这是一个宏，代表这是一个从init启动的进程，并且可以同init通信。宏的源码见platform/system/sepolicy/public/te\_macros

net\_domain -- 也是一个宏，可以进行通用的网络操作，比如读写TCP包，操作socket等。

allow dhcp self:capability { setgid setuid net\_admin net\_raw net\_bind\_service}; -- 这个就是具体的规则描述了。dhcp是source方， self是target方，capability是一个Class名，其后是可以做的操作。这句话是说允许dhcp这个进程对自己做setgid等操作。self关键字用来表示source可以对自己做什么操作。

除了上面例子里出现的关键字，有时我们还会看到typeattribute这个关键字：它代表将之前已经声明过的Type关联到一个之前声明过的Attribute。

> 更多语法见[TypeStatements - SELinux Wiki](https://selinuxproject.org/page/TypeStatements "TypeStatements - SELinux Wiki")

## ​​​​​​​4. **SeLinux的编译**

### **4**\*\*.\*\***1** **编译\*\*\*\*SeLinux** **Policy**

Android 8以上编译过程中将把/system 和 /vendor下的policy合并。这个逻辑体现在/platform/system/sepolicy/Android.mk。

具体policy位置：

<table><tbody><tr><td colspan="1" rowspan="1"><p><span>Location</span></p></td><td colspan="1" rowspan="1"><p><span>Contains</span></p></td></tr><tr><td colspan="1" rowspan="1"><p><span>system/</span><span>sepolicy/</span><span>public</span></p></td><td colspan="1" rowspan="1"><p><span>The&nbsp;platform's&nbsp;sepolicy&nbsp;API</span></p></td></tr><tr><td colspan="1" rowspan="1"><p><span>system/</span><span>sepolicy/</span><span>private</span></p></td><td colspan="1" rowspan="1"><p><span>Platform&nbsp;implementation&nbsp;details&nbsp;(vendors&nbsp;can&nbsp;ignore)</span></p></td></tr><tr><td colspan="1" rowspan="1"><p><span>system/</span><span>sepolicy/</span><span>vendor</span></p></td><td colspan="1" rowspan="1"><p><span>Policy&nbsp;and&nbsp;context&nbsp;files&nbsp;that&nbsp;vendors&nbsp;can&nbsp;use&nbsp;(vendors&nbsp;can&nbsp;ignore&nbsp;if&nbsp;desired)</span></p></td></tr><tr><td colspan="1" rowspan="1"><p><span>BOARD_</span><span>SEPOLICY_</span><span>DIRS</span></p></td><td colspan="1" rowspan="1"><p><span>Vendor&nbsp;sepolicy</span></p></td></tr><tr><td colspan="1" rowspan="1"><p><span>BOARD_</span><span>ODM_</span><span>SEPOLICY_</span><span>DIRS</span> <span>(Android&nbsp;9&nbsp;and&nbsp;higher)</span></p></td><td colspan="1" rowspan="1"><p><span>Odm&nbsp;sepolicy</span></p></td></tr><tr><td colspan="1" rowspan="1"><p><span>SYSTEM_</span><span>EXT_</span><span>PUBLIC_</span><span>SEPOLICY_</span><span>DIRS</span> <span>(Android&nbsp;11&nbsp;and&nbsp;higher)</span></p></td><td colspan="1" rowspan="1"><p><span>System_ext's&nbsp;sepolicy&nbsp;API</span></p></td></tr><tr><td colspan="1" rowspan="1"><p><span>SYSTEM_</span><span>EXT_</span><span>PRIVATE_</span><span>SEPOLICY_</span><span>DIRS</span> <span>(Android&nbsp;11&nbsp;and&nbsp;higher)</span></p></td><td colspan="1" rowspan="1"><p><span>System_ext&nbsp;implementation&nbsp;details&nbsp;(vendors&nbsp;can&nbsp;ignore)</span></p></td></tr><tr><td colspan="1" rowspan="1"><p><span>PRODUCT_</span><span>PUBLIC_</span><span>SEPOLICY_</span><span>DIRS</span> <span>(Android&nbsp;11&nbsp;and&nbsp;higher)</span></p></td><td colspan="1" rowspan="1"><p><span>Product's&nbsp;sepolicy&nbsp;API</span></p></td></tr><tr><td colspan="1" rowspan="1"><p><span>PRODUCT_</span><span>PRIVATE_</span><span>SEPOLICY_</span><span>DIRS</span> <span>(Android&nbsp;11&nbsp;and&nbsp;higher)</span></p></td><td colspan="1" rowspan="1"><p><span>Product&nbsp;implementation&nbsp;details&nbsp;(vendors&nbsp;can&nbsp;ignore)</span></p></td></tr></tbody></table>

编译过程：

1. 将Policy转化成CIL（ Common Intermediate Language ）文件，包括
2. system + system\_ext + product的public部分的policy
3. 合并private + public policy
4. 合并public + vendor and BOARD\_SEPOLICY\_DIRS policy
5. 将public部分的policy进行整理成对应版本policy，作为vendor policy的一部分。然后将public的policy传给合并后的public + vendor and BOARD\_SEPOLICY\_DIRS policy，告知其哪部分可以连接到platform的attribute
6. 创建一个mapping文件，连接platform和vendor的部分。
7. 合并所有的policy文件，整合mapping、platform和vender的policy，最终输出一个二进制的名为policy的文件。一般这个文件的位置在/sys/fs/sepolicy/policy。这个policy最终会被kernel加载。这个工作是在init进程初始化时进行的。也就是说是在运行时进行的。

### **4.2 预编译**

在SeLinux正式开启之前，init进程会收集所有的cil文件，将其编译成policy，这个过程需要花费1-2秒时间。为了更快完成此过程，会将CIL在编一阶段预编译，放在/vendor/etc/selinux/precompiled\_sepolicy和/odm/etc/selinux/precompiled\_sepolicy下，并且附有sha256 摘要。运行时会检查sha256是否有变化，如果没有变化就直接使用预编译的文件。

## **5. 实践**

### **5**\*\*.\*\***1** **开启****Se****L****i****n****u****x**

编译kernel时，指定CONFIG\_SECURITY\_SELINUX=y。

比如在kernel/arm64/arch/arm64/configs/defconfig中添加此配置。

在kernel/arm64/include/linux/selinux.h会对该配置进行判断：

```cpp
#ifdef CONFIG_SECURITY_SELINUXbool selinux_is_enabled(void);#elsestatic inline bool selinux_is_enabled(void){return false;}#endif	#endif 
```

### \*\*5.\*\***2** **设定SeLinux模式**

在厂商目录的BoardConfig.mk中（比如/device/google/trout/trout\_arm64/BoardConfig.mk）添加如下配置，可改变SeLinux模式

```cobol
BOARD_KERNEL_CMDLINE := androidboot.selinux=permissive
```

或

```cobol
BOARD_BOOTCONFIG := androidboot.selinux=permissive
```

上面两个配置只用于开发阶段。正式版本需要移除，否则无法通过CTS。即正式版本上，SeLinux是强制enforcing模式。

**非正式的修改方法：**

ALLOW\_PERMISSIVE\_SELINUX宏定义了是否启用permissivve模式。

在core/init/Android.bp中有对该宏设置：

```cobol
init_first_stage_cc_defaults {
	...
	cflags: [
        "-Wall",
        "-Wextra",
        "-Wno-unused-parameter",
        "-Werror",
        "-DALLOW_FIRST_STAGE_CONSOLE=0",
        "-DALLOW_LOCAL_PROP_OVERRIDE=0",
        "-DALLOW_PERMISSIVE_SELINUX=0",
        "-DREBOOT_BOOTLOADER_ON_PANIC=0",
        "-DWORLD_WRITABLE_KMSG=0",
        "-DDUMP_ON_UMOUNT_FAILURE=0",
        "-DSHUTDOWN_ZERO_TIMEOUT=0",
        "-DLOG_UEVENTS=0",
        "-DSEPOLICY_VERSION=30", // TODO(jiyong): externalize the version number
    ],
}
```

实际的生效位置：

```csharp
core/init/selinux.cppbool IsEnforcing() {if (ALLOW_PERMISSIVE_SELINUX) {return StatusFromProperty() == SELINUX_ENFORCING;    }return true;}
```

### **5**\*\*.\*\***3** **处理****异常****LOG**

厂商或供应上在添加了一些进程、访问行为、策略等之后，需要确认是否符合SeLinux规则。

如果有任何访问限制出现，则会打印log：

```cobol
02-05 15:06:30.448   450   450 E SELinux : avc:  denied  { find } for pid=3059 uid=10074 name=media.metrics scontext=u:r:testapp:s0:c512,c768 tcontext=u:object_r:mediametrics_service:s0 tclass=service_manager permissive=0
```

精简下log，只需avc:后面的即可

```cobol
avc:  denied  { find } for pid=3059 uid=10074 name=media.metrics scontext=u:r:testapp:s0:c512,c768 tcontext=u:object_r:mediametrics_service:s0 tclass=service_manager permissive=0
```

各字段含义：

{ find }：被组织的行为。

scontext（可以理解为source context的缩写）：该行为的执行者的context；

tcontext（可以理解为target context的缩写：）被访问者的context；

tclass：被访问者的class，详见本文之前的解释。

####  **使用auditallow****生成****TE**

ubuntu中，通过apt-get audit2allow安装audit2allow。

获取policy文件：

/sys/fs/selinux/policy

将上面的avc log保存在test.avc文件中，然后执行：

audit2allow -i test.avc -p policy > hynex.te

即生成TE文件。再将TE加入到SeLinux编译目录sepolicy即可修复异常log。
