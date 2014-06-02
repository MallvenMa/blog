---
layout: post
title: "JVM 之 Java 对象结构"
date: 2014-05-22 14:09:15 +0800
comments: true
categories: jvm
styles: [data-table]
keywords: jvm,对象结构,sa,hsdb
description: JVM 之 Java 对象结构 
---
##1. Java对象在内存中到底是什么样?
JAVA对象在内存中的表现形式就是一系列按照*某种规范*排列的字节(如果不清楚物理内存j的实现方式，请参考：[关于CPU寻址和物理内存结构][1] )。
那在JVM中这个规范是怎么定义的呢？
Hotspot虚拟机中，对象分为两大类：普通对象和数组对象，两类对象在内存中的布局可以分为三部分:`对象头，对象实例数据， 对齐填充` 。不同的是普通对象的对象头是由`MARK WORD，CLASS对象指针`两部分组成，而数组对象头是由`MARK WORD，CLASS对象指针，数组长度`三部分组成，如图:  
![对象结构](/images/blog/2014-05/20140524-object-arrayObject-structure.png)  

<!--more-->
####1.首先看一下对象头的具体结构(表格中的“长度“同时标明了32位和64位虚拟机):  

内容|长度|说明
:-----------:|:------------:|:---------------
  Mark Word  |  32/64bit  |  存储对象的hashCode或锁信息等。
  Class对象指针  |  32/64bit  |  存储到对象类型数据的指针
  数组长度  |  32/64bit  |  数组的长度（如果当前对象是数组）

Java对象头里的Mark Word里默认存储对象的HashCode，分代年龄和锁标记位。32位JVM的Mark Word的默认存储结构(无锁状态)如下：  

锁状态|25 bit| 4 bit|1 bit是否是偏向锁|2 bit锁标志位
:------:|:------:|:------|:-------|:------
 无锁状态|对象的hashCode|对象分代年龄|0|01

在运行期间Mark Word里存储的数据会随着锁标志位的变化而变化。Mark Word可能变化为存储以下4种数据：`轻量级锁`，`重量级锁`，`GC标记` ，`偏向锁`。本文只讨论对象结构，如果想详细了解对象锁，可以参考:[Java SE1.6中的Synchronized][2]  

####2.对象实例数据：  
实例数据包括自身的实例变量和从父类继承的实例变量，这些变量的排序顺序受虚拟机分配策略参数（FieldsAllocationStyle）和字段在源代码中定义的顺序影响。HotSpot虚拟机默认的分配策略为longs/doubles、ints、shorts/chars、bytes/booleans、oops（Ordinary Object Pointers）。从分配策略中可以看出，相同宽度的字段总是被分配到一起。在满足这个前提条件的情况下，在父类中定义的变量会出现在子类之前。如果CompactFields参数值为true（默认为true），那子类之中较窄的变量也可能会插入到父类变量的空隙之中。   

####3.对齐填充数据：  
对齐填充数据不是必须的，它仅仅起着占位的作用。HotSpot VM的自动内存管理系统要求对象起始地址必须是8字节的整数倍，换句话说就是对象的大小必须是8字节的整数倍。对象头部分正好似8字节的倍数（1倍或者2倍），因此当对象实例数据部分没有对齐的话，就需要通过对齐填充来补全。   


##2. 现在我们就通过实例来看一下JVM中的对象结构是不是和上面所述一样呢?  
说明:下面的实验我们将用到Hotspot提供的虚拟机调试工具ServiceAbility Agent。如果不太熟悉该工具可以参考:[借HSDB来探索HotSpot VM的运行时数据][3]  
环境:Ubuntu 14.04 32bit  
####1.先看一段简单代码：  
```java
public class Animal {
	public static void main(String args[]){
		Integer i = new Integer(1);
		Long l = new Long(2);
		Boolean b = new Boolean(true);
		System.out.println("");	
	}
}
```

main方法里面包含3个对象：`Integer`,`Long`,`Boolean` ，我们现在通过SA依次来看一下每一个对象的结构和大小：  
1.运行上面的程序，在`System.out.println("")`这行打个断点是虚拟机进程暂停  
2.使用JPS查看虚拟机进程PID:  
![jps截图](/images/blog/2014-05/20140524-objectsize-jps.png)	
3.启动SA(HSDB)  
`sudo java -cp $JAVA_HOME/lib/sa-jdi.jar sun.jvm.hotspot.HSDB`  
4.attach 上面的PID 3813,连上之后我们进入内存查看界面，如图:  

