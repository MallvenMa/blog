---
layout: post
title: "JVM 之 String 常量池 二"
date: 2014-05-29 20:58:21 +0800
comments: true
categories:  jvm 
---

上一篇文章[JVM 之 String 常量池 一](/blog/2014/05/27/runtime-string-pool-1)中我们了解到JVM中有一个叫String常量池的东西，String常量池到底是什么样？，又是怎样工作的呢？今天就来看一下。  
我们从哪里入手呢，要首先找到一个可以和常量池有交互的入口才可以？  
我们首先想到String里面有个intern方法，可以在运行时动态的向常量池中添加字符串。我们就从这个方法入手。  
先看String 的intern的源代码:  
```java
public native String intern();
```
这是一个native 方法，也就是这个方法不是用java实现的，要找到这个native 方法就要去[JDK的源代码](http://openjdk.java.net/)中找了，经过不懈的努力，终于找到了，它位于`openjdk/jdk/src/share/native/java/lang`目录中的`String.c `中。这个目录下还有好多其他java类中对应的native方法的实现，比如object类中的`hashCode`,`getClass`,`clone`等方法都在Object.c里面。   
String.c 中只有一个方法：  
```java
#include "jvm.h"
#include "java_lang_String.h"
JNIEXPORT jobject JNICALL
Java_java_lang_String_intern(JNIEnv *env, jobject this)
{
return JVM_InternString(env, this);
}
```
<!--more-->
`Java_java_lang_String_intern` 就是String类intern 对应的native方法，这里我们发现`Java_java_lang_String_intern` 调用了`JVM_InternString`这个方法。  
那么这个方法又在哪呢？  
根据经验我们发现`String.c` 引入了`jvm.h`，我们去`jvm.h`中去看一下。我们继续找到`jvm.h` 位于`openjdk/jdk/src/share/javavm/export` 下面，发现`jvm.h`中正好定义了我们要找的`JVM_InternString`  
```java
/*
* java.lang.String
* */
JNIEXPORT jstring JNICALL
JVM_InternString(JNIEnv *env, jstring str);
```  
这里只是定义,我们继续找这个定义对应的实现方法。根据经验应该位于`jvm.cpp`中，我们继续找到`jvm.cpp` 的位置，它位于：`openjdk/hotspot/src/share/vm/prims` 目录中。  
在`jvm.cpp` 中终于找到了`JVM_InternString`的具体实现:  
```java
JVM_ENTRY(jstring, JVM_InternString(JNIEnv *env, jstring str))
JVMWrapper("JVM_InternString");
JvmtiVMObjectAllocEventCollector oam;
if (str == NULL) return NULL;
oop string = JNIHandles::resolve_non_null(str);
oop result = StringTable::intern(string, CHECK_NULL);
return (jstring) JNIHandles::make_local(env, result);
JVM_END
```
从上面的代码我们可以看出，实际上`JVM_InternString` 又调用了`StringTable` 的 `intern` 方法。那就继续到`StringTable`中看看，`jvm.cpp` 引入了好多头文件，这下凭经验不好找了，简单粗暴的搜一下吧。macos 的搜索就是这么好用啊，最终找到`StringTable` 是在`symbolTable.hpp`  中被声明的，代码：  
```java
class StringTable : public Hashtable<oop, mtSymbol> {
  friend class VMStructs;
private:
  // The string table
  static StringTable* _the_table;
.......请自行查看源代码
```
我们先简单看一下`StringTable`这个类：   
1.  它继承了`Hashtable<oop, mtSymbol>`，它使用HashTable结构保持着对各个字符串实例的引用（所以常量池中的对象不会被垃圾收集器收集掉）。   
2. ` static StringTable* _the_table` : StringTable的实例变量，用它来来存储字符串常量，是通过`create_table()` 来实例化。  
3. `lookup(...)`: 用来查找常量池中是否包含某个实例。  
4. `basic_add(...)`: 往常量池中添加新实例。  
5. `intern()`: 判断常量池中是否有某个实例，有则返回该实例，没有则调用`basic_add` 添加。  

先看一下intern方法:  
```java

oop StringTable::intern(Handle string_or_null, jchar* name,
                        int len, TRAPS) {
  unsigned int hashValue = hash_string(name, len);
  int index = the_table()->hash_to_index(hashValue);
  oop found_string = the_table()->lookup(index, name, len, hashValue);

  // Found
  if (found_string != NULL) return found_string;

  debug_only(StableMemoryChecker smc(name, len * sizeof(name[0])));
  assert(!Universe::heap()->is_in_reserved(name) || GC_locker::is_active(),
         "proposed name of symbol must be stable");

  Handle string;
  // try to reuse the string if possible
  if (!string_or_null.is_null() && (!JavaObjectsInPerm || string_or_null()->is_perm())) {
    string = string_or_null;
  } else {
    string = java_lang_String::create_tenured_from_unicode(name, len, CHECK_NULL);
  }

  // Grab the StringTable_lock before getting the_table() because it could
  // change at safepoint.
  MutexLocker ml(StringTable_lock, THREAD);

  // Otherwise, add to symbol to table
  return the_table()->basic_add(index, string, name, len,
                                hashValue, CHECK_NULL);
}
```
第5行:检查该字符串是否存在，如果存在，就返回。  
第19行:创建一个String对象,创建过程可以参考`openjdk/hotspot/src/share/vm/classfile/javaClasses.cpp`中`java_lang_String`类的`create_tenured_from_unicode`方法。jvm创建对象并分配内存的过程基本也是这个模式。  
第27行: 将新创建的Sting对象添加到常量池`the_table()`中。  

###总结:  
String常量池对应的数据结构就是StringTable对象,也就是一个hashtable结构。所有的字符串常量都被StringTable所持。这下大家应该对String常量池有了一个基本的认识了。
