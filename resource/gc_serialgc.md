# [Inside HotSpot] Serial GC

## 1. 脑动触发垃圾回收
我们有这样一个类：
```java
class Foo{
    private static int _1MB = 1024*1024;

    private byte[] b = new byte[_1MB*40];
}
```
为了深入理解JVM Serial GC工作过程，还得"精心"安排一下Java堆划分：
```js
-Xmx100m -Xms100m -Xmn40m -Xlog:gc* -XX:+UseSerialGC
-----------------------------------------------------
Heap address: 0x00000000f9c00000, size: 100 MB, Compressed Oops mode: 32-bit
  def new generation   total 36864K, used 3277K
   eden space 32768K,  10% used
   from space 4096K,   0% used
   to   space 4096K,   0% used 
  tenured generation   total 61440K, used 40960K 
    the space 61440K,  66% used 
  Metaspace       used 572K, capacity 4500K, committed 4864K, reserved 1056768K
   class space    used 48K, capacity 388K, committed 512K, reserved 1048576K
```
上面的参数将设置新生代为40m，老年代为60m，然后使用Serial GC并输出gc信息，另外`-Xlog:gc`表示输出最基本的GC信息，`-Xlog:gc*`表示输出详细GC信息，`-Xlog:gc*=trace`表示输出最详细信息，包括卡表建立，引用地址等，不过这不适用于production级虚拟机。
再看看Foo对象，它占用40M空间，为了触发Full GC，可以new两个Foo：
```java
public class GCBaby {
    public static void main(String[] args) {
       new Foo();new Foo();
    }
}
```
编译执行，得到输出：
```js
 Heap address: 0x00000000f9c00000, size: 100 MB, Compressed Oops mode: 32-bit
 GC(0) Pause Young (Allocation Failure)
 GC(1) Pause Full (Allocation Failure)
 GC(1) Phase 1: Mark live objects
 GC(1) Phase 1: Mark live objects 1.136ms
 GC(1) Phase 2: Compute new object addresses
 GC(1) Phase 2: Compute new object addresses 0.170ms
 GC(1) Phase 3: Adjust pointers
 GC(1) Phase 3: Adjust pointers 0.435ms
 GC(1) Phase 4: Move objects
 GC(1) Phase 4: Move objects 0.208ms
 GC(1) Pause Full (Allocation Failure) 40M->0M(96M) 2.102ms
 GC(0) DefNew: 2621K->0K(36864K)
 GC(0) Tenured: 40960K->795K(61440K)
 GC(0) Metaspace: 562K->562K(1056768K)
 GC(0) Pause Young (Allocation Failure) 42M->0M(96M) 3.711ms
 GC(0) User=0.00s Sys=0.00s Real=0.00s

  def new generation   total 36864K, used 1638K 
   eden space 32768K,   5% used 
   from space 4096K,   0% used 
   to   space 4096K,   0% used 
  tenured generation   total 61440K, used 41755K
    the space 61440K,  67% used 
  Metaspace       used 608K, capacity 4500K, committed 4864K, reserved 1056768K
   class space    used 51K, capacity 388K, committed 512K, reserved 1048576K
```
这时候老年代使用了40M，说明Full GC清除了一个Foo，还剩下一个Foo()。上面的日志则是GC的详细过程，这篇文章将围绕上述过程展开。

## 2. 久任代的Full GC
```cpp
// hotspot\share\gc\serial\genMarkSweep.cpp
void TenuredGeneration::collect(bool   full,
                                bool   clear_all_soft_refs,
                                size_t size,
                                bool   is_tlab) {
  GenCollectedHeap* gch = GenCollectedHeap::heap();

  // Temporarily expand the span of our ref processor, so
  // refs discovery is over the entire heap, not just this generation
  ReferenceProcessorSpanMutator
    x(ref_processor(), gch->reserved_region());

  STWGCTimer* gc_timer = GenMarkSweep::gc_timer();
  gc_timer->register_gc_start();

  SerialOldTracer* gc_tracer = GenMarkSweep::gc_tracer();
  gc_tracer->report_gc_start(gch->gc_cause(), gc_timer->gc_start());

  gch->pre_full_gc_dump(gc_timer);

  GenMarkSweep::invoke_at_safepoint(ref_processor(), clear_all_soft_refs);

  gch->post_full_gc_dump(gc_timer);

  gc_timer->register_gc_end();

  gc_tracer->report_gc_end(gc_timer->gc_end(), gc_timer->time_partitions());
}
```

