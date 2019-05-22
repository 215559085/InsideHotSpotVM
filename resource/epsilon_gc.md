# [Inside HotSpot] Epsilon GC

## 1. Epislon GC简介
Epislon GC源于RedHat开发者Aleksey Shipilëv提交的一份JEP草案，该GC只做**内存分配**而不做**内存回收**(reclaim)，当堆空间耗尽关闭JVM即可，因为它不做任何垃圾回收工作，所以又叫No-op GC。也因为它简单，很适合用来入门OpenJDK GC源码，看看一个最小化可行的垃圾回收器应该具备哪些功能。

Epislon GC源码位于`gc/epsilon`:
```bash
hotspot/share/gc/epsilon
	epsilon_globals.hpp           # GC提供的一些JVM参数，如-XX:+UseEpsilonGC
	epsilonArguments.cpp
	epsilonArguments.hpp          # GC参数在JVM中的表示，是否使用TLAB等，是否开启EpsilonGC等
	epsilonBarrierSet.cpp
	epsilonBarrierSet.hpp
	epsilonCollectorPolicy.hpp    # 垃圾回收策略，堆初始化大小，最小，最大值，对齐等信息
	epsilonHeap.cpp			          # 包含堆初始化，内存分配，垃圾回收接口
	epsilonHeap.hpp               # 真正的堆表示，EpsilonGC独有
	epsilonMemoryPool.cpp
	epsilonMemoryPool.hpp
	epsilonMonitoringSupport.cpp  #
	epsilonMonitoringSupport.hpp  #
	epsilonThreadLocalData.hpp    #
	vmStructs_epsilon.hpp         #
```
另外为了启动EpislonGC需要添加JVM参数`-XX:+UnlockExperimentalVMOptions -XX:+UseEpsilonGC`，为了输出GC日志查看详细过程添加JVM参数`-Xlog:gc*=trace`(仅限fastdebug版JVM)

