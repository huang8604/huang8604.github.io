---
tags:
  - blog
  - 实战
categories: Android
title: Archer&ArcherIS AP签名方案总结
date: 2024-11-06T06:57:56.915Z
lastmod: 2024-11-05T01:34:02.089Z
---
```table-of-contents
style: nestedList # TOC style (nestedList|inlineFirstLevel)
maxLevel: 0 # Include headings up to the speficied level
includeLinks: true # Make headings clickable
debugInConsole: false # Print debug info in Obsidian console
```

### 关于AP端 HSM签名的配置说明

Archer 项目签名使用的加密狗来自YubiHSM\
具体的一些操作说明需要到官网寻找：\
[YubiHSM2](https://developers.yubico.com/YubiHSM2/)

#### 用于签名的HSM项目目前有:

1. Archer  A11
2. Archer  A13
3. ArcherIS

#### 编译的服务器分别在北京和沈阳

##### 北京服务器

上插着两个加密狗

###### 一个加密狗 用于 ArcherIS  ==serial=0016980259==

目前的usb serial no：

```shell
connector=yhusb://serial=0016980259
```

配置和密码：

```shell
[pkcs11_section]
engine_id = pkcs11
#dynamic_path = /usr/lib/ssl/engines/libpkcs11.so
MODULE_PATH = /usr/lib/x86_64-linux-gnu/pkcs11/yubihsm_pkcs11.so
INIT_ARGS = "connector=yhusb://serial=0016980259"
# PIN is the <key_id><key-password>, by default this is 0001password. Meaning the default key_id is '0001' and the default password is 'password'
# key_id = 0x2873 password = ef94cc600359806750df6595bbdf9652
# remove the 0x from the key_id and add the password to the end of it
# PIN = <key_id><password>
# PIN = <2873><ef94cc600359806750df6595bbdf9652>
# PIN = 5147bb4dc2069b1094a80b0311a91afd72f8
PIN = "5147bb4dc2069b1094a80b0311a91afd72f8"
init = 0
```

列出里面的pkcs11 key的情况：

```shell
builder@builder:~/archeris_hsm_apk_signer$ YUBIHSM_PKCS11_CONF=./conf/yubihsm_connector.conf keytool -list -keystore NONE -storetype PKCS11  -storepass 5147bb4dc2069b1094a80b0311a91afd72f8  -providerClass sun.security.pkcs11.SunPKCS11 -providerArg ./conf/hsm_config.conf 

Keystore type: PKCS11
Keystore provider: SunPKCS11-yubihsm-pkcs

Your keystore contains 9 entries

is_platform, PrivateKeyEntry, 
Certificate fingerprint (SHA-256): 4A:EE:97:7C:1C:10:E4:1A:59:38:8F:3A:20:27:17:55:3C:EF:7E:7D:30:B4:10:B1:40:15:09:FC:4F:57:7D:79
media, PrivateKeyEntry, 
Certificate fingerprint (SHA-256): 90:02:EB:69:82:F9:B8:F1:6A:0F:02:2F:03:0E:F8:AC:6E:78:27:41:86:D1:D8:0F:87:39:A1:2D:40:49:2F:1C
networkstack, PrivateKeyEntry, 
Certificate fingerprint (SHA-256): B7:7A:E1:E4:80:90:73:DC:C2:D1:09:BB:E8:28:81:65:29:BF:97:7C:33:07:73:53:8C:67:11:14:F8:69:17:49
partner, PrivateKeyEntry, 
Certificate fingerprint (SHA-256): 12:13:92:C9:91:78:7C:96:84:25:1C:A3:49:94:CF:E6:C8:E7:90:85:F7:E7:4D:15:A6:D9:D7:86:81:E1:51:51
platform, PrivateKeyEntry, 
Certificate fingerprint (SHA-256): D8:4F:01:E6:6F:5E:CD:05:4A:B3:67:0F:EA:5A:E1:FA:EA:A7:73:04:A2:9D:F1:C2:59:48:09:52:E5:FB:C4:6B
release, PrivateKeyEntry, 
Certificate fingerprint (SHA-256): 09:55:75:AE:AE:A3:AE:20:BB:B9:43:2C:1D:47:73:09:BC:E4:22:A0:F3:60:38:0F:E0:31:2F:2C:FE:27:D9:3F
shared, PrivateKeyEntry, 
Certificate fingerprint (SHA-256): 84:CE:CB:ED:38:A7:00:AB:D1:E4:71:0C:8C:51:0D:3F:77:D6:B2:CB:72:B3:96:94:83:7A:A8:8A:FE:C2:93:9B
webview, PrivateKeyEntry, 
Certificate fingerprint (SHA-256): 06:DC:3A:86:59:EF:66:95:5B:C6:18:37:A3:D1:F3:5D:44:18:01:59:43:0D:3B:27:ED:B6:A7:D5:80:0C:7A:EA
wrap_259, SecretKeyEntry, 


```

用途：

1. ArcherIS     AP apk 签名
2. ArcherIS     releasekey ota签名
3. Archer A13   AP apk 签名(计划中，验证中)

Archer A13  目前延用A11 的项目 release ota 采用直接用releasekey的方案

###### 一个加密狗 用于 Archer   ==551==

```
connector=yhusb://serial=0016499551
```

```shell
[pkcs11_section]
engine_id = pkcs11
#dynamic_path = /usr/lib/ssl/engines/libpkcs11.so
dynamic_path = /usr/lib/x86_64-linux-gnu/openssl-1.0.0/engines/libpkcs11.so
MODULE_PATH = /usr/lib/x86_64-linux-gnu/pkcs11/yubihsm_pkcs11.so
PIN = "60d4458874be39d674225d861b3080f02068"
INIT_ARGS = "connector=yhusb://serial=0016499551"
#INIT_ARGS = "connector=http://localhost:12345"
init = 0
```

```shell
builder@builder:~/hsm_apk_signer$ YUBIHSM_PKCS11_CONF=./conf/yubihsm_connector.conf keytool -list -keystore NONE -storetype PKCS11  -storepass 60d4458874be39d674225d861b3080f02068  -providerClass sun.security.pkcs11.SunPKCS11 -providerArg ./conf/hsm_config.conf 

Keystore type: PKCS11
Keystore provider: SunPKCS11-yubihsm-pkcs

Your keystore contains 0 entries

```

用途：

1. ArcherIS       CP SecureBoot 应该也是用的该加密狗。
2. Archer A11     CP SecureBoot 应该也是用的该加密狗。
3. Archer A13     CP SecureBoot 应该也是用的该加密狗。

##### 沈阳服务器

上插着一个加密狗

###### Archer A11的项目 ==561==

```
[pkcs11_section]
engine_id = pkcs11
#dynamic_path = /usr/lib/ssl/engines/libpkcs11.so
MODULE_PATH = /usr/lib/x86_64-linux-gnu/pkcs11/yubihsm_pkcs11.so
INIT_ARGS = "connector=yhusb://serial=0016499561"
PIN = "2873ef94cc600359806750df6595bbdf9652"
init = 0

```

```shell
builder@builder-PowerEdge-R740:~/hsm_apk_signer$ YUBIHSM_PKCS11_CONF=./conf/yubihsm_connector.conf keytool -list -keystore NONE -storetype PKCS11  -storepass 2873ef94cc600359806750df6595bbdf9652  -providerClass sun.security.pkcs11.SunPKCS11 -providerArg ./conf/hsm_config.conf 

Keystore type: PKCS11
Keystore provider: SunPKCS11-yubihsm-pkcs

Your keystore contains 6 entries

media, PrivateKeyEntry, 
Certificate fingerprint (SHA-256): 90:02:EB:69:82:F9:B8:F1:6A:0F:02:2F:03:0E:F8:AC:6E:78:27:41:86:D1:D8:0F:87:39:A1:2D:40:49:2F:1C
networkstack, PrivateKeyEntry, 
Certificate fingerprint (SHA-256): B7:7A:E1:E4:80:90:73:DC:C2:D1:09:BB:E8:28:81:65:29:BF:97:7C:33:07:73:53:8C:67:11:14:F8:69:17:49
platform, PrivateKeyEntry, 
Certificate fingerprint (SHA-256): D8:4F:01:E6:6F:5E:CD:05:4A:B3:67:0F:EA:5A:E1:FA:EA:A7:73:04:A2:9D:F1:C2:59:48:09:52:E5:FB:C4:6B
release, PrivateKeyEntry, 
Certificate fingerprint (SHA-256): 1A:0E:D4:92:DE:8E:A5:90:8A:51:BB:BA:80:70:F7:F7:CB:45:E0:3A:56:EB:19:D5:38:83:F3:EA:F0:2E:FA:AC
shared, PrivateKeyEntry, 
Certificate fingerprint (SHA-256): 84:CE:CB:ED:38:A7:00:AB:D1:E4:71:0C:8C:51:0D:3F:77:D6:B2:CB:72:B3:96:94:83:7A:A8:8A:FE:C2:93:9B
wrapkey, SecretKeyEntry, 
builder@builder-PowerEdge-R740:~/hsm_apk_signer$ 

```

用途：

1. Archer A11 AP端apk 签名使用\
   Archer A11 release key 签名 仍然采用的releasekey 文件的形式签名。

#### 对比总结

**从两个Usb key 导出的公钥来看，沈阳的加密狗561 和北京的加密狗259  里面存储的秘钥来看  meida platform share network 是一样的。**

##### 一 platform key 与其他key

**ArcherIS 采用来的 is\_platform 来作为platform key的签名，\
Archer 使用的是 platform 作为平台签名，北京259和沈阳561 的platform key 都可以使用\
其余 media share network 的key 两个项目共用**

##### 二 release key

**这两个加密狗里面 release key 是不一样的。\
ArcherIS 使用了加密狗里面的key 作为ota的签名\
Archer 使用的A11项目里面release key文件作为 release 签名(这个文件的来源目前不太清楚)**

##### 三 Archer   ==551== 的情况

**这个usb上面没有pkcs11的key**

### 签名方案

关于HSM的签名方案，概要的说就是原来放在build/security/ 下面的 私钥和证书对。 客户由于安全的因素，不外发相关的私钥。\
那么私钥通过写入 HSM 这种加密U盘(加密狗)，通过私钥去签名的工作，就交给了加密U盘来处理了。

那么在源码编译过程中使用了 build/security/  下面key的工作有以下几点（CP端的镜像签名没有采用里面的key，有额外的key，在这里不讨论）

1. APK的签名工作
2. ota包的签名工作

#### 过往方案选择说明

那么要做的工作就是把上面的两个签名流程，从本地签名修改到通过加密狗去签名。\
那么之前的工程师在修改签名流程的时候遇到了一个问题，就是Android 源码的编译是多线程的，一般CPU 核心越多，线程越多。可能在编译apk的时候，会有多个应用去同时签名，而加密狗是一个单线程的签名系统。这里就会遇到冲突问题。

目前采取的方案，就是启动一个签名服务，用来提供签名工作，传入签名文件，和签名秘钥别名。把这些需求加入队列中，按照顺序去签名。这样就不会有冲突问题。

签名服务上传了源码，在项目路径：

> \[!NOTE]\
> build/soong/cmd/hsm\_signer

#### APK的签名工作

hsm 上支持通过apksigner 进行 PKCS11 的签名，具体命令如下，里面包含了加密狗的 serial no，还有密码。

```shell
#!/bin/sh
sudo YUBIHSM_PKCS11_CONF=simt_connector.conf \
	/home/kailoma/Android/Sdk/build-tools/31.0.0/apksigner sign \
	--provider-class sun.security.pkcs11.SunPKCS11 \
	--provider-arg simt_yubihsm.conf \
	--ks NONE \
	--ks-pass pass:5147bb4dc2069b1094a80b0311a91afd72f8 \
	--ks-type pkcs11 \
	--ks-key-alias <key-alias> \
	--min-sdk-version 17 \
	--max-sdk-version 30 \
	--v1-signing-enabled true \
	--v2-signing-enabled true \
	-v \
	-out <path-to-output-apk-file>-signed.apk \
	-in <path-to-input-apk-file>.apk;
```

这部分实现是在加密服务里面，通常AOSP 编译是通过调用加密服务来实现的：

```shell
curl -d "key_alias=platform&apk_unsigned_path=/home/builder/huangwentao/temp/CIT.apk&apk_signed_path=/home/builder/huangwentao/temp/CIT_PLATFORM.apk" http://localhost:8886/hsm_apk_sign
```

这个例子里面就是：访问了 localhost 8886 端口的 hsm\_apk\_sign服务

然后在源码 SignApk.java 里面：

```java
    ProcessBuilder processBuilder = new ProcessBuilder();	
    processBuilder.command("bash", "-c", "curl -d \"key_alias=" + keyAlias + "&apk_unsigned_path=" + unsignedApkPath + "&apk_signed_path=" + signedApkPath + "\"  http://localhost:8886/hsm_apk_sign");	
    try {	
        Process process = processBuilder.start();	
        StringBuilder output = new StringBuilder();	
        BufferedReader reader = new BufferedReader(new InputStreamReader(process.getInputStream()));	
        String line;	
        while ((line = reader.readLine()) != null) {	
            output.append(line + "\n");	
        }
```

#### ota包的签名工作 采用官网提示进行签名

ota包的签名工作，不是调用apksigner

HSM官网的说明里面，pkcs11可以采用pkcs11-tool

```shell
export YUBIHSM_PKCS11_CONF=~/huangwentao/qcm6490-archer-IS-00034-dev-r/build/soong/cmd/hsm_signer/conf/yubihsm_connector.conf
 
#-m SHA256-RSA-PKCS
/usr/bin/pkcs11-tool --module /usr/lib/x86_64-linux-gnu/pkcs11/yubihsm_pkcs11.so --login -p 5147bb4dc2069b1094a80b0311a91afd72f8 --sign  --id 0032 -i sig.bin -o signed-new.bin

```

ota的签名里面的sig.bin 就是针对payload.bin进行的sha256  hash值。所以在签名的时候，不用再加-m SHA256-RSA-PKCS，否则验签会出错。

编译整包的时候，会有sig.bin 以及签名文件sig\_signed.bin 出来。我们通过 od -tx1 来查看二进制文件。

 \
sig.bin 二进制：

```shell
builder@builder:~/huangwentao/buildtemp/after$ cat sig.bin |od -tx1  
0000000 05 ef dd 77 4f 7f 1d 11 59 cc fc dd 9d c7 25 be  
0000020 2a a9 16 5b 5d 9a dd 32 5b 46 c6 f1 93 16 f0 8f  
0000040

```

sig.bin hash256后的二进制：

```shell
builder@builder:~/huangwentao/buildtemp/after$ openssl sha256 -binary sig.bin |od -tx1  
0000000 20 c3 0a eb ac 21 1f 23 11 a4 8e dd 66 6e ae 1f  
0000020 fc b5 72 fd 3f f8 15 09 e1 71 41 1d fb 21 93 e8  
0000040

```

签名添加 sha256后的签名  /usr/bin/pkcs11-tool --module /usr/lib/x86\_64-linux-gnu/pkcs11/yubihsm\_pkcs11.so --login -p 5147bb4dc2069b1094a80b0311a91afd72f8 --sign  --id 0032 -m SHA256-RSA-PKCS -i sig.bin -o sig\_signed256.bin\
然后通过公钥去解码，发现和sig.bin  hash 256的是一致的。

```shell
builder@builder:~/huangwentao/buildtemp/after$ openssl rsautl -verify -pubin -inkey key-pub.key -in sig_signed256.bin |od -tx1  
0000000 30 31 30 0d 06 09 60 86 48 01 65 03 04 02 01 05  
0000020 00 04 20 20 c3 0a eb ac 21 1f 23 11 a4 8e dd 66  
0000040 6e ae 1f fc b5 72 fd 3f f8 15 09 e1 71 41 1d fb  
0000060 21 93 e8
```

在main 函数里面添加参数，采用 pkcs11-tool 去签名

```python
@@ -1200,6 +1212,21 @@ def GenerateAbOtaPackage(target_file, output_file, source_file=None):
 
 def main(argv):
 
+  #ArcherIs code for yubihsm by 00169 at 20230423 begin
+  if os.path.exists("./.build_with_yubihsm") :
+    #wentao
+    cmd = ["pwd"]
+    common.RunAndCheckOutput(cmd)
+    print("hsm py version 002 yubihsm exist ,use yubihsm build tools pkcs11-tool instend ")
+    print("yubihsm before argv :",argv)
+    argv.insert(0, '-v')
+    argv.insert(1, '--payload_signer')
+    argv.insert(2, '/usr/bin/pkcs11-tool')
+    #argv.insert(3, '--payload_signer_args=--module /usr/lib/x86_64-linux-gnu/pkcs11/yubihsm_pkcs11.so --login -p 5147bb4dc2069b1094a80b0311a91afd72f8 --sign -m SHA256-RSA-PKCS --id 0032')
+    argv.insert(3, '--payload_signer_args=--module /usr/lib/x86_64-linux-gnu/pkcs11/yubihsm_pkcs11.so --login -p 5147bb4dc2069b1094a80b0311a91afd72f8 --sign  --id 0032')
+    print("yubihsm after args: ",argv)
+  #ArcherIs code for yubihsm by 00169 at 20230423 end
+
   def option_handler(o, a):
     if o in ("-k", "--package_key"):
       OPTIONS.package_key = a

```
