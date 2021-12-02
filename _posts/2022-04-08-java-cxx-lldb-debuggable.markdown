---
layout: post
category: android
title:  "Android sdk调试及动态加载调试"
tags: [java,c++,c,gdb,lldb]
---



代码之所以可以断点调试，是编译后源代码信息被保留下来或者未被擦除， 一般称这些信息是调试信息。 java代码编译后会携带调试信息一起加载的虚拟机里。cxx代码略有不同，它运行的库可以不携带调试信息，debugger从调试信息未被擦除的库读取调试信息并与运行库关联起来。当然, 理论上cxx也可以携带调试信息运行进行调试，但androidstudio可能没有这么做。


<!-- more -->

现在的Android开发，基本上都要借第三方的类库。开源的类库好一点，一般都提供源代码供查阅或者调试； 商业的就不太可能提供源代码了，并且release版本的sdk基本上都会擦除调试信息，所以调试sdk代码相比调试应用的代码也是麻烦许多。作为sdk的开发者，在帮助客户(开发者)解决问题的过程中，除了一般的日志输出的方式来定位问题，也可以尝试使用断点调试的方式来定位，但这并不是屡试不爽的方法，特别是调试cxx代码时，可能连附加进程都attach失败，更有些应用还会加入反调试代码。


##### 1. 携带调试信息的编译


```
    buildTypes {
        debug {
            minifyEnabled false
            externalNativeBuild {
                ndkBuild {
                    arguments 'V=1', "NDK_DEBUG=1"
                }
            }
        }
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
            externalNativeBuild {
                ndkBuild {
                    arguments 'V=1', "NDK_DEBUG=0"
                }
            }
        }
    }
```

执行gradle命令时，执行 ./gradlew assembleDebug , 就可以了。如果不是通用的gradle配置，比如调的是ndk-build命令，也是一样的加NDK_DEBUG=1参数。


```
    commandLine getNdkBuildCmd(), '-e','WEBP_SUPPORT=true', "V=1", "NDK_DEBUG=1"
```


##### 2. 验证是否携带调试信息(java)

借用apktool反编译jar或者apk, 查看smali字节码是否携带调试信息。通过第一步编译出来的基本上都可以调试，但如果不是自己打的包或者调试中遇到很奇怪的问题，可以通过此方法验证调试信息是否正常。像这样的是携带的。


<img src="../assets/2022-04-08_java_debuggable.png" style="width: 100%;display: block; height: auto;" />


其中， line代表源代码的行号，这应该是一行一行调试的原因； param行记录参数的名称、类型及对应的寄存器。像下面这样的就是没有携带调试信息。


<img src="../assets/2022-04-08_java_non_debuggable.png" style="width: 100%; height: auto;" />



##### 3. 验证是否携带调试信息(cxx)

前面提到过cxx运行的库没的携带调试信息， 所以通过运行的库是否携带调试信息来判断是否可正常调试是不正确的。
查看一个动态库是否包含调试信息的命令很多，有 file/nm/objdump/readelf ， 如何使用可以查看help或者百度。后面会介绍通过调试器是否正确加载调试符号来判断是否可以正常调试。


##### 4. 字节码调试(java汇编级调试)

