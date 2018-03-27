---
layout: post
category: program
title:  "DWZ结构学习和源代码研究"
tags: [dwz,javascript, jQuery]
---

### DWZ 是什么

DWZ富客户端框架(jQuery RIA framework)，是中国人自己开发的基于jQuery实现的Ajax RIA开源框架。 DWZ富客户端框架设计目标是简单实用、扩展方便、快速开发、RIA思路、轻量级。

DWZ框架支持用HTML扩展的方式来代替JavaScript代码，只要懂HTML语法，再参考DWZ使用手册就可以做Ajax开发。开发人员不写JavaScript的情况下，也能用Ajax做项目和使用各种UI组件。 基本可以保证程序员不懂JavaScript， 也能使用各种页面组件和Ajax技术。 如果有特定需求也可以扩展DWZ做定制化开发。

做Ajax项目时需要写大量的JavaScript才能达到满意的效果，国内很多程序员javascript不熟， 大大影响了开发速度。使用DWZ框架自动绑定JavaScript效果， 不需要开发人员去关心JavaScript怎么写，只要写标准HTML就可以了。DWZ简单扩展了HTML标准，给HTML定义了一些特别的class和attribute。DWZ框架会找到当前请求结果中的那些特别的class和attribute, 并自动关联上相应的js处理事件和效果。

DWZ基于jQuery，可以非常方便的定制特定需求的UI组件，并以jQuery插件的形式发布出来，如有需要也可做定制化开发。


### DWZ 特点与优点

**






在Android平台上开发unity3d游戏，有三种不同的编程语言java, c/c++, c#，有时免不了三种语言之前的互相调用。比如现在有一个已经完成某一功能的SDK, 游戏开发者需要调用此SDK，SDK被调用后完成功能后传递结果到C#，表面上看只涉及c#调用java, java调用c#, 并没有c/c++语言什么事儿，然而好像游戏开发人员需要调用java语言写好的SDK, java
很久以前我就想学习一下汇编语言，或者写个hello world什么的，但发现很难“找齐”一个完整的开发流程，后来也是在无意中（自己多年坚持下）找到的。在这个环境下，你可以尽情地“玩耍”，像学习c/c++语言一样来快速地学习汇编，c/c++语言这样的学习环境是很容易找到的，而汇编语言的就不那么容易了。所以这篇仅于记录，共勉有同等需求的人。


* 操作系统：linux(ubuntu)
* 编辑器：sublime (nasm 插件)
* 汇编器：nasm
* 链接器：ld
* 调试工具：gdb

#### 1. c#调用cxx




linux、ld、gdb一般的linux操作系统都会自带，如果没有可以使用相应的操作系统命令来安装，在ubuntu上面使用apt-get即可。

sublime 可能相对有点复杂， 请参考[***https://www.sublimetext.com/***](https://www.sublimetext.com/)

nasm 是开源的，安装方式更多选择一点， 可参考[***http://www.nasm.us/***](http://www.nasm.us/)。

sublime nasm 插件， 具体如何使用sublime来安装插件，就不再叙述了， 这里主要截图防止安装不对的插件。

![sublime_nasm](../assets/2018-02-18_sublime_nasm.png)

#### 2. c#调用java



sublime及sublime nasm 插件安装完毕后，可创建汇编文件进行编辑，且有“智能提示”功能，如下图所示

![sublime_edit](../assets/2018-02-18_sublime_edit.png)


写好汇编程序，源代码存放在一个叫hello.asm的文件中，它是一个32位的程序，那么怎样汇编、链接、运行、调试呢？

```
nasm -f elf32 hello.asm -g
ld -m elf_i386 hello.o
./a.out
gdb a.out
```

上面的4条命令分别对应汇编、链接、运行、调试。使用gdb调试如下

![nasm_gdb](../assets/2018-02-18_nasm_gdb.png)

反汇编之后与源代码一致。

#### 3. cxx调用c# 



#### 4. cxx调用java

#### 5. java调用c# 

#### 6. java调用c# 

程序码代码

```nasm
BITS 32
section .data
		msg db "Hello, world!", 0xA
		len equ $ - msg
section .text
global _start
_start:
        mov edx, len;
		mov ecx, msg;
		mov ebx, 1;
		mov eax, 4;
		int 0x80;

		mov ebx, 0;
		mov eax, 1;
		int 0x80;
```

这里是两个系统调用，分别是向1文件中(标准输出流)写内容和调用exit来退出程序。
看这个网站就会明白为什么这样写或者还可以调用更多的linux系统调用，[***http://syscalls.kernelgrok.com/***](http://syscalls.kernelgrok.com/)。

