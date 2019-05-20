# hotspot的启动流程与main方法调用
虚拟机的使命就是执行`public static void main(String[])`方法，从虚拟机创建到main方法执行会经过一系列流程。这篇文章详细讨论了执行命令行`java.exe HelloWorld`调用main函数输出经历了什么。源码使用`openjdk12`，操作系统为`windows 64bits`，其它系统和源码版本大同小异。

## java.base
首先要明白一个概念，`java.exe`大体上可以分为启动器部分和hotspot部分。
启动器负责执行一些命令行解析，环境初始化等任务，hotspot部分则是真正的虚拟机干活的地方。

启动器是用C++写的，如果不修改链接器入口点名字，执行`java.exe xxx`追根溯源必然会跟踪到`main`函数。这个main位于
`openjdk12\src\java.base\share\native\launcher\main.c`，这里就是java启动器的最终起源了：
```cpp
#ifdef JAVAW

char **__initenv;

int WINAPI
WinMain(HINSTANCE inst, HINSTANCE previnst, LPSTR cmdline, int cmdshow)
{
    int margc;
    char** margv;
    int jargc;
    char** jargv;
    const jboolean const_javaw = JNI_TRUE;

    __initenv = _environ;

#else /* JAVAW */
JNIEXPORT int
main(int argc, char **argv)
{
    int margc;
    char** margv;
    int jargc;
    char** jargv;
    const jboolean const_javaw = JNI_FALSE;
#endif /* JAVAW */

    // 处理传递给启动器的参数

    return JLI_Launch(margc, margv,
                   jargc, (const char**) jargv,
                   0, NULL,
                   VERSION_STRING,
                   DOT_VERSION,
                   (const_progname != NULL) ? const_progname : *margv,
                   (const_launcher != NULL) ? const_launcher : *margv,
                   jargc > 0,
                   const_cpwildcard, const_javaw, 0);
}
```
如果用户执行的是`javaw.exe`就进入`WinMain`入口，否则`java.exe`进入`main`入口。

在main中会处理启动器的参数比如这种`-XX:+UnlockDiagnosticVMOptions -XX:+PauseAtExit `，处理完之后调用`JLI_Launcher`。多说一点，启动器代码也分为系统相关和系统无关，像`java.base/linux`,`java.base/windows`这种就是平台相关，`java.base/share`就是平台无关代码。

## java.base -> JLI_Launcher
JLI_Launcher位于`openjdk12\src\java.base\share\native\libjli\java.c`，
```cpp
JNIEXPORT int JNICALL
JLI_Launch(int argc, char ** argv,              /* main argc, argc */
        int jargc, const char** jargv,          /* java args */
        int appclassc, const char** appclassv,  /* app classpath */
        const char* fullversion,                /* full version defined */
        const char* dotversion,                 /* dot version defined */
        const char* pname,                      /* program name */
        const char* lname,                      /* launcher name */
        jboolean javaargs,                      /* JAVA_ARGS */
        jboolean cpwildcard,                    /* classpath wildcard */
        jboolean javaw,                         /* windows-only javaw */
        jint     ergo_class                     /* ergnomics policy */
);
```
JLI_Launcher做了很多重要的事情

1. 定位jre
2. 解析命令行，比如传入`-XX:SDKGJ`导致虚拟机退出就是在这里发生的。
3. 加载jvm.dll，获取JNI_CreateJavaVM等函数地址

第三点非常重要，它是启动器调用hotspot JNI的桥梁。说着这么夸张，其实做起来是非常简单的，就是`LoadLibrary()`加载jvm.dll然后`GetProcAddress()`运行时获取JNI_CreateJavaVM地址转化为函数指针，对应linux的`dlopen,dlsym`。然后经过一些中转，启动器会走到JavaMain。

