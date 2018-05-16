---
layout: post
category: android
title:  "Android上开发unity3d各语言的互相调用"
tags: [android,unity3d,c#/java/native]
---

android平台上开发app, 使用java语言，这算是“最简单”的模式了，简单是因为只使用一种语言，不存在语言互相调用的问题。android平台上开发cocos2dx游戏，就需要两种语言了，java和c/c++; android平台上开发unity3d游戏，可能就需要三种语言了，java、c/c++、c#， 是最“复杂”的模式了。






现就三种语言如何相互调用、堆栈如何访问、对象可否传递问题




不过支持使用native语言来开发，如cocos2dx游戏开发，这样

在Android平台上开发unity3d游戏，有三种不同的编程语言java, c/c++, c#，有时免不了三种语言之前的互相调用。比如现在有一个已经完成某一功能的SDK, 游戏开发者需要调用此SDK，SDK被调用后完成功能后传递结果到C#，表面上看只涉及c#调用java, java调用c#, 并没有c/c++语言什么事儿，然而好像游戏开发人员需要调用java语言写好的SDK, java

<!-- more -->


及堆栈访问

### c#调用c/c++

c#语言运行在.net环境中，java运行在jvm中，应该有一些类似的地方，它们在访问本地代码(native code)的功能上，都是语言本身提供的功能。

c#语言：在函数定义时在前面增加“extern”关键字，调用c/c++的函数只能是 static。

```
		[DllImport("paysdk_bridge", EntryPoint = "add")]
		private static extern int add(int a, int b);
```

paysdk_bridge是库名称, 不要加前缀lib和后缀so; EntryPoint定义的是native的函数名。

c/c++语言：编译时保证函数不被移除或者更名，在c++环境下特别注意，在函数声明时可以这样：

```
#ifdef __cplusplus
extern "C" {
#endif

int add(int a, int b);

#ifdef __cplusplus
}
#endif
```



这个应该是语言本身的功能，




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

