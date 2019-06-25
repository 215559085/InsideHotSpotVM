# [Inside HotSpot] HotSpot源码结构
```bash
➜  hotspot pwd
/Users/kelthuyang/Desktop/openjdk12/src/hotspot
➜  hotspot tree -d    
.
├── cpu                        # cpu架构相关的代码
├── os                         # 操作系统相关的代码
├── os_cpu                     # cpu+操作系统相关代码
└── share                      # 共享代码，虚拟机的大部分实现
    ├── adlc
    │   ├── Doc
    │   └── Test
    ├── aot                    # AOT编译支持
    ├── asm                    # 汇编器，用来将类似汇编的代码编译成机器相关的二进制代码
    ├── c1                     # Client编译器，即C1即时编译器
    ├── ci 
    ├── classfile              # 解析字节码文件到虚拟机，包括类加载器，字节码文件解析器等
    ├── code         
    ├── compiler               # 编译器接口，提供一个通用编译器接口
    ├── gc                     # 垃圾回收器
    │   ├── cms                # 并发标记清除gc
    │   ├── epsilon            # 只分配不释放内存的gc，可以作为开发gc的模板
    │   ├── g1                 # g1gc
    │   │   ├── c1  
    │   │   └── c2 
    │   ├── parallel           # 并行gc
    │   ├── serial             # 串行gc
    │   ├── shared             # 所有gc共有的部分，比如可进行gc的堆结构，卡表，分带堆等
    │   │   ├── c1
    │   │   ├── c2
    │   │   └── stringdedup
    │   ├── shenandoah         # redhat开发的shenandoah gc
    │   │   ├── c1
    │   │   ├── c2
    │   │   └── heuristics
    │   └── z                  # zgc
    │       ├── c1
    │       └── c2
    ├── include
    ├── interpreter            # 高性能字节码解释器，解释字节码走的是本地代码；也包括废弃的低效cpp解释器
    ├── jfr
    │   ├── dcmd
    │   ├── instrumentation
    │   ├── jni
    │   ├── leakprofiler
    │   │   ├── chains
    │   │   ├── checkpoint
    │   │   ├── sampling
    │   │   └── utilities
    │   ├── metadata
    │   ├── periodic
    │   │   └── sampling
    │   ├── recorder
    │   │   ├── checkpoint
    │   │   │   └── types
    │   │   │       └── traceid
    │   │   ├── repository
    │   │   ├── service
    │   │   ├── stacktrace
    │   │   ├── storage
    │   │   └── stringpool
    │   ├── support
    │   ├── utilities
    │   └── writers
    ├── jvmci                  # JVM编译器接口
    ├── libadt                 # 抽象数据类型，比如字典，集合等
    ├── logging   
    ├── memory   
    │   └── metaspace
    ├── metaprogramming        # 元编程的一些type trait
    ├── oops                   # 对象模型
    ├── opto                   # Server编译器，即大名鼎鼎的C2优化编译器
    ├── precompiled
    ├── prims                  # JDK里面JNI函数的实现+JVMTI的实现+白盒测试
    │   └── wbtestmethods
    ├── runtime                # 运行时
    │   └── flags
    ├── services
    └── utilities
```