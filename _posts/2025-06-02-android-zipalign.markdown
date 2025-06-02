---
layout: post
category: android
title:  "zipalign 超 2G windows 上无法对齐问题"
tags: [android，c++，zipalign，windows]
---

android apk 文件上线或者安装到手机前，需要对文件进行签名， 现在主流 v2 签名方式需要先对 apk 文件进行对齐。 sdk 中提供的 zipalign 工具就是用来对齐 apk 的。windows 平台下的 zipalign 工具默认只能对齐小于 2G 的文件，超过 2G 进行将会出错， 报这个的错误，“Unable to open 'xxx.apk' as zip archive”， 此文章就是提供解决此问题的方法。

<!-- more -->

咋一看错误好像是文件不是合法的 zip 格式，但通过其它工具 file，zipinfo 等可以检查到 zip 格式是正常的。zip 格式的中内目录信息和 EOCD 都位于文件结尾处，且超过 2G 就有问题，unsigned int 最大值确实只能表示到 2G，所以应该是类型表示不足导致的。

通过分析 aosp 中相关源代码，位于 build/make/tools/zipalign/ ，可以很快地发现 fseek/ftell 函数在 windows 平台上返回值是 long 类型的，long 类型是 4 个字节，找出其替代函数 fseeko/ftello ，返回值是8个字节。

tools/zipalign/ZipEntry.cpp 文件改动如下:

<img src="../assets/2025-06-02_zipentry.png" style="width: 100%; height: auto;" />

tools/zipalign/ZipFile.cpp 文件改动如下:

<img src="../assets/2025-06-02_zipfile.png" style="width: 100%; height: auto;" />

改动之后需要重新编译 zipalign ，编译后的就会支持超 2G 文件的对齐。

zipalign 的编译虽然需要的是 windows 平台的，但并不是在 windows 平台上编译，具体编译文档，参考官方的吧， 位于 sdk/docs/howto_build_SDK.txt。 以下是官方链接

```
https://cs.android.com/android/platform/superproject/main/+/main:sdk/docs/howto_build_SDK.txt
```

当然编译sdk，需要下载 aosp 源代码，这个过程还是挺繁琐的，仅仅是想使用编译好的而省去编译过程，可以从这个链接处下载，这是我编译的，经过对比测试与 linux 平台上 sdk 中的 zipalign 对齐的结果完全一致。

```
https://gitee.com/litylee/superslim/releases/download/1.0.0/zipalign.exe
```

另外，如果 apk 文件超过 4G 也会报这个错误“Unable to open 'xxx.apk' as zip archive”， 有时也遇到低于 4G 甚至低 2G 的文件也会报这个错，不超过4G大多是 entry 条目超过了 65535，这时去使用 linux 平台下的 zipalign 也会报这个错误，通常是因为这个文件是 zip64 格式而非 zip 格式。虽然 android 官方平台没有明确指出 apk 必须是 zip 格式(非 zip64)，但 zipalign 确实不支持 zip64 格式，暂时也未发现安装到 android 设备上的 apk 有超过4G 或者是 zip 64 格式的。
