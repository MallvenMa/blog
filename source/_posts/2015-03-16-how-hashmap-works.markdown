---
layout: post
title: "How HashMap Works"
date: 2015-03-16 13:57:42 +0800
comments: true
categories: java
keywords: HashMap,Source
description: How HashMap Works
---
###1.The Structure of HashMap
As we know,all structures of object in Java are based on array and reference,HashMap is same too. A HashMap structure consist of two parts:Array and LinkList(Figure 1).   

![HashMap Structure][1]   

<!--more-->
When we put a new item to a HashMap, First of all, HashMap will use hash algorithm to calculate the hashcode of the given key. Then use this hashcode to mod the capacity of the array to get the index of the array which will store the reference of this object. If the space of this index was already used by other object, HashMap will use linklist structure to store these object, the new one will be added before to the old one, And the  first one's reference is stored in this specified index of the array.  

fragments of HashMap source code  
Array:  
```java
 transient Entry<K,V>[] table = (Entry<K,V>[]) EMPTY_TABLE;
``` 

LinkList Node:  

```java
 Entry(int h, K k, V v, Entry<K,V> n) {
          value = v;
          next = n;//point to the next entry 
          key = k;
          hash = h;
   }
```


###2.How hash function find the bucket location of the given key  
From figure 1 ,we can know that If we want to find an object from HashMap,we must use the hashcode of the key to find the bucket’s location in the entry array.This is how HashMap in Java does:  
```java
static int indexFor(int h, int length) {
        // assert Integer.bitCount(length) == 1 : "length must be a non-zero power of 2";
        return h & (length-1);
 }
```
h & (length-1) means h mod (length-1), but bits operation is more efficient than mod operation.  
but we notice that there is a line of comment “length must be a non-zero power of 2”,why power of 2? and why (length-1) not length.  
####2.1  why is length-1?
Okay, because h & (length-1) equals h % length.  
####2.2  why the length must be power of 2
If we want to get a value from HashMap efficiently,the ideal situation is that every bucket only has one entry,so we can get the object directly without  traversing the linklist.  So we need the value are stored in the array evenly. But how HashMap make a hashcode evenly? The implementation in Java:  

#####1.First,choose a suitable length to hash table array:  
We will do a contrast experiment to find the advantange of using the power of 2 as the length of array.   

condition 1 : `length = 2^4 = 16, hash = 13 or hash = 12`  
 
![contrast experiment][2]  

condition 2 : `length = 15, hash = 13 or hash = 12`  

![contrast experiment][3]   

From figure 2 we can find that when we use 15 as the length of array hash=13 and hash=12 will get the same index=12, this is hash collision, beacuse if we use 2^N as the length, we can guarantee length-1 has more 1 bits data in the binary representation than no use 2^N as the length. So use power of 2 as length can reduce hash collision between two different hashcode.  
#####2.Second : bit shift  
```java
h ^= (h >>> 20) ^ (h >>> 12);
return h ^ (h >>> 7) ^ (h >>> 4);
```
These two lines  can ensure that every change of bit of hashcode can affect the final result of bucket index calculating.  

###3.The capacity of HashMap
####3.1 How many entries can be hold by an HashMap? only restricted by the system memory?  
The answer is no, the maximum capacity of a HashMap is 2^30, this value is defined in the class file of HashMap   
```java
static final int MAXIMUM_CAPACITY = 1 << 30;
```
####3.2 What is the default capacity of HashMap?
The default capactity is 2^4, we can specify the initial capacity in either of the constructors with arguments, but must be a power of two. If we specify a capacity not a power of two, HashMap will find a number of a power of two closet to the specified capacity  as the final capacity.  
```java
// Find a power of 2 >= toSize
int capacity = roundUpToPowerOf2(toSize);
```
####3.3 How HashMap to resize
HashMap has a variable named loadFactor to regulate when it needs to resize, if the amount of entries holded by HashMap over capacity*loadFactor, the resize function will be called.   
```java
threshold = (int) Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);
```
```java
if ((size >= threshold) && (null != table[bucketIndex])) {
            resize(2 * table.length);
            hash = (null != key) ? hash(key) : 0;
            bucketIndex = indexFor(hash, table.length);
        }
```












[1]:/images/blog/2015-03/20150316-hashmap-structure.png
[2]:/images/blog/2015-03/20150316-hashmap-hash-to-index-1.png
[3]:/images/blog/2015-03/20150316-hashmap-hash-to-index-2.png
