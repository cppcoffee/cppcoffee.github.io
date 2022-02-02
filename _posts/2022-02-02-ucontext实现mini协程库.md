---
layout: post
title:  "ucontext实现mini协程库"
subtitle: ""
description: "ucontext实现mini协程库"
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

```c
typedef enum {
    COROUTINE_READY,    // 准备就绪
    COROUTINE_RUNNING,  // 运行中
    COROUTINE_SUSPEND,  // 暂停中
    COROUTINE_DEAD,     // 结束
} coroutine_status_e;
```


### coroutine 结构体

```c
typedef struct {
    ucontext_t              main;
    ucontext_t              ctx;

    coroutine_pt            func;
    void                   *ud;

    void                   *stack;
    size_t                  stack_size;

    coroutine_status_e       status;

    int                     stack_id;

    unsigned                done:1;
} coroutine_t;
```


### coroutine_create

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

### coroutine_yield

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

## 优化

协程运行会频繁的调用 `swapcontext` 与 `getcontext`，精简 `ucontext` 调用的汇编指令会是优化的关键

1. 移除 `swapcontext` 的 sig_flags 操作
2. 移除删除参数寄存器 (x64 上面是 RDI, RDX, RCX, R8, R9 and RSI) 操作
3. 移除浮点数寄存器操作


## References

[https://github.com/cppcoffee/coroutine](https://github.com/cppcoffee/coroutine)
[https://man7.org/linux/man-pages/man3/swapcontext.3.html](https://man7.org/linux/man-pages/man3/swapcontext.3.html)
[https://github.com/cloudwu/coroutine](https://github.com/cloudwu/coroutine)

