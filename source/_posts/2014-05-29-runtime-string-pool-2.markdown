---
layout: post
title: "JVM 之 String 常量池 二"
date: 2014-05-29 20:58:21 +0800
comments: true
categories:  jvm
keywords: jvm,string,pool,stringtable,hashtable,常量池,字符串池
description: JVM 之 String 常量池 二 
---

上一篇文章[JVM 之 String 常量池 一](/blog/2014/05/27/runtime-string-pool-1)中我们了解到JVM中有一个叫String常量池的东西，String常量池到底是什么样？，又是怎样工作的呢？今天就来看一下。  
做任何事情都要首先找一个入口。我们怎么才能找到常量池的入口呢？  
我们首先想到String里面有个intern方法，可以在运行时向常量池中动态添加字符串。我们就从这个方法入手。  
先看String 的intern的源代码:  
```java
public native String intern();
```
这是一个native 方法，也就是这个方法不是用java实现的，要找到这个native 方法就要去[JDK的源代码](http://openjdk.java.net/)中查找，它位于`openjdk/jdk/src/share/native/java/lang`目录中的`String.c `中。这个目录下还有许多其他java类中对应的native方法的实现，例如object类中的`hashCode`,`getClass`,`clone`等方法都在`Object.c`里面。   
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
`Java_java_lang_String_intern` 就是String类intern 对应的native方法,然而`Java_java_lang_String_intern` 只是调用了`JVM_InternString`这个方法。  
那么这个方法又在哪呢？  
根据经验我们发现`String.c` 引入了`jvm.h`，去`jvm.h`中去看一下。`jvm.h` 位于`openjdk/jdk/src/share/javavm/export` 下面，`jvm.h`中正好定义了我们要找的`JVM_InternString`  
```java
/*
* java.lang.String
* */
JNIEXPORT jstring JNICALL
JVM_InternString(JNIEnv *env, jstring str);
```  
这里只是定义,我们继续找这个定义对应的实现方法。相关实现在`jvm.cpp`中,`jvm.cpp` 位于：`openjdk/hotspot/src/share/vm/prims` 目录中。`jvm.cpp` 中`JVM_InternString`的具体实现如下:  
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
从上面的代码我们可以看出，实际上`JVM_InternString` 又调用了`StringTable` 的 `intern` 方法。`StringTable` 是在`symbolTable.hpp`  中被声明的，代码：  
```java
class StringTable : public Hashtable<oop, mtSymbol> {
  friend class VMStructs;
private:
  // The string table
  static StringTable* _the_table;
.......
```
我们先简单看一下`StringTable`这个类：   
1.  继承了Hashtable<oop, mtSymbol>，在Hashtable中，字符串被包装成HashtableEntry对象存储，同时为了解决hash碰撞的问题，HashtableEntry对象被设计为链表结构。最后HashTable使用数组_buckets来存储这些HashtableEntry。   
2. ` static StringTable* _the_table` : StringTable的实例变量,在`create_table()` 中被实例化。  
3. `lookup(...)`: 用来查找常量池中是否包含某个实例。  
4. `basic_add(...)`: 往常量池中添加新实例。  
5. `intern(...)`: 判断常量池中是否有某个实例，有则返回该实例，没有则调用`basic_add` 添加。  

看一下intern方法:  
```java

oop StringTable::intern(Handle string_or_null, jchar* name,
                        int len, TRAPS) {
  unsigned int hashValue = hash_string(name, len);
  int index = the_table()->hash_to_index(hashValue);
  oop string = the_table()->lookup(index, name, len, hashValue);

  // Found
  if (string != NULL) return string;

  // Otherwise, add to symbol to table
  return the_table()->basic_add(index, string_or_null, name, len,
                                hashValue, CHECK_NULL);
}
```
第3行:首先调用hash_string()计算字符串的hash值。hash值计算规则:`s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]` 例如：`“a”.hashCode() = 97`。代码:  
```java
int StringTable::hash_string(jchar* s, int len) {
  unsigned h = 0;
  while (len-- > 0) {
    h = 31*h + (unsigned) *s;
    s++;
  }
  return h;
}
```
第4行:把hash值转换为数组的下标。转换规则：`hash % table_size`。代码:  
```java
int hash_to_index(unsigned int full_hash) {
    int h = full_hash % _table_size;
    assert(h >= 0 && h < _table_size, "Illegal hash value");
    return h;
  }
```
第5行:检查该字符串是否存在，如果存在，就返回。代码:
```java
oop StringTable::lookup(int index, jchar* name,
                        int len, unsigned int hash) {
  for (HashtableEntry* l = bucket(index); l != NULL; l = l->next()) {
    if (l->hash() == hash) {
      if (java_lang_String::equals(l->literal(), name, len)) {
        return l->literal();
      }
    }
  }
  return NULL;
}
```
说明:  
1.首先从数组_buckets获得当前下标对应的HashtableEntry。   
2.判断该entry的hash值和字符串值是否都相等(是不是很眼熟)，如果都相等则返回该entry中存储的字符串对象。  
3.如果(2)条件不成立则继续循环next entry。  

第11行:调用StringTable的`basic_add(...)`方法将字符串添加到常量池。代码:  
```java
oop StringTable::basic_add(int index, Handle string_or_null, jchar* name,
                           int len, unsigned int hashValue, TRAPS) {
  debug_only(StableMemoryChecker smc(name, len * sizeof(name[0])));
  assert(!Universe::heap()->is_in_reserved(name) || GC_locker::is_active(),
         "proposed name of symbol must be stable");

  Handle string;
  // try to reuse the string if possible
  if (!string_or_null.is_null() && string_or_null()->is_perm()) {
    string = string_or_null;
  } else {
    string = java_lang_String::create_tenured_from_unicode(name, len, CHECK_NULL);
  }

  // Allocation must be done before grapping the SymbolTable_lock lock
  MutexLocker ml(StringTable_lock, THREAD);

  assert(java_lang_String::equals(string(), name, len),
         "string must be properly initialized");

  // Since look-up was done lock-free, we need to check if another
  // thread beat us in the race to insert the symbol.

  oop test = lookup(index, name, len, hashValue); // calls lookup(u1*, int)
  if (test != NULL) {
    // Entry already added
    return test;
  }

  HashtableEntry* entry = new_entry(hashValue, string());
  add_entry(index, entry);
  return string();
}
```

第12行:创建一个String对象,创建过程可以参考`openjdk/hotspot/src/share/vm/classfile/javaClasses.cpp`中`java_lang_String`类的`create_tenured_from_unicode`方法。后面要单独讲对象的创建过程，这里就不展开了，后面文章写完了，会把链接贴过来。  
第30行:创建一个HashtableEntry对象。  
第31行: 将新创建的Sting对象添加到常量池`_buckets`中。代码:  
```java
inline void BasicHashtable::add_entry(int index, BasicHashtableEntry* entry) {
  entry->set_next(bucket(index));
  _buckets[index].set_entry(entry);
  ++_number_of_entries;
}
```
说明:  
1.首先从`_buckets`数组获得index位置的HashtableEntry 记为oldEntry。   
2.将oldEntry设置为entry的`_next`(HashtableEntry设计为链表结构,就是用在这里)  
3.将entry设置到_buckets的index位置  

###总结:  
String常量池对应的数据结构就是StringTable对象,也就是一个Hashtable结构。hashtable的结构是数组+链表。hashtable一直持有字符串的引用，因此字符串池中的对象，不会被垃圾收集器回收掉。  
Hashtable 的结构看起来应该是这样的：  
![Hasttable](/images/blog/2014-06/20140602-hashtable.png)
