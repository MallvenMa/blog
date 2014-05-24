---
layout: post
title: "JVM 之 Java Object Size"
date: 2014-05-22 14:09:15 +0800
comments: true
categories: jvm
styles: [data-table] 
---
####1. Java对象在内存中到底是什么样?
JAVA对象在内存中的表现形式就是一系列按照*某种规范*排列的字节(如果不清楚物理内存j的实现方式，请参考：[关于CPU寻址和物理内存结构][1] )。
那在JVM中这个规范是怎么定义的呢？
Hotspot虚拟机中，对象分为两大类：普通对象和数组对象，两类对象在内存中的布局可以分为三部分:`对象头，对象实例数据， 对齐填充` 。不同的是普通对象的对象头是由`MARK WORD，CLASS对象指针`两部分组成，而数组对象头是由`MARK WORD，CLASS对象指针，数组长度`三部分组成，如图:  
![对象结构](/images/blog/2014-05/20140524-object-arrayObject-structure.png)  
1. 首先看一下对象头的具体结构  

内容1111111|长度|说明
:-----------:|:-----------:|:---------------
ss|ss|ss 












[1]:http://blog.zarue.com/blog/2014/05/21/cpu-and-memory/

