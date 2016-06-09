---
layout: post
category: program
title:  "no implementation found in native"
tags: [gcc,编译,编程]
---

### 一个"no implementation found in native"问题的调查过程
1. 前要(一个java类RemoteDesktopJni, 功能实现在c/cxx层)

    1.1 项目里有一个Java类, 但其功都是用c/cxx代码实现的, 即native方法调用。

    1.2 java native链接到本地函数名, 使用的是RegisterNatives()方法。

1. 起因(proguard 删除未使用的方法)

    2.1 基于保护代码考虑, 要把java代码使用proguard做混淆处理.

    2.2 proguard有删除无用方法和类的功能, 如果一些native 方法未直接/间接使用，将从类中删除

    2.3 若真有native方法删除, 将导致RegisterNatives()中止, 原因找不到对应的java方法。

    2.4 修改java native链接到本地函数方式，不使用RegisterNatives()方法, 使用javah式的规范。

1. no implementation found in native错误出现

    3.1 javah 生成头文件*.h, 复制*.h > *.cpp, 删除不必要define, 把之前的实现代码copy到对应新方法中.

    3.2 ndk-build生成libremotedesktop.so, 运行安装, 发现当执行某个native方法(nativeSetColor)时, 崩溃抛出错误

    3.3 对比*.h、*.cpp方法签名, 并未发现有不一致的现象(结果证实有问题的)

    3.4 还发现此类的另一个native方法(nativeCreateCxxObject)方法已正常执行过

    3.5 基于2.3、2.4、2.5现象, 感到震惊、奇怪, Why, 思考中...

    3.6 突然想到, 之前链接库时经常遇到xxxx方法找不到(库和头文件不对应时), 常用命令nm, objdump来检查so中是否真有那个方法。

    3.7 objdump -T libremotedesktop.so | grep nativeSetColor, 输出

            00070ad1 g    DF .text  0000007c _Z80Java_com_oray_sunlogin_plugin_remotedesktop_RemoteDesktopJni_nativeSetColorP7_JNIEnvP8_jobjectb

    3.8 方法也存在, 感觉没错啊(有问题, 方法名与你定义的不一样)

    3.9 Google "no implementation found", 发现有人说, jni方法的声明要放在 extern "C" {}; 方法内

    3.10 检查*.h文件, 发现确实声明的方法在extern "C" {};中。

    3.11 没有思路了, 不过不太明白Google为什么有人说jni方法声明一定要话extern "C" {}中, 所以去Google extern。

    3.12  简单说一下, cxx中extern "C"声明中的方法, 编译器不对方法做任何处理；cxx 编译器为了解决cxx方法多态性和重载, 不得不在编译时在方法名前后做一些必要的特殊处理, 像3.7那样.

    3.13 回头看看3.7方法名确实不对, 可我也做了extern "C"的处理了(javah 做的), 摇头思考中...

    3.14 突然想起3.4, 为什么native方法nativeCreateCxxObject可以正常找到, 执行objdump -T libremotedesktop.so | grep nativeCreateCxxObject,

    3.15 00071b95 g    DF .text  0000008c Java_com_oray_sunlogin_plugin_remotedesktop_RemoteDesktopJni_nativeCreateCxxObject, 方法没被cxx编译做任何处理

    3.16 综合3.7、3.15 就是同样的在extern "C" 中声明的方法, 一个被编译器按cxx 方式处理了(不正常), 一个没按cxx方式处理

    3.17 仔细对比两个方法nativeSetColor和nativeCreateCxxObject, 发现nativeSetColor定义时有一个bool类型变量, 而声明时是jboolean。

    3.18 转到jboolean的定义, typedef unsigned char   jboolean;, 原来如此。

4. 找出原因(bool, jboolean不是同一类型, 与jint和int不一样)

    4.1 把nativeSetColor方法定义中的bool 换成jboolean, 重新编译、运行, 问题消失

    4.2 确定问题是头文件*h 和 *.cpp 声明和定义的方法签名不一致

1. 回顾(方法找不到, 检测签名, 为什么转了一大圈才得出结果)
    5.1 从1.2到2.4, 复制参数类型, 就认为复制的东西不会有问题(复制前是正常的)
    5.2 想当然的认为jboolean和bool是一样的, 就像jint和int那样
    5.3 5.1和5.2加起来就导致了3.3检查大打折扣。
