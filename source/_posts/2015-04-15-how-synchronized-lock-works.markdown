---
layout: post
title: "Usage of Synchronized in Java"
date: 2015-04-15 11:40:31 +0800
comments: true
categories: java
keywords: synchronized, lock, jvm, java
---
### 1. How to use synchronized keyword
In Java, we have three ways to use `synchronized` to keep thread safe  
>
>synchronize the class  
>synchronize the object  
>synchronize a private object  
>

<!--more-->

### 2. Examples of How to use synchronized
We will write some codes to illustrate how synchronized works  

```java
   public class SynchronizedUsageTest{

    private Object syncObject = new Object();
    private static int i = 0;

    //no lock
    public  void incWithNoLock(){
        System.out.println("incWithNoLock start");
        System.out.println("incWithNoLock i="+i++);
        System.out.println("incWithNoLock end");
    }

    //synchronize object
    public synchronized void incWithSynchronizedObject(){
        System.out.println("incWithSynchronizedObject start");
        System.out.println("incWithSynchronizedObject i="+i++);
        System.out.println("incWithSynchronizedObject end");
    }

    //synchronize object
    public  void incWithSynchronizedObject2(){
        synchronized (this) {
            System.out.println("incWithSynchronizedObject2 start");
            System.out.println("incWithSynchronizedObject2 i="+i++);
            System.out.println("incWithSynchronizedObject2 end");
        }
    }

    //synchronize class
    public static synchronized void incWithSynchronizedClass(){
        System.out.println("incWithSynchronizedClass start");
        System.out.println("incWithSynchronizedClass i="+i++);
        System.out.println("incWithSynchronizedClass end");
    }

    //synchronize class 2
    public void incWithSynchronizedClass2(){
        synchronized (SynchronizedUsageTest.class) {
            System.out.println("incWithSynchronizedClass2 start");
            System.out.println("incWithSynchronizedClass2 i="+i++);
            System.out.println("incWithSynchronizedClass2 end");
        }
    }


    //synchronize private object 2
    public void incWithSynchronizedPrivateObject(){

        synchronized (syncObject) {
            System.out.println("incWithSynchronizedPrivateObject start");
            System.out.println("incWithSynchronizedPrivateObject i="+i++);
            System.out.println("incWithSynchronizedPrivateObject end");
        }

    }

    //test the effect of synchronized of class
    public void testSynchronizeClass(){
        incWithSynchronizedClass();
        incWithSynchronizedClass2();
    }

    //test the effect of synchronized of object
    public void testSynchronizeObject(){
        incWithSynchronizedObject();
        incWithSynchronizedObject2();
    }

    //test the effect of synchronized of object and Class
    public void testSynchronizeObjectAndClass(){
        incWithSynchronizedObject();
        incWithSynchronizedClass();
    }

    //test the effect of synchronized of private object and Class
    public void testSynchronizePrivateObjectAndClass(){
        incWithSynchronizedPrivateObject();
        incWithSynchronizedClass();
    }

    //test the effect of synchronized of private object and Class
    public void testSynchronizePrivateObjectAndObject(){
        incWithSynchronizedPrivateObject();
        incWithSynchronizedObject();
    }

}

```
#### 2.1 Test the effect of add synchronized to class
Result:
```java  
>incWithSynchronizedClass start  
>incWithSynchronizedClass i=0  
>incWithSynchronizedClass end  

>incWithSynchronizedClass2 start  
>incWithSynchronizedClass2 i=1  
>incWithSynchronizedClass2 end  

>incWithSynchronizedClass start  
>incWithSynchronizedClass i=2  
>incWithSynchronizedClass end  

>incWithSynchronizedClass2 start  
>incWithSynchronizedClass2 i=3  
>incWithSynchronizedClass2 end  

>incWithSynchronizedClass start  
>incWithSynchronizedClass i=4  
>incWithSynchronizedClass end  

>incWithSynchronizedClass2 start  
>incWithSynchronizedClass2 i=5  
>incWithSynchronizedClass2 end   
```
From the result we can konw that the codes are executed one by one in serial, so `static synchronized` and `synchronized(xxx.class)` have the same effect.   