```cpp
// hotspot\share\gc\serial\genMarkSweep.cpp
void GenMarkSweep::invoke_at_safepoint(ReferenceProcessor* rp, bool clear_all_softrefs) {
  GenCollectedHeap* gch = GenCollectedHeap::heap();

  // hook up weak ref data so it can be used during Mark-Sweep
  assert(ref_processor() == NULL, "no stomping");
  assert(rp != NULL, "should be non-NULL");
  set_ref_processor(rp);
  rp->setup_policy(clear_all_softrefs);

  gch->trace_heap_before_gc(_gc_tracer);

  // When collecting the permanent generation Method*s may be moving,
  // so we either have to flush all bcp data or convert it into bci.
  CodeCache::gc_prologue();

  // Increment the invocation count
  _total_invocations++;

  // Capture used regions for each generation that will be
  // subject to collection, so that card table adjustments can
  // be made intelligently (see clear / invalidate further below).
  gch->save_used_regions();

  allocate_stacks();

  mark_sweep_phase1(clear_all_softrefs);

  mark_sweep_phase2();

  // Don't add any more derived pointers during phase3
#if COMPILER2_OR_JVMCI
  assert(DerivedPointerTable::is_active(), "Sanity");
  DerivedPointerTable::set_active(false);
#endif

  mark_sweep_phase3();

  mark_sweep_phase4();

  restore_marks();

  // Set saved marks for allocation profiler (and other things? -- dld)
  // (Should this be in general part?)
  gch->save_marks();

  deallocate_stacks();

  // If compaction completely evacuated the young generation then we
  // can clear the card table.  Otherwise, we must invalidate
  // it (consider all cards dirty).  In the future, we might consider doing
  // compaction within generations only, and doing card-table sliding.
  CardTableRS* rs = gch->rem_set();
  Generation* old_gen = gch->old_gen();

  // Clear/invalidate below make use of the "prev_used_regions" saved earlier.
  if (gch->young_gen()->used() == 0) {
    // We've evacuated the young generation.
    rs->clear_into_younger(old_gen);
  } else {
    // Invalidate the cards corresponding to the currently used
    // region and clear those corresponding to the evacuated region.
    rs->invalidate_or_clear(old_gen);
  }

  CodeCache::gc_epilogue();
  JvmtiExport::gc_epilogue();

  // refs processing: clean slate
  set_ref_processor(NULL);

  // Update heap occupancy information which is used as
  // input to soft ref clearing policy at the next gc.
  Universe::update_heap_info_at_gc();

  // Update time of last gc for all generations we collected
  // (which currently is all the generations in the heap).
  // We need to use a monotonically non-decreasing time in ms
  // or we will see time-warp warnings and os::javaTimeMillis()
  // does not guarantee monotonicity.
  jlong now = os::javaTimeNanos() / NANOSECS_PER_MILLISEC;
  gch->update_time_of_last_gc(now);

  gch->trace_heap_after_gc(_gc_tracer);
}
```
## 1. 阶段1：标记存活对象
Serial GC的Full GC第一阶段对应GC日志的
```js
 GC(1) Phase 1: Mark live objects
```
它会遍历GC Root标记可达对象，处理`java.lang.ref.*`特殊引用类型，清除字符串常量池无效字符串等等，这一阶段占用了标记清除大部分的时间。下面是阶段一最重要的小步骤。

