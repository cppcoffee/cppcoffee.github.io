---
layout: post
title:  "Lock-Free Stack Implement"
subtitle: ""
description: "Lock-Free Stack Implement"
date: 2021-04-07
author: "Sharp Liu"
categories: datastructure
---

## {{ page.title }}

### 无锁链式栈

栈是一种 LIFO （Last In First Out） 的数据结构，常见的实现有数组的方式，操作数组索引进行出入栈；还有另外一种是链式栈实现，操作指针进行出入栈。本文将讨论的是基于链式栈实现无锁操作。

链式栈是一种单向链表的结构体，每个节点有一个 next 指针，指向当前栈的下一个栈节点。

最基本操作：栈的初始化、入栈、出栈。


### 实现

#### 结构体

栈的结构体需要有一个指针指向当前栈的栈顶，由于只需要原子操作一个栈顶指针，实现起来将会变得简单。

栈结构体和栈节点的结构体定义如下：

```c
// 链式栈节点结构体
typedef struct node_s  node_t;
struct node_s {
    node_t  *next;  // 下一个栈节点
    void    *value; // 存储的值
};

// 链式栈结构体
typedef struct stack_s  stack_t;
struct stack_s {
    node_t  *top;   // 永远指向栈顶的指针
};
```


#### 初始化

初始化不需要原子操作，直接将栈顶指针设置为空：

```c
void stack_init(stack_t *s)
{
    s->top = NULL;
}
```


#### push/压栈

压栈操作是将栈顶指针设置为新压入的栈节点。

```c
int stack_push(stack_t *s, void *x)
{
    node_t  *node, *p;

    node = (node_t *) malloc(sizeof(node_t));
    if (node == NULL) {
        return -ENOMEM;
    }

    node->value = x;

    for ( ;; ) {
        p = s->top;
        // 新栈节点的下一个栈是栈顶
        node->next = p;

        // 设置 top 指针为新节点
        if (__sync_bool_compare_and_swap(&(s->top), p, node)) {
            break;
        }
    }
}
```


#### pop/出栈

出栈操作是将栈顶指针设置为栈顶的下一个栈节点

```c
void *stack_pop(stack_t *s)
{
    node_t  *p;
    void    *v;

    for ( ;; ) {
        p = s->top;

        if (p == NULL) {
            return NULL;
        }

        // 设置栈顶指针为栈顶的下一个栈节点
        if (__sync_bool_compare_and_swap(&(s->top), p, p->next)) {
            break;
        }
    }

    v = p->value;

    free(p);

    return v;
}
```


### Rust 实现

完整代码链接放在文末 **参考** 字段

```
use std::sync::atomic::{AtomicPtr, Ordering};

unsafe impl<T: Send> Sync for Stack<T> {}

struct Node<T: Send> {
    next: AtomicPtr<Node<T>>,
    value: Option<T>,
}

impl<T: Send> Node<T> {
    fn new(x: T) -> Self {
        Self {
            next: AtomicPtr::default(),
            value: Some(x),
        }
    }

    fn sentry() -> Self {
        Self {
            next: AtomicPtr::default(),
            value: None,
        }
    }
}

pub struct Stack<T: Send> {
    top: AtomicPtr<Node<T>>,
}

impl<T: Send> Stack<T> {
    pub fn new() -> Self {
        let dummy = Box::into_raw(Box::new(Node::sentry()));

        Stack {
            top: AtomicPtr::new(dummy),
        }
    }

    pub fn push(&self, x: T) {
        unsafe { self.try_push(x) }
    }

    unsafe fn try_push(&self, x: T) {
        let node = Box::leak(Box::new(Node::new(x)));

        loop {
            let top_ptr = self.top.load(Ordering::Acquire);
            node.next.store(top_ptr, Ordering::Relaxed);

            if self
                .top
                .compare_exchange(top_ptr, node, Ordering::Release, Ordering::Relaxed)
                .is_ok()
            {
                break;
            }
        }
    }

    pub fn pop(&self) -> Option<T> {
        unsafe { self.try_pop() }
    }

    unsafe fn try_pop(&self) -> Option<T> {
        loop {
            let top_ptr = self.top.load(Ordering::Acquire);
            let next_ptr = (*top_ptr).next.load(Ordering::Acquire);

            if next_ptr.is_null() {
                return None;
            }

            if self
                .top
                .compare_exchange(top_ptr, next_ptr, Ordering::Release, Ordering::Relaxed)
                .is_ok()
            {
                let mut node = Box::from_raw(top_ptr);
                return node.value.take();
            }
        }
    }
}

impl<T: Send> Drop for Stack<T> {
    fn drop(&mut self) {
        let mut p = self.top.load(Ordering::Relaxed);

        while !p.is_null() {
            unsafe {
                let next = (*p).next.load(Ordering::Relaxed);
                Box::from_raw(p);
                p = next;
            }
        }
    }
}
```


### 性能测试

lib.rs Stack 与 标准库的 Mutex<LinkedList> 类型进行压测对比

笔记本电脑 CPU 参数如下:

```
machdep.cpu.brand_string: Intel(R) Core(TM) i5-4278U CPU @ 2.60GHz
machdep.cpu.core_count: 2
machdep.cpu.thread_count: 4
```


#### 压测描述

```
stack_loop_n(n)：一个 stack 对象，循环 n 次入栈出栈
stack_thread_n_m(n, m)：同一个 stack 对象， n 个线程入栈和出栈，循环 m 次数据
```


#### 结果对比

|    压测类型    | 总耗时 | 平均耗时 |
| ------------- | ----- | ------- |
| stack_loop_n(100000) | 56.523467ms | 565ns |
| list_loop_n(100000)  | 67.573497ms | 675ns |
| stack_thread_n_m(2, 100000) | 115.590207ms | 577ns |
| list_thread_n_m(2, 100000)  | 161.359683ms | 806ns |
| stack_thread_n_m(4, 100000) | 440.585874ms | 1.101µs |
| list_thread_n_m(4, 100000)  | 562.439723ms | 1.406µs |
| stack_thread_n_m(8, 100000) | 1.886768172s | 2.358µs |
| list_thread_n_m(8, 100000)  | 2.120945074s | 2.651µs |


### 参考

[Implementing Lock-Free Queues (1994)](http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.53.8674)

[https://gcc.gnu.org/onlinedocs/gcc/_005f_005fatomic-Builtins.html](https://gcc.gnu.org/onlinedocs/gcc/_005f_005fatomic-Builtins.html)

[https://github.com/cppcoffee/stack-rs](https://github.com/cppcoffee/stack-rs)

