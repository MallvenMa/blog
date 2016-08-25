---
layout: post
title: "Layout of Java Object"
date: 2014-05-22 14:09:15 +0800
comments: true
categories: jvm
styles: [data-table]
keywords: jvm,layout,java,object,sa,hsdb
description: Structure of Java Object
---
## 1. What do a Java object looks like in the memory?
Java Object represents as a series of bytes sorted by some  *specific rules*(If you do not familiar with the physical memory structure, please refer:[CPU Addressing and Physical Memory Structure][1]).  
But what is the definition of these *specific rules* in JVM？  
In hotspot，there are two types of object：normal object and array，the memory layout of each object consists of three parts: `object head, fields, padding`, normal object's header contains `MARK WORD, CLASS POINTER`, but array's object header contains `MARK WORD, CLASS POINTER, ARRAY LENGTH`. (Figure 1)   
![Object Layout](/images/blog/2014-05/20140524-object-arrayObject-structure.png)  

<!--more-->
#### 1.This is a table about the details of object layout
title|length|description
:-----------:|:------------:|:---------------
  Mark Word  |  32/64bit  | used to store object's hashcode and lock type
  Class Pointer |  32/64bit  |  point to the class of this object 
  Array length  |  32/64bit  |  length of array（only exists in array object）


This is a table of the details of a 32-bit JVM's  `MARK WORD` with none lock,it contains `HashCode`,`Object Generation`,`Lock Type`.    


Lock Status|25 bit| 4 bit|1 bit(biased lock)|2 bit(lock type)
:------:|:------:|:------|:-------|:------
 no lock|hashCode|object generation|0|01


Datas represented by `MARK WORD` will be different if `Lock Status` is different, `MARK WORD` can change to represent four type datas:`Lightweight Lock`, `Heavyweight Lock`, `GC Mrak`, `Biased Lock`. This article only talk about object layout, If you want to know more about `Object Lock`, please refer to [Java SE1.6 ynchronized][2]  

#### 2.Instance Data：  
Instance data includes instance variables of  itself  and instance variables extend from its parents.  the order of these variables are effected by jvm `FieldsAllocationStyle` and their orders in source code.  The default allocation strategy in HotSpot is longs/doubles, ints, shorts/chars, bytes/booleans, oops(Orginary Object Pointers). From these  allocation stragegies we can find that those fields has same width will be allocated  together. With this precondation, fields extends from parents will display before child's  autogeneic variables, but if variable `Compactfields` is set to true(defalut is true), child's variables with narrow width will be insert into parents' variable's gap.   

#### 3.Padding Data：  
Padding data is not necessary, They are only placeholders. HotSpot's `Automatic Memory Management System` requires the object's size must integral multiple of 8 bytes, object header is exactly integral multiple of 8 bytes(1 times or 2 times), So it will need padding data if instance data could not align as 8 bytes.  

## 2. Observe object layout in run-time   
Note:In this experiment we will use  `ServiceAbility Agent`(a debug tool provide by HostSpot) to observe object layout in run-time. If you don't familiar with it,please refer to :[Use HSDB to explore HotSpot VM's run-time data][3]  
Environment:Ubuntu 14.04 32bit  
#### 1.a sample class：  
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

main function contains three objects ：`Integer`,`Long`,`Boolean`, Now we use SA to inspect each object's structure and size：  
1.runing the program above，add a breakpoint at  `System.out.println("")`  
2.use command `JPS` to query JVM's PID:  
![jps](/images/blog/2014-05/20140524-objectsize-jps.png)	
3.start `SA`(HSDB)  
`sudo java -cp $JAVA_HOME/lib/sa-jdi.jar sun.jvm.hotspot.HSDB`  
4.attach PID 3813,open `Stack Memory View`，Figure 1:  

![memory inspector](/images/blog/2014-05/20140524-hsdb-memory.png)

This red rectangle shows us the three objects we decliared in the  main function,the left hand is the addresses of objects in JVM  and the  right hand is the notes about these addresses.  
The addresses of these three objects are ：  
>Integer i &nbsp;&nbsp;&nbsp; 0xa0159410  
>Long l  &nbsp;&nbsp;&nbsp; 0xa0159520  
>Boolean b &nbsp;&nbsp;&nbsp;0xa0159650  

We will inspect `Integer i` first,select menu:`Tool-->Inspect`,input `0xa0159410` in the textfield to get this object，Figure 2:  
![Layout of Integer](/images/blog/2014-05/20140524-hsdb-memory-integer.png)

Use the same way to get `Long l` object，Figure 3：  
![Layout of Long](/images/blog/2014-05/20140524-hsdb-memory-long.png)

also `Boolean b` ,Figure 4:  
![Layout of Boolean](/images/blog/2014-05/20140524-hsdb-memory-boolean.png)

Conclusion:Object instance data  only contains autogeneic instance variables and variables extend from its' parents，In the source code of `Integer,Long,Boolean` we can know that these three classes only have one `value` variable，so except  _mark(MARK_WORD),_metadata._klass，we only can find a `value` variable in the memory inspector.  

#### 2.a complex class:
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
The memory layout of `Animal` object, Figure 5:  
![Memory layout of Animal](/images/blog/2014-05/20140524-hsdb-memory-animal.png)

From `figure 5` we can find that three Objects are all  in Animal's memory layout.  

#### 3.a class with extended variables:
```java
public class Tiger extends Animal{
	public Double weight;
	public static void main(String args[]){
		Tiger tiger = new Tiger();
		System.out.println("");	
	}
}
```
HSDB shows us the result，Figure 6:  
![Memory Layout of Tiger](/images/blog/2014-05/20140524-hsdb-memory-tiger.png)

From figure 6 we can find that variables extends from parents will stored in children's class too.  

#### 4 .Tiger with array:
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
This is the result ,Figure 7:  
![Memory layout of array](/images/blog/2014-05/20140524-hsdb-memory-array.png)

It is weird that there is no length variable for childs array!  It is wrong about  we said before?  

use `inspect` command to inspect this object's size:  
`inspect 0xa01ba470` Figure 8：  
![array size](/images/blog/2014-05/20140524-hsdb-memory-array-size.png)

We can do some inferences :  
1. if this object has no length variable，the size should be：size=4+4=8.  
2. if this object has length variable，the size should be ：size=4+4+4+4(padding)=16, correspond with the size showd in Figure 8, so the length variable is exists，but why it does not display on the panel ，i think it would be a bug of SA plugin.  

Conclusion: SA plugin can show the Run-Time data of JVM, It is a great tool to help us to learn the implements of  JVM.  

Bibliographies:  
[http://rednaxelafx.iteye.com/blog/1847971][3]  
[http://ifeve.com/java-synchronized/][2]  
[http://icyfenix.iteye.com/blog/1145044](http://icyfenix.iteye.com/blog/1145044)


[1]: /blog/2014/05/21/cpu-and-memory/
[2]: http://ifeve.com/java-synchronized/
[3]: http://rednaxelafx.iteye.com/blog/1847971
