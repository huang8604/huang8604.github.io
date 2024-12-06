---
title: Android13 GMS相关RKP流程简述-CSDN博客
source: https://blog.csdn.net/qixi119220/article/details/135879056
author: 
published: 
created: 2024-11-18
description: 
tags:
  - clippings
  - GMS
  - blog
date: 2024-11-18T01:18:38.261Z
lastmod: 2024-11-18T01:21:13.261Z
---
远程密钥配置 (RKP) 包括一种新型密钥管理，可将配置过程从工厂内配置转移到无线配置。这是通过将工厂内公钥提取和无线证书配置与短期证书相结合来完成的。作为 Android 认证基础设施的一部分，RKP 加强了 Android 信任链的安全性，并提高了发生漏洞时设备信任的可恢复性。

对于需要支持RKP的项目，如果在认证时未支持，会导致GTS fail

<table><tbody><tr><td colspan="3"><a>arm64-v8a&nbsp;GtsGmscoreHostTestCases</a></td></tr><tr><th>Test</th><th>Result</th><th>Details</th></tr><tr><td>com.google.android.gts.security.AttestationRootHostTest#testEcAttestationChainRemProvLengthTee</td><td><p>fail</p></td><td><p>java.lang.AssertionError: on-device tests failed:</p></td></tr><tr><td>com.google.android.gts.security.AttestationRootHostTest#testRsaAttestationChainRemProvLengthTee</td><td><p>fail</p></td><td><p>java.lang.AssertionError: on-device tests failed:</p></td></tr></tbody></table>

<table><tbody><tr><td colspan="3"><a>armeabi-v7a&nbsp;GtsGmscoreHostTestCases</a></td></tr><tr><th>Test</th><th>Result</th><th>Details</th></tr><tr><td>com.google.android.gts.security.AttestationRootHostTest#testEcAttestationChainRemProvLengthTee</td><td><p>fail</p></td><td><p>java.lang.AssertionError: on-device tests failed:</p></td></tr><tr><td>com.google.android.gts.security.AttestationRootHostTest#testRsaAttestationChainRemProvLengthTee</td><td><p>fail</p></td><td><p>java.lang.AssertionError: on-device tests failed:</p></td></tr></tbody></table>

所以基于认证需求，需要在项目开始时，向Google申请RKP。

1\. 需要mada所属公司开通android-partner-api@mada公司邮箱地址，并使用该邮箱创建google账号。后续需要使用这个邮箱作为账户名，登录GCP，管理RKP项目。

完整步骤：

[https://docs.partner.android.com/gms/building/integrating/att-keys/Setting-Up-Partner-API?hl=zh-cn](https://docs.partner.android.com/gms/building/integrating/att-keys/Setting-Up-Partner-API?hl=zh-cn "https://docs.partner.android.com/gms/building/integrating/att-keys/Setting-Up-Partner-API?hl=zh-cn")

2\. 在GCP上的项目，需要和公司ID绑定，公司ID是向Google提交项目的账号，和实验室有关。如果一个公司有多个合作的3pl实验室，则会有多个公司ID，eg.mada公司名-3pl实验室名。后续上传json时，GCP上的project需要和company id绑定，所以针对不同的company id，需要创建不同的GCP project。公司ID可以在APA上查询到（[https://partner.android.com/approvals/](https://partner.android.com/approvals/ "https://partner.android.com/approvals/")）

![](https://i-blog.csdnimg.cn/blog_migrate/977aef0f1e031ae8ca03b1482170d456.png)

![](https://i-blog.csdnimg.cn/blog_migrate/311f2396576e6a9e3a83186ffa993891.png)

3\. 根据上面链接步骤创建OAuth2 凭证和服务帐户凭据，创建过程中生成的凭证文件务必保存好，后续上传json需要用到。

4\. json的生成和上传

完整步骤：

[https://docs.partner.android.com/gms/building/integrating/att-keys/rkp?hl=zh-cn#dev-best-practices](https://docs.partner.android.com/gms/building/integrating/att-keys/rkp?hl=zh-cn#dev-best-practices "https://docs.partner.android.com/gms/building/integrating/att-keys/rkp?hl=zh-cn#dev-best-practices")

我的项目使用豆荚的写号方案，json由工具直接提取。我司只需要按照Google的脚本上传即可。

我司选择链接中方法2上传json

##### 选项 2：将`device_info_uploader.py`与 GCP 服务帐户结合使用

使用`csrs.json`文件和您在Service Account Credentials中下载的`my-project-0000000000-ffffffffffff.json`密钥文件运行`device_info_uploader.py`脚本。示例命令如下所示：

> ```cobol
> ./device_info_uploader.py --credentials-keyfile /secure/storage/my-project-0000000000-ffffffffffff.json --json-csr csrs.json --company-id COMPANY_ID 
> ```

命令说明， 

`device_info_uploader.py是从google网站上直接下载的脚本`

`/secure/storage/my-project-0000000000-ffffffffffff.json是创建GCP项目时生成的秘钥`

`csrs.json是从写key后的设备中提取的json文件`

`COMPANY_ID是前面提到的公司ID`

如果账号一切正常，配置正常，网络正常，GCP API也正常启用，直接上传json时会报这个错

#### 公司 ID 的用户权限被拒绝

当当前用户没有上传指定公司 ID 的 CSR 的必要权限时，会出现此错误。

> 消息：此 CSR 未上传。不允许用户 {USER} 上传公司 ID {COMPANY\_ID} 的设备信息。请提交错误或联系 TAM 以获得权​​限。
>
> 代码：7（权限被拒绝）403 禁止

这是由于GCP项目尚未和公司ID绑定，最快的方法是联系TAM帮忙绑定权限。绑定后，即可正常上传json了。

其他报错参考：

[https://docs.partner.android.com/gms/building/integrating/att-keys/rkp-errors?hl=zh-cn](https://docs.partner.android.com/gms/building/integrating/att-keys/rkp-errors?hl=zh-cn "https://docs.partner.android.com/gms/building/integrating/att-keys/rkp-errors?hl=zh-cn")
