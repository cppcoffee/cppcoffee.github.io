---
layout: post
title:  "Process crash print stacktrace -- C Library"
subtitle: ""
description: "Process crash print stacktrace -- C Library"
date: 2021-04-25
author: "Sharp Liu"
categories: system program
---

{{ page.title }}


### 简述

在使用非内存安全，直接操作内存指针的计算机语言进行开发时，不免会碰到操作野指针、回收再访问的内存等等让进程崩溃的情况。

进程 crash 后，如果有开启 coredump 功能，linux 系统会 dump 进程相关信息到文件中。
在不安装 debug-info 源码包查看 coredump 产生的 core 崩溃的堆栈信息，可以使用如下 gdb 命令：

```shell
# batch mode 下，执行 bt 打印堆栈
gdb -batch -c ./coredump-nginx-pid -ex bt /bin/nginx
```

coredump 开启后，碰到 crash 的进程占用较大内存时，导致 dump 进程数据到磁盘过程过长，机械磁盘负载会持续飙高。

但如果限制了 coredump 次数与 coredump 文件的大小，会导致某些条件的 coredump 无法被发现。

本文描述开发 crash 输出栈信息到 C Library 的实现。


### crash 调用栈信息输出

如果进程 crash 后，将导致 crash 的调用栈信息输出到文件，这样可以方便查找问题。

主要逻辑如下：
1. 进程启动后，调用库的初始化，注册进程 crash 的信号处理
2. 当 crash 发生后，调用信号处理函数
3. 在处理函数中，将调用栈输出到 stderr

该 C Library 依赖 libbfd (Binary File Descriptor library)，使用它来解析 elf sections，找出调用栈函数名和代码行。

libbfd 由 binutils package 提供，在 CentOS 中，可以使用下列命令行进行安装：

```shell
yum install binutils-devel -y
```


### 主要实现

下列代码将 crash 的信息输出到 stderr。

更详细的代码见文末 github libstacktrace 仓库链接


```c
#define _GNU_SOURCE
#include <execinfo.h>
#include <assert.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <limits.h>
#include <signal.h>
#include <unistd.h>

#include "symbol_table.h"


// The max number of levels in the stack trace
#define STACK_TRACE_MAX_LEVELS  100
#define BUFFER_LENGTH           4096


typedef void (*signal_handler_t)(int signo, siginfo_t *info, void *ctx);

static void register_crash_handlers();
static int backtrace_symbol_write(int fd, const char *text, void *addr);


// store process full path.
static char program_path[PATH_MAX];
// the current program binary symbol table.
static symbol_table_t symtab;


// initialize stacktrace library.
int init_stacktrace()
{
    int n;

    n = readlink("/proc/self/exe", program_path, PATH_MAX);
    if (n < 0 || n >= PATH_MAX) {
        return -1;
    }

    program_path[n] = '\0';

    if (symbol_table_build(program_path, &symtab) != 0) {
        return -2;
    }

    register_crash_handlers();

    return 0;
}


static void
stack_trace_dump()
{
    int             i, btl;
    char          **strings;
    void           *stack[STACK_TRACE_MAX_LEVELS + 1];
    const char     *msg = " - STACK TRACE: \n";

    if (write(STDERR_FILENO, program_path, strlen(program_path)) == -1) {
        return;
    }

    if (write(STDERR_FILENO, msg, strlen(msg)) == -1) {
        return;
    }

    memset(stack, 0, sizeof(stack));

    if ((btl = backtrace(stack, STACK_TRACE_MAX_LEVELS)) > 2) {
        strings = backtrace_symbols(stack, btl);
        if (strings != NULL) {
            for (i = 2; i < btl; i++) {
                backtrace_symbol_write(STDERR_FILENO, strings[i], stack[i]);
            }

            free(strings);

        } else {
            backtrace_symbols_fd(stack + 2, btl - 2, STDERR_FILENO);
        }
    }
}


// Reset a signal handler to the default handler.
static void
signal_reset_default(int signo)
{
    struct sigaction act;

    act.sa_handler = SIG_DFL;
    act.sa_flags   = SA_NODEFER | SA_ONSTACK | SA_RESETHAND;
    sigemptyset(&(act.sa_mask));

    assert(sigaction(signo, &act, NULL) == 0);
}


static void
signal_crash_handler(int signo, siginfo_t *siginfo, void *data)
{
    stack_trace_dump();

    signal_reset_default(signo);
    // throw signal to default handler.
    raise(signo);
}


static void
set_signal(int signo, signal_handler_t handler)
{
    struct sigaction act;

    act.sa_handler   = NULL;
    act.sa_sigaction = handler;
    act.sa_flags     = SA_SIGINFO;
    sigemptyset(&(act.sa_mask));

    assert(sigaction(signo, &act, NULL) == 0);
}


static void
register_crash_handlers()
{
    set_signal(SIGBUS,  signal_crash_handler);
    set_signal(SIGSEGV, signal_crash_handler);
    set_signal(SIGILL,  signal_crash_handler);
    set_signal(SIGTRAP, signal_crash_handler);
    set_signal(SIGFPE,  signal_crash_handler);
    set_signal(SIGABRT, signal_crash_handler);
}


static int
backtrace_symbol_format(char *buf, size_t len, const char *prefix, frame_record_t fr)
{
    int     n;
    char   *p = buf;

    // file name
    if (fr.filename != NULL) {
        n = snprintf(p, len, "%s %s", prefix, fr.filename);

    } else {
        n = snprintf(p, len, "%s ??", prefix);
    }

    p += n;
    len -= n;

    // function name
    if (fr.functionname != NULL && *fr.functionname != '\0') {
        n = snprintf(p, len, " %s()", fr.functionname);

    } else {
        n = snprintf(p, len, " ??");
    }

    p += n;
    len -= n;

    // line
    if (fr.line != 0) {
        n = snprintf(p, len, ":%u", fr.line);

        p += n;
        len -= n;
    }

    // discriminator
    if (fr.discriminator != 0) {
        n = snprintf(p, len, " (discriminator %u)\n", fr.discriminator);

    } else {
        n = snprintf(p, len, "\n");
    }

    p += n;
    len -= n;

    return p - buf;
}


static int
backtrace_symbol_write(int fd, const char *text, void *addr)
{
    frame_record_t  fr;
    int             n;
    char            buf[BUFFER_LENGTH + 1];

    if (symbol_table_find(&symtab, addr, &fr)) {
        n = backtrace_symbol_format(buf, BUFFER_LENGTH, text, fr);

    } else {
        n = snprintf(buf, BUFFER_LENGTH, "%s\n", text);
    }

    buf[n] = '\0';

    if (write(fd, buf, strlen(buf)) == -1) {
        return -1;
    }

    return 0;
}
```


