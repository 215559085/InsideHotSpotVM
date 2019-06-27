# [Inside HotSpot] C1编译器优化：条件表达式消除

## 1. 条件传送指令
日常编程中有很多**根据某个条件对变量赋不同值**这样的模式，比如：
```cpp
int cmov(int num) {
    int result = 10;
    if(num<10){
        result = 1;
    }else{Q
        result = 0;
    }
    return result;
}
```
如果不进行编译优化会产出`cmp-jump`组合，即根据cmp比较的结果进行跳转。可以使用`gcc -O0`查看：
```asm
cmov(int):
        push    rbp
        mov     rbp, rsp
        mov     DWORD PTR [rbp-20], edi
        int result = 10;
        mov     DWORD PTR [rbp-4], 10
        cmp     DWORD PTR [rbp-20], 9
        jg      .L2
        mov     DWORD PTR [rbp-4], 1
        jmp     .L3
.L2:
        mov     DWORD PTR [rbp-4], 0
.L3:
        mov     eax, DWORD PTR [rbp-4]
        pop     rbp
        ret
```
如果num>9就为result赋1，否则赋0。正因为跳转是条件的，CPU必须要等到条件成立才执行后面的指令（即数据依赖于条件），这会让处理器变慢，所以CPU通常会使用分支预测算法，提前执行某个概率更高的分支，即使这个时候条件没有成立。但问题是，这是**预测**不是**断言**，如果它预测条件成立进入A分支给R赋值V1，实际却可能是条件失败，应该继续B分支给R赋V2，那么CPU还需要撤销给R赋值V1。而条件传送系列指令(cmovcc,setcc)则不同，它根据EFLAGS的位**条件性的**选择给R赋V1还是V2，这样就没有分支预测失败的性能惩罚。回到上面的例子，使用gcc8的`-O3`可以看到产出的条件传送指令：
```asm
cmov(int):
        xor     eax, eax
        cmp     edi, 9
        setle   al
        ret
```
产出变短了，code cache能容纳下更多代码不说，setle也会根据ZF结果，即小于等于9则给al赋1，否则赋0。我们还顺便证明了C++有条件表达式消除，那么HotSpot有吗？还真有，但是别高兴的太早，它可能和你想的大相径庭...

