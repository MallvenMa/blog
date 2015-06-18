---
layout: post
title: "How HashMap Works"
date: 2015-03-16 13:57:42 +0800
comments: true
categories: java
keywords: HashMap
description: How HashMap Works
---
###1.The Structure Of HashMap
As we know,all structures of object in Java are based on array and reference,HashMap is same too. A HashMap structure consist of two parts:Array and LinkList(Figure 1).   
 
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
Okey,because h & (length-1) equals h % length.  
####2.2  why the length must be power of 2
If we want to get a value from HashMap efficiently,the ideal situation is that every bucket only has one entry,so we can get the object directly without  traversing the linklist.  So we need the value are stored in the array evenly. But how HashMap make a hashcode evenly? The implementation in Java:  

#####1.First,choose the suitable length of array:  
We will do a contrast experiment to see the advantange of using the power of2 as the length of array.  
 

From figure 2 we can find that use power of 2 as length can reduce the collision between two different hashcode.  
Second:  
```java
h ^= (h >>> 20) ^ (h >>> 12);
return h ^ (h >>> 7) ^ (h >>> 4);
```
These two line codes can  make the hashcode disperse more evenly.  