## jaba.base -> JLI_Launcher -> JavaMain
JavaMain维护hotspot的一个生命周期，它沟通java启动器与hotspot世界，完成java.exe的功能：
```cpp
int JNICALL
JavaMain(void * _args)
{
	...

     /* 初始化虚拟机 */
    start = CounterGet();
    /* 这里初始化虚拟机调用的就是之前提到的在jvm.dll里面获取到的JNI_CreateJavaVM函数指针
     * 可以说，这里的JNI_CreateJavaVM是hotspot世界最先出现的地方
     */
    if (!InitializeJVM(&vm, &env, &ifn)) {
        JLI_ReportErrorMessage(JVM_ERROR1);
        exit(1);
    }

    /* 对虚拟机启动进行性能profiling */
    if (JLI_IsTraceLauncher()) {
        end = CounterGet();
        JLI_TraceLauncher("%ld micro seconds to InitializeJVM\n",
               (long)(jint)Counter2Micros(end-start));
    }

    ret = 1;

    /* 加载main函数所在的类 */
    mainClass = LoadMainClass(env, mode, what);
    CHECK_EXCEPTION_NULL_LEAVE(mainClass);

    /* 对GUI程序的支持 */
    appClass = GetApplicationClass(env);
    mainArgs = CreateApplicationArgs(env, argv, argc);
    if (dryRun) {
        ret = 0;
        LEAVE();
    }
    PostJVMInit(env, appClass, vm);

    /* 获取main方法id */
    mainID = (*env)->GetStaticMethodID(env, mainClass, "main",
                                       "([Ljava/lang/String;)V");

    /* main方法调用 */
    (*env)->CallStaticVoidMethod(env, mainClass, mainID, mainArgs);

    /* 启动器的返回值(非System.exit退出) */
    ret = (*env)->ExceptionOccurred(env) == NULL ? 0 : 1;

    LEAVE();
}
```
JavaMain这个函数做了我们通常意义上所认为启动器应该做的事情，它：

+ 初始化虚拟机，
+ 获取main所在的类
+ 调用main方法
+ 处理返回值

到这里java启动器流程基本上已经清晰了，但是旅程并未结束。除了java启动器外，本文还想探究一下main方法的调用。

## jaba.base -> JLI_Launcher -> JavaMain -> CallStaticVoidMethod
首先，欢迎来到hotspot的世界。前面说到main方法的调用是这么一行代码：
```cpp
(*env)->CallStaticVoidMethod(env, mainClass, mainID, mainArgs);
```
那么它是怎么进入hotspot的世界的呢，要回答这个问题得看看env这是个什么东西。
env类似于这样一个结构：
```cpp
struct P{
    void (*jni_f1)(int,int);
    void (*jni_f2)();
    void (*jni_f3)(double);
};
P* env;
```
然后`(*env)->jni_f1(3,4)`调用的就是这个jni_f1函数指针，这些指针指向的是`hotspot/share/primes/jni.cpp`里面的入口点。

## jaba.base -> JLI_Launcher -> JavaMain -> CallStaticVoidMethod -> JavaCalls::call
回到上面的代码，CallStaticVoidMethod函数指针指向的就是JNI里面的函数：
```cpp
JNI_ENTRY(void, jni_CallStaticVoidMethod(JNIEnv *env, jclass cls, jmethodID methodID, ...))
  JNIWrapper("CallStaticVoidMethod");
  HOTSPOT_JNI_CALLSTATICVOIDMETHOD_ENTRY(env, cls, (uintptr_t) methodID);
  DT_VOID_RETURN_MARK(CallStaticVoidMethod);

  va_list args;
  va_start(args, methodID);
  JavaValue jvalue(T_VOID);
  JNI_ArgumentPusherVaArg ap(methodID, args);
  jni_invoke_static(env, &jvalue, NULL, JNI_STATIC, methodID, &ap, CHECK);
  va_end(args);
JNI_END

static void jni_invoke_static(JNIEnv *env, JavaValue* result, jobject receiver, JNICallType call_type, jmethodID method_id, JNI_ArgumentPusher *args, TRAPS) {
  methodHandle method(THREAD, Method::resolve_jmethod_id(method_id));

  // 创建java调用的参数
  ResourceMark rm(THREAD);
  int number_of_parameters = method->size_of_parameters();
  JavaCallArguments java_args(number_of_parameters);
  args->set_java_argument_object(&java_args);

  assert(method->is_static(), "method should be static");

  args->iterate( Fingerprinter(method).fingerprint() );

  // 初始化返回值类型
  result->set_type(args->get_ret_type());

  // main方法调用
  JavaCalls::call(result, method, &java_args, CHECK);

  // 返回值转换
  if (result->get_type() == T_OBJECT || result->get_type() == T_ARRAY) {
    result->set_jobject(JNIHandles::make_local(env, (oop) result->get_jobject()));
  }
}
```
jni_CallStaticVoidMethod只是处理了一下可变参数，其他工作交给jni_invoke_static。这个函数会把之前传入的命令行参数转换为虚拟机里面的oop对象，然后最终通过`JavaCalls::call`调用了main函数。这不是个特例，java的所有方法调用都是通过`JavaCalls::call`调用的，它会创建解释执行所需的栈帧，然后识别hotspot的模板解释器入口点，进入这个入口点执行字节码，当然这都是后话，如果有的话...


