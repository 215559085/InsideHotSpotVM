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

低观点下的安全点是一页内存，如果JVM需要让所有线程STW，就将这页内存设置为不可读不可写，然后等待其他线程执行安全点访问代码(`汇编指令test`)，然后抛出页访问冲突异常。最后该线程的异常处理器捕获访问冲突异常，主动阻塞自身。反之，如果不需要安全点，只需将这页内存设置为可读可写，这时候安全点饭访问代码相当于一个空操作。




## 2. 创建安全点
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

