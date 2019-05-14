# [Inside HotSpot] JVMTI

## 1. 最简示例
JVMTI(JVM Tool Interface)提供给外部提供一个接口让外部可以查看虚拟机内部的状态或者控制虚拟机一些行为。首先来一个最简示例：
```cpp
// jvmdbg.cpp
#include <jni.h>
#include <jvmti.h>

#include <iostream>

JNIEXPORT jint JNICALL Agent_OnLoad(JavaVM *jvm, char *options, void *reserved) {
    std::cout<<"hello jvmti\n";
    return JNI_OK;
}

JNIEXPORT void JNICALL Agent_OnUnload(JavaVM *vm) {
    std::cout << "goodbye jvmti\n";
}

```
然后编译成动态链接库(`.dll`,`.so`)，当使用`java -agentpath=C:/Path/To/jvmdbg.dll App`的时候会输出上面的信息。Agent_OnLoad会在虚拟机可以工作的时候调用，Agent_OnUnload在虚拟机不能工作时调用。

## 2. 赋予JVMTI功能
JVMTI可以做很多事情，但是在做之前我们得给她这个能力。这个意思有点像权限：
```cpp
JNIEXPORT jint JNICALL Agent_OnLoad(JavaVM *jvm, char *options, void *reserved) {
    jvm->GetEnv((void **) &jvmti, JVMTI_VERSION_11);
    jvmtiCapabilities capa;
    capa.can_tag_objects = 1;
    capa.can_generate_field_modification_events = 1;
    capa.can_generate_field_access_events = 1;
    jvmti->AddCapabilities(&capa);
    return JNI_OK;
}
```
jvmtiCapabilities是个结构体，里面包含了所有能力，简单起见，可以把所有字段全部设置为1（true）

## 3. 获取所有线程
JVMTI与虚拟机沟通的桥梁是事件与回调：当虚拟机内部发生某个事件的时候，会调用用户注册的回调函数。我们可以在Agent_OnLoad中绑定事件与回调，然后等待事件发生调用回调即可。举个例子，我们想在虚拟机创建线程的时候获取线程的名字，就可以注册创建线程事件然后回调获取线程信息：
```cpp
static void JNICALL callbackJvmtiEventThreadStart(jvmtiEnv *jvmti,
                                                  JNIEnv *jni_env,
                                                  jthread thread) {
    jvmtiThreadInfo threadInfo;
    jvmti->GetThreadInfo(0,&threadInfo);
    std::cout<<"Thread name:"<<threadInfo.name<<"\n";
    std::cout<<"Priority:"<<threadInfo.priority<<"\n";
    std::cout<<"Daemon:"<<threadInfo.is_daemon<<"\n";
}

JNIEXPORT jint JNICALL Agent_OnLoad(JavaVM *jvm, char *options, void *reserved) {
    jvm->GetEnv((void **) &jvmti, JVMTI_VERSION_11);
    jvmtiCapabilities capa;
    capa.can_signal_thread = 1;
    jvmti->AddCapabilities(&capa);
    // 绑定线程启动回调
    jvmtiEventCallbacks callbacks;
    callbacks.ThreadStart = callbackJvmtiEventThreadStart;
    jvmti->SetEventCallbacks(&callbacks, (jint) sizeof(callbacks));
    // 注册线程启动事件
    jvmti->SetEventNotificationMode(JVMTI_ENABLE, JVMTI_EVENT_THREAD_START, (jthread) nullptr);
    return JNI_OK;
}
```
其他响应事件和可执行功能都是一样的套路，请参看[引用手册](https://docs.oracle.com/javase/1.5.0/docs/guide/jvmti/jvmti.html#GetThreadState)获取详细信息。