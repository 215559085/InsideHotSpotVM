# [Inside HotSpot] JVM新日志接口-xlog

JVM的日志记录比较杂乱，为此[JEP158](http://openjdk.java.net/jeps/158)引入了统一的日志记录接口`-xlog`。该接口可以分级别，分组件，分等级的记录用户想要的信息。
它的详细语法如下：
```cpp
-Xlog[:option]
    option         :=  [<what>][:[<output>][:[<decorators>][:<output-options>]]]
                       'help'
                       'disable'
    what           :=  <selector>[,...]
    selector       :=  <tag-set>[*][=<level>]
    tag-set        :=  <tag>[+...]
                       'all'
    tag            :=  name of tag
    level          :=  trace
                       debug
                       info
                       warning
                       error
    output         :=  'stderr'
                       'stdout'
                       [file=]<filename>
    decorators     :=  <decorator>[,...]
                       'none'
    decorator      :=  time
                       uptime
                       timemillis
                       uptimemillis
                       timenanos
                       uptimenanos
                       pid
                       tid
                       level
                       tags
    output-options :=  <output_option>[,...]
    output-option  :=  filecount=<file count>
                       filesize=<file size>
                       parameter=value
```

## tag
tag可以记录表示哪一方面的数据，比如加上`-xlog:gc`表示记录gc信息：
```cs
[4.429s][info][gc] GC(1) Pause Full (Allocation Failure) 1M->1M(123M) 30.103ms
[4.430s][info][gc] GC(0) Pause Young (Allocation Failure) 8M->1M(123M) 40.066ms
[5.244s][info][gc] GC(3) Pause Full (Allocation Failure) 1025M->1M(1147M) 135.748ms
```
然后加上`-xlog:class*`可以记录类信息，`*`表示表示所有和class相关的信息；`-xlog:os`记录os模块相关信息：
```cs
[0.281s][info][os] SafePoint Polling address, bad (protected) page:0x000002083a8c0000, good (unprotected) page:0x000002083a8c1000
```
## level
level可以输出对于级别，比如`-xlog:gc=trace`
```cs
[0.113s][info][gc] Using Serial
[0.900s][trace][gc] GC(0) Young invoke=1 size=1073741840
[0.900s][trace][gc] GC(0) Tenured: promo attempt is safe: available(1412104192) >= av_promo(0), max_promo(8596720)
[0.907s][trace][gc] GC(0) TenuredGeneration::should_collect: because should_allocate(134217730)
[0.907s][trace][gc] GC(1) Old invoke=1 size=1073741840
[0.922s][trace][gc] GC(1) Restoring 14124 marks
[0.942s][info ][gc] GC(1) Pause Full (Allocation Failure) 1M->1M(123M) 15.989ms
```
比trace低的都会输出。

## file
使用file可以将输出dump到文件,比如`-xlog:gc*=trace:file`

## and more
JEP上提到的`-xlog:rt,-xlog:compiler`在hotspot12中还没有实现，可以使用`-xlog:all`输出目前虚拟机使用到该接口的所有logging信息。不过个人感觉目前这个东西最大的用处就是输出GC信息。。