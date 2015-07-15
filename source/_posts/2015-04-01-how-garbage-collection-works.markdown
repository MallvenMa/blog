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

#####5.1.1 Young Generation Collection
Figure 1 illustrates the operation of young generation using the serial collection, when `Eden` is full, `Minor Gc` will be executed on `Young Generation`. The live objects in `Eden` are copied to the empty survivor space,labed `To` in the figure, except for ones too large to fit comfortably in the `To` space. Such objects are driectly copied to `Old Generation`. The live objects relatively young  in survivor `From` are also copied to `To` while objects that relatively old are copied to `Old Generation`. But if `To` survivor space becomes full, the objects from `Eden` and `From` that have not been copied to it are tenured, regardless how many young generation collections they have survived, these objects will be copied to `Old Generation`. After this process, all objects remaining in `Eden` and `From` are not live, the memory they use will be deallocated.  
 
![serial collector][1]  

After the young generation collections, `Eden` and `From` are empty and `To` holds all the live objects from `Eden` and `From`, `From` and `To` would swap their roles after the young generation collection, one of them will always be empty.  
  
![serial collector after][2]  

#####5.1.2 Old Generation Collection
Old Generation Collector use `Mark-Sweep-Compact` algorithm ,the collecting process has three phases:  

1. Mark Phase: collector identifies which objects are still alive.  
2. Sweep Phase: deallocate the space used by objects not alive.  
3. Compact Phase: the objects still alive are slided towards the begining of Old Generation, the opposite end of Old Generation will leave free. After that live objects are at one side while free spaces at the other side, the future allocations into the Old Generation can use `Bump-the-pointer` technique.  
 
Figure 3 illustrates `Old Generation` collection using the serial collector.  

![serial collector old generation][3]  


####5.2 Parallel Garbage Collector

#####5.2.1 Young Generation Collection
Parallel Garbage Collector used in `Young Generation` is a parallel version of algorithm utilized by the `Serial Collector`,It is still a `Stop-the-World` and copying collector, but can  perform the young generation collection in parallel by using many cpus to reduce the stop time and increase the application throughput.  
Figure 4 illustrates the difference between Serial Collector and Parallel Collector in young generation  

![parallel young][4]  

#####5.2.2 Old Generation Collection
Old generation collection for parallel collector is done using the same serial Mark-Sweep-Compact algorithm as the serial collector.  
  

####5.3 Parallel Compaction Collector

#####5.3.1 Young Generation Collection
Algorithm used by Parallel Compacting Collector in `Young Generation` is as same as the algorithm utilized by the Parallel Garbage collector.  
  
 

#####5.3.2 Old Generation Collection 
Pararllel Compaction Collection in old generation has three phases:  
First, each generation is logically divided into fixed-sized regions  
1. Mark: the initial set of live object directly reachable from the application code is divided among garbage collection threads, and then all live objects are marked in parallel. As an object is marked as live, the data of the region where the object in will be updated with information about the size and location of the object.  
2. Summary: this phase operates on region not objects, Due to compactions from prvious collections, it is typical that some portion of the left side of the generation will become dense, containing mostly live objects, The amount of space can be revovered from these dense regions is not worth the cost of compacting them. So the first thing Summary phase does is to examine the density of the regions, starting with the leftmostly ones, until it reaches a point where the space that could be recovered from a region and those to the right of it is worth the cost of computing those regions. The regions to the left of that point are referred to as the dense frefix, and no objects are moved in those regions. The regions to the right of that point will be compacted, eliminating all the dead objects. The summary phase calculates and stores the new location of the first byte of live data for each compacted region.  
3. Compaction: in this phase, garbage collection threads will use the summary data to identify the regions which need to be filled, and the threads can idependently copy data into that regions. This produces a heap that is densely packed on one end, with a single large empty block at the other end.  
 




[1]:/images/blog/2015-04/20150401-serial-collector.png
[2]:/images/blog/2015-04/20150401-serial-collector-after.png
[3]:/images/blog/2015-04/20150401-serial-collector-old-generation.png
[4]:/images/blog/2015-04/20150401-parallel-collector-young.png

