---
layout: post
title: "JVM 之 Java对象创建[初始化]"
date: 2014-06-15 10:01:00 +0800
comments: true
categories: jvm
keywords: jvm,对象,初始化,加载,连接
description: JVM 之 Java对象创建 
---
上一篇文章简单介绍了类的加载和连接阶段，今天来简单看一下类的初始化过程。  
还是使用上文的例子:  
```java
Class claszz = clt.loadClass("Animal");
Animal animal = (Animal)claszz.newInstance();
```
`newInstance`的native方法在:src/share/native/sun/reflect/NativeAccessors.c  
```java
JNIEXPORT jobject JNICALL Java_sun_reflect_NativeConstructorAccessorImpl_newInstance0
(JNIEnv *env, jclass unused, jobject c, jobjectArray args)
{
return JVM_NewInstanceFromConstructor(env, c, args);
}
```
<!--more-->
`JVM_NewInstanceFromConstructor`在jvm.cpp 中的实现：   
```java
JVM_ENTRY(jobject, JVM_NewInstanceFromConstructor(JNIEnv *env, jobject c, jobjectArray args0))
JVMWrapper("JVM_NewInstanceFromConstructor");
oop constructor_mirror = JNIHandles::resolve(c);
objArrayHandle args(THREAD, objArrayOop(JNIHandles::resolve(args0)));
oop result = Reflection::invoke_constructor(constructor_mirror, args, CHECK_NULL);
jobject res = JNIHandles::make_local(env, result);
if (JvmtiExport::should_post_vm_object_alloc()) {
JvmtiExport::post_vm_object_alloc(JavaThread::current(), result);
}
return res;
JVM_END
```
第4行：创建构造方法参数数组。   
第5行：`invoke_constructor(...)`位于/src/share/vm/runtime/reflection.cpp,这个方法包含了对象的创建过程。  
```java
oop Reflection::invoke_constructor(oop constructor_mirror, objArrayHandle args, TRAPS) {
  oop mirror             = java_lang_reflect_Constructor::clazz(constructor_mirror);
  int slot               = java_lang_reflect_Constructor::slot(constructor_mirror);
  bool override          = java_lang_reflect_Constructor::override(constructor_mirror) != 0;
  objArrayHandle ptypes(THREAD, objArrayOop(java_lang_reflect_Constructor::parameter_types(constructor_mirror)));

  instanceKlassHandle klass(THREAD, java_lang_Class::as_klassOop(mirror));
  methodOop m = klass->method_with_idnum(slot);
  if (m == NULL) {
    THROW_MSG_0(vmSymbols::java_lang_InternalError(), "invoke");
  }
  methodHandle method(THREAD, m);
  assert(method->name() == vmSymbols::object_initializer_name(), "invalid constructor");

  // Make sure klass gets initialize
  klass->initialize(CHECK_NULL);

  // Create new instance (the receiver)
  klass->check_valid_for_instantiation(false, CHECK_NULL);
  Handle receiver = klass->allocate_instance_handle(CHECK_NULL);

  // Ignore result from call and return receiver
  invoke(klass, method, receiver, override, ptypes, T_VOID, args, false, CHECK_NULL);
  return receiver();
}
```
直接从为对象分配内存开始看src/share/vm/oops/instanceKlass.cpp  
```java
Handle receiver = klass->allocate_instance_handle(CHECK_NULL);
```
继续看:`allocate_instance(...)`  
```java
instanceOop instanceKlass::allocate_instance(TRAPS) {
  bool has_finalizer_flag = has_finalizer(); // Query before possible GC
  int size = size_helper();  // Query before forming handle.

  KlassHandle h_k(THREAD, as_klassOop());

  instanceOop i;

  i = (instanceOop)CollectedHeap::obj_allocate(h_k, size, CHECK_NULL);
  if (has_finalizer_flag && !RegisterFinalizersAtInit) {
    i = register_finalizer(i, CHECK_NULL);
  }
  return i;
}
```
第9行:`i = (instanceOop)CollectedHeap::obj_allocate(h_k, size, CHECK_NULL);`位于collectedHeap.inline.hpp中:  
```java
oop CollectedHeap::obj_allocate(KlassHandle klass, int size, TRAPS) {
  debug_only(check_for_valid_allocation_state());
  assert(!Universe::heap()->is_gc_active(), "Allocation during gc not allowed");
  assert(size >= 0, "int won't convert to size_t");
  HeapWord* obj = common_mem_allocate_init(size, false, CHECK_NULL);
  post_allocation_setup_obj(klass, obj, size);
  NOT_PRODUCT(Universe::heap()->check_for_bad_heap_word_value(obj, size));
  return (oop)obj;
}
```
看一下`common_mem_allocate_init(size, false, CHECK_NULL);`  
```java
HeapWord* CollectedHeap::common_mem_allocate_init(size_t size, bool is_noref, TRAPS) {
  HeapWord* obj = common_mem_allocate_noinit(size, is_noref, CHECK_NULL);
  init_obj(obj, size);
  return obj;
}
```
继续看`common_mem_allocate_noinit(...)`  
```java
HeapWord* CollectedHeap::common_mem_allocate_noinit(size_t size, bool is_noref, TRAPS) {

  // Clear unhandled oops for memory allocation.  Memory allocation might
  // not take out a lock if from tlab, so clear here.
  CHECK_UNHANDLED_OOPS_ONLY(THREAD->clear_unhandled_oops();)

  if (HAS_PENDING_EXCEPTION) {
    NOT_PRODUCT(guarantee(false, "Should not allocate with exception pending"));
    return NULL;  // caller does a CHECK_0 too
  }

  // We may want to update this, is_noref objects might not be allocated in TLABs.
  HeapWord* result = NULL;
  if (UseTLAB) {
    result = CollectedHeap::allocate_from_tlab(THREAD, size);
    if (result != NULL) {
      assert(!HAS_PENDING_EXCEPTION,
             "Unexpected exception, will result in uninitialized storage");
      return result;
    }
  }
  bool gc_overhead_limit_was_exceeded = false;
  result = Universe::heap()->mem_allocate(size,
                                          is_noref,
                                          false,
                                          &gc_overhead_limit_was_exceeded);
  if (result != NULL) {
    NOT_PRODUCT(Universe::heap()->
      check_for_non_bad_heap_word_value(result, size));
    assert(!HAS_PENDING_EXCEPTION,
           "Unexpected exception, will result in uninitialized storage");
    return result;
  }


  if (!gc_overhead_limit_was_exceeded) {
    // -XX:+HeapDumpOnOutOfMemoryError and -XX:OnOutOfMemoryError support
    report_java_out_of_memory("Java heap space");

    if (JvmtiExport::should_post_resource_exhausted()) {
      JvmtiExport::post_resource_exhausted(
        JVMTI_RESOURCE_EXHAUSTED_OOM_ERROR | JVMTI_RESOURCE_EXHAUSTED_JAVA_HEAP,
        "Java heap space");
    }

    THROW_OOP_0(Universe::out_of_memory_error_java_heap());
  } else {
    // -XX:+HeapDumpOnOutOfMemoryError and -XX:OnOutOfMemoryError support
    report_java_out_of_memory("GC overhead limit exceeded");

    if (JvmtiExport::should_post_resource_exhausted()) {
      JvmtiExport::post_resource_exhausted(
        JVMTI_RESOURCE_EXHAUSTED_OOM_ERROR | JVMTI_RESOURCE_EXHAUSTED_JAVA_HEAP,
        "GC overhead limit exceeded");
    }

    THROW_OOP_0(Universe::out_of_memory_error_gc_overhead_limit());
  }
}
```
第15行，如果启用了UseTLAB则优先在TLAB上分配：allocate_from_tlab(...)  
```java
HeapWord* CollectedHeap::allocate_from_tlab(Thread* thread, size_t size) {
  assert(UseTLAB, "should use UseTLAB");

  HeapWord* obj = thread->tlab().allocate(size);
  if (obj != NULL) {
    return obj;
  }
  // Otherwise...
  return allocate_from_tlab_slow(thread, size);
}
```
如果在TLAB分配失败则调用`allocate_from_tlab_slow(...)`，该方法会重新计算TLAB的大小，然后重新创建一个新的TLAB用于分配该对象。   
如果`allocate_from_tlab_slow(...)`也没成功，则调用  
```java
Universe::heap()->mem_allocate(size,
                                          is_noref,
                                          false,
                                          &gc_overhead_limit_was_exceeded);
```
在共享内存区域分配内存。在该区域分配内存需要加锁，所以速度要比在TLAB上分配效率低一些。该方法位于：/src/share/vm/gc_implementation/parallelScavenge/parallelScavengeHeap.cpp  
```java
eapWord* ParallelScavengeHeap::mem_allocate(
                                     size_t size,
                                     bool is_noref,
                                     bool is_tlab,
                                     bool* gc_overhead_limit_was_exceeded) {
  assert(!SafepointSynchronize::is_at_safepoint(), "should not be at safepoint");
  assert(Thread::current() != (Thread*)VMThread::vm_thread(), "should not be in vm thread");
  assert(!Heap_lock->owned_by_self(), "this thread should not own the Heap_lock");

  // In general gc_overhead_limit_was_exceeded should be false so
  // set it so here and reset it to true only if the gc time
  // limit is being exceeded as checked below.
  *gc_overhead_limit_was_exceeded = false;

  HeapWord* result = young_gen()->allocate(size, is_tlab);

  uint loop_count = 0;
  uint gc_count = 0;

  while (result == NULL) {
    // We don't want to have multiple collections for a single filled generation.
    // To prevent this, each thread tracks the total_collections() value, and if
    // the count has changed, does not do a new collection.
    //
    // The collection count must be read only while holding the heap lock. VM
    // operations also hold the heap lock during collections. There is a lock
    // contention case where thread A blocks waiting on the Heap_lock, while
    // thread B is holding it doing a collection. When thread A gets the lock,
    // the collection count has already changed. To prevent duplicate collections,
    // The policy MUST attempt allocations during the same period it reads the
    // total_collections() value!
    {
      MutexLocker ml(Heap_lock);
      gc_count = Universe::heap()->total_collections();

      result = young_gen()->allocate(size, is_tlab);

      // (1) If the requested object is too large to easily fit in the
      //     young_gen, or
      // (2) If GC is locked out via GCLocker, young gen is full and
      //     the need for a GC already signalled to GCLocker (done
      //     at a safepoint),
      // ... then, rather than force a safepoint and (a potentially futile)
      // collection (attempt) for each allocation, try allocation directly
      // in old_gen. For case (2) above, we may in the future allow
      // TLAB allocation directly in the old gen.
      if (result != NULL) {
        return result;
      }
      if (!is_tlab &&
          size >= (young_gen()->eden_space()->capacity_in_words(Thread::current()) / 2)) {
        result = old_gen()->allocate(size, is_tlab);
        if (result != NULL) {
          return result;
        }
      }
      if (GC_locker::is_active_and_needs_gc()) {
        // GC is locked out. If this is a TLAB allocation,
        // return NULL; the requestor will retry allocation
        // of an idividual object at a time.
        if (is_tlab) {
          return NULL;
        }

        // If this thread is not in a jni critical section, we stall
        // the requestor until the critical section has cleared and
        // GC allowed. When the critical section clears, a GC is
        // initiated by the last thread exiting the critical section; so
        // we retry the allocation sequence from the beginning of the loop,
        // rather than causing more, now probably unnecessary, GC attempts.
        JavaThread* jthr = JavaThread::current();
        if (!jthr->in_critical()) {
          MutexUnlocker mul(Heap_lock);
          GC_locker::stall_until_clear();
          continue;
        } else {
          if (CheckJNICalls) {
            fatal("Possible deadlock due to allocating while"
                  " in jni critical section");
          }
          return NULL;
        }
      }
    }

    if (result == NULL) {

      // Generate a VM operation
      VM_ParallelGCFailedAllocation op(size, is_tlab, gc_count);
      VMThread::execute(&op);

      // Did the VM operation execute? If so, return the result directly.
      // This prevents us from looping until time out on requests that can
      // not be satisfied.
      if (op.prologue_succeeded()) {
        assert(Universe::heap()->is_in_or_null(op.result()),
          "result not in heap");

        // If GC was locked out during VM operation then retry allocation
        // and/or stall as necessary.
        if (op.gc_locked()) {
          assert(op.result() == NULL, "must be NULL if gc_locked() is true");
          continue;  // retry and/or stall as necessary
        }

        // Exit the loop if the gc time limit has been exceeded.
        // The allocation must have failed above ("result" guarding
        // this path is NULL) and the most recent collection has exceeded the
        // gc overhead limit (although enough may have been collected to
        // satisfy the allocation).  Exit the loop so that an out-of-memory
        // will be thrown (return a NULL ignoring the contents of
        // op.result()),
        // but clear gc_overhead_limit_exceeded so that the next collection
        // starts with a clean slate (i.e., forgets about previous overhead
        // excesses).  Fill op.result() with a filler object so that the
        // heap remains parsable.
        const bool limit_exceeded = size_policy()->gc_overhead_limit_exceeded();
        const bool softrefs_clear = collector_policy()->all_soft_refs_clear();
        assert(!limit_exceeded || softrefs_clear, "Should have been cleared");
        if (limit_exceeded && softrefs_clear) {
          *gc_overhead_limit_was_exceeded = true;
          size_policy()->set_gc_overhead_limit_exceeded(false);
          if (PrintGCDetails && Verbose) {
            gclog_or_tty->print_cr("ParallelScavengeHeap::mem_allocate: "
              "return NULL because gc_overhead_limit_exceeded is set");
          }
          if (op.result() != NULL) {
            CollectedHeap::fill_with_object(op.result(), size);
          }
          return NULL;
        }

        return op.result();
      }
    }

    // The policy object will prevent us from looping forever. If the
    // time spent in gc crosses a threshold, we will bail out.
    loop_count++;
    if ((result == NULL) && (QueuedAllocationWarningCount > 0) &&
        (loop_count % QueuedAllocationWarningCount == 0)) {
      warning("ParallelScavengeHeap::mem_allocate retries %d times \n\t"
              " size=%d %s", loop_count, size, is_tlab ? "(TLAB)" : "");
    }
  }

  return result;
}

```
首先在young_gen 分配，如果young 分配失败，则触发一次GC，然后重新尝试从young上分配，如果再分配失败，则从old 上分配。  
Java对象初始化并分配内存的过程基本就是这样了。这个过程中的很多细节我也还没有弄明白，接下来弄明白了再补过来吧。  
 	
