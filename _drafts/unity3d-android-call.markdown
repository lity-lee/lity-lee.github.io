---
layout: post
category: android
title:  "Android上开发unity3d各语言的互相调用"
tags: [android,unity3d,c#/java/native]
---

android平台上开发app, 使用java语言，这算是“最简单”的模式了，简单是因为只使用一种语言，不存在语言互相调用的问题。android平台上开发cocos2dx游戏，就需要两种语言了，java和c/c++; android平台上开发unity3d游戏，可能就需要三种语言了，java、c/c++、c#， 是最“复杂”的模式了。

<!-- more -->

### c#调用c/c++

c#语言运行在.net环境中，java运行在jvm中，应该有一些类似的地方，它们在访问本地代码(native code)的功能上，都是语言本身提供的功能。

c#语言：在函数定义时在前面增加“extern”和“static”关键字。

```
		public delegate void FunctionDelete(int a);
		[DllImport("paysdk_bridge", EntryPoint = "nativeMethod")]
		private static extern void externNativeMethod(int a, char b, bool c, IntPtr d, FunctionDelete e, ref int os);
```

paysdk_bridge是编译c/c++生成的库名称, 不要加前缀lib和后缀so; EntryPoint是c/c++定义的函数名。调用此函数的代码如下：

```
			int a = 100;
			char b = 'A';
			bool c = true;
			IntPtr d = Marshal.StringToHGlobalAnsi("abcdefg");
			FunctionDelete e = callFromCxx;
			externNativeMethod(a, b, c, d, e, ref a);
			Debug.Log ("a value:" + a);
```

c/c++语言：编译时保证函数不被移除或者更名，在c++环境下特别注意，在函数声明时可以这样：

```
#ifdef __cplusplus
extern "C" {
#endif
JNIEXPORT void JNICALL nativeMethod(int a, char b, bool c, void* d, void* e, int* os);
#ifdef __cplusplus
}
#endif
```

nativeMethod的实现代码如下：

```
	__android_log_print(ANDROID_LOG_VERBOSE, "native", "a: %d, b: %c, c: %d, d: %s, e: %d, os: %d", a, b, c, d, e, *os);
	*os = 200;
```

从C#传到c/c++的参数类型, 可以是基本类型(int, char, bool)、c#的引用(参数os)、委托类型(参数e)、堆指针(参数d)。这时c#的引用类型，可以理解为栈指针。

<font color="red">当然有人说可以传c#的对象过去，我不是说不可以，只是其本质传的是序列化后的内存，即上面我说的堆指针。</font>

### 2. c#调用java

C#语言本身没有提供访问java的特性，应该是Unity3d引擎封装的。猜测它的实现应该是调用c/c++， 然后c/c++通过jni接口调用java来实现的，但看不到源码，只是猜测。主要分为两种形式，第1种是通过AndroidJavaClass和AndroidJavaObject对象；第2种是通过AndroidJNI，相比我更喜欢第2种方式，它更jni一点。

```
		public static IntPtr newJavaInstance(string clazz) 
		{
			IntPtr jclazz = AndroidJNI.FindClass (clazz);
			IntPtr jConstructor = AndroidJNI.GetMethodID (jclazz, "<init>", "()V");
			IntPtr jret = AndroidJNI.NewObject (jclazz, jConstructor, new jvalue[0]);
			return jret;
		}
```

因为相对简单，且通过看unity3d的文档就能明白，就不重墨介绍啦。

### 3. c/c++调用c\#　

c#语言本身没有提供像java语言的jni，c/c++不能像jni那样调用C#。但C#可以传递一个委托到c/c++中，c/c++也就可以通过委托来调用C#了。这里可以把委托和函数划等号。

在“c#调用c/c++”部分(上面), 我已经传了一个委托到c/c++中，且


### 4. c/c++调用java

c/c++调用java是通过jni进行的, 使用JNIEn里定义的函数即可。这里一个简单的创建java对象实例的代码：

```
jobject C2DXCxxJavaObject::newJavaInstance(JNIEnv* env, const char* clazz)
{
    jclass jclazz = env->FindClass(clazz);
    jmethodID jConstructor = env->GetMethodID(jclazz, "<init>", "()V");
    return env->NewObject(jclazz, jConstructor);
}
```

### 5. java调用c\#

Java语言没有提供接口可以直接调用C#, 相应c#也没有提供接口供Java语言来调用。也可通过曲线的方式来解决这个问题，java去调用c/c++, c/c++调用C#来间接达到调用的目的，相当与是第6部分和第3部分的合并，这里不再赘述。

### 6. java调用c/c++

这也是Jni规范的一部分。主要几部分吧

* 在java类函数定义成native
* 编译成class, 使用javah生成头文件h
* 按照头文件中的函数签名写实现


### 7. java、c/c++、c\# 堆栈可访问性

||Java栈|Java堆|c/c++栈|c/c++堆|c\#栈|c\#堆|
|java访问|OK|OK|NG|NG|NG|NG|
|c/c++访问|NG|OK|OK|OK|OK|NG|
|c\#访问|NG|OK|OK|OK|OK|OK|






