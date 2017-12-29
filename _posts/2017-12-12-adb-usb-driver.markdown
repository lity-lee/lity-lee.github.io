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
![dumpactivity](../assets/2017-12-16_dumpactivity.png)

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

## adbs端和adbd端，usb通信原理探索

#### 1. 识别usb设备, 找到USB设备信息

```
lsusb
```
![usb_device_info](../assets/2017-12-12_usb_device_info.png)

记录ID，访问网站查看usb设备类型（设备是什么）

[***http://www.linux-usb.org***](https://usb-ids.gowdy.us/read/UD/)

可以确实设备的Vendors和设备类型(打印机／Mass Storage等)


#### 2. Linux内核识别设备

```
udevadm monitor --kernel
```
![usb_kernel_monitor](../assets/2017-12-16_usb_kernel_monitor.png)

接着去查询一下device的信息

```
udevadm info -q all -p
```
![usb_kernel_info](../assets/2017-12-16_usb_kernel_info.png)

#### 3. adbs 访问的device文件

启动adbs，查看一下进程访问的device文件

```
adb start-server
ps -ef | grep adb
lsof -p $pid
```
![adb_access_device_file](../assets/2017-12-16_adb_access_device_file.png)


#### 4. adbs源代码分析

调用栈

```
main(adb.c)
main_adb(adb.c)
[
    usb_vendors_init(usb_vendors.c)
    usb_init(usb_linux.c)
]
-----new thread----
device_poll_thread(usb_linux.c)
find_usb_device(usb_linux.c)
kick_disconnected_devices(usb_linux.c)

```
从usb_vendors.c文件中，可以知道vendor信息是被“固化”adbs里面。（<font color="red">那是否就可以解释为啥linux不需要adb驱动呢）</font>

![adbs_init_vendors](../assets/2017-12-16_adbs_init_vendors.png)

usb_linux.c文件的函数列表

![adbs_usb_linux_functions](../assets/2017-12-16_adbs_usb_linux_functions.png)

里面读取函数，主要封装linux usb 通用的访问device的方式。

#### 5. adbd 访问的device文件

```
adb shell
su (root权限)
lsof > /sdcard/lsof.data
adb pull /sdcard/lsof.data
less lsof.data
```

![adbd_access_file](../assets/2017-12-16_adbd_access_file.png)

#### 6. adbd源代码分析

调用栈

```
main(adb.c)
main_adb(adb.c)
usb_init(usb_linux_client.c)
usb_adb_init(usb_linux_client.c)
```

usb_adb_init的内容, 可以确定adbd确实访问了<font color="red">/dev/android_adb</font>文件（设备结点） 
![adbd_usb_adb_init](../assets/2017-12-16_adbd_usb_adb_init.png)

usb_linux_client.c文件定义读取函数，从实现上看它主要通过/dev/android_adb文件与外界通信。

![adbd_usb_linux_client_function](../assets/2017-12-16_adbd_usb_linux_client_function.png)

#### 7. 串联adbs和adbd(android kernel)

```
init(android.c)
usb_composite_register[&android_usb_driver](android.c)
android_bind(android.c)
usb_add_config[cdev, &android_config_driver](android.c)
android_bind_config(android.c)
adb_function_add(f_adb.c)
misc_deregister[&adb_device](f_adb.c)

static struct miscdevice adb_device = {
  	.minor = MISC_DYNAMIC_MINOR,
  	.name = shortname,
  	.fops = &adb_fops,
};
static const char shortname[] = "android_adb";

```

android.c和f_adb.c代码所在路径是/drivers/usb/gadget/
<font color="red">注意：这里的代码是kernel的源代码，不是Android的源代码（aosp）。如果你也下载了linux kernel, 会发现没有这两个文件的。</font>




