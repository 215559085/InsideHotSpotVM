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

## 3. 分配内存

## 4. 垃圾回收
前面已经提到EpsilonGC只管分配不管释放，所以垃圾回收接口极其简单，就只需要记录一下GC计数信息即可：
```cpp
// hotspot\share\gc\epsilon\epsilonHeap.cpp
void EpsilonHeap::collect(GCCause::Cause cause) {
  log_info(gc)("GC request for \"%s\" is ignored", GCCause::to_string(cause));
  _monitoring_support->update_counters();
}

```
然后？就没啦！大功告成，我们也可以试着20分钟自制HotSpot垃圾回收器

## 引用
\[1\] [Build Your Own GC in 20 Minutes](https://shipilev.net/jvm/diy-gc/kennke-fosdem-2019.webm)
\[2\] [Do It Yourself (OpenJDK) Garbage Collector](https://shipilev.net/jvm/diy-gc/#_epsilon_gc)
\[3\] [JEP 318: Epsilon: A No-Op Garbage Collector (Experimental)](https://openjdk.java.net/jeps/318)