# [Inside HotSpot] C1编译器HIR的构造

## 1. 简介
这篇文章可以说是Christian Wimmer硕士论文[Linear Scan Register Allocation for the Java HotSpot™ Client Compiler](http://compilers.cs.uni-saarland.de/ssasem/talks/Christian.Wimmer.pdf)的不完整翻译，这篇论文详细论述了HotSpot JIT编译器的架构，然后描述了C1编译器（研究用，细节和Sun的Client编译器生产级实现有些许出入）中线性扫描寄存器分配算法的设计和实现。
C1编译器内部使用HIR，LIR做为中间表示并进行系列优化和寄存器分配。字节码到HIR的构造是最先完成的步骤，其中HIR是一个平台无关的图结构IR，它的构造位于[`hotspot\share\c1\c1_IR.cpp`](http://hg.openjdk.java.net/jdk/jdk12/file/06222165c35f/src/hotspot/share/c1/c1_IR.cpp)，整个源码的层次如下：

![Linear Scan Register Allocation for the Java HotSpot™ Client Compiler](c1_instr_hierarchy.png)

上面是不完整的类层次图，列举了最重要的类。所有的指令都是继承自[Instruction](http://hg.openjdk.java.net/jdk/jdk12/file/06222165c35f/src/hotspot/share/c1/c1_Instruction.cpp)：

```cpp
class Instruction: public CompilationResourceObj {
 private:
    int          _id;                              // 指令id
    int          _use_count;                       // 值的使用计数
    int          _pin_state;                       // pin的原因
    ValueType*   _type;                            // 指令类型
    Instruction* _next;                            // 下一个指令（如果下一个是BlockEnd则为null）
    Instruction* _subst;                           // 替换指令，如果有的话。。
    LIR_Opr      _operand;                         // LIR operand信息
    unsigned int _flags;                           // flag信息

    ValueStack*  _state_before;                    // Copy of state with input operands still on stack (or NULL)
    ValueStack*  _exception_state;                 // Copy of state for exception handling
    XHandlers*   _exception_handlers;              // Flat list of exception handlers covering this instruction

    protected:
    BlockBegin*  _block;                           // 包含该指令的Block的指针
  ..
};
```

## 2. HIR的设计
BlockBegin表示一个基本块的开始，BlockEnd表示结束，两者有一个指针相连，这样可以快速的遍历控制流图，而不需要经过基本块中间的指令。BlockBegin的字段predecessors表示前驱基本块，由于前驱可能是多个，所以是BlockList结构，BlockList是`GrowableArray<BlockBegin*>`，即多个BlockBegin组成的可扩容数组。同理，BlockEnd的successors表示多个后继基本块。

![](c1_cfg.png)

基本块除了BlockBegin，BlockEnd之外就是主体部分，这一部分由具体的Instruction子类组成，每个Instruction有一个next指针（见上面源码），以此可以构成Instruction链表。

## 3. 高观点层次的字节码到HIR构造
在[[Inside HotSpot] C1编译器工作流程及中间表示](https://www.cnblogs.com/kelthuzadx/p/10740453.html)我们提到build_hir()这个函数会执行HIR的构造，HIR的优化。从字节码到HIR的构造最终调用的是[GraphBuilder](http://hg.openjdk.java.net/jdk/jdk12/file/06222165c35f/src/hotspot/share/c1/c1_GraphBuilder.cpp)。

GraphBuilder首先使用BlockListBuilder遍历字节码构造所有基本块，然后储存为一个链表结构，即之前提到的`GrowableArray<BlockBegin*>`，但是这个时候的基本块只有BlockBegin，不包括具体的字节码。第二步GraphBuilder用一个ValueStack作为操作数栈和局部变量表，模拟执行字节码，构造出对应的HIR，填充之前空的基本块：

![](c1_hir_construct.png)

上图是非常直观的构造过程，上面的**local variable**和**operand stack**位于ValueStack里面，当执行iload_1时候，局部变量表的索引1位置的变量i8压入操作数栈；执行iload_0压入i7到操作数栈（下文用栈代替），imul弹出栈顶两个值，然后构造出HIR指令`i11 = i8 * i7`，然后根据imul的语义生成的i11入栈；istore_1弹出栈的数据保存到局部变量表；iconst_1构造出HIR指令`i12 = 1`然后压入i12,；isub弹出栈顶做减法，构造出`i13 = i7 - i12`，将i13压入栈；最后istore_0弹出栈顶i13写入局部变量表。

## 4. 源码层次的字节码到HIR构造
明白高观点下字节码是如何构造出HIR的，源码层次也变得很容易理解了。ValueStack表示用于模拟字节码执行的操作数栈和局部变量表：
```cpp
// hotspot\share\c1\c1_ValueStack.hpp
class ValueStack: public CompilationResourceObj {
 public:
  enum Kind {
    Parsing,             // During abstract interpretation in GraphBuilder
    CallerState,         // Caller state when inlining
    StateBefore,         // Before before execution of instruction
    StateAfter,          // After execution of instruction
    ExceptionState,      // Exception handling of instruction
    EmptyExceptionState, // Exception handling of instructions not covered by an xhandler
    BlockBeginState      // State of BlockBegin instruction with phi functions of this block
  };

 private:
  IRScope* _scope;                               // the enclosing scope
  ValueStack* _caller_state;
  int      _bci;
  Kind     _kind;

  Values   _locals;                              // the locals
  Values   _stack;                               // the expression stack
  Values   _locks;                               // the monitor stack (holding the locked values)
...
};
```
然后GraphBuilder的模拟过程如下：
```cpp
// hotspot\share\c1\c1_GraphBuilder.cpp
// 加载局部变量表的值到栈
void GraphBuilder::load_local(ValueType* type, int index) {
  Value x = state()->local_at(index);
  assert(x != NULL && !x->type()->is_illegal(), "access of illegal local variable");
  push(type, x);
}
// 储存栈的值到局部变量表
void GraphBuilder::store_local(ValueType* type, int index) {
  Value x = pop(type);
  store_local(state(), x, index);
}
// 逻辑运算
void GraphBuilder::logic_op(ValueType* type, Bytecodes::Code code) {
  Value y = pop(type);
  Value x = pop(type);
  push(type, append(new LogicOp(code, x, y)));
}
// 比较运算
void GraphBuilder::compare_op(ValueType* type, Bytecodes::Code code) {
  ValueStack* state_before = copy_state_before();
  Value y = pop(type);
  Value x = pop(type);
  ipush(append(new CompareOp(code, x, y, state_before)));
}
```
按照JVM虚拟机规范上字节码的语义来。比如上面的logic_op即从操作数栈弹出两个值，然后做逻辑运算，具体逻辑运算类型用code表示。除了这些常规的指令模拟外，还有些特殊指令可能会在模拟之外增加一些内容，比如在分层编译模式下goto和if指令可能增加回边计数等profiling信息。