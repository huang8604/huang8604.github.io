---
tags:
  - blog
  - Framework
title: 开机时提示 your device is corrupt？
date: 2025-09-11T01:46:50.766Z
lastmod: 2025-11-04T04:20:26.000Z
---
[avb校验相关与块校验原理](/avb%E6%A0%A1%E9%AA%8C%E7%9B%B8%E5%85%B3%E4%B8%8E%E5%9D%97%E6%A0%A1%E9%AA%8C%E5%8E%9F%E7%90%86)

```
adb reboot recovery
adb root
adb shell
reboot "dm-verity enforcing"
```
