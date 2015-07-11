---
layout: post
title: "What the Java's memory model looks like"
date: 2015-03-24 14:12:42 +0800
comments: true
categories: jvm
keywords: jvm,memory,java
description: What the Java's memory model looks like
---
###1. Java's memory model
From Java 2 Standard Edition, Java performs automatic memory management. In order to enhance the garbage collection's efficiency, Java bring in generational memory model.   
Figure 1 is a picture I find from "pointsoftware.com"  

![java memeory model][1]  

<!--more-->
  
Heap Space: we must be familar with this area, The objects we create are stored  in this area. To improve the efficiency of garbage collection, this area is divided into two separate areas:`Young Generation` and `Old Generation`, For the same reason `Young Generation` is divided into three parts : `Eden`, `Survivor 0 (From)`, `Survivor 1 (To)`. The default ratios of these three areas is `8:1:1`, We can also change it by specify parameter  `-XX:SurvivorRatio` in command line while jvm start up.   `Old Generation` holds the objects that have survived after several circulations(default value is 15) of garbage collections and objects have big size such as arrays.   
   
Method Area:  also known as the `Permanent Generation`, All class data are loaded into this memory space, This include Runtime Constant Pool, Field, Method Data, Code. Note: Method Area has been removed from jvm in JDK8.  The data stored in Method area before are stored in a new memeory  space same as `Native Area` named `Metaspace`.   
  
Native Area: This area is  used by threads and holds the references to the code and object data in the heap, `Native Area` also stores the local variables of primitive types.   
  
  
###2. How an object is allocated
Figure 2 is a picture I find from ifeve.com 
 
![allocate memory for objects][2]  

####Allocating Steps

1. if the object is big or an array, it will be directly allocated on the `Old Generation`.  
2. if it is a normal object, Jvm will try to allocate it on the `Eden`.  
3. if `Eden` is full, minorGC will be triggered, live objects in `Eden` and one  `Survivor[From]` will be moved to another  `Survivor[To]`. The old enough objects(default is 15 years old) will be moved to `Old Generation`. After that, these two Survivor will exchange their role.  
4. if  `To Survivor` is full, the objects in `To Survivor` will be moved to `Old Generation`  
5. if `Old Generation` is full, `Full GC` will be invoked.  
6. if heap also has no enough memeory after `Full GC`, jvm will throw `OutOfMemoryException`  

####Note:
In order to improve the efficiency of object allocation, Java use `TLAB(Thread Local Allocation Buffer)` technology to allocate memory, Because allocate data on `Eden` directly can cause concurrent problems, so JVM need to lock the memory before the allocation and unlock the memory after the allocation, this will make memory allocation a bottleneck.  

#####So, what is the `TLAB` ?  
JVM use 1% of the `Eden` area as the `TLAB` space, each Thread will be given a specified space to allocate their own object, so objects are allocated in their own Thread, thus, there is no lock needed during the allocation, so `TLAB` will be more efficient than the direct allocation.  Only the space of the thread is exhausted and need to increase new spaces the synchronized lock need to be added.  




[1]:/images/blog/2015-03/20150324-RuntimeDataAreas_JVM_Model.png
[2]:/images/blog/2015-03/20150324-eden_survivor.png

