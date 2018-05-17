---
layout: post
category: android
title:  "Android上开发unity3d各语言的互相调用"
tags: [android,unity3d,c#/java/native]
---

android平台上开发app, 使用java语言，这算是“最简单”的模式了，简单是因为只使用一种语言，不存在语言互相调用的问题。android平台上开发cocos2dx游戏，就需要两种语言了，java和c/c++; android平台上开发unity3d游戏，可能就需要三种语言了，java、c/c++、c#， 是最“复杂”的模式了。

<!-- more -->

### 1. c#调用c/c++

c#语言运行在.net环境中，java运行在jvm中，应该有一些类似的地方，它们在访问本地代码(native code)的功能上，都是语言本身提供的功能。

c#语言：在函数定义时在前面增加“extern”和“static”关键字。

```
		public delegate void FunctionDelete(int a, char b, bool c, IntPtr d, IntPtr os);
		[DllImport("paysdk_bridge", EntryPoint = "nativeMethod")]
		private static extern void externNativeMethod(int a, char b, bool c, IntPtr d, FunctionDelete e, ref int os);
```

paysdk_bridge是编译c/c++生成的库名称, 不要加前缀lib和后缀so; EntryPoint是c/c++定义的函数名。调用此函数的代码如下：

```
			int a = 100;
			externNativeMethod(100, 'A', true, Marshal.StringToHGlobalAnsi("abcdefg"), callFromCxx, ref a);
			Debug.Log ("(log from c#)a:" + a);
```

c/c++语言：编译时保证函数不被移除或者更名，在c++环境下特别注意，在函数声明时可以这样：

```
#ifdef __cplusplus
extern "C" {
#endif
typedef void (* CallCSharp)(int a, char b, bool c, void* d, void* os);
JNIEXPORT void JNICALL nativeMethod(int a, char b, bool c, void* d, void* e, int* os);
#ifdef __cplusplus
}
#endif
```

nativeMethod的实现代码如下：

```
	__android_log_print(ANDROID_LOG_VERBOSE, "Unity", "(log from c/c++)a: %d, b: %c, c: %d, d: %s, e: %d, os: %d", a, b, c, d, e, *os);
	*os = 200;

    CallCSharp p = (CallCSharp)e;
    int cs = 100;
    p(100, 'A', false, (void *)"abcdefg", &cs);

    __android_log_print(ANDROID_LOG_VERBOSE, "Unity", "(log from c/c++)cs:%d", cs);
```

从c\#传到c/c++的参数类型, 可以是基本类型(int, char, bool)、c#的引用(参数os)、委托类型(参数e)、堆指针(参数d)。这里c#的引用类型，可以理解为栈指针。

<font color="red">当然有人说可以传c#的对象过去，我不是说不可以，只是其本质传的是序列化后的内存，即上面我说的堆指针。</font>

### 2. c#调用java

c\#语言本身没有提供访问java的特性，应该是unity3d引擎封装的。猜测它的实现应该是调用c/c++， 然后c/c++通过jni接口调用java，但看不到源码，只是猜测。主要分为两种形式，第1种是通过AndroidJavaClass和AndroidJavaObject对象；第2种是通过AndroidJNI，相比我更喜欢第2种方式(它更像jni)，下面的代码是第2种方式的。

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

c\#语言本身没有提供像java语言的jni，c/c++不能像jni那样调用c\#。但c\#可以传递一个委托到c/c++中，c/c++也就可以通过委托来调用c\#了。这里可以把委托和函数划等号。

在“c#调用c/c++”部分(上面), 已经有c/c++调用c\#的代码了。c\#传一个委托类型的函数，传到c/c++里面就是 void* 类型， 转换成函数指针然后调用。c\#函数的回调方法的实现代码。

```
		public static void callFromCxx (int a, char b, bool c, IntPtr d, IntPtr os)
		{
			Debug.Log("(log from c#)a:" + a + ", b:" + b + ", c:" + c + ", d:" + Marshal.PtrToStringAnsi(d));
			Marshal.Copy (new int[1]{ 200 }, 0, os, 1);
		}
```

这里先记录一下“c#调用c/c++”部分和这部分“c/c++调用c\#”代码里印的日志

```
05-17 19:43:23.433 31979-32022/com.mob.paysdk.demo V/Unity: (log from c/c++)a: 100, b: A, c: 1, d: abcdefg, e: -873921712, os: 100
05-17 19:43:23.448 31979-32022/com.mob.paysdk.demo I/Unity: (log from c#)a:100, b:C, c:False, d:abcdefg
05-17 19:43:23.449 31979-32022/com.mob.paysdk.demo V/Unity: (log from c/c++)cs:200
05-17 19:43:23.455 31979-32022/com.mob.paysdk.demo I/Unity: (log from c#)a:200
```

从后面两行日志可以得出：c/c++是可以访问c\#的栈空间的; 同样c\#也可以访问c/c++的栈空间。下面有一张表更清楚地描述可访问性。

### 4. c/c++调用java

c/c++调用java是通过jni进行的, 使用JNIEnv里定义的函数即可。这里一个简单的创建java对象实例的代码：

```
jobject C2DXCxxJavaObject::newJavaInstance(JNIEnv* env, const char* clazz)
{
    jclass jclazz = env->FindClass(clazz);
    jmethodID jConstructor = env->GetMethodID(jclazz, "<init>", "()V");
    return env->NewObject(jclazz, jConstructor);
}
```

### 5. java调用c\#

unity3d引擎提供了java调用c\#的接口，叫SendMessage。它是异步调用的，无法立即获取返回值，在某些应用场景下可能是不适用的(限于网站有很多例子，这里就不介绍了)。

Java语言没有提供接口可以直接调用c\#, 相应c\#也没有提供接口供Java语言来调用。可通过曲线的方式来解决这个问题，java去调用c/c++, c/c++调用c\#间接达到调用目的，相当于是“java调用c/c++”和“c/c++调用c\#”的合并，这里不再赘述。

### 6. java调用c/c++

这是Jni规范的一部分。

java语言：定义方法为native函数, 这里类名不重要，看方法前面的native关键字。

```
public class OnPayListener<T> extends Object implements com.mob.paysdk.OnPayListener<T> {
	static {
		System.loadLibrary("paysdk_bridge");
	}
	private native boolean nativeOnWillPay(String tId, T order, MobPayAPI api, long callback);
	private native void nativeOnPayEnd(int result, T order, MobPayAPI api, long callback);
}
```

编译成class, 使用javah命令生成.h头文件, 头文件内容：

```
#ifndef _Included_com_mob_paysdk_unity_OnPayListener
#define _Included_com_mob_paysdk_unity_OnPayListener
#ifdef __cplusplus
extern "C" {
#endif
JNIEXPORT jboolean JNICALL Java_com_mob_paysdk_unity_OnPayListener_nativeOnWillPay
  (JNIEnv *, jobject, jstring, jobject, jobject, jlong);

JNIEXPORT void JNICALL Java_com_mob_paysdk_unity_OnPayListener_nativeOnPayEnd
  (JNIEnv *, jobject, jint, jobject, jobject, jlong);
#ifdef __cplusplus
}
#endif
#endif
```

写cpp实现文件，最好是复制过来，注意函数签名，必须一致：

```
JNIEXPORT jboolean JNICALL Java_com_mob_paysdk_unity_OnPayListener_nativeOnWillPay
  (JNIEnv *env, jobject jthiz, jstring jticket, jobject jOrder, jobject jApi, jlong callback)
{
}

JNIEXPORT void JNICALL Java_com_mob_paysdk_unity_OnPayListener_nativeOnPayEnd
  (JNIEnv *env, jobject jthiz, jint jresult, jobject jOrder, jobject jApi, jlong  callback)
{
}
```

这样，在调用native函数时就可以调用c/c++代码了。由于java语言本身没有c/c++的指针，c\#语言的引用（ref）， 所以c/c++是无法访问java的栈空间的; java也无法访问c/c++的栈空间。由于jni规范很强劲，提供的接口可以直接访问java对象，认为c/c++可以访问java的堆；java无法直接访问c/c++的堆，当然这里不包括java的nio。下面有一张表更清楚地描述可访问性。

### 7. java、c/c++、c\# 堆栈互相可访问性

这里对三语言堆栈互相访问做一个总结：OK表示可以访问（访问指的是读取和写入）；NG表示不能访问；-- 表示同一语言。

||Java栈|Java堆|c/c++栈|c/c++堆|c\#栈|c\#堆|
|java访问|--|--|NG|NG|NG|NG|
|c/c++访问|NG|OK|--|--|OK|NG|
|c\#访问|NG|OK|OK|OK|--|--|