## 2. EpsilonGC创建
虚拟机在创建早期会调用GCArguments::initialize()初始化GC参数，然后创建中期会配置好堆空间并调用GCArguments::create_heap()创建堆：
```cpp
// hotspot\share\runtime\thread.cpp
// 创建早期
jint Threads::create_vm(JavaVMInitArgs* args, bool* canTryAgain) {
  ...
  jint ergo_result = Arguments::apply_ergo(); // GCArguments::initialize()
  
}
// hotspot\share\memory\universe.cpp
// 创建中期
jint Universe::initialize_heap() {
  _collectedHeap = create_heap();			  // GCArguments::create_heap()
  jint status = _collectedHeap->initialize();
  ...
}
```
首先我们创建GCArguments的子类用来表示EpsilonGC的GC参数：
```cpp
class EpsilonArguments : public GCArguments {
public:
  virtual void initialize(); 
  virtual size_t conservative_max_heap_alignment();
  virtual CollectedHeap* create_heap();
};
```
GCArguments::initialize()可以初始化GC参数，不过并没有什么可以初始化的，Epsilon GC只是简单检查了一下是否有`-XX:+UseEpsilonGC`。
```cpp
CollectedHeap* EpsilonArguments::create_heap() {
  return create_heap_with_policy<EpsilonHeap, EpsilonCollectorPolicy>();
}
```
GCArguments::create_heap()更简单，直接根据堆的类型(EpsilonHeap，真正的Java堆空间表示)和垃圾回收器策略(EpsilonCollectorPolicy，堆初始化大小，最小，最大值，对齐等信息)创建堆。这个新创建的堆是EpsilonHeap，关于Java堆体系可以参见[Java分代堆](gc_heap_overview.md)，简单来说，每种垃圾回收器都有一个独属于自己的可垃圾回收堆，它需要继承自CollectedHeap：
```cpp
// \hotspot\share\gc\epsilon\epsilonHeap.hpp
class EpsilonHeap : public CollectedHeap {
  friend class VMStructs;
private:
  EpsilonCollectorPolicy* _policy;
  SoftRefPolicy _soft_ref_policy;
  EpsilonMonitoringSupport* _monitoring_support;
  MemoryPool* _pool;
  GCMemoryManager _memory_manager;
  ContiguousSpace* _space;
  VirtualSpace _virtual_space;
  size_t _max_tlab_size;
  size_t _step_counter_update;
  size_t _step_heap_print;
  int64_t _decay_time_ns;
  volatile size_t _last_counter_update;
  volatile size_t _last_heap_print;

public:
  static EpsilonHeap* heap();

  EpsilonHeap(EpsilonCollectorPolicy* p) :
          _policy(p),
          _memory_manager("Epsilon Heap", "") {};

  virtual jint initialize();
  virtual void post_initialize();
  virtual void initialize_serviceability();

  virtual GrowableArray<GCMemoryManager*> memory_managers();
  virtual GrowableArray<MemoryPool*> memory_pools();

  virtual size_t max_capacity() const { return _virtual_space.reserved_size();  }
  virtual size_t capacity()     const { return _virtual_space.committed_size(); }
  virtual size_t used()         const { return _space->used(); }
  ...
};
```
CollectedHeap是一个抽象基类，里面有很多纯虚函数需要子类重写，创建完堆之后JVM会初始化堆，初始化分两步走：EpsilonHeap::initialize和EpsilonHeap::post_initialize。initialize是重头戏，它做了最重要的工作，包括堆内存的申请，barrier的设置：
```cpp

jint EpsilonHeap::initialize() {
  size_t align = _policy->heap_alignment();
  size_t init_byte_size = align_up(_policy->initial_heap_byte_size(), align);
  size_t max_byte_size  = align_up(_policy->max_heap_byte_size(), align);

  // 向虚拟内存空间reserve内存然后commit部分
  // [------------------------------|------------------]
  // [           committed          |      reserved    ]
  // 0                           init_byte_size      max_byte_size
  // low/low_boundary            high                high_boundary
  ReservedSpace heap_rs = Universe::reserve_heap(max_byte_size, align);
  _virtual_space.initialize(heap_rs, init_byte_size);

  MemRegion committed_region((HeapWord*)_virtual_space.low(),          (HeapWord*)_virtual_space.high());
  MemRegion  reserved_region((HeapWord*)_virtual_space.low_boundary(), (HeapWord*)_virtual_space.high_boundary());

  initialize_reserved_region(reserved_region.start(), reserved_region.end());
  
  // 用ContiguousSpace表示这片(连续)堆内存
  _space = new ContiguousSpace();
  _space->initialize(committed_region, /* clear_space = */ true, /* mangle_space = */ true);

  // 计算最大tlab大小
  _max_tlab_size = MIN2(CollectedHeap::max_tlab_size(), align_object_size(EpsilonMaxTLABSize / HeapWordSize));
  _step_counter_update = MIN2<size_t>(max_byte_size / 16, EpsilonUpdateCountersStep);
  _step_heap_print = (EpsilonPrintHeapSteps == 0) ? SIZE_MAX : (max_byte_size / EpsilonPrintHeapSteps);
  _decay_time_ns = (int64_t) EpsilonTLABDecayTime * NANOSECS_PER_MILLISEC;

  // 开启监控，可以感知空间used，capacity等信息
  _monitoring_support = new EpsilonMonitoringSupport(this);
  _last_counter_update = 0;
  _last_heap_print = 0;

  // 创建barrier
  BarrierSet::set_barrier_set(new EpsilonBarrierSet());

  // 完成初始化，输出配置信息
  ...

  return JNI_OK;
}

void EpsilonHeap::post_initialize() {
  CollectedHeap::post_initialize();
}

void EpsilonHeap::initialize_serviceability() {
  // post_initialize会调用该方法，将堆空间加入内存池管理
  _pool = new EpsilonMemoryPool(this);
  _memory_manager.add_pool(_pool);
}
```

## 3. 内存分配
EpsilonGC支持普通内存分配和TLAB内存分配，前者接口是mem_allocate()，后者是allocate_new_tlab()。

