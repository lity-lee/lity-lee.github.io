---
layout: post
category: android
title:  "adb使用和usb通信"
tags: [adb,usb,driver]
---

## adb使用的技巧

#### 1. 查询当前展示的Activity

```shell
adb shell dumpsys activity top | head -n 10
```
![perl_cn_charater1](/assets/2017-12-16_dumpactivity.png)

#### 2. 备份和还原所有安装的apk

* 找出设备上安装的第三方apk包名:	adb shell pm list packages -3
* 依据apk包名找出apk:			adb shell pm path $pkg
* 导出apk文件:				adb shell pull $path
* 把导出的apk文件安装到新设备里: adb install -r $file

脚本：

```shell
#!/bin/sh
echo -n "" > apks
adb shell pm list packages -3 | sed -E "s/\r$//" \
| while read line
do 
    pkg=${line#*:}
    line=$(adb shell pm path "$pkg"&)
    path=${line#*:}
    echo $pkg:$path >> apks
done
cat apks
cat apks | sed -E "s/\r$//" | while read line
do
    pkg=${line%:*}
    path=${line#*:}
    adb pull "$path" "$pkg.apk" 
    echo $path
done
rm apks
#echo $apks;
```

<font color="red">疑惑， 第7行通过包名查询apk安装路径时， 在命令最后添加一个＆字符， 不添加无法正常找出所有apk路径， 不知道什么原由。</font> 


#### 3. adb调试wifi模式和usb模式切换

手机端(root)

* 安装terminal
* 切wifi: setprop service.adb.tcp.port 5555
* 切wifi重启依然生效：setprop persist.adb.tcp.port 5555
* 切usb: setprop service.adb.tcp.port -1
* stop adbd
* start adbd

电脑上切换wifi

* 切wifi: adb tcpip 5555
* 切usb: adb usb

电脑端连接设备

* wifi模式: adb connect 手机ip:5555
* usb模式: 通过usb连接手机

## adb s端和adbd端，usb通信原理探索

#### 1. 识别usb设备, 找到USB设备信息

```
lsusb
```
![usb_device_info](/assets/usb_device_info.png)

记录ID，访问网站查看usb设备类型（设备是什么）

[***http://www.linux-usb.org***](https://usb-ids.gowdy.us/read/UD/)

可以确实设备的Vendors和设备类型(打印机／Mass Storage等)


#### 2. Linux内核识别设备

```
udevadm monitor --kernel
```
![usb_kernel_info](/assets/2017-12-16_usb_kernel_monitor.png)

接着去查询一下device的信息

```
udevadm info -q all -p
```
![usb_kernel_info](/assets/2017-12-16_usb_kernel_info.png)

启动adbs后，查看一下进程访问的设备文件

```
adb start-server
ps -ef | grep adb
lsof -p $pid
```
![usb_kernel_info](/assets/2017-12-16_adb_access_device_file.png)




#### 3. adb server端





