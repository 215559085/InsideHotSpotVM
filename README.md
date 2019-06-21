# Inside HotSpot VM

关于HotSpot(OpenJDK12)的内幕，对某个部分感兴趣的朋友可以创建Issue说明。Github Just-In-Time更新，[博客](https://www.cnblogs.com/kelthuzadx/)滞后更新。(注:WIP表示未完成的文章)

### 解释器
+ [模板解释器](resource/template_interpreter.md)
  1. 简介
  2. 解释器的两种实现
  3. 解释器
      1. 抽象解释器
      2. 模板解释器
  4. 解释器生成器
      1. 生成器与解释器的关系
      2. 示例：数组越界异常例程生成
    
### 编译器
+ [方法编译](resource/compile_method.md)
  1. 编译器代理
  2. nmethod
  3. 栈上替换(On Stack Replacement)
  4. 编译重放(ReplayCompiles)
+ [C1编译器中间表示](resource/c1_compile.md)
  1. C1编译器线程
  2. 中间表示简介
  3. 构造HIR
  4. 构造LIR
+ [C1编译器HIR的构造](resource/c1_construct_hir.md)
  1. 简介
  2. HIR的设计
  3. 高观点层次的字节码到HIR构造
  4. 源码层次的字节码到HIR构造
+ [C1编译器优化：条件表达式消除](resource/c1opt_conditional_elimination.md)
  1. 条件传送指令
  2. C1编译器中的条件表达式消除
  3. CEE示例
+ [C1编译器优化：全局值编号(GVN)](resource/c1opt_gvn.md)
  1. 值编号
  2. C1编译器的全局值编号
  3. 示例：公共子表达式消除（成功)
  4. 示例：代数恒等式变换（失败）
  5. 循环不变代码外提（成功但受限）

### 运行时
+ [加载字节码到JVM](resource/class_loader.md) **WIP**
  1. 字节码文件解析器
  2. 类加载器
      1. Bootstrap类加载器 **WIP**
+ [Java的方法调用](resource/java_call.md)
  1. 方法调用模块入口
  2. 寻找调用方法
  3. 建立栈帧
  4. Java方法调用
  5. 总结
  6. 附1：使用hsdis查看对应的汇编表示 
  7. 附2：解释器入口点

## GC
+ [安全点](resource/safepoint.md)
  1. 安全点简介
  2. 创建安全点
  3. 线程局部握手
    1. 白盒测试API
    2. 基于线程握手的实现
+ [Java分代堆](resource/gc_heap_overview.md)
  1. 宇宙初始化
  2. 创建Java堆
  3. 初始化Java堆
  4. JVM分代堆详细结构
      1. CollectedHeap
      2. GenCollectedHeap
      3. SerialHeap
  5. 分代堆中的卡表代
+ [Epsilon GC](resource/gc_epsilongc.md)
  1. Epsilon GC简介
  2. EpsilonGC创建
  3. 内存分配
      1. 普通内存分配
      2. TLAB内存分配
  4. 垃圾回收
+ [Serial垃圾回收器 (一) Full GC](resource/gc_serialgc_fullgc.md)
  0. Serial垃圾回收器Full GC
  1. 阶段1：标记存活对象
  2. 阶段2：计算对象新地址
  3. 阶段3：调整对象指针
  4. 阶段4：移动对象
+ [Serial垃圾回收器 (二) Minor GC](resource/gc_serialgc_minorgc.md)
  1. DefNewGeneration垃圾回收
  2. 快速扫描闭包(FastScanClosure)
      1. 新生代到To survivor的复制
      2. GC屏障
  3. 快速成员处理闭包(FastEvacuateFollowersClosure)
+ [UseParallelGC和UseParallelOldGC的区别](resource/gc_parallel_vs_parallelold.md)

### 其他
+ [hotspot的启动流程与main方法调用](resource/java_main.md)
+ [JVM新日志接口-xlog](resource/xlog.md)
+ [JVMTI(JVM Tool Interface)](resource/jvmti.md)
+ [Visual Studio2017编译调试OpenJDK12](resource/building_win.md)
+ [Xcode编译调试OpenJDK12](resource/building_osx.md)

### 附注
1. 文章中JVM,jvm,hotspot,HotSpot等都表示的OpenJDK12的HotSpot虚拟机实现；From,from,From survivor一个意思;To，To survivor一个意思；
2. 代码的部分干扰比如断言，繁琐不重要的日志记录已清除
3. 强烈建议**每一章按照自上而下的顺序阅读**，因为下一篇文章很有可能需要上一篇文章的知识
