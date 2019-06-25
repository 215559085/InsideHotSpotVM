为了观察C1构造HIR，我们准备一段Java代码：
```java
// bytecode
   0: iload_0
   1: sipush        354
   4: if_icmple     18
   7: getstatic     #2                  // Field k:I
  10: iconst_2
  11: iadd
  12: putstatic     #2                  // Field k:I
  15: goto          26
  18: getstatic     #2                  // Field k:I
  21: iconst_1
  22: iadd
  23: putstatic     #2                  // Field k:I
  26: return
// source code
package com.github.kelthuzadx;

public class C1 {
    public static int k = 0;
    public static void F(int i){
        if(i>354){
            k+=2;
        }else{
            k+=1;
        }
    }
    public static void main(String[] argc) throws InterruptedException {
        F(32);
    }
}
```
回到源码上，`c1/c1_GraphBuilder.cpp`会负责HIR的构造工作：
```cpp
// hotspot\share\c1\c1_GraphBuilder.cpp
GraphBuilder::GraphBuilder(Compilation* compilation, IRScope* scope)
  : _scope_data(NULL)
  , _compilation(compilation)
  , _memory(new MemoryBuffer())
  , _inline_bailout_msg(NULL)
  , _instruction_count(0)
  , _osr_entry(NULL)
{
  int osr_bci = compilation->osr_bci();
  
  // 找出基本块，连接它们构造出控制流图(CFG)
  // 此时基本块不包含代码，所以不完整
  BlockListBuilder blm(compilation, scope, osr_bci);
  BlockList* bci2block = blm.bci2block();
  BlockBegin* start_block = bci2block->at(0);

  push_root_scope(scope, bci2block, start_block);

  // 设置入口基本块
  _initial_state = state_at_entry();
  start_block->merge(_initial_state);

  // 用SSA指令填充每个基本块

  // 模拟字节码执行的时候用的操作数栈
  _vmap        = new ValueMap();
  switch (scope->method()->intrinsic_id()) {
  // 如果是特殊方法就特殊处理
  case vmIntrinsics::_dabs          : // fall through
  case vmIntrinsics::_dsqrt         : // fall through
  case vmIntrinsics::_dsin          : // fall through
  case vmIntrinsics::_dcos          : // fall through
  case vmIntrinsics::_dtan          : // fall through
  case vmIntrinsics::_dlog          : // fall through
  case vmIntrinsics::_dlog10        : // fall through
  case vmIntrinsics::_dexp          : // fall through
  case vmIntrinsics::_dpow{...}
  case vmIntrinsics::_Reference_get:{...}

  default:
  // 否则正常模拟执行
    scope_data()->add_to_work_list(start_block);
    iterate_all_blocks();
    break;
  }

  _start = setup_start_block(osr_bci, start_block, _osr_entry, _initial_state);

  eliminate_redundant_phis(_start);

  NOT_PRODUCT(if (PrintValueNumbering && Verbose) print_stats());
  // for osr compile, bailout if some requirements are not fulfilled
  if (osr_bci != -1) {
    BlockBegin* osr_block = blm.bci2block()->at(osr_bci);
    if (!osr_block->is_set(BlockBegin::was_visited_flag)) {
      BAILOUT("osr entry must have been visited for osr compile");
    }

    // check if osr entry point has empty stack - we cannot handle non-empty stacks at osr entry points
    if (!osr_block->state()->stack_is_empty()) {
      BAILOUT("stack not empty at OSR entry point");
    }
  }
}
```
第一步BlockListBuilder找出基本块，然后链接起来构成控制流图。BlockListBuilder做了三件事：

1)使用set_entries创建入口基本块和异常基本块
```cpp
void BlockListBuilder::set_entries(int osr_bci) {
  // 创建入口基本块
  BlockBegin* std_entry = make_block_at(0, NULL);
  if (scope()->caller() == NULL) {
    std_entry->set(BlockBegin::std_entry_flag);
  }
  if (osr_bci != -1) {
    BlockBegin* osr_entry = make_block_at(osr_bci, NULL);
    osr_entry->set(BlockBegin::osr_entry_flag);
  }
  // 创建异常基本块，异常在HIR中叫做XHandler
  XHandlers* list = xhandlers();
  const int n = list->length();
  for (int i = 0; i < n; i++) {
    XHandler* h = list->handler_at(i);
    BlockBegin* entry = make_block_at(h->handler_bci(), NULL);
    entry->set(BlockBegin::exception_entry_flag);
    h->set_entry_block(entry);
  }
}
```
bci表示字节码偏移，`make_block_at()`是个重要的函数，它寻找字节码对应的基本块，如果没有就创建一个，然后设置该基本块的前驱基本块（如果有的话）。

2)set_leader找出每个基本块
leader表示基本块的第一个指令，set_leader目的就是找出所有基本块。方法也很简单，就是遍历字节码，发现有if，go这样可以产生回边的字节码时创建一个基本块。在示例代码中，set_entries会设置字节码偏移0为入口基本块B0；然后遍历到if_icmple时设置7为B1，设置18为基本块B2；接着到goto时设置26为基本块B3，最后的效果如下：

3)mark_loops标记循环


BlockListBuilder找出了所有基本块的起始并连接成了控制流图，但是基本块里面还是空的，接下来就需要iterate_all_blocks处理每个基本块。它会模拟运行字节码，用SSA指令填充基本块。

可以使用`-XX:+PrintIRDuringConstruction`查看这一步生成的控制流图：
```CPP
B0 (SV) [0, -1] sux: B1 B2
empty stack
inlining depth 0
__bci__use__tid____instr____________________________________
locals size: 1 stack size: 0
  1    0    i5     354
. 4    0     6     if i4 <= i5 then B2 else B1

B1 (V) [7, -1] sux: B3 pred: B0
empty stack
inlining depth 0
__bci__use__tid____instr____________________________________
locals size: 1 stack size: 0
  7    0    a7     <instance 0x0000018a7fd50358 klass=java/lang/Class>
. 7    0    i8     a7._104 (I) k
  10   0    i9     2
  11   0    i10    i8 + i9
. 12   0    i12    a7._104 := i10 (I) k
. 15   0     13    goto B3

B2 (V) [18, -1] sux: B3 pred: B0
empty stack
inlining depth 0
__bci__use__tid____instr____________________________________
locals size: 1 stack size: 0
  18   0    a14    <instance 0x0000018a7fd50358 klass=java/lang/Class>
. 18   0    i15    a14._104 (I) k
  21   0    i16    1
  22   0    i17    i15 + i16
. 23   0    i19    a14._104 := i17 (I) k
. 26   0     20    goto B3

B3 (V) [26, -1] pred: B1 B2
empty stack
inlining depth 0
__bci__use__tid____instr____________________________________
locals size: 1 stack size: 0
. 26   0    v21    return
```
sux表示后继基本块，pred表示前驱基本块。bci表示字节码偏移，即SSA指令对应的字节码的偏移。instr表示生成的SSA指令，bci表示对应的字节码偏移。比如bci 7表示`getstatic`，对应SSA如下:
```cpp
  7    0    a7     <instance 0x0000018a7fd50358 klass=java/lang/Class>
. 7    0    i8     a7._104 (I) k
```
第一行获取k所类的Class对象，然后得到k的值，这正是getstatic的语义。