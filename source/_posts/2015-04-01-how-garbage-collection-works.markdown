---
layout: post
title: "How Garbage Collection Work"
date: 2015-04-01 11:32:34 +0800
comments: true
keywords: garbage collection,jvm,memory
categories: jvm 
---
###1.Disadvantage of Explicit Memory Management
Before the advent of Automatic Memory Mangement, programmers are  responsible for the memeory management, they allocate and deallocate memeory by their codes. 
But sometimes manage memory by hand will cause many memeory related problems when  programs become relative complexed. eg:   
####1.1. Dangling References  
When we deallocate the space used by an object to which some other objects still has a reference. If the object with that reference tries to access the original object, but the space has been reallocated to a new object, the result would be unpredictable.  

####1.2. Space Leaks
If one object has a reference of another object, we deallocate the space used by the first object but forgot to free the space used by the second object. The second object could not be referenced by other objects or be deallocated, it will become a no-reachable object, if there are many objects like this in our program, the physical memory will be exhausted soon.  

<!--more-->

###2. Automatic Memory Management
In order to resolve the problems caused by Explicit Memory Mnangement, we introduce the Automatic Memory Management concept.  With `Automatic Memory Management` we do not concern the Memory Management problems, they are managed by a program called `Garbage Collector` , we can only concentrate on the logical thing. The world is  becoming more easier.  

###3. The Responsibility of Garbage Collector
1. allocate space for objects.  
2. ensuring that any referenced objects remain in memory
3. deallocate memory used by objects that are no long reachable from references in executing code.  
 
###4. How Many Garbage Collectors JVM Have
1. Serial Garbage Collector
2. Parallel Garbage Collector
3. Parallel Compacting Garbage Collector
4. Concurrent Mark-Sweep Garbage Collector
5. Garbage-First(G1) Collector [JDK7 update 4]

###5. How These Garbage Collectors Works
####5.1 Serial Garbage Collector

####5.1.1 Young Generation Collection
Figure 1 illustrates the operation of young generation using the serial collection, when `Eden` is full, `Minor Gc` will be executed on `Young Generation`. The live objects in `Eden` are copied to the empty survivor space,labed `To` in the figure, except for ones too large to fit comfortably in the `To` space. Such objects are driectly copied to `Old Generation`. The live objects relatively young  in survivor `From` are also copied to `To` while objects that relatively old are copied to `Old Generation`. But if `To` survivor space becomes full, the objects from `Eden` and `From` that have not been copied to it are tenured, regardless how many young generation collections they have survived, these objects will be copied to `Old Generation`. After this process, all objects remaining in `Eden` and `From` are not live, the memory they use will be deallocated.  
 
![serial collector][1]  

After the young generation collections, `Eden` and `From` are empty and `To` holds all the live objects from `Eden` and `From`, `From` and `To` would swap their roles after the young generation collection, one of them will always be empty.  
  
![serial collector after][2]  

####5.1.2 Old Generation Collection
Figure 3 illustrates `Old Generation` collection using the serial collector.  

![serial collector old generation][3]  
.  
.  
####5.2 Parallel Garbage Collector






[1]:/images/blog/2015-04/20150401-serial-collector.png
[2]:/images/blog/2015-04/20150401-serial-collector-after.png
[3]:/images/blog/2015-04/20150401-serial-collector-old-generation.png

