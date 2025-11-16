# 自动开关机脚本

reboot\_test.sh

```shell

#Navy add for reboot/factory-reset test

#adb reboot
#adb reboot factory-reset

#默认开关机测试次数
rebootNum=5
#开关机设备ID号
deviceId=null
#开关机间隔时间
rebootInterval=30

if [ $# = 1 ];then
   rebootNum=$1
elif [ $# = 2 ];then
   rebootNum=$1
   deviceId=$2
fi

i=1
while [ $i -le $rebootNum ]
do
if [[ $deviceId =~ "null" ]];then
   echo "reboot test num" $i
   adb reboot
else
   echo "reboot "$deviceId" test num "$i
   adb -s $deviceId reboot
fi
sleep $rebootInterval
let i++
done
```


---

> 作者: [wentao](https://github.com/huang8604)  
> URL: https://huang8604.github.io/posts/%E8%87%AA%E5%8A%A8%E5%BC%80%E5%85%B3%E6%9C%BA%E8%84%9A%E6%9C%AC/  

