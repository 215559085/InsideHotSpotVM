# [Inside HotSpot] UseParallelGC和UseParallelOldGC的区别

JVM的垃圾回收器和命名真是个重灾区，一大堆`-XX:+UseParallel`，`-XX:+UseParallelOldGC`，`-XX:+UseParNewGC`，`-XX:+UseConcMarkSweepGC`咋一看很容易混淆，而且JDK升个级某个GC就可能不见了。那么这些神仙参数到底都是干什么的呢，我们先来看看到底都有哪些类型的GC：
```cpp
//hotspot\share\gc\shared\gc_globals.hpp 
product(bool, UseConcMarkSweepGC, false,                                  
      "Use Concurrent Mark-Sweep GC in the old generation")             
product(bool, UseSerialGC, false,                                         
      "Use the Serial garbage collector")                                               
product(bool, UseG1GC, false,                                             
      "Use the Garbage-First garbage collector")  
product(bool, UseParallelGC, false,                                      
      "Use the Parallel Scavenge garbage collector")                                         
product(bool, UseParallelOldGC, false,                                    
      "Use the Parallel Old garbage collector")     
experimental(bool, UseEpsilonGC, false,                                   
      "Use the Epsilon (no-op) garbage collector")                                       
experimental(bool, UseZGC, false,                                         
      "Use the Z garbage collector")          
```
首先，好消息是`ParNewGC`在[JDK9中弃用了，JDK10中已经完全移除](https://bugs.openjdk.java.net/browse/JDK-8151084)了，它的理想代替物是G1GC。然后openjdk12中现存的GC有：

+ ConcMarkSweepGC
+ SerialGC
+ G1GC：
+ ParallelGC
+ ParallelOldGC
+ EpsilonGC
+ ZGC
+ ShenandoahGC ( OpenJDK12上游的新GC，我的源码拉的早，就没有它了)

它们在源码中都有对应的独立目录：
```bash
λ tree .
├─gc
│  ├─cms	  # UseConcMarkSweepGC
│  ├─epsilon  # UseEpsilonGC
│  ├─g1       # UseG1GC
│  ├─parallel # UseParallelGC && UseParallelOldGC
│  ├─serial   # UseSerialGC
│  ├─shared   # 所有GC共享的代码
│  └─z        # UseZGC
```
本文将要简要分析Parallel GC和ParallelOld GC的区别。
要想找不同很简单：对着源码目录搜索一下UseParallelGC/UseParallelOldGC标志，可以得到所有源码使用，而且找出来的结果通常是两者伴随出现的，看来方法是没问题的。我们重点关注几个地方，首先看看`parallelArgument.cpp`，它会负责GC早期的参数处理（可以参见[EpsilonGC示例](gc_epsilongc.md)）：
```cpp
// hotspot\share\gc\parallel\parallelArguments.cpp
void ParallelArguments::initialize() {
  GCArguments::initialize();
  assert(UseParallelGC || UseParallelOldGC, "Error");
  // Enable ParallelOld unless it was explicitly disabled (cmd line or rc file).
  if (FLAG_IS_DEFAULT(UseParallelOldGC)) {
    FLAG_SET_DEFAULT(UseParallelOldGC, true);
  }
  FLAG_SET_DEFAULT(UseParallelGC, true);
  ...
}
```
这段代码告诉我们，除非显式指定`-XX:-UseParallelOldGC`，否则都开启Parallel Old。第二个地方是GCConfiguration：
```cpp
// hotspot\share\gc\shared\gcConfiguration.cpp
GCName GCConfiguration::young_collector() const {
  if (UseG1GC) {
    return G1New;
  }
  // 如果开启UseParallelGC则新年代使用ParallelScavenge
  if (UseParallelGC) {
    return ParallelScavenge;
  }

  if (UseConcMarkSweepGC) {
    return ParNew;
  }

  if (UseZGC) {
    return NA;
  }

  return DefNew;
}

GCName GCConfiguration::old_collector() const {
  if (UseG1GC) {
    return G1Old;
  }

  if (UseConcMarkSweepGC) {
    return ConcurrentMarkSweep;
  }
  // 如果开启UseParallelOldGC则老年代使用ParallelOld，否则使用SerialOld
  if (UseParallelOldGC) {
    return ParallelOld;
  }

  if (UseZGC) {
    return Z;
  }

  return SerialOld;
}
```
通过简单的字符串搜索就能知道:

+ `+UseParallelGC` = `新生代ParallelScavenge + 老年代ParallelOld`
+ `+UseParallelOldGC` = 同上
+ `-UseParallelOldGC` = `新生代ParallelScavenge + 老年代SerialOld`

ParallelOld和SerialOld字面上意思是老年代并行处理和老年代串行处理，关于这两个的区别也可以通过字符串搜索一窥究竟：
```cpp
//hotspot\share\gc\parallel\parallelScavengeHeap.cpp
void ParallelScavengeHeap::do_full_collection(bool clear_all_soft_refs) {
  if (UseParallelOldGC) {
    bool maximum_compaction = clear_all_soft_refs;
    // ParallelOld使用PSParallelCompact做full gc
    PSParallelCompact::invoke(maximum_compaction);
  } else {
  	// 关闭ParallelOld则使用PSMarkSweep做full gc
    PSMarkSweepProxy::invoke(clear_all_soft_refs);
  }
}
```
PSMarkSweepProxy是一个命名空间，它做的唯一一件事情就是把调用转发到PSMarkSweep类的同名方法，比如PSMarkSweepProxy::do_a()实际调用的是PSMarkSweep::do_a()。PSMarkSweep和[Serial GC Full GC](gc_serialgc_fullgc.md)提到的算法几乎一样，都是串行地分四个阶段对老年代做标记-压缩，稍有不同的是PSMarkSweep支持UseAdaptiveSizePolicy参数，它可以自适应的调整新生代和老年代的大小。

总的来说，Parallel GC和Parallel Old GC说的是不一样的事情，**前者表示并行分代式垃圾回收器**，其老年代和新生代都是多线程并行操作。而**后者只是老年代是否使用并行的一个选项**(默认开启)，如果关闭则老年代退化为串行操作。足见Hotspot命名功力是多么的不忍直视...