## 2. C1编译器中的条件表达式消除
在[[Inside HotSpot] C1编译器工作流程及中间表示](https://www.cnblogs.com/kelthuzadx/p/10740453.html)中我们提到OpenJDK12的C1编译器将字节码转化为机器码的过程分为多个阶段：

![](c1_ir.png)

第一个阶段它生成HIR，一种C1内部用到的表示字节码的图结构。该阶段同时也会对它进行优化，其中就包括本文要讨论的**条件表达式消除**(Conditional expression elimination)，这一部分代码位于[c1_Optimizer.cpp](http://hg.openjdk.java.net/jdk/jdk12/file/06222165c35f/src/hotspot/share/c1/c1_Optimizer.cpp#l95)。但是在研究之前还需要做一些准备工作，我们无法直接查看虚拟机JIT产出的本地代码，需要下载[hsdis-amd64.dll](https://files.cnblogs.com/files/kelthuzadx/hsdis-amd64.zip)，将它放在`jdk/bin/server/`目录下，然后虚拟机加上参数`-XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly`。另外，为了确保虚拟机用C1编译方法且做了条件表达式消除，还要加上`-Xcomp -XX:TieredStopAtLevel=1 -XX:+DoCEE`。

## 3. CEE示例
准备好后我们可以开始尝试，准备一段和上面C++差不多的代码：
```java
package com.github.kelthuzadx;

public class C1Optimizations {
    public static int conditionalExpressionElimination(int depend){
        if(depend<10){
            return 1;
        }else{
            return 0;
        }
    }

    public static void main(String[] args) {
         conditionalExpressionElimination(1024);
    }
}
```
然后加上参数查看汇编形式的产出（实际上还是本地代码的，只是汇编形式输出而已）：
```asm
; Simplified
com/github/kelthuzadx/C1Optimizations.conditionalExpressionElimination(I)I 
  0x000001afb0a691c0: mov    %eax,-0x9000(%rsp)
  0x000001afb0a691c7: push   %rbp
  0x000001afb0a691c8: sub    $0x30,%rsp                     ;*iload_0 

  0x000001afb0a691cc: cmp    $0xa,%edx

  0x000001afb0a691cf: jge    L1             ;*if_icmpge
  0x000001afb0a691d5: mov    $0x1,%eax
  0x000001afb0a691da: add    $0x30,%rsp
  0x000001afb0a691de: pop    %rbp
  0x000001afb0a691df: mov    0x120(%r15),%r10               ; 安全点
  0x000001afb0a691e6: test   %eax,(%r10)                    ; 轮询
  0x000001afb0a691e9: retq                                  ;*ireturn 

L1:  
  0x000001afb0a691ea: mov    $0x0,%eax
  0x000001afb0a691ef: add    $0x30,%rsp
  0x000001afb0a691f3: pop    %rbp
  0x000001afb0a691f4: mov    0x120(%r15),%r10               ; 安全点
  0x000001afb0a691fb: test   %eax,(%r10)                    ; 轮询
  0x000001afb0a691fe: retq   
  0x000001afb0a691ff: nop
  0x000001afb0a69200: nop
```
并没有优化？HotSpot不认为上面的Java代码是条件表达式。它认为的条件表达式是三元表达式，有些出乎意料，但是也请继续吧:
```java
package com.github.kelthuzadx;

public class C1Optimizations {
    public static int conditionalExpressionElimination(int depend){
        return depend>10?1:0;
    }

    public static void main(String[] args) {
         conditionalExpressionElimination(1024);
    }
}
```
要确定是**HotSpot认为这是条件表达式**而不是**我们认为这是条件表达式**，得加上`-XX:+PrintIR -XX:+PrintLIRWithAssembly`辅证（也可使用 -XX:+PrintCEE），它产出C1 HIR及其汇编表示。首先看看原始HIR：
```js
IR after parsing
B4 [0, 0] -> B0 sux: B0
empty stack
inlining depth 0
__bci__use__tid____instr____________________________________
. 0    0     14    std entry B0

B0 (SV) [0, 3] -> B2 B1 sux: B2 B1 pred: B4
empty stack
inlining depth 0
__bci__use__tid____instr____________________________________
  1    0    i5     10
. 3    0     6     if i4 <= i5 then B2 else B1

B1 (V) [6, 7] -> B3 sux: B3 pred: B0
empty stack
inlining depth 0
__bci__use__tid____instr____________________________________
  6    0    i7     1
. 7    0     8     goto B3
                   stack [0:i7]

B3 (V) [11, 11] pred: B1 B2Stack:
0  i11 [ i7 i9] 

stack [0:i11]
inlining depth 0
__bci__use__tid____instr____________________________________
. 11   0    i12    ireturn i11

B2 (V) [10, 11] -> B3 sux: B3 pred: B0
empty stack
inlining depth 0
__bci__use__tid____instr____________________________________
  10   0    i9     0
. 11   0     10    goto B3
                   stack [0:i9]
```
根据右边助记指令还是很好理解: 如果`i4<i5`即`depend<10`则返回1，反之返回0。接着查看条件表达式消除之后的HIR:
```js
IR after CEE
B4 [0, 0] -> B0 sux: B0
empty stack
inlining depth 0
__bci__use__tid____instr____________________________________
. 0    0     14    std entry B0

B0 (SV) [0, 3] -> B3 sux: B3 pred: B4
empty stack
inlining depth 0
__bci__use__tid____instr____________________________________
  1    0    i5     10
  3    0    i15    0
  3    0    i16    1
  3    0    i17    i4 <= i5 ? i15 : i16
. 3    0     18    goto B3
                   stack [0:i17]

B3 (V) [11, 11] pred: B0
stack [0:i17]
inlining depth 0
__bci__use__tid____instr____________________________________
. 11   0    i12    ireturn i17
```
i17的值视条件而定，如果条件`i4<i5`则i17为0，否则为1，最后返回。这里的模式正是条件传送指令的工作机制，HIR也有明显的改变，经过一番折腾，我们成功的让HotSpot对条件表达式做了一次消除优化。不过还没有结束，再看看HIR经过一系列步骤最终生成的本地代码：
```js
[Disassembling for mach='i386:x86-64']
   2 std_entry 
  0x0000026a64f209a0: mov    %eax,-0x9000(%rsp)
  0x0000026a64f209a7: push   %rbp
  0x0000026a64f209a8: sub    $0x30,%rsp

  0x0000026a64f209ac: cmp    $0xa,%edx

  0x0000026a64f209af: mov    $0x0,%eax
  0x0000026a64f209b4: jle    0x0000026a64f209bf
  0x0000026a64f209ba: mov    $0x1,%eax

  0x0000026a64f209bf: add    $0x30,%rsp
  0x0000026a64f209c3: pop    %rbp
  0x0000026a64f209c4: mov    0x120(%r15),%r10
  0x0000026a64f209cb: test   %eax,(%r10)
  0x0000026a64f209ce: retq   
```
很遗憾，即便高层次上HIR做了条件表达式消除，后面到低层次LIR再到本地代码生成也没有产出cmov系列指令。