![内存查看](/images/blog/2014-05/20140524-hsdb-memory.png)

红色方框中的内容就是main方法中我们new的三个对象，通过右边的注释我们也可以看出来。  
三个对象在内存中的地址分别是：  
>Integer i &nbsp;&nbsp;&nbsp; 0xa0159410  
>Long l  &nbsp;&nbsp;&nbsp; 0xa0159520  
>Boolean b &nbsp;&nbsp;&nbsp;0xa0159650  

首先我们先看Integer i 对象,选择SA中的菜单:`Tool-->Inspect`子菜单,在address中输入`0xa0159410`可以得到这个对象，如图:  
![Integer对象内存结构](/images/blog/2014-05/20140524-hsdb-memory-integer.png)

按照刚才得方法，接下来我们看一下Long l 对象，如图：  
![Long对象内存结构](/images/blog/2014-05/20140524-hsdb-memory-long.png)

再看一下Boolean b 对象,如图:  
![Boolean对象内存结构](/images/blog/2014-05/20140524-hsdb-memory-boolean.png)

小结:因为对象实例数据只包括实例变量和从父类继承得实例变量，通过查看Integer,Long,Boolean 的源代码可知，这几个类只有value 一个实例变量，所以除了_mark(MARK_WORD),_metadata._klass(class对象指针)，我们只能看到一个value变量。

####2.现在我们看一个复杂点的:
```java
public class Animal {
	Integer age = new Integer(1);
	Long height = new Long(2);
	Boolean sex = new Boolean(true);
	public static void main(String args[]){
		System.out.println("");	
	}
}
```  
我们看一下Animal的对象结构,如图:  
![Animal对象内存结构](/images/blog/2014-05/20140524-hsdb-memory-animal.png)

由图可以看出，三个实例变量都在对象Animal里面了。

####3.我们再看一个继承的:
```java
public class Tiger extends Animal{
	public Double weight;
	public static void main(String args[]){
		Tiger tiger = new Tiger();
		System.out.println("");	
	}
}
```
用HSDB查看结果，如图:  
![Tiger对象内存结构](/images/blog/2014-05/20140524-hsdb-memory-tiger.png)

由图可见，从父类继承的变量也是存在子类里面的。

####4.最后看一个数组的:
```java
public class Tiger extends Animal{
	public Double wight;
	public Integer[] childs = new Integer[0];
	public static void main(String args[]){
		Tiger tiger = new Tiger();
		System.out.println("");	
	}
}
```
用HSDB查看,结果如图:  
![数组对象内存结构](/images/blog/2014-05/20140524-hsdb-memory-array.png)

不幸的事情出现了，为什么这个数组对象没有显示length变量呢？难道我们之前说的数组对象的结构式错的？  
用inspect 查看一下这个对象的size:  
`inspect 0xa01ba470` 结果如图所示：  
![数组对象size](/images/blog/2014-05/20140524-hsdb-memory-array-size.png)

我们来算一下:  
1. 如果这个对象没有length变量，那么大小应该是：size=4+4=8.  
2. 如果这个对象有length，那么大小应该是：size=4+4+4+4(补全)=16,和刚才截图中的一样。所以这个length肯定是有的，至于为什么没显示出来，我觉得有可能是这个SA插件得bug。  



总结：通过SA我们可以查看JVM内部的实时数据，这对我们学习JVM是一个很有利的工具，了解对象的结构是万里长征迈出的第一步。  
参考文章:  
[http://rednaxelafx.iteye.com/blog/1847971][3]  
[http://ifeve.com/java-synchronized/][2]  
[http://icyfenix.iteye.com/blog/1145044](http://icyfenix.iteye.com/blog/1145044)


[1]: http://blog.zarue.com/blog/2014/05/21/cpu-and-memory/
[2]: http://ifeve.com/java-synchronized/
[3]: http://rednaxelafx.iteye.com/blog/1847971
