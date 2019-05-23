# Inside-HotSpot

关于HotSpot(OpenJDK12)的内幕，对某个部分感兴趣的朋友可以创建Issue说明。Github Just-In-Time更新，[博客](https://www.cnblogs.com/kelthuzadx/)滞后更新。(注:WIP表示未完成的文章)

### 解释器
+ [模板解释器](resource/template_interpreter.md)

### JIT编译器
+ [C1编译器工作流程及中间表示](resource/c1_compile.md)
+ [C1编译器HIR的构造](resource/c1_construct_hir.md)
+ [C1编译器优化：条件表达式消除](resource/c1opt_conditional_elimination.md)
+ [C1编译器优化：全局值编号(GVN)](resource/c1opt_gvn.md)

### 运行时
+ [Java的方法调用](resource/java_call.md)

## GC
+ [Epsilon GC](resource/epsilon_gc.md)
+ [Java分代堆](resource/gc_heap_overview.md) **WIP**
+ [Serial垃圾回收器](resource/gc_serialgc.md) **WIP**

### 其他
+ [hotspot的启动流程与main方法调用](resource/java_main.md)
+ [JVM新日志接口-xlog](resource/xlog.md)
+ [JVMTI(JVM Tool Interface)](resource/jvmti.md)
+ [Visual Studio2017编译调试openjdk12](resource/building.md)

### 附注
1. 文章中JVM,jvm,hotspot,HotSpot等都表示的OpenJDK12的HotSpot虚拟机实现；From,from,From survivor一个意思;To，To survivor一个意思；
2. 代码的部分干扰比如断言，繁琐不重要的日志记录已清除
3. 强烈建议每一章按照自上而下的顺序阅读，因为下一篇文章很有可能需要上一篇文章的只是