如果一个jar或者apk里没有携带调试信息，像2中的第二张图那样，真的就不能断点调试了吗？答案是可以调试。需要借助一个androidstudio插件，smalidea, 下载地址: [https://bitbucket.org/JesusFreke/smali/downloads/](https://bitbucket.org/JesusFreke/smali/downloads/) 。插件安装后，用AndroidStudio导入需要调试的apk， 如下图：


<img src="../assets/2022-04-08_profile_debug_apk.png" style="width: 100%; height: auto;" />

这种调试是基于java汇编级别的，不同于源代码调试，查看变量值需要查看变量对应寄存器。且此插件不是官方的，其它更新迭代速度跟不上AndroidStudio, 到2022年4月9号仅支持AndroidStudio 4.0及以下版本。


##### 5. 附加调试(attach)

调试的设备是具有root权限的模拟器或者root过的手机，理论上可以调试任何程序。附加进程时勾选“Show all processes”, 选择需要调试的进程即可。


<img src="../assets/2022-04-08_root_debug_all.png" style="width: 100%; height: auto;" />


非root的设备，做一个空的android application项目, 把包名指定成和需要调试的应用包名一样。应用一般都是不可调试的，需要改其AndroidManifest.xml文件， 在 application 节点加入 android:debuggable="true"。

```
<application android:debuggable="true" />
```

修改工具就是apktool。 有的应用做了防apktoo处理，使用时直接报错，因为这时的目的是改资源而不是改代码，可以加 -s 选项不反编译代码。加了选项还是报错了，确实也啥好方法了，去看apktool的源代码吧或许能找方法解决方法。

```
apktool d  -s
```

<img src="../assets/2022-04-08_nonroot_debug.png" style="width: 100%; height: auto;" />


##### 6. 组织代码

AndriodStudio创建一个模块modulue也很方便，稍微大一点的项目基本都要模块化。把所有模块放一个AS窗口下，可以方便阅读调试整个sdk的所有代码。上面第5步，非root设备需要做一个空的应用，这时其它可以所sdk的所有modulue都“link"过来。

```
include ':app', ':slice-app',
        ':modulue1',
        ':modulue2', 
        ':modulue3', 

project(":modulue1").projectDir = new File("/data/projects/sdk/modulue1")
project(":modulue2").projectDir = new File("/data/projects/sdk/modulue2")
project(":modulue3").projectDir = new File("../sdk/modulue3")

```

##### 7. 设置断点和时机

在第6步已经把所有sdk代码放在了一个AS窗口里，所以在需要调试的代码行放置断点即可。因为此时不是通过AS工具栏的debug启动调试的，而是在设备桌面点了应用的图标启动的，再去attach进程，可能需要调试的代码已经执行过去了。可以通过am来启动

```
adb shell am start -D -W com.stickgame.los2/com.DBGame.DiabloLOL.DiabloLOL
```

-D 选项是调试启动， -W选项是启动后暂停直到attach上之后代码继续执行，后面的参数是应用的launcher，可以在AndroidManifest.xml里找到。还有另外的方法，在代码里加入

```
android.os.Debug.waitForDebugger();
```

代码执行到此方法时，也会暂停直到attach上之后，函数才会返回。所以断点代码行执行时机应该在这行代码之后，就可以正常调试了。

Android系统调用过来的函数，有的做了耗时处理，超过一定时间就中止进程执行，比如 Service.onStartCommand 函数。




编译带有调试信息的java，




Android开发中Java代码和Native代码调试

Java代码调试






可调试的代码和非可调试

在客户端到服务端的传输中，需要实现一套安全有效的加密方式。这里在别人的基础了介绍一套解决方案。原来是用java实现的，现在改用python实现，主要是验证目的，可以在不调试代码的情况下通过抓包来解密数据，便于快速定位问题。https在安全性上没有问题，理论上是无法破解的，但在获取证书环节，有可能被做“手脚”，如中间人攻击。https相对http效率不高，所以我们基于http实现一个安全的加密方式。之所以用python实现，是因为python是一个高层次的结合了解释性、编译性、互动性和面向对象的脚本语言，且类库非常丰富。



AndroidStudio断点调试动态加载应用
java代码调试
可调试java代码是jvm运行的是带调试信息的字节码

1. 带调试信息的字节码, 一般assembleDebug编译出来的就是。确认代码是否可以调试


2. 如果编译出来的没有调试信息的字节码是否可调试(常用第三方的sdk)呢? 
可以. 

AS里文件菜单里有"Profile or Debug APK" , 把需要调试的APK导入

3. root环境下和非root环境下调试
非root环境需要修改AndroidManifest.xml 


native调试，cxx代码可以调试并