### 3.1 普通内存分配
```cpp
HeapWord* EpsilonHeap::mem_allocate(size_t size, bool *gc_overhead_limit_was_exceeded) {
  *gc_overhead_limit_was_exceeded = false;
  return allocate_work(size);
}

HeapWord* EpsilonHeap::allocate_work(size_t size) {
  // 无锁并发分配
  HeapWord* res = _space->par_allocate(size);

  // 如果分配失败，循环扩容
  while (res == NULL) {
    MutexLockerEx ml(Heap_lock);
    // 先扩容（之前virtual space有一部分是reserved但是没有committed）
    // 剩余可用空间大小
    size_t space_left = max_capacity() - capacity();
    // 需要空间大小
    size_t want_space = MAX2(size, EpsilonMinHeapExpand);

    if (want_space < space_left) {
      bool expand = _virtual_space.expand_by(want_space);
    } else if (size < space_left) {
      bool expand = _virtual_space.expand_by(space_left);
    } else {
      // 如果扩容失败则分配失败，返回null
      return NULL;
    }
    // 扩容成功，设置virtual_space的committed尾为新大小
    _space->set_end((HeapWord *) _virtual_space.high());
    // 再次尝试分配内存
    res = _space->par_allocate(size);
  }

  size_t used = _space->used();

  // 如果分配成功，更新计数器
  {
    size_t last = _last_counter_update;
    if ((used - last >= _step_counter_update) && Atomic::cmpxchg(used, &_last_counter_update, last) == last) {
      _monitoring_support->update_counters();
    }
  }

  // 输出堆占用情况
  {
    size_t last = _last_heap_print;
    if ((used - last >= _step_heap_print) && Atomic::cmpxchg(used, &_last_heap_print, last) == last) {
      log_info(gc)("Heap: " SIZE_FORMAT "M reserved, " SIZE_FORMAT "M (%.2f%%) committed, " SIZE_FORMAT "M (%.2f%%) used",
                   max_capacity() / M,
                   capacity() / M,
                   capacity() * 100.0 / max_capacity(),
                   used / M,
                   used * 100.0 / max_capacity());
    }
  }

  return res;
}
```
为了看到扩容的发生，我们可以修改一下代码，在循环扩容处添加日志记录：
```cpp
    log_info(gc)("Heap expansion: committed %lluM, needs %lluM, reserved %lluM",
        capacity() / M, 
        want_space/M,
        max_capacity() / M);
```
编译得到JVM，然后准备一段Java代码：
```java
package com.github.kelthuzadx;
class Foo{
    private static int _1MB = 1024*1024;
    private byte[] b = new byte[_1MB*200];
}
public class GCBaby {
    public static void main(String[] args) {
       new Foo();new Foo();
    }
}
```
启动JVM时添加参数`-XX:+UnlockExperimentalVMOptions  -Xms128m -Xmx512m  -XX:+UseEpsilonGC -Xlog:gc*=info `，最终我们可以看到为了分配400M的对象，初始大小128M的堆进行了3次扩容：
```bash
[8.904s][info][gc] Heap expansion: committed 128M, needs 128M, reserved 512M
[8.904s][info][gc] Heap: 512M reserved, 256M (50.00%) committed, 207M (40.51%) used

[9.010s][info][gc] Heap expansion: committed 256M, needs 128M, reserved 512M
[9.010s][info][gc] Heap expansion: committed 384M, needs 128M, reserved 512M
[9.011s][info][gc] Heap: 512M reserved, 512M (100.00%) committed, 407M (79.57%) used
```
### 3.2 TLAB内存分配
```cpp
HeapWord* EpsilonHeap::allocate_new_tlab(size_t min_size,
                                         size_t requested_size,
                                         size_t* actual_size) {
  Thread* thread = Thread::current();

  // Defaults in case elastic paths are not taken
  bool fits = true;
  size_t size = requested_size;
  size_t ergo_tlab = requested_size;
  int64_t time = 0;

  if (EpsilonElasticTLAB) {
    ergo_tlab = EpsilonThreadLocalData::ergo_tlab_size(thread);

    if (EpsilonElasticTLABDecay) {
      int64_t last_time = EpsilonThreadLocalData::last_tlab_time(thread);
      time = (int64_t) os::javaTimeNanos();

      assert(last_time <= time, "time should be monotonic");

      // If the thread had not allocated recently, retract the ergonomic size.
      // This conserves memory when the thread had initial burst of allocations,
      // and then started allocating only sporadically.
      if (last_time != 0 && (time - last_time > _decay_time_ns)) {
        ergo_tlab = 0;
        EpsilonThreadLocalData::set_ergo_tlab_size(thread, 0);
      }
    }

    // If we can fit the allocation under current TLAB size, do so.
    // Otherwise, we want to elastically increase the TLAB size.
    fits = (requested_size <= ergo_tlab);
    if (!fits) {
      size = (size_t) (ergo_tlab * EpsilonTLABElasticity);
    }
  }

  // Always honor boundaries
  size = MAX2(min_size, MIN2(_max_tlab_size, size));

  // Always honor alignment
  size = align_up(size, MinObjAlignment);

  // Check that adjustments did not break local and global invariants
  assert(is_object_aligned(size),
         "Size honors object alignment: " SIZE_FORMAT, size);
  assert(min_size <= size,
         "Size honors min size: "  SIZE_FORMAT " <= " SIZE_FORMAT, min_size, size);
  assert(size <= _max_tlab_size,
         "Size honors max size: "  SIZE_FORMAT " <= " SIZE_FORMAT, size, _max_tlab_size);
  assert(size <= CollectedHeap::max_tlab_size(),
         "Size honors global max size: "  SIZE_FORMAT " <= " SIZE_FORMAT, size, CollectedHeap::max_tlab_size());

  if (log_is_enabled(Trace, gc)) {
    ResourceMark rm;
    log_trace(gc)("TLAB size for \"%s\" (Requested: " SIZE_FORMAT "K, Min: " SIZE_FORMAT
                          "K, Max: " SIZE_FORMAT "K, Ergo: " SIZE_FORMAT "K) -> " SIZE_FORMAT "K",
                  thread->name(),
                  requested_size * HeapWordSize / K,
                  min_size * HeapWordSize / K,
                  _max_tlab_size * HeapWordSize / K,
                  ergo_tlab * HeapWordSize / K,
                  size * HeapWordSize / K);
  }

  // All prepared, let's do it!
  HeapWord* res = allocate_work(size);

  if (res != NULL) {
    // Allocation successful
    *actual_size = size;
    if (EpsilonElasticTLABDecay) {
      EpsilonThreadLocalData::set_last_tlab_time(thread, time);
    }
    if (EpsilonElasticTLAB && !fits) {
      // If we requested expansion, this is our new ergonomic TLAB size
      EpsilonThreadLocalData::set_ergo_tlab_size(thread, size);
    }
  } else {
    // Allocation failed, reset ergonomics to try and fit smaller TLABs
    if (EpsilonElasticTLAB) {
      EpsilonThreadLocalData::set_ergo_tlab_size(thread, 0);
    }
  }

  return res;
}

```

## 4. 垃圾回收
前面已经提到EpsilonGC只管分配不管释放，所以垃圾回收接口极其简单，就只需要记录一下GC计数信息即可：
```cpp
// hotspot\share\gc\epsilon\epsilonHeap.cpp
void EpsilonHeap::collect(GCCause::Cause cause) {
  log_info(gc)("GC request for \"%s\" is ignored", GCCause::to_string(cause));
  _monitoring_support->update_counters();
}

```
然后？就没啦！大功告成。如果想做一个有实际垃圾回收效果的GC可以继续阅读[Do It Yourself (OpenJDK) Garbage Collector](https://shipilev.net/jvm/diy-gc/#_epsilon_gc)，这篇文章在Epislon GC上增加了一个基于标记-压缩(Mark-Compact)算法的垃圾回收机制。

## 引用
\[1\] [Build Your Own GC in 20 Minutes](https://shipilev.net/jvm/diy-gc/kennke-fosdem-2019.webm)
\[2\] [Do It Yourself (OpenJDK) Garbage Collector](https://shipilev.net/jvm/diy-gc/#_epsilon_gc)
\[3\] [JEP 318: Epsilon: A No-Op Garbage Collector (Experimental)](https://openjdk.java.net/jeps/318)