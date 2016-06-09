---
layout: post
category: "android"
title:  "Android中的一些小问题 "
tags: [android,编程]
---

1. LocalActivityManager，销毁子activity时的bug
    ActivityGroup，是用来管理多个Activity的：可以通过LocalActivityManager.startActivity()创建新的Activity；通过LocalActivityManager.destroyActivity()来销毁Activity， 然有时用LocalActivityManager跳转画面，界面会出现莫名其妙的错误，很有可能是这个bug引起的。此bug在api-10之前版本都存在(api-15此bug已经不存在了, 中间版本我未找到源码不确定)。
 在api-8中错误代码是LocalActivityManager.java的383行(Map移除是通过key, 而不是value)。

1. ImageButton在ListItem中，ListView的onItemClickListener事件监听不到
    一般遇到这种情况，都是先把ListItem中有可能获得焦点的View(如Button)设置android:focusable="false",就可以监听到了。
 然如果ListItem中有ImageButton，这样设置也是监听不到。原因是ImageButton的构造方法中增加了setFocusable(true);这行代码。
 
1. Gallery中滚动条滚动时无法显示出来
    原因是：Gallery.onTouchEvent()方法中用GestureDetector类来对触摸事件进行处理, 未调用super.onTouchEvent(), 主要是未调用awakenScrollBars()。

1. View的滚动条
    Android中我们经常看到ScrollView的滚动条、ListView的滚动条、GridView的滚动条，其实Android控件从View开始就已经有了水平滚动条和竖直滚动条。 然一般情况下，怎么设置一个View都无法让其显示出滚动条来。原因是我们需要重写六个方法，分别是：水平相关的
 computeHorizontalScrollExtent()、computeHorizontalScrollOffset()、computeHorizontalScrollRange()，
 竖直相关的computeVerticalScrollExtent()、computeVerticalScrollOffset()、computeVerticalScrollRange()，只有当这些方法的返回值满足一定条件时滚动条才会显示出来。
    例如：computeHorizontalScrollRange()返回值只有大于View.getWidth()时水平滚动条才会显示出来。
