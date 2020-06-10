---
title: 探究coroutine（二）
date: 2020-06-10 23:08:41
tags: coroutine
categories: linux
---

## previously

在上一篇我们简单了解了linux下的调用约定：每个CPU上有16个通用寄存器，其中6个在调用函数的时候用作参数传递，多余6个的参数通过栈来传递。进程的栈空间每调用一层函数就向低地址方向扩展一个栈帧。

这两篇文章的目的是讲清楚协程在线程的栈上切换的流程。本文分析两个开源项目：云风的coroutine和腾讯的libco。我们只关注其栈的切换逻辑。

## coroutine

coroutine的地址为：[cloudwu/coroutine](https://github.com/cloudwu/coroutine) 。 这个项目实现的非常简洁，核心就两个文件 `coroutine.h` 和`coroutine.c`.  头文件提供的接口：

<!-- more -->

```c
struct schedule;
// coroutine执行体
typedef void (*coroutine_func)(struct schedule *, void *ud);
// 打开一个coroutine调度者
struct schedule * coroutine_open(void);
void coroutine_close(struct schedule *);

// 创建协程，参数为 协程执行的函数和函数的参数
int coroutine_new(struct schedule *, coroutine_func, void *ud);
// 恢复编号为id的协程继续执行，此处发生上下文切换
void coroutine_resume(struct schedule *, int id);
// 通过协程id查询协程状态
int coroutine_status(struct schedule *, int id);
int coroutine_running(struct schedule *);
// 协程让出CPU，做上下文切换
void coroutine_yield(struct schedule *);
```

Coroutine实现的是stackful类型的协程，在一个schedule中可以有多个coroutine对象。与之相对应的为stackless 协程，stackless的协程只能在调用中切换，yield时恢复为其调用者的栈。

在云风实现的coroutine中，重要的一点是schedule结构在堆上分配，schedule中定义了一个1024*1024的共享栈。

```c
 #define STACK_SIZE (1024*1024)
 #define DEFAULT_COROUTINE 16
 struct coroutine;
 struct schedule {
   char stack[STACK_SIZE]; // 共享栈
   ucontext_t main;        // coroutine 使用的是 ucontext系列函数：makecontext/setcontext/swapcontext/getcontext
   int nco;
   int cap;
   int running;
   struct coroutine **co;    // schedule管理的所有coroutine
 };
// 协程结构， func 为要执行的调用， ud 为参数
struct coroutine {
	coroutine_func func;
	void *ud;
	ucontext_t ctx;
	struct schedule * sch;
	ptrdiff_t cap;
	ptrdiff_t size;
	int status;
	char *stack;
};
```

coroutine中使用的是系统提供的ucontext系列函数：makecontext/setcontext/swapcontext/getcontext, 这些函数是libc提供的用户上下文操作函数。

我们看看resume的时候如何切换

```c
 void 
 coroutine_resume(struct schedule * S, int id) {
   assert(S->running == -1);
   assert(id >=0 && id < S->cap);
   struct coroutine *C = S->co[id];
   if (C == NULL)
     return;
   int status = C->status;
   switch(status) {
   case COROUTINE_READY:
     getcontext(&C->ctx);
     C->ctx.uc_stack.ss_sp = S->stack;  // 关键点，指定使用 schedule 上的 stack 作为栈空间
     C->ctx.uc_stack.ss_size = STACK_SIZE; // stack 的大小
     C->ctx.uc_link = &S->main;
     S->running = id; 
     C->status = COROUTINE_RUNNING;
     uintptr_t ptr = (uintptr_t)S;
     makecontext(&C->ctx, (void (*)(void)) mainfunc, 2, (uint32_t)ptr, (uint32_t)(ptr>>32));  // mainfunc实际上是一个封装，mainfunc退出标志着协程退出
     swapcontext(&S->main, &C->ctx);  // swapcontext导致C->ctx 换入并开始执行，实际上就是执行mainfunc，当前的栈保存在S->main
     break;
   case COROUTINE_SUSPEND:
     memcpy(S->stack + STACK_SIZE - C->size, C->stack, C->size);
     S->running = id; 
     C->status = COROUTINE_RUNNING;
     swapcontext(&S->main, &C->ctx);
     break;
   default:
     assert(0);
   }
 }

```

首次创建的coroutine处于COROUTINE_READY状态，调用getcontext初始化协程的ucontext_t结构。然后关键点是`C->ctx.uc_stack.ss_sp = S->stack` 指定使用共享栈，其大小为STACK_SIZE。 makecontext给C->ctx绑定了用户函数封装，最后调用swapcontext将协程的栈换入，并且开始执行，实际回去执行mainfunc。maincfunc中根据S->running查到对应的coroutine，并且开始执行其func, 如下所示：

```c
 static void
 mainfunc(uint32_t low32, uint32_t hi32) {
   uintptr_t ptr = (uintptr_t)low32 | ((uintptr_t)hi32 << 32);
   struct schedule *S = (struct schedule *)ptr;  // 合并两个参数得到 schedule 对象指针
   int id = S->running;   // 查询当前正在执行的coroutine id
   struct coroutine *C = S->co[id];  // 取出 running状态的 coroutine
   C->func(S,C->ud);  // 执行coroutine指定的有用户调用
   _co_delete(C);
   S->co[id] = NULL;
   --S->nco;
   S->running = -1;
 }
```

这里的关键点是`C->func(S, C->ud)`在用户函数内，用户要适时让出CPU，调用coroutine_yield。如果用户函数直接执行结束，那么当前协程就执行完成销毁了。

coroutine_yield 中首先是保存当前用户函数的栈内容，然后再次调用swapcontext, 以激活上次执行的栈空间。

```c
 static void
 _save_stack(struct coroutine *C, char *top) {
   // top 参数是 S->stack + STACK_SIZE, schedule在堆上分配，在resume的时候，指定了STACK_SIZE, 我猜测，运行的时候会从
   // S->stack + STACK_SIZE 向下占用，模拟栈向下扩展的特性
   char dummy = 0;   // 巧妙的通过定义一个局部变量，取起地址，这样 top - &dummy 就是当前栈的所有内容
   assert(top - &dummy <= STACK_SIZE);
   if (C->cap < top - &dummy) {
     free(C->stack);
     C->cap = top-&dummy;
     C->stack = malloc(C->cap);  // 首次执行一定会分配，后面根据大小重分配
   }
   C->size = top - &dummy;
   memcpy(C->stack, &dummy, C->size);
 }
 
 void
 coroutine_yield(struct schedule * S) {
   int id = S->running;
   assert(id >= 0);
   struct coroutine * C = S->co[id];
   assert((char *)&C > S->stack);
   _save_stack(C,S->stack + STACK_SIZE);  // 保存栈内容到C分配的栈空间上
   C->status = COROUTINE_SUSPEND;
   S->running = -1;
   swapcontext(&C->ctx , &S->main);  // swapcontext 将导致 S->main 的执行，并将当前上下文保存在C->ctx
 }
```

## libco

libco项目就略显复杂，我们只看其做context swap的操作。libco的切换使用了汇编，在coctx_swap.S中直接操作寄存器来完成切换。

libco中自定义了ctx结构

```c
struct coctx_t
{
#if defined(__i386__)
	void *regs[ 8 ];
#else
	void *regs[ 14 ];
#endif
	size_t ss_size;
	char *ss_sp;
	
};
```

这个结构中定义了void \*数组用于保存寄存器的内容（x86_64下寄存器为8字节，void \* 也是8字节)。x86_64总共16个寄存器，但是此处只定义了14位置，其中rsp为栈顶寄存器，不需要保存，另外r10寄存器没有保存，这个我还没有搞清楚为什么。

 下面只截取了x86_64架构下的汇编执行流程：

