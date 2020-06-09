---
title: 探究coroutine（一）
date: 2020-06-08 22:26:33
tags: linux asm coroutine
---



## what is coroutine

在很多高级语言中有协程，比如golang中的goroutine，lua中的coroutine，boost中的fiber。协程是比线程更小的运行单位，通常支持协程的语言都是在一个线程中运行多个协程，线程中某一时刻只有一个协程在运行。当协程进入阻塞调用时，切换上线文，把pending的其他协程还原到线程栈内执行下一个协程。

协程让我们写的代码看起来是同步的，但是实际运行时却是异步。

## coroutine impl

如何实现协程？实际上协程的切换就是线程上线文的切换。为了搞清楚上下文切换，我们首先需要搞清楚进程内如何调用子过程。

<!--more--> 

### registers

每个CPU上有16个寄存器（只考虑x86_64), 这些寄存器是运行时刻暂存数据，以及调用函数传递参数的。每个寄存器有各自的作用，比如rsp是栈寄存器，rax用于存函数返回值。所谓的上下文就是某一时刻这些寄存器的“镜像”。

在x86_64 上，这些寄存器位： rax， rbx， rcx， rdx， rsi， rdi，rbp， rsp， r8-r16, 最后的8个寄存器是在64位CPU扩展上才有的。为了兼容32位程序机器码， 这些寄存器也分别提供了32位兼容的名称，即：eax，ebx， ecx,等等。

在64位机上执行函数调用的时候，会使用其中的六个寄存器来传递参数（google的c++编码规范规定函数调用不超过6个参数，如果超过六个参数就需要通过压栈的方式传递参数）

| 第一参数 | 第二参数 | 第三参数 | 第四参数 | 第五参数 | 第六参数 |
| -------- | -------- | -------- | -------- | -------- | -------- |
| rdi      | rsi      | rdx      | rcx      | r8       | r9       |

示例

```c
// file: test.c
long test(long a, long b, long c, long d, long e, long f) {
 long x = a + b;
 long y = c + d;
 long z = e + f;
 long result = x + y + z;
 return result;
}
```

我们定义如上一个函数，执行 `gcc -Og -S test.c`,  生成汇编文件 test.s 内容如下

```asm
	.file	"test.c"
	.text
	.globl	test
	.type	test, @function
test:
.LFB0:
	.cfi_startproc
	addq	%rsi, %rdi  # 第一、第二参数相加的和存入rdi
	addq	%rcx, %rdx  # 第三、第四参数相加的和存入rdx
	addq	%r9, %r8    # 第五、第六参数相加的和存入r8
	addq	%rdi, %rdx  # 前两个和相加存入 rdx
	leaq	(%rdx,%r8), %rax  # rdx与r8 相加 放入 rax， rax是返回值寄存器
	ret
	.cfi_endproc
.LFE0:
	.size	test, .-test
	.ident	"GCC: (GNU) 4.8.5 20150623 (Red Hat 4.8.5-39)"
	.section	.note.GNU-stack,"",@progbits
```



### process space

我们知道每个进程有各自的进程空间，其布局如下

{% asset_img process_space.png ProcessSpace %}

栈空间是从高地址向低地址方向扩展的。

通常每次调用函数，就会导致栈空间向下扩展一个栈帧。在x86_64的架构下，如果函数内完全不需要再栈上分配局部变量、或者函数内没有继续调用其他函数，那么栈甚至不用向下扩展。

比如，我们在main函数中调用刚才定义的test函数

```c
// file : main.c
#include <stdio.h>
long test(long, long, long, long, long, long);

int main() {
 long a = 1;
 long b = 2;
 long c = 3;
 long d = 4;
 long e = 5;
 long f = 6;
 long x = test(a, b, c, d, e, f);
 printf("%ld\n", x);
 return 0;
}
```

使用gcc生成汇编 `gcc -Og -S main.c`, 输出如下：

```asm
	.file	"main.c"
	.section	.rodata.str1.1,"aMS",@progbits,1
.LC0:
	.string	"%ld\n"
	.text
	.globl	main
	.type	main, @function
main:
.LFB11:
	.cfi_startproc
	subq	$8, %rsp
	.cfi_def_cfa_offset 16
	movl	$6, %r9d  //按照过程调用约定，向寄存器中存入参数
	movl	$5, %r8d
	movl	$4, %ecx
	movl	$3, %edx
	movl	$2, %esi
	movl	$1, %edi
	call	test      // 调用test函数, rip 指向下一条指令，并且将rip的值pushq，压栈
	movq	%rax, %rsi  // 返回后执行的地址
	movl	$.LC0, %edi
	movl	$0, %eax
	call	printf
	movl	$0, %eax
	addq	$8, %rsp
	.cfi_def_cfa_offset 8
	ret
	.cfi_endproc
.LFE11:
	.size	main, .-main
	.ident	"GCC: (GNU) 4.8.5 20150623 (Red Hat 4.8.5-39)"
	.section	.note.GNU-stack,"",@progbits
```

在汇编之后的文件中，`call test` 会把下一条指令的地址压栈，test中ret指令会popq，将压栈的地址出栈，放入rip（程序指针寄存器）。

实际运行看看，首先编译： `gcc -Og -ggdb -o main main.c test.c`， 使用`objdump -d main` 查看main和test两个函数的机器码与汇编的对应

```asm
000000000040052d <main>:
  40052d:       48 83 ec 08             sub    $0x8,%rsp
  400531:       41 b9 06 00 00 00       mov    $0x6,%r9d
  400537:       41 b8 05 00 00 00       mov    $0x5,%r8d
  40053d:       b9 04 00 00 00          mov    $0x4,%ecx
  400542:       ba 03 00 00 00          mov    $0x3,%edx
  400547:       be 02 00 00 00          mov    $0x2,%esi
  40054c:       bf 01 00 00 00          mov    $0x1,%edi
  400551:       e8 1c 00 00 00          callq  400572 <test>
  400556:       48 89 c6                mov    %rax,%rsi
  400559:       bf 20 06 40 00          mov    $0x400620,%edi
  40055e:       b8 00 00 00 00          mov    $0x0,%eax
  400563:       e8 a8 fe ff ff          callq  400410 <printf@plt>
  400568:       b8 00 00 00 00          mov    $0x0,%eax
  40056d:       48 83 c4 08             add    $0x8,%rsp
  400571:       c3                      retq   

0000000000400572 <test>:
  400572:       48 01 f7                add    %rsi,%rdi
  400575:       48 01 ca                add    %rcx,%rdx
  400578:       4d 01 c8                add    %r9,%r8
  40057b:       48 01 fa                add    %rdi,%rdx
  40057e:       4a 8d 04 02             lea    (%rdx,%r8,1),%rax
  400582:       c3                      retq   
  400583:       66 2e 0f 1f 84 00 00    nopw   %cs:0x0(%rax,%rax,1)
  40058a:       00 00 00 
  40058d:       0f 1f 00                nopl   (%rax)

```

编译之后，所有的代码有了存储地址，test函数位于 400572 处，所以main中调用指令是`callq 400572 <test>`。我们在test函数中定个断点，在断点处的rsp（栈定寄存器）一定是`400551:       e8 1c 00 00 00          callq  400572 <test>` 这条代码的下一个位置`400556`

{% asset_img rsp.png RSP %}

在断点处，我们发现`rsp`寄存器的地址为0x7fffffffe418， 该处存储的内容为0x400556, 刚好是`callq test`下一条指令的地址。

## Next episode

下篇预告： 下一篇文章我们分析两个开源协程库（腾讯开源的协程库libco和云风写的简单协程coroutine）的实现