### 1.1 遍历GC Root(process_roots())
JVM在`process_string_table_roots()`和`process_roots()`中会遍历所有类型的GC Root，然后使用`XX::oops_do(root_closure)`从该GC Root出发标记所有存活对象。`XX`表示GC Root类型，`root_closure`表示**标记存活对象**的方法。GC模块有很多闭包(closure)，它们代表的是一段代码、一种行为。root_closure就是一个`MarkSweep::FollowRootClosure`闭包。这个闭包很强大，给它一个对象，就能标记这个对象，迭代标记对象的成员，以及对象所在的栈的所有对象及其成员：
```cpp
// hotspot\share\gc\serial\markSweep.cpp
void MarkSweep::FollowRootClosure::do_oop(oop* p)       { follow_root(p); }

template <class T> inline void MarkSweep::follow_root(T* p) {
  // 如果引用指向的对象不为空且未标记
  T heap_oop = RawAccess<>::oop_load(p);
  if (!CompressedOops::is_null(heap_oop)) {
    oop obj = CompressedOops::decode_not_null(heap_oop);
    if (!obj->mark_raw()->is_marked()) {
      mark_object(obj);   // 标记对象
      follow_object(obj); // 标记对象的成员 
    }
  }
  follow_stack();       // 标记引用所在栈
}

// 如果对象是数组对象则标记数组，否则标记对象的成员
inline void MarkSweep::follow_object(oop obj) {
  if (obj->is_objArray()) {
    MarkSweep::follow_array((objArrayOop)obj);
  } else {
    obj->oop_iterate(&mark_and_push_closure);
  }
}

// 标记引用所在的整个栈
void MarkSweep::follow_stack() {
  do {
    // 如果待标记栈不为空则逐个标记
    while (!_marking_stack.is_empty()) {
      oop obj = _marking_stack.pop();
      follow_object(obj);
    }
    // 如果对象数组栈不为空则逐个标记
    if (!_objarray_stack.is_empty()) {
      ObjArrayTask task = _objarray_stack.pop();
      follow_array_chunk(objArrayOop(task.obj()), task.index());
    }
  } while (!_marking_stack.is_empty() || !_objarray_stack.is_empty());
}

// 标记数组的类型的Class和数组成员，比如String[] p = new String[2]
// 对p标记会同时标记java.lang.Class，p[1],p[2]
inline void MarkSweep::follow_array(objArrayOop array) {
  MarkSweep::follow_klass(array->klass());
  if (array->length() > 0) {
    MarkSweep::push_objarray(array, 0);
  }
}
```
既然走到这里了不如看看JVM是如何标记对象的：
```cpp
inline void MarkSweep::mark_object(oop obj) {
  // 获取对象的mark word
  markOop mark = obj->mark_raw();
  // 设置gc标记
  obj->set_mark_raw(markOopDesc::prototype()->set_marked());
  // 垃圾回收器视情况保留对象的gc标志
  if (mark->must_be_preserved(obj)) {
    preserve_mark(obj, mark);
  }
}
```
对象的mark work有32bits或者64bits，取决于CPU架构和UseCompressedOops：
```js
// hotspot\share\oops\markOop.hpp
32 位mark lword：
          hash:25 ------------>| age:4    biased_lock:1 lock:2 (normal object)
          JavaThread*:23 epoch:2 age:4    biased_lock:1 lock:2 (biased object)
          size:32 ------------------------------------------>| (CMS free block)
          PromotedObject*:29 ---------->| promo_bits:3 ----->| (CMS promoted object)

最后的lock2位有不同含义：
          [ptr             | 00]  locked             ptr指向栈上真正的对象头
          [header      | 0 | 01]  unlocked           普通对象头
          [ptr             | 10]  monitor            碰撞锁
          [ptr             | 11]  marked             GC标记
```
原来垃圾回收的标记就是对每个对象mark word最后两位置为11，可是如果最后两位用于其他用途怎么办？比如这个对象的最后两位表示碰撞锁，那GC就不能对它进行标记了，所以垃圾回收器还需要视情况在额外区域保留GC标志。回到之前的话题，GC Root有很多，有的是我们耳熟能详的，有的则是略微少见，它们都包含可进行标记的引用，会视情况进行单线程标记或者并发标记：

+ 所有已加载的类(`ClassLoaderDataGraph::roots_cld_do`)
+ 所有Java线程当前栈帧的引用和虚拟机内部线程(`Threads::possibly_parallel_oops_do`)
+ JVM内部使用的引用(`Universe::oopds_do`和`SystemDictionary::oops_do`)
+ JNI handles(`JNIHandles::oops_do`)
+ 所有synchronized锁住的对象引用(`ObjectSynchronizer::oops_do`)
+ java.lang.management对象(`Management::oops_do`)
+ JVMTI导出(`JvmtiExport::oops_do`)
+ AOT代码的堆(`AOTLoader::oops_do`)
+ code cache(`CodeCache::blobs_do`)
+ String常量池(`StringTable::oops_do`)

最后JVM使用CAS(Atomic::cmpxchg)自旋锁等待标记任务。如果任务全部完成，即标记线程和完成计数相等，就结束阻塞。

### 1.2 处理java.lang.ref.*特殊引用类型(ReferenceProcessor)
当对象标记完成后jvm还会使用`ref_processor()->process_discovered_references()`对特殊引用类型做一些处理。所谓特殊引用类型即：

+ 弱引用：
+ 软引用：
+ 虚引用：
+ final引用:重写了finalize()方法的引用。

如果待回收队列里面存在final引用，就使用`DefNewGeneration::FastKeepAliveClosure`闭包将对象再次标记为存活，然后放入fianlize队列等待Java层面的FinalizeThread调用该对象的finalize()方法，当调用完成后下次GC该对象就会被老老实实回收：
```cpp
size_t ReferenceProcessor::process_final_keep_alive_work(DiscoveredList& refs_list,
                                                         OopClosure*     keep_alive,
                                                         VoidClosure*    complete_gc) {
  DiscoveredListIterator iter(refs_list, keep_alive, NULL);
  while (iter.has_next()) {
    // 让引用关联的对象复活
    iter.load_ptrs(DEBUG_ONLY(false /* allow_null_referent */));
    iter.make_referent_alive();
    java_lang_ref_Reference::set_next_raw(iter.obj(), iter.obj());
    // 加入finalize队列等待FinalizeThread调用该对象的finalize方法
    iter.enqueue();
    iter.next();
  }
  iter.complete_enqueue();
  complete_gc->do_void();
  refs_list.clear();
  return iter.removed();
}
```
所以说重写了finalize()的方法不得不到析构的语义，还会耽误GC回收对象。



## 1. 阶段2：计算对象新地址