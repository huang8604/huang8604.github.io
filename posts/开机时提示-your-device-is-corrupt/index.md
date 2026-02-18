# 开机时提示 Your Device Is Corrupt？

[avb校验相关与块校验原理](/avb%E6%A0%A1%E9%AA%8C%E7%9B%B8%E5%85%B3%E4%B8%8E%E5%9D%97%E6%A0%A1%E9%AA%8C%E5%8E%9F%E7%90%86)

```
adb reboot recovery
adb root
adb shell
reboot "dm-verity enforcing"
```


---

> 作者: [wentao](https://github.com/huang8604)  
> URL: https://huang8604.github.io/posts/%E5%BC%80%E6%9C%BA%E6%97%B6%E6%8F%90%E7%A4%BA-your-device-is-corrupt/  

