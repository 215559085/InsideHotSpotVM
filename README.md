# Inside-HotSpot

关于HotSpot(OpenJDK12)的内幕，对某个部分感兴趣的朋友可以创建Issue说明。Github Just-In-Time更新，[博客](https://www.cnblogs.com/kelthuzadx/)滞后更新。(注:WIP表示未完成的文章)
有几点要注意下~：

1. 文章中JVM,jvm,hotspot,HotSpot等都表示的OpenJDK12的HotSpot虚拟机实现
2. 代码的部分干扰比如断言已清除

### 构建与调试
+ [Visual Studio2017编译调试openjdk12](resource/building.md)

### 解释器
+ [[Inside HotSpot] 模板解释器](resource/template_interpreter.md)

### JIT编译器
+ [[Inside HotSpot] C1编译器工作流程及中间表示](resource/c1_compile.md)
+ [[Inside HotSpot] C1编译器HIR的构造](resource/c1_construct_hir.md)
+ [[Inside HotSpot] C1编译器优化：条件表达式消除](resource/c1opt_conditional_elimination.md)
+ [[Inside HotSpot] C1编译器优化：全局值编号(GVN)](resource/c1opt_gvn.md)

### 运行时
+ [[Inside HotSpot] Java的方法调用](resource/java_call.md)

## GC
+ [[Inside HotSpot] Epsilon GC](resource/epsilon_gc.md) **WIP**
+ [[Inside HotSpot] Java分代堆](resource/gc_heap_overview.md) **WIP**
+ [[Inside HotSpot] Serial垃圾回收器Full GC](resource/gc_serialgc_markcompact.md)

### 其他
+ [hotspot的启动流程与main方法调用](resource/java_main.md)
+ [JVM新日志接口-xlog](resource/xlog.md)
+ [JVMTI(JVM Tool Interface)](resource/jvmti.md)

