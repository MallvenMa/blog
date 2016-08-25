---
layout: post
title: "How Locks in Java Work"
date: 2015-04-10 11:38:18 +0800
comments: true
categories: jvm
keywords: lock,jvm,java 
---
#### 1. Lock Types in Java
There are four lock types in java(From JDK1.6): 

>None-Lock  
>Biased-Lock  
>LightWeight-Lock  
>HeavyWeight-Lock  
  
#### 2. where is the lock type stored in  
Lock type is stored in the `MARK WORD` in the head data of an object:  
Table 1 is a table of the details of a 32-bit JVM's  `MARK WORD` with no lock,it contains `HashCode`,`Object Generation`,`Lock Type` , The data holded by `Mark Word` will change while lock type changes during Runtime.    

<!--more-->

Lock Status|25 bit| 4 bit|1 bit(biased lock)|2 bit(lock type)
:------:|:------:|:------|:-------|:------
 no lock|hashCode|object generation|0|01
 biased lock|    |		   |1|01
 lightweight lock|      |          |  |00
 heavyweight lock|      |          |  |11
 	
  
Lock type of an object can change with the amount of threads who wants to use this object.  
Process of how lock types upgrade  
`Biased Lock > LightWeight Lock > HeavyWeight Lock`   

#### 2. How Locks Work
##### 2.1 No Lock
1. When to use `no lock`  
There is no `synchronized` keyword in the object
2. How no lock  works ? 
Noting to say about it  
  
##### 2.2 Biased Lock
1. When to use `biased lock`  
If an object with lock only used by one thread, jvm will use `biased lock` model to keep thread safe.  
2. How biased lock works?   
When a thread want to access an object with lock, the first thing to do is to check `lock type` and `biased lock` in object's `Mark Word` to find is there a lock on this object, if there is no lock on it, it will try to use `CAS` to change `Mark Word`'s `Thread ID`, if  success, this thread get the lock. The `biased lock` of an object will not be removed until the collision happen. If the thread this object is used is not alive, thread2 use `CAS` to occupy this object. If the thread is still alive, jvm will suspend this thread and turn to `lightweight lock` model.   

##### 2.3 LightWeight Lock
1. When to use `ligthweight lock`  
If an object with lock only used by several threads, and thread can get lock within several self-spin time.  
2. How lightweight lock works?   
If a thread want to access an object which is used by another thread, it will use `self-spin` to try to get the lock, if it can get the control of the object in less than 10 `self-spin` times,  it will use `CAS` operation to `Displaced Mark Word`. If it couldn't get the lock, the lock type will upgrade to `heavyweight lock`.  

##### 2.4 HeavyWeight Lock
1. When to use `heavyweight lock`
If the  application runs in multiple model, the collision between in threads are serious, the lock type will upgrade to `heavyweight lock`.  
2. How heavyweight lock works?  
Heavyweight lock use `semaphore` structure to keep thread safe, when an object is used by a thread, other threads which want to access this object are suspended by system, when the lock released by the thread , other therads will be notified to compete the lock.  
 
Note : if a lock type is upgraded to a high level type, it could not be degraded to the lower ones.  



 











 

 
