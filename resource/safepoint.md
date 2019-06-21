# [Inside HotSpot] 安全点

## 1. 安全点简介
垃圾回收通常有一个过程：虚拟机线程设置安全点(Safepoint)，然后等待其他Java线程走到安全点并阻塞。这种主动阻塞的行为叫做STW（Stop the world）。

高观点来说，安全点相当于指定一个地方，所有执行Java代码的线程都在这停止了（只剩下一条虚拟机线程可以走），整个虚拟机的状态是确定的，之所以只是执行Java代码的线程是因为执行native代码不会改变JVM状态，所以无需阻塞（native代码通过JNI访问JVM或者从native返回到java层还是会阻塞）。

安全点对很多操作来说是必要的，举个例子， JIT编译器如果编译完一段代码，可能会做栈上替换（OSR），即将解释执行的栈替换为本地代码的栈，如果替换过程没有一个同步操作，很可能造成解释器栈分配了新对象引用，而本地代码栈缺少新对象。又如jstack可以dump出线程栈的数据，如果这时候Java线程还在走，那很可能会得错误的数据。等等情况都需要一个全局停顿（STW）机制，而HotSpot正是用安全点实现STW的。

上面的例子还揭露了一个事实：并不是只有GC需要安全点，所有需要虚拟机状态保持一致的操作都需要安全点，这些操作包括：

+ 线程dump，jstack死锁检查，堆dump
+ Deoptimization
+ GC
+ 偏向锁revoke
+ 类重定义
+ 线程停止
+ ....

完整列表可以参见`runtime/vmOperations.hpp`。

低观点下的安全点是一页内存，如果JVM需要让所有线程STW，就将这页内存设置为不可读不可写，然后等待其他线程执行安全点访问代码(`汇编指令test`)，然后抛出页访问冲突异常。最后该线程的异常处理器捕获访问冲突异常，主动阻塞自身。反之，如果不需要安全点，只需将这页内存设置为可读可写，这时候安全点访问代码相当于一个空操作。

