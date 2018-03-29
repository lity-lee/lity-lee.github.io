---
layout: post
category: linux
title:  "gdb远程调试的一般步骤"
tags: [gdb,调试,编程]
---

gdb远程调试，主要用手机或者嵌入式设备。gdbserver与gdb平台和版本要对应，否则无法通信。

<!-- more -->

### 在远程主机上，用gdbserver开启调试服务

```
gdbserver :port debug_process）
```

port为端口号; debug_process是需要调试的二进制程序(带调试信息的)

### 在调试机(通常是pc)开启gdb客户端

```
gdb或者arm-linux-androideabi-gdb.exe
```

### 加载符号symbols

```
file bin或者so
```

bin是带调试信息生成的二进制文件,　即gcc里加-g。

so用于调试动态链接库时用,　gcc时也需加-g。

### 连接到gdbserver服务机

```
target remote ip:port
```

ip为远程主机,　手机或者嵌入式设备的ip; port 为gdbserver开启时的端口号。
