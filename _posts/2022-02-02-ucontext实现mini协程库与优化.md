---
layout: post
title:  "ucontext实现mini协程库与优化"
subtitle: ""
description: "ucontext实现mini协程库与优化"
date: 2022-02-02
author: "Sharp Liu"
categories: system program
---

{{ page.title }}
## 简介

Linux 下提供 `ucontext` 系列 API 来实现协程（coroutine）操作，协程可以由开发者实现调度。

`ucontent` 是 `setjmp`/`longjmp` 的高级版，支持携带参数调用。

`ucontext` APIs:

```
#include <ucontext.h>

int getcontext(ucontext_t *ucp);
int setcontext(const ucontext_t *ucp);

void makecontext(ucontext_t *ucp, void (*func)(), int argc, ...);
int swapcontext(ucontext_t *restrict oucp, const ucontext_t *restrict ucp);
```

使用 ucontext 系列 API 实现协程库需要实现基本的 coroutine `yield`/`resume` 接口，其中

- `resume`: 重新执行协程暂停的位置
- `yield`: 在当前点暂停协程的执行


## 实现

### coroutine 状态

协程状态分成四种，定义四种协程状态

1. 准备就绪(ready)
2. 运行中(resume)
3. 暂停中(yield)
4. 运行完成(done)

```c
typedef enum {
    COROUTINE_READY,
    COROUTINE_RUNNING,
    COROUTINE_SUSPEND,
    COROUTINE_DEAD,
} coroutine_status_e;
```


### coroutine 结构体

协程结构体需要包含协程栈大小和协程相关状态，使用 stack_id 用于解决使用 valgrind 跟踪出现的栈变动警告。

```c
typedef struct {
    ucontext_t              main;
    ucontext_t              ctx;

    // 协程执行入口函数与参数
    coroutine_pt            func;
    void                   *ud;

    // 协程栈指针与栈大小
    void                   *stack;
    size_t                  stack_size;

    // 协程运行状态
    coroutine_status_e       status;

    int                     stack_id;

    // 协程是否运行完成
    unsigned                done:1;
} coroutine_t;
```


### coroutine_create

创建协程，指定协程运行函数的入口与参数，还有协程运行需要的栈大小。

如果指定栈大小为0，就使用 `SIGSTKSZ` 定义的大小。

```c
coroutine_t *
coroutine_create(coroutine_pt fn, void *ud, size_t stack_size)
{
    coroutine_t  *co;
    size_t        size;

    if (stack_size == 0) {
        stack_size = SIGSTKSZ;
    }

    size = sizeof(*co) + stack_size;

    co = malloc(size);
    if (co == NULL) {
        return NULL;
    }

    memset(co, 0, sizeof(*co));

    // 设置协程执行入口函数和参数
    co->func = fn;
    co->ud = ud;

    // 栈与栈大小
    co->stack = co + 1;
    co->stack_size = stack_size;

    co->status = COROUTINE_READY;

    co->stack_id = VALGRIND_STACK_REGISTER(co, (void *) co + size);

    return co;
}
```


### coroutine_resume

协程切换/调度，恢复协程运行，并更新协程状态。

```c
int
coroutine_resume(coroutine_t *co)
{
    switch (co->status) {
    case COROUTINE_READY:
        if (getcontext(&co->ctx) == -1) {
            return CO_ERROR;
        }

        co->status = COROUTINE_RUNNING;

        co->ctx.uc_stack.ss_sp = co->stack;
        co->ctx.uc_stack.ss_size = co->stack_size;
        co->ctx.uc_stack.ss_flags = 0;
        co->ctx.uc_link = &co->main;

        // 协程主入口 coroutine_mainfunc
        makecontext(&co->ctx, (void (*)(void)) coroutine_mainfunc, 1, co);

        if (swapcontext(&co->main, &co->ctx) == -1) {
            return CO_ERROR;
        }

        break;

    case COROUTINE_SUSPEND:
        co->status = COROUTINE_RUNNING;

        if (swapcontext(&co->main, &co->ctx) == -1) {
            return CO_ERROR;
        }

        break;

    default:
        /* unreachable */
        assert(0);
    }

    if (co->done) {
        coroutine_destroy(co);
    }

    return CO_OK;
}
```


### coroutine_mainfunc

协程运行的入口函数，间接的调用传递的入口函数，并设置协程完成标识位。

```c
static void
coroutine_mainfunc(void *data)
{
    coroutine_t  *co = data;

    co->func(co->ud);

    co->done = 1;
}
```


### coroutine_yield

协程暂停，切换到 main context 运行

```c
int
coroutine_yield(coroutine_t *co)
{
    co->status = COROUTINE_SUSPEND;

    if (swapcontext(&co->ctx, &co->main) == -1) {
        return CO_ERROR;
    }

    return CO_OK;
}
```


## 协程优化

协程运行会频繁的调用 `swapcontext` 与 `getcontext`，如果继续使用 `ucontext` 系列结构，那么精简 `ucontext` 调用的汇编指令会是优化的关键

1. 移除 `swapcontext` 内部调用设置的 `sig_flags` API 操作
2. 移除参数寄存器 (x64 上面是 RDI, RDX, RCX, R8, R9 and RSI) 操作
3. 移除浮点数寄存器操作


### ucontext_i.h

定义寄存器存储的偏移量

