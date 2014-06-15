---
layout: post
title: "JVM 之 Java对象创建[加载和连接]"
date: 2014-06-14 08:53:49 +0800
comments: true
categories: jvm
keywords: jvm,对象,初始化,加载,连接
description: JVM 之 Java对象创建
---
Java对象的生命周期可以分为：加载，验证，准备，解析，初始化，使用，卸载 八个阶段。其中验证，准备，解析又被统称为连接，今天主要是简单看一下加载和连接，下一篇文章讲初始化。  
![对象生命周期](/images/blog/2014-06/20140614-java-load-pic.png)  
说明：  
>本文章所涉及的代码Gist地址:[点击查看](https://gist.github.com/zarue/0d5f83fa8458a9298b9d)  
>本文使用是Jdk6u-Hotspot
<!--more-->
要想了解jvm底层的对象创建过程，还是要首先找到一个入口。new关键字无疑是最先想到的，但是看着这段代码“new Animal()” ,我还是找不到什么有用的信息，于是又想到了另一种方式。自定义classloader。  
```java
public class Animal {
	
	String name;
	
	Integer age;
	
	public void run(){
		
	}
	
}
```
关于自定义classloader 的文章，一搜一大堆，这里就不多说了。我自己定义了一个简单的Classloader。代码：  
```java
public class ClassLoaderTest extends ClassLoader {
	@Override
	protected Class<?> findClass(String name) throws ClassNotFoundException {
		byte[] classBytes = null;
		try {
			InputStream is = new FileInputStream(new File(name));
			ByteArrayOutputStream bos = new ByteArrayOutputStream();
			byte[] buffer = new byte[256];
			int length = 0;
			while((length = is.read(buffer))!=-1){
				bos.write(buffer, 0, length);
			}
			classBytes = bos.toByteArray();
			bos.close();
			is.close();
		} catch (Exception e) {
			e.printStackTrace();
		}
		if(classBytes==null){
			throw new ClassNotFoundException("class not found:"+name);
		}
		return defineClass(name, classBytes, 0, classBytes.length);
	}

	public static void main(String args[]){
		ClassLoaderTest clt = new ClassLoaderTest();
		try {
			Class claszz = clt.loadClass("Animal");
			Animal animal = (Animal)claszz.newInstance();
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
}
```
findClass（） 方法主要是加载class文件,然后调用defineClass（...）。  
看一下defineClass(...)的代码:  
```java
protected final Class<?> defineClass(String name, byte[] b, int off, int len,
                                         ProtectionDomain protectionDomain)
        throws ClassFormatError
    {
        protectionDomain = preDefineClass(name, protectionDomain);

        Class c = null;
        String source = defineClassSourceLocation(protectionDomain);

        try {
            c = defineClass1(name, b, off, len, protectionDomain, source);
        } catch (ClassFormatError cfe) {
            c = defineTransformedClass(name, b, off, len, protectionDomain, cfe,
                                       source);
        }

        postDefineClass(c, protectionDomain);
        return c;
    }
```
第5行：`preDefineClass(name, protectionDomain);`主要是进行一些预处理，比如检查名字是否合法，证书是否正确等等。有兴趣的自己看一下。  
第11行：`defineClass1(...)`调用了native方法，这个方法位于：/share/native/java/lang/ClassLoader.c中。  
```java
Java_java_lang_ClassLoader_defineClass1(JNIEnv *env,
                                        jobject loader,
                                        jstring name,
                                        jbyteArray data,
                                        jint offset,
                                        jint length,
                                        jobject pd,
                                        jstring source)
{
    jbyte *body;
    char *utfName;
    jclass result = 0;
    char buf[128];
    char* utfSource;
    char sourceBuf[1024];

    if (data == NULL) {
        JNU_ThrowNullPointerException(env, 0);
        return 0;
    }
 if (length < 0) {
        JNU_ThrowArrayIndexOutOfBoundsException(env, 0);
        return 0;
    }

    body = (jbyte *)malloc(length);

    if (body == 0) {
        JNU_ThrowOutOfMemoryError(env, 0);
        return 0;
    }

    (*env)->GetByteArrayRegion(env, data, offset, length, body);

    if ((*env)->ExceptionOccurred(env))
        goto free_body;

    if (name != NULL) {
        utfName = getUTF(env, name, buf, sizeof(buf));
        if (utfName == NULL) {
            JNU_ThrowOutOfMemoryError(env, NULL);
            goto free_body;
        }
        VerifyFixClassname(utfName);
    } else {
        utfName = NULL;
    }

    if (source != NULL) {
        utfSource = getUTF(env, source, sourceBuf, sizeof(sourceBuf));
        if (utfSource == NULL) {
            JNU_ThrowOutOfMemoryError(env, NULL);
            goto free_utfName;
        }
    } else {
        utfSource = NULL;
    }
    result = JVM_DefineClassWithSource(env, utfName, loader, body, length, pd, utfSource);

    if (utfSource && utfSource != sourceBuf)
        free(utfSource);

 free_utfName:
    if (utfName && utfName != buf)
        free(utfName);

 free_body:
    free(body);
    return result;
}
```
第33行：`(*env)->GetByteArrayRegion(env, data, offset, length, body);`把传进来的clas对应的字节数组复制给body。`GetByteArrayRegion`此函数将Java传来的字节数组data，复制offset->length长度的数据给body。  
第39行：把name转为UTF格式。  
第58行：`JVM_DefineClassWithSource(env, utfName, loader, body, length, pd, utfSource);`该方法位于:`/share/vm/prims/jvm.cpp`   
```java
JVM_ENTRY(jclass, JVM_DefineClassWithSource(JNIEnv *env, const char *name, jobject loader, const jbyte *buf,
 jsize len, jobject pd, const char *source))
JVMWrapper2("JVM_DefineClassWithSource %s", name);

return jvm_define_class_common(env, name, loader, buf, len, pd, source, true, THREAD);
JVM_END
``` 
调用了`jvm_define_class_common(...) `方法:  
```java
static jclass jvm_define_class_common(JNIEnv *env, const char *name,
                                      jobject loader, const jbyte *buf,
                                      jsize len, jobject pd, const char *source,
                                      jboolean verify, TRAPS) {
  if (source == NULL)  source = "__JVM_DefineClass__";

  assert(THREAD->is_Java_thread(), "must be a JavaThread");
  JavaThread* jt = (JavaThread*) THREAD;

  PerfClassTraceTime vmtimer(ClassLoader::perf_define_appclass_time(),
                             ClassLoader::perf_define_appclass_selftime(),
                             ClassLoader::perf_define_appclasses(),
                             jt->get_thread_stat()->perf_recursion_counts_addr(),
                             jt->get_thread_stat()->perf_timers_addr(),
                             PerfClassTraceTime::DEFINE_CLASS);

  if (UsePerfData) {
    ClassLoader::perf_app_classfile_bytes_read()->inc(len);
  }

  // Since exceptions can be thrown, class initialization can take place
  // if name is NULL no check for class name in .class stream has to be made.
  symbolHandle class_name;
  if (name != NULL) {
    const int str_len = (int)strlen(name);
    if (str_len > symbolOopDesc::max_length()) {
      // It's impossible to create this class;  the name cannot fit
      // into the constant pool.
      THROW_MSG_0(vmSymbols::java_lang_NoClassDefFoundError(), name);
    }
    class_name = oopFactory::new_symbol_handle(name, str_len, CHECK_NULL);
  }

  ResourceMark rm(THREAD);
  ClassFileStream st((u1*) buf, len, (char *)source);
  Handle class_loader (THREAD, JNIHandles::resolve(loader));
  if (UsePerfData) {
    is_lock_held_by_thread(class_loader,
                           ClassLoader::sync_JVMDefineClassLockFreeCounter(),
                           THREAD);
  }
  Handle protection_domain (THREAD, JNIHandles::resolve(pd));
  klassOop k = SystemDictionary::resolve_from_stream(class_name, class_loader,
                                                     protection_domain, &st,
                                                     verify != 0,
                                                     CHECK_NULL);

  if (TraceClassResolution && k != NULL) {
    trace_class_resolution(k);
  }

  return (jclass) JNIHandles::make_local(env, Klass::cast(k)->java_mirror());
}

```
第35行：把传入的字节数组转换为`ClassFileStream `对象，以后所有的class数据的提取，分析，验证，转换等都将依托于该对象，和之前的数组没有关系了。  
第43行：调用`SystemDictionary::resolve_from_stream(...)`进行class对象的解析工作。该方法位于：`share/vm/classfile/systemDictionary.cpp`  
看一下这个方法里面的几行主要代码：  
```java
instanceKlassHandle k = ClassFileParser(st).parseClassFile(class_name,
                                                             class_loader,
                                                             protection_domain,
                                                             parsed_name,
                                                             verify,
                                                             THREAD);
```
调用了ClassFileParser的parseClassFile方法，这个方法里面完成Class对象的构建过程。代码位置:`src/share/vm/classfile/classFileParser.cpp`  
说道这里就要了解一下Class文件的格式了，[Java语言规范](http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html)中是这么规定的,其实单讲这个格式也够写一篇文章的，这里不深入，有兴趣的自己网上搜吧。  
```java
ClassFile {
    u4             magic;
    u2             minor_version;
    u2             major_version;
    u2             constant_pool_count;
    cp_info        constant_pool[constant_pool_count-1];
    u2             access_flags;
    u2             this_class;
    u2             super_class;
    u2             interfaces_count;
    u2             interfaces[interfaces_count];
    u2             fields_count;
    field_info     fields[fields_count];
    u2             methods_count;
    method_info    methods[methods_count];
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```
再回到parseClassFile方法：  
```java
u4 magic = cfs->get_u4_fast();
  guarantee_property(magic == JAVA_CLASSFILE_MAGIC,
                     "Incompatible magic value %u in class file %s",
                     magic, CHECK_(nullHandle));
```
对照上面的Class格式规范，首先获取magic标识(4字节)，然后判断它是不是等于CAFEBABE，如果不等于，则说明不是一个正确的Class文件格式。  
```java
// Version numbers
  u2 minor_version = cfs->get_u2_fast();
  u2 major_version = cfs->get_u2_fast();
  if (!is_supported_version(major_version, minor_version)) {
    if (name.is_null()) {
      Exceptions::fthrow(
        THREAD_AND_LOCATION,
        vmSymbolHandles::java_lang_UnsupportedClassVersionError(),
        "Unsupported major.minor version %u.%u",
        major_version,
        minor_version);
    } else {
      ResourceMark rm(THREAD);
      Exceptions::fthrow(
        THREAD_AND_LOCATION,
        vmSymbolHandles::java_lang_UnsupportedClassVersionError(),
        "%s : Unsupported major.minor version %u.%u",
        name->as_C_string(),
        major_version,
        minor_version);
    }
    return nullHandle;
  }

```
获得Class文件的版本号，分为主版本号和次版本号。然后判断当前的JVM版本支不支持此版本的Class文件的解析。  
接下来就是常量池了，每一个Class对象里面都有一个对应的常量池(ConstantPoolOop)对象，用来存放Class中的常量。看一下`ClassFileParser::parse_constant_pool(.)`中的部分代码。   
```java
parse_constant_pool_entries(cp, length, CHECK_(nullHandle));
``` 
调用parse_constant_pool_entries(...)对象来创建常量池项：  
看一下` ClassFileParser::parse_constant_pool_entries(constantPoolHandle cp, int length, TRAPS)`中的部分代码：  
```java
case JVM_CONSTANT_Class :
        {
          cfs->guarantee_more(3, CHECK);  // name_index, tag/access_flags
          u2 name_index = cfs->get_u2_fast();
          cp->klass_index_at_put(index, name_index);
        }
        break;
```
常量池对象constantPoolOop使用一个`typeArrayOop`来存储常量池项，回到上面的代码，首先判断剩余的字节数是否满足一个`JVM_CONSTANT_Class`要求的字节数(通过Class文件规范可知是3),然后是获得u2长度的字节数作为name的真实值在常量池中对应的index(Animal的常量池可以参考[Animal常量池](https://gist.github.com/zarue/0d5f83fa8458a9298b9d#file-3-animal-javap))。  
再来看一个：  
```java
case JVM_CONSTANT_Utf8 :
        {
          cfs->guarantee_more(2, CHECK);  // utf8_length
          u2  utf8_length = cfs->get_u2_fast();
          u1* utf8_buffer = cfs->get_u1_buffer();
          assert(utf8_buffer != NULL, "null utf8 buffer");
          // Got utf8 string, guarantee utf8_length+1 bytes, set stream position forward.
          cfs->guarantee_more(utf8_length+1, CHECK);  // utf8 string, tag/access_flags
          cfs->skip_u1_fast(utf8_length);

          // Before storing the symbol, make sure it's legal
          if (_need_verify) {
            verify_legal_utf8((unsigned char*)utf8_buffer, utf8_length, CHECK);
          }

          if (AnonymousClasses && has_cp_patch_at(index)) {
            Handle patch = clear_cp_patch_at(index);
            guarantee_property(java_lang_String::is_instance(patch()),
                               "Illegal utf8 patch at %d in class file %s",
                               index, CHECK);
            char* str = java_lang_String::as_utf8_string(patch());
            // (could use java_lang_String::as_symbol instead, but might as well batch them)
            utf8_buffer = (u1*) str;
            utf8_length = (int) strlen(str);
          }

          unsigned int hash;
          symbolOop result = SymbolTable::lookup_only((char*)utf8_buffer, utf8_length, hash);
          if (result == NULL) {
            names[names_count] = (char*)utf8_buffer;
            lengths[names_count] = utf8_length;
            indices[names_count] = index;
            hashValues[names_count++] = hash;
            if (names_count == SymbolTable::symbol_alloc_batch_size) {
              oopFactory::new_symbols(cp, names_count, names, lengths, indices, hashValues, CHECK);
              names_count = 0;
            }
          } else {
            cp->symbol_at_put(index, result);
          }
        }
        break;
```
这是Utf-8类型常量的处理方式，首先检查`SymbolTable`中是否存在该字符串，如果存在就返回已经存在的字符串对象。如果不存在，首先调用`oopFactory::new_symbols(...)`创建一个symbol对象，然后将它加入到SymbolTable中。这样就保证了同样的符号在jvm中仅仅会存在一个对象，可以大大节省存储空间。这里和之前讲的字符串池是一样的处理方式。   
常量池项解析完成之后，我们回到`ClassFileParser::parse_constant_pool(.)`方法，接下来是对转换完成的常量池项进行检查，如果全部检查通过，则返回该常量池对象。  
然后再回到`ClassFileParser::parseClassFile(...)`方法。  
接下来会接着解析`access_flag`,`this_class`,`super_class`等项,具体看源代码即可。
全都解析，验证完了之后，会创建一个描述这个类的对象即:instanceKlass来存储上面解析出来的各个项。instanceKlass的结构如下：  
```java
    instanceKlass layout:
    [header                     ] klassOop
    [klass pointer              ] klassOop
    [C++ vtbl pointer           ] Klass
    [subtype cache              ] Klass
    [instance size              ] Klass
    [java mirror                ] Klass
    [super                      ] Klass
    [access_flags               ] Klass
    [name                       ] Klass
    [first subklass             ] Klass
    [next sibling               ] Klass
    [array klasses              ]
    [methods                    ]
    [local interfaces           ]
    [transitive interfaces      ]
    [number of implementors     ]
    [implementors               ] klassOop[2]
    [fields                     ]
    [constants                  ]
    [class loader               ]
    [protection domain          ]
    [signers                    ]
    [source file name           ]
    [inner classes              ]
    [static field size          ]
    [nonstatic field size       ]
    [static oop fields size     ]
    [nonstatic oop maps size    ]
    [has finalize method        ]
    [deoptimization mark bit    ]
    [initialization state       ]
    [initializing thread        ]
    [Java vtable length         ]
    [oop map cache (stack maps) ]
    [EMBEDDED Java vtable             ] size in words = vtable_len
    [EMBEDDED static oop fields       ] size in words = static_oop_fields_size
    [         static non-oop fields   ] size in words = static_field_size - static_oop_fields_size
    [EMBEDDED nonstatic oop-map blocks] size in words = nonstatic_oop_map_size
```
然后是初始化静态变量(准备阶段):  
```java
 // Initialize static fields
    this_klass->do_local_static_fields(&initialize_static_field, CHECK_(nullHandle));
``` 
然后是一系验证，验证通过之后返回该对象。  
最后再回到`jvm_define_class_common(..)`方法，看如下代码:  
`(jclass) JNIHandles::make_local(env, Klass::cast(k)->java_mirror());`  
首先是把klassOop 转为Klass 对象，然后加入到当前线程中，然后转换为class对象返回。

