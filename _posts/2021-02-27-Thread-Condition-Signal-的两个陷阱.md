---
layout: post
title:  "Thread Condition Signal 的两个陷阱"
subtitle: ""
description: "Thread Condition Signal 的两个陷阱"
date: 2021-02-27
author: "Sharp Liu"
categories: system program
---
## Thread Condition Signal

当接触 线程条件信号 时，通常是实现生产者和消费者的场景。翻看 man 手册后，很疑惑为什么 cond 需要依赖外部的 mutex？

在 man 手册中没有 example  可以参考，很容易不假思索的写成下面这样子有陷阱的代码：

```c
// producer
pthread_mutex_lock(&mutex);
pthread_cond_signal(&cond);
pthread_mutex_unlock(&mutex);


// consumer
pthread_mutex_lock(&mutex);
pthread_cond_wait(&cond, &mutex);
pthread_mutex_unlock(&mutex);
```

这样子写会步入 *信号丢失的陷阱* 中。


### 信号丢失的陷阱

当 signal 发生于 wait 之前，信号就会丢失

```flow
      +   +----------+    +----------+
      |   | producer |    | consumer |
      |   +----------+    +----------+
      |
      |   +----------+
      |   |   lock   |
      |   +----------+
      |   +----------+
      |   |  signal  |
      |   +----------+
Time  |   +----------+
      |   |  unlock  |
      |   +----------+
      |-------------------------------
      |                   +----------+
      |                   |   lock   |
      |                   +----------+
      |                   +----------+
      |                   |   wait   |
      |                   +----------+
      |                   +----------+
      |                   |  unlock  |
      v                   +----------+
```

这里是一个生产者，一个消费者的场景。
producer 优先执行，导致了信号丢失，consumer 一直在  wait 中。


### 虚假唤醒的陷阱

man `pthread_cond_broadcast` 文档中，`Multiple Awakenings by Condition Signal` 段落提到的 `spurious wakeup` 问题。

考虑到一个生产者，多个消费者的场景:

一个线程尝试等待条件变量，另一个线程并发执行到了 `pthread_cond_signal`，第三个线程已经在等待中。

如下伪代码实现与执行步骤(末尾数字)：

```c
pthread_cond_wait(mutex, cond):
    value = cond->value; /* 1 */
    pthread_mutex_unlock(mutex); /* 2 */
    pthread_mutex_lock(cond->mutex); /* 10 */
    if (value == cond->value) { /* 11 */
        me->next_cond = cond->waiter;
        cond->waiter = me;
        pthread_mutex_unlock(cond->mutex);
        unable_to_run(me);
    } else
        pthread_mutex_unlock(cond->mutex); /* 12 */
        pthread_mutex_lock(mutex); /* 13 */


pthread_cond_signal(cond):
    pthread_mutex_lock(cond->mutex); /* 3 */
    cond->value++; /* 4 */
    if (cond->waiter) { /* 5 */
        sleeper = cond->waiter; /* 6 */
        cond->waiter = sleeper->next_cond; /* 7 */
        able_to_run(sleeper); /* 8 */
    }
    pthread_mutex_unlock(cond->mutex); /* 9 */
```

调用一次 `pthread_cond_signal`，导致了多个 consumer 线程在 `pthread_cond_wait` 或者 `pthread_cond_timedwait` 返回，这现象称为 `spurious wakeup`。


### 解决方法

当实现 Thread Condition Signal 逻辑时，外部的 mutex 锁是为了保证正确性，加入一个条件变量以保证唤醒信号不会丢失。

如下正确的写法：

```c
// producer
pthread_mutex_lock(&mutex);
condition_ = true;
pthread_cond_signal(&cond, &mutex);
pthread_mutex_unlock(&mutex);


// consumer
pthread_mutex_lock(&mutex);
while (!condition_) {
    pthread_cond_wait(&cond);
}
condition_ = false;
pthread_mutex_unlock(&mutex);
```

如果是多个生产者多个消费者的情况，可以将条件改成 count 计数器。


### 参考

[https://man7.org/linux/man-pages/man3/pthread_cond_broadcast.3p.html](https://man7.org/linux/man-pages/man3/pthread_cond_broadcast.3p.html)

[https://code.woboq.org/userspace/glibc/nptl/pthread_cond_wait.c.html](https://code.woboq.org/userspace/glibc/nptl/pthread_cond_wait.c.html)


