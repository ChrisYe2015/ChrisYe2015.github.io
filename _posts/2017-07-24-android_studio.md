---
layout: post
title: Android Studio 断点调试
categories: Android
description: Android Studio 断点调试
keywords: Andorid, Android Studio,断点调试
---
原文地址：http://blog.csdn.net/wzy_1988/article/details/51778755

Android Studio包含一个debugger程序，可以在模拟器和真机上调试android应用。比如设置断点，在运行时去检查变量和表达式的值。

## 设置断点
设置断点的方法：单机左侧需要调试的代码所处的侧边栏：
![](/images/posts/android/android_断点调试.png)

## 开始调试
点击红色箭头指向的按钮,即可进行代码调试,如下图所示: 
![](/images/posts/android/android_断点调试1.jpeg)
调试界面如下所示: 
![](/images/posts/android/android_断点调试2.jpeg)

## 单步调试
Step Over(F6)
代表程序直接执行当前行代码.(ps:如果该行是函数调用,则直接执行完函数的全部代码) 
![](/images/posts/android/android_断点调试3.jpeg)

Step Into(F5)
代表程序执行当前行代码(ps:如果该行有自定义方法,则运行进入自定义方法,不会进入官方类库的方法)
![](/images/posts/android/android_断点调试4.jpeg)

Step Out(F7)
跳出Step Into进入的方法.例如我们感觉进入的方法没有问题,不需要执行后续代码,就可以通过Step Out跳出当前进入的代码.
![](/images/posts/android/android_断点调试5.jpeg)