```asm
 .globl coctx_swap
 #if !defined( __APPLE__ )
 .type  coctx_swap, @function
 #endif
 coctx_swap:
    leaq (%rsp),%rax  // 加载当前的栈地址
     movq %rax, 104(%rdi)  // $rsp 地址 regs[13]
     movq %rbx, 96(%rdi)  // regs[12]
     movq %rcx, 88(%rdi)  // regs[11]
     movq %rdx, 80(%rdi)  // regs[10] 
     movq 0(%rax), %rax  // 加载返回地址
     movq %rax, 72(%rdi)  // 当前的返回地址 放在 regs[9]
     movq %rsi, 64(%rdi)  // regs[8]
     movq %rdi, 56(%rdi)  // regs[7]
     movq %rbp, 48(%rdi)  // regs[6]
     movq %r8, 40(%rdi)  // regs[5]
     movq %r9, 32(%rdi)   // regs[4]
     movq %r12, 24(%rdi)  // regs[3]
     movq %r13, 16(%rdi)  // regs[2]
     movq %r14, 8(%rdi)  // regs[1]
     movq %r15, (%rdi)  // regs[0]
     xorq %rax, %rax  // 清零
 
     movq 48(%rsi), %rbp  //
     movq 104(%rsi), %rsp  // 使用之前保存的栈地址
     movq (%rsi), %r15
     movq 8(%rsi), %r14
     movq 16(%rsi), %r13
     movq 24(%rsi), %r12
     movq 32(%rsi), %r9
     movq 40(%rsi), %r8
     movq 56(%rsi), %rdi
     movq 80(%rsi), %rdx
     movq 88(%rsi), %rcx
     movq 96(%rsi), %rbx
     leaq 8(%rsp), %rsp  // 栈地址向上推进了
     pushq 72(%rsi)  //  rsi 压栈, 导致 rsp - 8，ret 将从之前保存的返回地址, 开始执行
 
     movq 64(%rsi), %rsi  // 
   ret
```

从`xorq %rax, %rax`, 将代码分成两段，前半段是保存当前的所有寄存器值，后半段是恢复之前保存的寄存器值。

首先`leaq (%rsp),%rax`  加载当前的栈地址到rax中，实际上此时M[rsp]处保存的值为返回地址（回忆上一篇，调用过程时call指令将rip地址压栈）。然后`movq 0(%rax), %rax ` 加载M[$rax],  `movq %rax, 72(%rdi)`  作用是将返回地址放在regs[9]  （rdi参数为ctx地址）。这一段的其他指令都是在保存寄存器的值。

第二段，从给定的处于pending状态的ctx（rsi寄存器传递进来的）恢复所有寄存器。对等的，`movq 104(%rsi), %rsp` 将pending时使用的栈恢复到rsp寄存器。`leaq 8(%rsp), %rsp` 该条指令将恢复后的栈顶向上收缩8字节，然后指令`pushq 72(%rsi)` 将 regs[9]中保持的返回地址压栈，这样当前执行流程就返回到上一次保持的执行地址去了。