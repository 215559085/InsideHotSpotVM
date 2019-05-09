# [Inside HotSpot] 虚拟机中的Java堆

## 1. 宇宙初始化
JVM在启动的时候会初始化各种结构，比如模板解释器，类加载器，当然也包括这篇文章的主题，Java堆。为了简单起见，本文所述Java堆均为分代堆(Generational Heap)，分代堆相信不用多说，新生代老年代堆模型都是融入每个Javer灵魂的东西。在讨论分代堆之前，我们先从头说起。

Java堆初始化会经过一个调用链：
```js
JNI_CreateJavaVM(prims/jni.cpp)
  ->JNI_CreateJavaVM_inner
    ->Threads::create_vm(runtime/thread.cpp)
      ->init_globals(runtime/init.cpp)
        ->universe_init(memory/universe.cpp)
          ->Universe::initialize_heap()
```
Universe模块（宇宙模块？）会负责高层次的Java堆的创建与初始化：
```cpp
jint Universe::initialize_heap() {
  // 创建Java堆
  _collectedHeap = create_heap();
  // 初始化Java堆
  jint status = _collectedHeap->initialize();
  if (status != JNI_OK) {
    return status;
  }
  // 使用的GC，如[38.500s][info][gc] Using G1
  log_info(gc)("Using %s", _collectedHeap->name());

  ThreadLocalAllocBuffer::set_max_size(Universe::heap()->max_tlab_size());

  if (UseCompressedOops) {
    if ((uint64_t)Universe::heap()->reserved_region().end() > UnscaledOopHeapMax) {
      Universe::set_narrow_oop_shift(LogMinObjAlignmentInBytes);
    }
    if ((uint64_t)Universe::heap()->reserved_region().end() <= OopEncodingHeapMax) {
      Universe::set_narrow_oop_base(0);
    }
    AOTLoader::set_narrow_oop_shift();

    Universe::set_narrow_ptrs_base(Universe::narrow_oop_base());

    LogTarget(Info, gc, heap, coops) lt;
    if (lt.is_enabled()) {
      ResourceMark rm;
      LogStream ls(lt);
      Universe::print_compressed_oops_mode(&ls);
    }
    Arguments::PropertyList_add(new SystemProperty("java.vm.compressedOopsMode",
                                                   narrow_oop_mode_to_string(narrow_oop_mode()),
                                                   false));
  }

  // TLAB初始化
  if (UseTLAB) {
    assert(Universe::heap()->supports_tlab_allocation(),
           "Should support thread-local allocation buffers");
    ThreadLocalAllocBuffer::startup_initialization();
  }
  return JNI_OK;
}
```

## 2. 创建Java堆
在`Universe::initialize_heap()`JVM初始化宇宙模块时调用create_heap()创建堆，这个函数会进一步调用位于`memory/allocation`模块的AllocateHeap，`memory/allocation`包括了堆分配，堆释放，堆重分配API。虚拟机所有的内存分配都是调用这些API进行的。但是这些APIs实际还没有做分配动作，它们只是包装一下底层分配，比如分配失败做一些处理工作，真正的内存分配是位于`runtime/os`模块。
上面的堆分配，堆释放，堆重分配API在OS模块中都有对应的低层次API：

![](mem_api.png)

说到`runtime/os`是低层次内存分配，那它到底有多低呢？打开源码看看，其实没有太低。并没有像OS这个名字一样使用操作系统的VirtualAlloc，sbrk，而是使用C/C++语言运行时的`malloc()/free()`进行分配/释放的。

内存是分配了，但是得有一个类来表示这片已分配、可GC的堆区。JVM有很多垃圾回收器，每个垃圾回收器处理的堆结构都是不一样的，比如G1GC处理的堆是由Region组成，CMS处理由老年代新生代组成的分代堆。这些不同的堆类型都继承自`gc/share/CollectedHeap`，抽象基类CollectedHeap表示所有堆都拥有的一些属性：

![](gc_heap_hierarchy.png)

```cpp
// hotspot\share\gc\shared\collectedHeap.hpp
class CollectedHeap : public CHeapObj<mtInternal> {
 private:
  GCHeapLog* _gc_heap_log;                  // GC日志
  MemRegion _reserved;                      // ？？？
 protected:
  bool _is_gc_active;                       // 是否正在GC。如果是stop-the-world就是true

  unsigned int _total_collections;          // Minor GC次数
  unsigned int _total_full_collections;     // Full GC次数

  GCCause::Cause _gc_cause;                 // 当前引发GC的原因
  GCCause::Cause _gc_lastcause;             // 上次引发GC的原因
  PerfStringVariable* _perf_gc_cause;       // perf上面两个
  PerfStringVariable* _perf_gc_lastcause;

  // functions
  ...
};
```
这样的堆是不能满足GC需求的。说好的新生代老年代呢，别急。上图继承模型中GenCollectedHeap正是分代堆的实现，它继承自CollectedHeap：
```cpp
//hotspot\share\gc\shared\genCollectedHeap.hpp
class GenCollectedHeap : public CollectedHeap {
public:
  enum GenerationType {
    YoungGen,
    OldGen
  };

protected:
  Generation* _young_gen;
  Generation* _old_gen;

private:
  GenerationSpec* _young_gen_spec;
  GenerationSpec* _old_gen_spec;

  // The singleton CardTable Remembered Set.
  CardTableRS* _rem_set;

  // The generational collector policy.
  GenCollectorPolicy* _gen_policy;

  SoftRefGenPolicy _soft_ref_gen_policy;

  // The sizing of the heap is controlled by a sizing policy.
  AdaptiveSizePolicy* _size_policy;

  GCPolicyCounters* _gc_policy_counters;

  // Indicates that the most recent previous incremental collection failed.
  // The flag is cleared when an action is taken that might clear the
  // condition that caused that incremental collection to fail.
  bool _incremental_collection_failed;

  // In support of ExplicitGCInvokesConcurrent functionality
  unsigned int _full_collections_completed;

  // functions
  ...
};
```