####2.2 test the effect of add synchronized to object
Result:  
```java
>incWithSynchronizedObject i=0  
>incWithSynchronizedObject end  

>incWithSynchronizedObject2 start  
>incWithSynchronizedObject2 i=1  
>incWithSynchronizedObject2 end  

>incWithSynchronizedObject start  
>incWithSynchronizedObject i=2  
>incWithSynchronizedObject end  

>incWithSynchronizedObject2 start  
>incWithSynchronizedObject2 i=3  
>incWithSynchronizedObject2 end  

>incWithSynchronizedObject start  
>incWithSynchronizedObject i=4  
>incWithSynchronizedObject end  

>incWithSynchronizedObject2 start  
>incWithSynchronizedObject2 i=5  
>incWithSynchronizedObject2 end  
```
From the result we can know that the codes are executed one by one in serial, so `synchronized` and `synchronized(this)` have the same effect.  

#### 2.3 test the effect of add  synchronized to object and Class
Result:   
```java
>incWithSynchronizedObject start  
>incWithSynchronizedObject i=0  
>incWithSynchronizedObject end  

>incWithSynchronizedClass start  
>incWithSynchronizedObject start  
>incWithSynchronizedClass i=1  
>incWithSynchronizedClass end  

>incWithSynchronizedObject i=2  
>incWithSynchronizedObject end  

>incWithSynchronizedClass start  
>incWithSynchronizedObject start  
>incWithSynchronizedClass i=3  
>incWithSynchronizedClass end  

>incWithSynchronizedObject i=4  
>incWithSynchronizedObject end  

>incWithSynchronizedClass start  
>incWithSynchronizedClass i=5  
>incWithSynchronizedClass end  
```
From the result we can see that the codes are executed in a random way not in serial, so we can get the result of synchronized of object and class can not keep thread safe.  

#### 2.4 test the effect of add synchronized to private object and Class
Result:  
```java
>incWithSynchronizedPrivateObject start  
>incWithSynchronizedPrivateObject i=0  
>incWithSynchronizedPrivateObject end  

>incWithSynchronizedClass start  
>incWithSynchronizedClass i=1  
>incWithSynchronizedPrivateObject start  
>incWithSynchronizedClass end  

>incWithSynchronizedPrivateObject i=2  
>incWithSynchronizedPrivateObject end  

>incWithSynchronizedClass start  
>incWithSynchronizedPrivateObject start  
>incWithSynchronizedClass i=3  
>incWithSynchronizedClass end  

>incWithSynchronizedPrivateObject i=4  
>incWithSynchronizedPrivateObject end  

>incWithSynchronizedClass start  
>incWithSynchronizedClass i=5  
>incWithSynchronizedClass end   
```
From the result we can see that the codes are executed in a random way not in serial, so we can get the result of synchronized of class and private object can not keep thread safe.   

#### 2.5 test the effect of add synchronized to private object and object
Result:  
```java 
>incWithSynchronizedPrivateObject start  
>incWithSynchronizedPrivateObject i=0  
>incWithSynchronizedPrivateObject end  

>incWithSynchronizedObject start  
>incWithSynchronizedObject i=1  
>incWithSynchronizedObject end  

>incWithSynchronizedPrivateObject start  
>incWithSynchronizedPrivateObject i=2  
>incWithSynchronizedPrivateObject end  

>incWithSynchronizedObject start  
>incWithSynchronizedPrivateObject start  
>incWithSynchronizedObject i=3  
>incWithSynchronizedObject end  

>incWithSynchronizedPrivateObject i=4  
>incWithSynchronizedPrivateObject end  

>incWithSynchronizedObject start  
>incWithSynchronizedObject i=5  
>incWithSynchronizedObject end  
```

From the result we can see that the codes are executed in a random way not in serial, so we can get the result of synchronized of object and private object can not keep thread safe.   


#### 2.6 What will happen  if has no locks
Result:  
```java
>incWithNoLock start  
>incWithNoLock i=0  
>incWithNoLock end  
>incWithNoLock start  
>incWithNoLock i=1  
>incWithNoLock end  

>incWithNoLock start  

>incWithNoLock i=2  
>incWithNoLock end  
```
Wowooooooo.. zzzZZZZZ