## 2. 创建安全点
安全点的创建位于`safepointMechanism`：
```cpp
// hotspot\share\runtime\safepointMechanism.cpp
void SafepointMechanism::default_initialize() {
  if (ThreadLocalHandshakes) {
    set_uses_thread_local_poll();

    // Poll bit values
    intptr_t poll_armed_value = poll_bit();
    intptr_t poll_disarmed_value = 0;

    if (!USE_POLL_BIT_ONLY)
    {
      // Polling page
      const size_t page_size = os::vm_page_size();
      const size_t allocation_size = 2 * page_size;
      char* polling_page = os::reserve_memory(allocation_size, NULL, page_size);
      os::commit_memory_or_exit(polling_page, allocation_size, false, "Unable to commit Safepoint polling page");
      MemTracker::record_virtual_memory_type((address)polling_page, mtSafepoint);

      char* bad_page  = polling_page;
      char* good_page = polling_page + page_size;

      os::protect_memory(bad_page,  page_size, os::MEM_PROT_NONE);
      os::protect_memory(good_page, page_size, os::MEM_PROT_READ);

      log_info(os)("SafePoint Polling address, bad (protected) page:" INTPTR_FORMAT ", good (unprotected) page:" INTPTR_FORMAT, p2i(bad_page), p2i(good_page));
      os::set_polling_page((address)(bad_page));

      // Poll address values
      intptr_t bad_page_val  = reinterpret_cast<intptr_t>(bad_page),
               good_page_val = reinterpret_cast<intptr_t>(good_page);
      poll_armed_value    |= bad_page_val;
      poll_disarmed_value |= good_page_val;
    }

    _poll_armed_value    = reinterpret_cast<void*>(poll_armed_value);
    _poll_disarmed_value = reinterpret_cast<void*>(poll_disarmed_value);
  } else {
    const size_t page_size = os::vm_page_size();
    char* polling_page = os::reserve_memory(page_size, NULL, page_size);
    os::commit_memory_or_exit(polling_page, page_size, false, "Unable to commit Safepoint polling page");
    os::protect_memory(polling_page, page_size, os::MEM_PROT_READ);
    MemTracker::record_virtual_memory_type((address)polling_page, mtSafepoint);

    log_info(os)("SafePoint Polling address: " INTPTR_FORMAT, p2i(polling_page));
    os::set_polling_page((address)(polling_page));
  }
}
```
使用`-Xlog:os*=trace`查看生成的安全点详细地址：
```cpp
[7.450s][info ][os] SafePoint Polling address, bad (protected) page:0x0000000102c30000, good (unprotected) page:0x0000000102c31000
```
safepointMechanism有个上面没有提到的特性：线程局部握手。该特性来源于[JEP 312: Thread-Local Handshakes](http://openjdk.java.net/jeps/312)。简单来说，如果要执行某个需要全局停顿(Stop the world)的操作，用安全点机制需要1）所有线程全部停止2）然后执行操作3）最后所有线程唤醒；线程局部握手可以让到达每个线程各自的局部安全点的线程立刻执行某个操作（这个操作就叫做握手（handshake）），然后立刻唤醒。线程局部握手的好处是执行某个操作的线程不需要等待其他线程都到安全点再继续。需要这种的情况不多但是存在，可以参见JEP提案，最详细的内容可以参见JEP312的代码改变。

只举一例，JVM可以输出每个线程的调用栈，但是该需求并不需要让所有线程都到某个全局点停顿然后统一输出最后统一启动，理想的情况应该是各个线程到某个可以输出调用栈的点进行输出再启动即可，很明显，这种情况就是我们提到的线程局部握手的完美应用场景，实际上只要使用了`-XX:-ThreadLocalHandshakes`虚拟机就是这么做的。

## 3. 线程局部握手
### 3.1 白盒测试API
前面提到线程局部握手应用于白盒测试的情景，所谓白盒测试是指测试者完全了解程序，希望看到测试程序走的路径和自己预料的一样，区别于黑盒测试测试者不关心走的路径，只希望最后输出的结果和自己预料的一样。HotSpot也支持[白盒测试](https://wiki.openjdk.java.net/display/HotSpot/The+WhiteBox+testing+API)。如果已经编译过hotspot，然后执行
```bash
$ make build-test-lib
```
生成的wb.jar位于`openjdk12/build/macosx-x86_64-server-fastdebug/support/test/lib`。现在来试试白盒API。写个java程序，该程序会遍历所有线程的栈：
```java
package com.github.kelthuzadx;
import sun.hotspot.WhiteBox;
public class WBTest {
    static WhiteBox wb = WhiteBox.getWhiteBox();
    static void redundant(){
        wb.handshakeWalkStack(Thread.currentThread(),false/*is_all_thread?*/);
    }
    public static void main(String[] args) {
        redundant();
    }
}
```
附带加上命令行`-Xbootclasspath/a:/Users/kelthuyang/Desktop/openjdk12/build/macosx-x86_64-server-fastdebug/support/test/lib/wb.jar -XX:+UnlockDiagnosticVMOptions -XX:+WhiteBoxAPI`，可以得到类似下面的输出：
```cpp
"main" #1 prio=5 os_prio=31 cpu=736.70ms elapsed=0.81s tid=0x00007fb500800800 nid=0x2403 waiting on condition  [0x000070000629c000]
   java.lang.Thread.State: RUNNABLE
Thread: 0x00007fb500800800  [0x2403] State: _running _has_called_back 0 _at_poll_safepoint 0
   JavaThread state: _thread_blocked
  at sun.hotspot.WhiteBox.handshakeWalkStack(Native Method)
  at com.github.kelthuzadx.WBTest.redundant(WBTest.java:10)
  at com.github.kelthuzadx.WBTest.main(WBTest.java:14)
```
### 3.2 基于线程握手的实现
Java层面的白盒测试API会调用HotSpot中`prims/whitebox.cpp`对应的入口。比如上面的`WhiteBox.handshakeWalkStack`会执行WB_HandshakeWalkStack：
```cpp
WB_ENTRY(jint, WB_HandshakeWalkStack(JNIEnv* env, jobject wb, jobject thread_handle, jboolean all_threads))
  // 输出线程调用栈的闭包，这是最终输出实现，不过要想被JVM使用还会经过重重包装
  class TraceSelfClosure : public ThreadClosure {
    jint _num_threads_completed;
    void do_thread(Thread* th) {
      assert(th->is_Java_thread(), "sanity");
      JavaThread* jt = (JavaThread*)th;
      ResourceMark rm;
      jt->print_on(tty);
      jt->print_stack_on(tty);
      tty->cr();
      Atomic::inc(&_num_threads_completed);
    }

  public:
    TraceSelfClosure() : _num_threads_completed(0) {}

    jint num_threads_completed() const { return _num_threads_completed; }
  };
  // 输出线程栈的闭包
  TraceSelfClosure tsc;
  //输出所有线程调用栈
  if (all_threads) {
    Handshake::execute(&tsc);
  } else {
  // 只输出某一个线程调用栈
    oop thread_oop = JNIHandles::resolve(thread_handle);
    if (thread_oop != NULL) {
      JavaThread* target = java_lang_Thread::thread(thread_oop);
      Handshake::execute(&tsc, target);
    }
  }
  return tsc.num_threads_completed();
WB_END
```
假如某个Java线程想做一些操作，比如GC，比如dump调用栈，Java线程是没有能力的，它会把该操作封装成`VM_xxx`然后投递到VM线程（jstack可以看到这个线程）的任务队列。VM线程从队列里面获取`VM_xxx`并执行。基于上面的认识，如果启用了线程局部握手，Java线程会投递一个VM_HandshakeOneThread，如果没有就会投递VM_HandshakeFallbackOperation。这两个操作的区别正如我们之前提到的，前者在各个线程特设的一个点停顿，执行操作再启动，后者需要全局安全点停顿，执行操作，全局启动：
```cpp
// hotspot\share\runtime\vmThread.cpp
void VMThread::loop() {
  while(true) {
    ...
    // 阻塞在队列头，等待其他线程投递VM_XX
    { MutexLockerEx mu_queue(VMOperationQueue_lock,
                             Mutex::_no_safepoint_check_flag);
      ...
    } 

    // 执行VM操作
    { HandleMark hm(VMThread::vm_thread());
      EventMark em("Executing VM operation: %s", vm_operation()->name());
      // 如果需要位于安全点
      if (_cur_vm_operation->evaluate_at_safepoint()) {
        _vm_queue->set_drain_list(safepoint_ops); // ensure ops can be scanned
        ...
        // 安全点开始
        SafepointSynchronize::begin();
        ...
        // 执行操作
        evaluate_operation(_cur_vm_operation);
        ...
        // 安全点结束
        SafepointSynchronize::end();
        ...

      } else {  
        //否则不需要位于安全点
        ...
        evaluate_operation(_cur_vm_operation);
        ...
      }
    }
    //  通知Java线程VM操作执行完了
    { MutexLockerEx mu(VMOperationRequest_lock,
                       Mutex::_no_safepoint_check_flag);
      VMOperationRequest_lock->notify_all();
    }
    ...
  }
}
```
VM_HandshakeOneThread走的是else分支，即不需要全局安全点就能执行操作。