### 崩溃栈输出

在 example.c 中，有个访问空指针的代码，导致 crash，完整代码如下：

```shell
#include <stdio.h>
#include "stacktrace.h"


static void bar()
{
    // 访问空指针，导致 crash
    char *p = 0;
    *p = 'a';
}


static void foo()
{
    bar();
}


int main()
{
    // 初始化 library
    init_stacktrace();

    foo();

    return 0;
}
```

运行后，example 进程 crash 输出：

```shell
[root@localhost libstacktrace]# ./example
/home/sharp/libstacktrace/example - STACK TRACE:
/lib64/libc.so.6(+0x36450) [0x7fca8db52450]
./example() [0x4036f0] example.c bar()
./example() [0x403703] example.c foo()
./example() [0x40371d] ?? main()
/lib64/libc.so.6(__libc_start_main+0xf5) [0x7fca8db3e555]
./example() [0x402c9a] ?? _start()
Segmentation fault
```

如果使用 -g 编译，crash 输出更详细，包括崩溃的具体代码行：

```shell
[root@localhost libstacktrace]# ./example
/home/sharp/libstacktrace/example - STACK TRACE:
/lib64/libc.so.6(+0x36450) [0x7fa21a2e0450]
./example() [0x4036f0] /home/sharp/libstacktrace/example.c bar():8
./example() [0x403703] /home/sharp/libstacktrace/example.c foo():15
./example() [0x40371d] /home/sharp/libstacktrace/example.c main():24
/lib64/libc.so.6(__libc_start_main+0xf5) [0x7fa21a2cc555]
./example() [0x402c9a] ?? _start()
Segmentation fault
```


### 参考

[https://github.com/cppcoffee/libstacktrace](https://github.com/cppcoffee/libstacktrace)

[https://man7.org/linux/man-pages/man1/gdb.1.html](https://man7.org/linux/man-pages/man1/gdb.1.html)

[https://man7.org/linux/man-pages/man1/addr2line.1.html](https://man7.org/linux/man-pages/man1/addr2line.1.html)

[https://github.com/apache/trafficserver/blob/master/src/tscore/signals.cc](https://github.com/apache/trafficserver/blob/master/src/tscore/signals.cc)

[https://sourceware.org/binutils/docs/bfd/](https://sourceware.org/binutils/docs/bfd/)

