# [Inside HotSpot] 方法编译

## 1. 编译器代理
HotSpot中编译器与解释器通过一个编译队列（Compilerueue），当解释器认为某个方法需要做即时编译，它会创建一个编译任务（CompilerTask）然后投递到编译队列；另一边编译器线程（C1/C2 CompilerThread）阻塞在编译队列上，当发现有任务时再唤醒执行编译。

为了降低代码耦合，解释器不会直接创建编译任务，这个过程会交给编译器代理（CompileBroker）完成。
```cpp
// hotspot\share\compiler\compileBroker.cpp
nmethod* CompileBroker::compile_method(const methodHandle& method, int osr_bci,
                                         int comp_level,
                                         const methodHandle& hot_method, int hot_count,
                                         CompileTask::CompileReason compile_reason,
                                         DirectiveSet* directive,
                                         Thread* THREAD) {

  // 选择编译器,选osr编译还是普通编译等
  ...

  // 开始编译
  if (method->is_native()) {
    // native方法
    ...
  } else {
    // 普通java方法
    if (!should_compile_new_jobs()) {
      CompilationPolicy::policy()->delay_compilation(method());
      return NULL;
    }
    bool is_blocking = !directive->BackgroundCompilationOption || ReplayCompiles;
    compile_method_base(method, osr_bci, comp_level, hot_method, hot_count, compile_reason, is_blocking, THREAD);
  }

  // 返回nmethod
  if (osr_bci == InvocationEntryBci) {
    CompiledMethod* code = method->code();
    if (code == NULL) {
      return (nmethod*) code;
    } else {
      return code->as_nmethod_or_null();
    }
  }
  return method->lookup_osr_nmethod_for(osr_bci, comp_level, false);
}
void CompileBroker::compile_method_base(const methodHandle& method,
                                        int osr_bci,
                                        int comp_level,
                                        const methodHandle& hot_method,
                                        int hot_count,
                                        CompileTask::CompileReason compile_reason,
                                        bool blocking,
                                        Thread* thread) {
  ...
  CompileTask* task     = NULL;
  CompileQueue* queue  = compile_queue(comp_level);
  // 同步区域
  {
    MutexLocker locker(MethodCompileQueue_lock, thread);
    // 再次检查是否已经
    if (compilation_is_in_queue(method)) {
      return;
    }
    // 再次检查方法是否早已加入编译队列了
    if (compilation_is_complete(method, osr_bci, comp_level)) {
      return;
    }
    // 创建compile_id
    int compile_id = assign_compile_id(method, osr_bci);
    if (compile_id == 0) {
      return;
    }
    // 都没有问题，那就创建编译任务->初始化->加入编译队列->唤醒编译线程
    task = create_compile_task(queue,
                               compile_id, method,
                               osr_bci, comp_level,
                               hot_method, hot_count, compile_reason,
                               blocking);
  }

  // 如果主线程需要等待编译完成则阻塞等待
  if (blocking) {
    wait_for_completion(task);
  }
}
```

## 2. nmethod
nmethod（native method）表示编译后的java方法。严格来说，它不完全表示**编译后的java方法**，应该说是**Java方法的本地代码表示**，因为主线程并不一定要等到方法编译完（除非禁止后台编译或者启用ReplayCompiles），它是可以提前返回的。这个时候返回的nmethod就不是编译完成的java方法，而是正在编译的方法。

因为nmethod会被多个线程使用，所以如果某个时候编译线程编译完该方法，编译线程必须先获取Patching_lock锁，这个锁用于给nmethod设置（Install）编译好的代码：
```cpp
// hotspot\share\runtime\mutexLocker.hpp
extern Mutex*   Patching_lock;                   // a lock used to guard code patching of compiled code

// hotspot\share\code\nmethod.hpp
class nmethod : public CompiledMethod {
  ...
}

// hotspot\share\oops\method.cpp
void Method::set_code(const methodHandle& mh, CompiledMethod *code) {
  MutexLockerEx pl(Patching_lock, Mutex::_no_safepoint_check_flag);
  mh->_code = code; 
  int comp_level = code->comp_level();
  if (comp_level > mh->highest_comp_level()) {
    mh->set_highest_comp_level(comp_level);
  }

  OrderAccess::storestore();
  mh->_from_compiled_entry = code->verified_entry_point();
  OrderAccess::storestore();
  if (!mh->is_method_handle_intrinsic())
    mh->_from_interpreted_entry = mh->get_i2c_entry();
}
```
如果再进一步，比如C1就会在编译的最后一个阶段（codeinstall）使用Method::set_code将J生成的本地代码安装到nmethod上，不过这算是后话。

## 3. OSR栈上替换
前面在创建编译任务的时候会区分普通编译方法和OSR编译方法。OSR（On Stack Replacement）即栈上替换，它用于快速从解释器模式切换到JIT生成的本地代码。它的用处有限但是效果巨大，举个例子，如果一个方法只执行1次，但是里面有个循环执行千万次，对这个方法做JIT编译后如果要等待第二次方法的执行才会使用JIT生成的本地代码，那么JIT编译就完全失去了作用，因为方法不会调用第二次，这时候就需要使用OSR，在第10000次循环执行后JIT得到本地代码，用本地代码的栈桢替换解释器栈桢，在10001次使用本地代码。 可以写个简单的实例体验OSR的效果
```java
public class Main {
    // Timing:1875 Result:2051657985(Without OSR)
    // Timing:277 Result:2051657985(With OSR)
    public static void main(String[] args) {
        long start = System.currentTimeMillis();
        int sum = 0;
        for(int i=0;i<999999999;i++){
            sum +=i;
        }
        System.out.println("Timing:"+(System.currentTimeMillis()-start)+" Result:"+sum);
    }
}
```
当开启OSR(`-Xcomp -XX:+UseOnStackReplace`)使用了277毫秒，关闭OSR则是1875毫秒

## 4. ReplayCompiles