```c
#define SIG_BLOCK 0
#define SIG_SETMASK 2
#define _NSIG8 8
#define oRBP 120
#define oRSP 160
#define oRBX 128
#define oR8 40
#define oR9 48
#define oR10 56
#define oR11 64
#define oR12 72
#define oR13 80
#define oR14 88
#define oR15 96
#define oRDI 104
#define oRSI 112
#define oRDX 136
#define oRAX 144
#define oRCX 152
#define oRIP 168
#define oEFL 176
#define oFPREGS 224
#define oSIGMASK 296
#define oFPREGSMEM 424
#define oMXCSR 448
```


### lightweight_getcontext

轻量级的 getcontext 实现

```c
lightweight_getcontext.S
#include "ucontext_i.h"


.globl lightweight_getcontext;
.type lightweight_getcontext, @function;
lightweight_getcontext:
.cfi_startproc;
	/* Save the preserved registers, the registers used for passing
	   args, and the return address.  */
	movq	%rbx, oRBX(%rdi)
	movq	%rbp, oRBP(%rdi)
	movq	%r12, oR12(%rdi)
	movq	%r13, oR13(%rdi)
	movq	%r14, oR14(%rdi)
	movq	%r15, oR15(%rdi)

	movq	%rdi, oRDI(%rdi)
	movq	%rsi, oRSI(%rdi)
	movq	%rdx, oRDX(%rdi)
	movq	%rcx, oRCX(%rdi)
	movq	%r8, oR8(%rdi)
	movq	%r9, oR9(%rdi)

	movq	(%rsp), %rcx
	movq	%rcx, oRIP(%rdi)
	leaq	8(%rsp), %rcx		/* Exclude the return address.  */
	movq	%rcx, oRSP(%rdi)

	/* We have separate floating-point register content memory on the
	   stack.  We use the __fpregs_mem block in the context.  Set the
	   links up correctly.  */
	leaq	oFPREGSMEM(%rdi), %rcx
	movq	%rcx, oFPREGS(%rdi)
	/* Save the floating-point environment.  */
	fnstenv	(%rcx)
	stmxcsr oMXCSR(%rdi)

	/* Formerly here: a call to sigprocmask.
	   Deleted because unnecessary for our application. */

	/* All done, return 0 for success.  */
	xorl	%eax, %eax
	ret
.cfi_endproc;
```


### lightweight_swapcontext

轻量级的 swapcontext 实现，移除了注册信号的系统调用

```c
#include "ucontext_i.h"


.globl lightweight_swapcontext;
.type lightweight_swapcontext, @function;
lightweight_swapcontext:
.cfi_startproc;
	/* Save the preserved registers, the registers used for passing args,
	   and the return address.  */
	movq	%rbx, oRBX(%rdi)
	movq	%rbp, oRBP(%rdi)
	movq	%r12, oR12(%rdi)
	movq	%r13, oR13(%rdi)
	movq	%r14, oR14(%rdi)
	movq	%r15, oR15(%rdi)

	/* Don't bother saving  and restoring argument registers */
	movq	%rdi, oRDI(%rdi)
	movq	%rsi, oRSI(%rdi)
	movq	%rdx, oRDX(%rdi)
	movq	%rcx, oRCX(%rdi)
	movq	%r8, oR8(%rdi)
	movq	%r9, oR9(%rdi)

	movq	(%rsp), %rcx
	movq	%rcx, oRIP(%rdi)
	leaq	8(%rsp), %rcx		/* Exclude the return address.  */
	movq	%rcx, oRSP(%rdi)


	/* We have separate floating-point register content memory on the
	   stack.  We use the __fpregs_mem block in the context.  Set the
	   links up correctly.  */
	leaq	oFPREGSMEM(%rdi), %rcx
	movq	%rcx, oFPREGS(%rdi)
	/* Save the floating-point environment.  */
	fnstenv	(%rcx)
	stmxcsr oMXCSR(%rdi)

	/* Formerly here: a call to sigprocmask.
	   Deleted because unnecessary for our application. */

	/* Restore the floating-point context.  Not the registers, only the
	   rest.  */
	movq	oFPREGS(%rsi), %rcx
	fldenv	(%rcx)
	ldmxcsr oMXCSR(%rsi)

	/* Load the new stack pointer and the preserved registers.  */
	movq	oRSP(%rsi), %rsp
	movq	oRBX(%rsi), %rbx
	movq	oRBP(%rsi), %rbp
	movq	oR12(%rsi), %r12
	movq	oR13(%rsi), %r13
	movq	oR14(%rsi), %r14
	movq	oR15(%rsi), %r15

	/* The following ret should return to the address set with
	getcontext.  Therefore push the address on the stack.  */
	movq	oRIP(%rsi), %rcx
	pushq	%rcx

	/* Setup registers used for passing args--don't bother with this  */
	movq	oRDI(%rsi), %rdi
	movq	oRDX(%rsi), %rdx
	movq	oRCX(%rsi), %rcx
	movq	oR8(%rsi), %r8
	movq	oR9(%rsi), %r9

	/* Setup finally  %rsi.  */
	movq	oRSI(%rsi), %rsi

	/* Clear rax to indicate success.  */
	xorl	%eax, %eax
	ret
.cfi_endproc
```


## References

[https://github.com/cppcoffee/coroutine](https://github.com/cppcoffee/coroutine)

[https://man7.org/linux/man-pages/man3/swapcontext.3.html](https://man7.org/linux/man-pages/man3/swapcontext.3.html)

[https://github.com/cloudwu/coroutine](https://github.com/cloudwu/coroutine)

[https://rethinkdb.com/blog/making-coroutines-fast/](https://rethinkdb.com/blog/making-coroutines-fast/)

