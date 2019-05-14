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

