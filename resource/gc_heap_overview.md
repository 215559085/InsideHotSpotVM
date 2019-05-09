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
### 2.1 使用os::malloc()分配内存
在`Universe::initialize_heap()`JVM初始化宇宙模块时调用create_heap()创建堆，这个函数会进一步调用位于`memory/allocation`模块的AllocateHeap，但是这些APIs实际还没有做分配动作，它们只是包装底层分配，处理一下分配失败，真正的内存分配是位于底层`runtime/os`模块：

![](mem_api.png)

说到`runtime/os`是底层内存分配，那它到底有多底层？打开源码看看，并没有像OS这个名字一样使用操作系统的VirtualAlloc，sbrk，而是使用C/C++语言运行时的`malloc()/free()`进行分配/释放的。

### 2.2 堆在JVM中的表示
内存是分配了，但是得有一个类来表示这片已分配、可GC的堆区。JVM有很多垃圾回收器，每个垃圾回收器处理的堆结构都是不一样的，比如G1GC处理的堆是由Region组成，CMS处理由老年代新生代组成的分代堆。这些不同的堆类型都继承自`gc/share/CollectedHeap`，抽象基类CollectedHeap表示所有堆都拥有的一些属性：

![](gc_heap_hierarchy.png)

```cpp
// hotspot\share\gc\shared\collectedHeap.hpp
class CollectedHeap : public CHeapObj<mtInternal> {
 private:
  GCHeapLog* _gc_heap_log;                  // GC日志
  MemRegion _reserved;                      // 堆内存表示
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
上面的**_reserved**就表示Java堆这片连续的地址，它包含堆的其实地址和大小，即`[start,start+size]`。然而这样的堆是不能满足GC需求的，Full GC处理老年代，Minor GC处理新生代，可是这两个“代”都没有在CollectedHeap中体现。翻翻上图继承模型，GenCollectedHeap才是分代堆：
```cpp
//hotspot\share\gc\shared\genCollectedHeap.hpp
class GenCollectedHeap : public CollectedHeap {
public:
  enum GenerationType {
    YoungGen,
    OldGen
  };

protected:
  Generation* _young_gen;     // 新生代
  Generation* _old_gen;       // 老年代
  ...
};
```
看到GenCollectedHeap里面的`_young_gen`和`_old_gen`基本就稳了。它继承自CollectedHeap，其中CollectedHeap里面的_reserved表示整个堆，GenCollectedHeap的新生代和老年代进一步划分_reserved。这个划分工作发生在堆初始化中

## 2. 初始化Java堆

```cpp
// hotspot\share\gc\shared\genCollectedHeap.cpp
jint GenCollectedHeap::initialize() {
  // While there are no constraints in the GC code that HeapWordSize
  // be any particular value, there are multiple other areas in the
  // system which believe this to be true (e.g. oop->object_size in some
  // cases incorrectly returns the size in wordSize units rather than
  // HeapWordSize).
  guarantee(HeapWordSize == wordSize, "HeapWordSize must equal wordSize");

  // Allocate space for the heap.

  char* heap_address;
  ReservedSpace heap_rs;

  size_t heap_alignment = collector_policy()->heap_alignment();

  heap_address = allocate(heap_alignment, &heap_rs);

  if (!heap_rs.is_reserved()) {
    vm_shutdown_during_initialization(
      "Could not reserve enough space for object heap");
    return JNI_ENOMEM;
  }

  initialize_reserved_region((HeapWord*)heap_rs.base(), (HeapWord*)(heap_rs.base() + heap_rs.size()));

  _rem_set = create_rem_set(reserved_region());
  _rem_set->initialize();
  CardTableBarrierSet *bs = new CardTableBarrierSet(_rem_set);
  bs->initialize();
  BarrierSet::set_barrier_set(bs);

  ReservedSpace young_rs = heap_rs.first_part(_young_gen_spec->max_size(), false, false);
  _young_gen = _young_gen_spec->init(young_rs, rem_set());
  heap_rs = heap_rs.last_part(_young_gen_spec->max_size());

  ReservedSpace old_rs = heap_rs.first_part(_old_gen_spec->max_size(), false, false);
  _old_gen = _old_gen_spec->init(old_rs, rem_set());
  clear_incremental_collection_failed();

  return JNI_OK;
}
```