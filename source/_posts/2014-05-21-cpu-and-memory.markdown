---
layout: post
title: "CPU Addressing and Physical Memory Structure"
date: 2014-05-21 20:35:18 +0800
comments: true
categories: system
keywords: cpu,memory,addressing,bus
description: CPU Addressing and Physical Memory Structure
---
####What we are talking in this article are concept model, differ with the  specific implement
### 1. Several concepts of CPU
1. Data Bus：Data Bus are used to transfer data between CPU and other components(eg:memory)。  
2. Data Bus Width：Data Bus Width represent the number of bits that  Registers can hold, namely:bits of CPU can hold. for example an "32-bit CPU" or an "64-bit CPU". Number of bits of CPU can hold could not represent CPU's addressing abilities.   
3. Address Bus：Address Bus is used to specify a physical address。  
4. Address Bus Width: The width of the Address Bus determines the amount of memory the system can address.For example, a system with a 32-bit address bus can address 0-2^32-1（0x00000000-0xFFFFFFFF) locations. If each memory address holds one byte, the addressable memory space  is 4GiB.    
5. Control Bus: Control Bus are used to transmit instructions to other components   

There are many articles about BUS  on the internet ，For example：  

* [Structures of CPU Bus](http://share.onlinesjtu.com/mod/tab/view.php?id=253)  
* [Width of CPU, Addressing abilities, Instructions，Width of Register，Width of Operation System](http://my.oschina.net/u/158589/blog/70813)
<!--more-->
### 2. Several concepts of memory
1. Memory cell: The memory cell is the fundamental building block of computer memory,every memory cell can store 1 bit.  
2. Memory unit: The memory unit consists of memory cells，each memory unit can store one byte or one word, but how many byte a word represent? it depend on the specific implemention.  
3. Memory unit address：the index of memory nuit，alway a hexadecimal number. CPU use this index to locate the data it needs.  
4. Memory device：Memory device is a physical component of computer consists of memory units.  

There are many articles about the details of memory on the internet，For example：  

* [Memory](http://www.baike.com/wiki/%E5%AD%98%E5%82%A8%E5%99%A8)  

Conclusion: Comprehend the structures of CPU and Memory can help us to dispel the confusions about  how our program works on the computer and understand the principle of CPU addressing.  
