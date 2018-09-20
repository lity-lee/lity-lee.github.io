---
layout: post
category: mlai
title:  "特别的python开发环境"
tags: [cygwin,python,pycharm,pycrypto‎]
---

windows开发python, 基础环境的安装很简单，大概有两种方式。1、可以直接从[官方](https://www.python.org)下载, 安装即可；2、在cygwin终端里使用apt-cyg安装。然而这两种方案差异性还是挺大的，主要是都不“完美”。方案1在使用需要编译的模块时（比如：pycrypto‎），需要c++编译环境, 而windows编译器需要安装VS, VS太大了且我在安装界面也没找到编译工具；方案2编译倒没有问题，因为cygwin环境下安装gcc是很容易的，但IDE断点调试却成问题。基于上面2种方案取优补短介绍一种“更完美”的开发环境。

<!-- more -->

这套开发环境所需软件预览

1. cygwin + python
2. pycharm ide
3. pydevd remote debug

<p><font color="red">如果您仅仅是学习一个python语法，并不使用cygwin, 这套开发环境并不适合，会显得大材小用了，为了它你还得装一个cygwin环境。</font></p>

如果您是一个资深PC工程师或者电脑上安装过Visual Studio, 这套开发环境也不适合，你可以build需要编译的python模块。

### cygwin + python 

从上面那两句可以看到，使用cygwin + python 代替 windows + python的原因是编译问题。有一次我需要编写一个aes加解密的脚本，需要pycrypto‎模块，使用cygwin + python是很容易安装pycrypto‎模块‎的。

```
pip3 install pycrypto -v
Successfully installed pycrypto-2.6.1
```

加上"-v"参数，可以看到gcc的编译过程。windows + python 因为缺少编译器而无法安装

```
    running build_ext
    warning: GMP or MPIR library not found; Not building Crypto.PublicKey._fastmath.
    building 'Crypto.Random.OSRNG.winrandom' extension
    error: Microsoft Visual C++ 14.0 is required. Get it with "Microsoft Visual
C++ Build Tools": http://landinghub.visualstudio.com/visual-cpp-build-tools

```

### pycharm IDE

pycharm确实是一个很不错的python开发环境，有智能提示、断点调试，这些是我比较在意的。安装pycharm需要配置python解释器的路径，配置成cygwin里python的路径即可，你可以通过下面的命令找到python在windows上的绝对位置。

```
cygpath -aw $(which python3)
```

在pycharm中配置python选项：file->settings->Project Interpreter, 在里面填写即可，如下面：

![pycharm_settings]({{ site.url }}/assets/2018-08-24_pycharm_settings.png)

pycharm + python(cygwin版本)在编写python脚本没有问题，运行也没有问题，就是不能断点调试，调试时提示这个错误：

```
pydev debugger: warning: trying to add breakpoint to file that does not exist:  (will have no effect)
```

在网上有人说，可以解决这个问题，通过修改pydevd有代码。我试用过是可以解决，但好像仅仅是不提示这个错误了，断点依然无法暂停程序。

另外想说两点的是，pycharm + python(windows版本) 不存在这个问题，断点可以暂停程序；python本身是支持断点调试的，缺点是命令行式的调试，不方便。

```
$ python3 -m pdb /home/litl/shbin/mobrsa
> /home/litl/shbin/mobrsa(10)<module>()
-> import math
(Pdb)
```


### pydevd remote debug

pydevd是远程调试工具，在这里我并不是用来远程调试的，只是本地使用“远程”调试方式。既然是远程调试，就分为客户端和用户端。有的语言(可能还区别调试工具)ide是客户端，运行程序是服务器，像jdb，gdb, lldb。而有的语言ide是服户端，运行程序是客户端，像php 的xdebug。其实两者区别并不是很大，值得注意的是谁是服务端，先启动谁。pydevd就属于后者。

安装python模块pydevd
```
pip3 install pydevd -v
```


pycharm 作为ide, 在远程调试中扮演服务器角色，创建远程调试配置，如下图：

![pycharm_remote_debug]({{ site.url }}/assets/2018-08-24_pycharm_remote_debug.png)

从上面的图上也可以看出来，需要在调试的脚本里加入一段代码(最好在开头位置吧)：

```
import pydevd
pydevd.settrace('localhost', port=6565, stdoutToServer=True, stderrToServer=True)
```

经常使用c-s模式，应该能理解上面的这段代码。剩余的就是先点击pycharm ide里的debug按钮，控制台会输出类似这样的信息：

```
Starting debug server at port 6565
Use the following code to connect to the debugger:
import pydevd
pydevd.settrace('localhost', port=6565, stdoutToServer=True, stderrToServer=True)
Waiting for process connection...
Connected to pydev debugger (build 182.4129.34)
```

然后在cygwin终端执行脚本：

```
python3 ~/shbin/mobrsa
```

调试代码界面如下：

![pycharm_debugging]({{ site.url }}/assets/2018-08-24_pycharm_debugging.png)

<font color="red">这里特别注意，我也是在浪费了几个小时才发现的，pycharm免费版本(Community)是没有远程调试功能的，只有专业版(Professional)注册后才有。</font>