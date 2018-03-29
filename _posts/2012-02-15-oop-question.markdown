---
layout: post
category: program
title:  "面向对象的三个疑惑"
tags: [编程,java, oop]
---

<!-- more -->

### 面向对象的三个小问题

 1. 继承于抽象类的非抽象类必须实现抽象类中定义的抽象方法吗？

 1. 父类里的私有属性，在子类对象里存不存在？对象对应于内存中的一片空间，那么子类对象的内存空间中有没有保存父类中声明的私有属性的值？

 1. 父类中声明了一个保护方法a()和一个私有方法b(),a()方法会调用b()方法；子类有一公用方法c(),c()方法会调用a()方法，若创建
一个子类对象,调用其c()方法会不会报错？

问题1答案：如下，自己得结论吧，我较赞同不一定。
在包com.lity.java.apis.abstracttest中定义一个类，如：

```
public abstract class Animal {
	 abstract void run();
	 public void crazyRun() {
		  run();
	 }
}
```
run()方法声明成包内的，并把此包打包成jar文件。打包的jar构建到项目中去，并写一个Dog类继承于Animal类：

```
public class Dog extends Animal {
    public static void main(String[] args) {
        Dog dog = new Dog();
        dog.crazyRun();
    }
}
```

这时eclipse并不提示你去实现run()方法，但当你运行此程序时会抛出错误，原因是找不到run()方法，就添加一个run()方法。
根据我们一般的思路这似乎就是重写，而你在run()方法上面加上“@Override"，就会出现编译错误；若把run()方法的访问权限改为private也可以正常运行。

问题2答案：存在。不继承并不是没有。

问题3答案：如下，此程序符合面向对象语法、没有编译错误、没有运行错误。父类私有的方法，子类无法直接访问，但可间接执行。

```
package com.lity.java.apis.pri;
class Animal extends Object {
	private void internal() {
	System.out.println("run away");
	}
	protected void run() {
		internal();
	}
}
class Dog extends Animal {
	public void crazyRun() {
		run();
	}
}
public class Method {
	public static void main(String[] args) {
		Dog dog = new Dog();
		dog.crazyRun();
	}
}
```
