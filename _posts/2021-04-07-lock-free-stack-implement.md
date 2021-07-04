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

这里采用 Rust 实现，**crossbeam-epoch** crate 来解决无锁结构体的 ABA 问题和内存回收问题。


#### 结构体

栈的结构体需要有一个指针指向当前栈的栈顶，由于只需要原子操作一个栈顶指针，实现起来将会变得简单。

栈结构体和栈节点的结构体定义如下：

```c
// 链式栈节点结构体
struct Node<T: Send> {
    next: Atomic<Node<T>>,  // 下一个节点
    value: Option<T>,       // 存储的值
}

// 栈对象结构体
pub struct Stack<T: Send> {
    top: Atomic<Node<T>>,
}
```


#### 初始化

初始化不需要原子操作，这里提供两个方法：

```c
impl<T: Send> Node<T> {
    // 普通节点
    fn new(v: T) -> Self {
        Self {
            next: Atomic::null(),
            value: Some(v),
        }
    }

    // 哨兵节点
    fn sentinel() -> Self {
        Self {
            next: Atomic::null(),
            value: None,
        }
    }
}
```


#### push/压栈

压栈操作是将栈顶指针设置为新压入的栈节点。

```c
pub fn push(&self, v: T) {
    unsafe { self.try_push(v) }
}

unsafe fn try_push(&self, v: T) {
    let guard = &epoch::pin();
    let node = Owned::new(Node::new(v)).into_shared(guard);

    loop {
        let top_ptr = self.top.load(Acquire, guard);
        // 新节点的下一个节点指向栈顶
        (*node.as_raw()).next.store(top_ptr, Relaxed);

        // 设置 top 为新节点
        if self
            .top
            .compare_exchange(top_ptr, node, Release, Relaxed, guard)
            .is_ok()
        {
            break;
        }
    }
}
```


#### pop/出栈

出栈操作是将栈顶指针设置为栈顶的下一个栈节点

```c
pub fn pop(&self) -> Option<T> {
    unsafe { self.try_pop() }
}

unsafe fn try_pop(&self) -> Option<T> {
    let guard = &epoch::pin();

    loop {
        let top_ptr = self.top.load(Acquire, guard);
        let next_ptr = (*top_ptr.as_raw()).next.load(Acquire, guard);

        if next_ptr.is_null() {
            return None;
        }

        // 设置栈顶指针为栈顶的下一个栈节点
        if self
            .top
            .compare_exchange(top_ptr, next_ptr, Release, Relaxed, guard)
            .is_ok()
        {
            let top_ptr = top_ptr.as_raw() as *mut Node<T>;
            return (*top_ptr).value.take();
        }
    }
}
```

完整代码链接放在文末 **参考** 字段


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

[https://lib.rs/crates/crossbeam-epoch](https://lib.rs/crates/crossbeam-epoch)

[https://github.com/cppcoffee/stack-rs](https://github.com/cppcoffee/stack-rs)

