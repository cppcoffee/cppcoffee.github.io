---
layout: post
title:  "Lock-Free Queues Implement"
subtitle: ""
description: "Lock-Free Queues Implement"
date: 2021-03-25
author: "Sharp Liu"
categories: datastructure
---

## {{ page.title }}

队列是一种FIFO的抽象数据结构，这里提到的无锁队列实现是 `Implementing Lock-Free Queues(1994)` 这篇论文提出来的。

无锁队列操作依靠 CPU 的 CAS (Compare And Swap) 指令，CAS 对应的 Intel CPU 指令是 `lock cmpxchg`，前缀 `lock` 表明这是一条原子操作指令。

现在许多新语言都有自带 CAS 相关函数；底层基础库也有提供内建函数，例如 GCC 提供的内建 CAS 函数：

```
bool __sync_bool_compare_and_swap (type *ptr, type oldval, type newval, ...)
type __sync_val_compare_and_swap (type *ptr, type oldval, type newval, ...)
```


### 队列结构体

实现无锁队列需要有两个指针：一个 head 指针，指向队列头部；一个 tail 指针，指向队列尾部。

节点结构体有一个 next 指针，指向下一个节点，形成链式队列。

```c
// 队列节点结构体
typedef struct node_s  node_t;
struct node_s {
    node_t  *next;
    void    *value;
};

// 队列结构体
typedef struct queue_s  queue_t;
struct queue_s {
    node_t  *head;
    node_t  *tail;
};
```


### 初始化

论文提到初始化的时候生成一个 dummy 节点作为 head 和 tail 的初始值。

dummy 节点为了防止在空队列或只有一个节点的时候出现边界问题。

初始化的实现就如下：

```c
int queue_init(queue_t *q)
{
    node_t *dummy = (node_t *) malloc(sizeof(node_t));

    if (dummy == NULL) {
        return -ENOMEM;
    }

    memset(dummy, 0, sizeof(node_t));

    q->head = q->tail = dummy;

    return 0;
}
```

函数中需要判断内存分配错误。


### 入队列

根据论文的伪代码实现如下：

```c
int enqueue(queue_t *q, void *x)
{
    node_t  *node, *tail, *next;

    node = (node_t *) malloc(sizeof(node_t));
    if (node == NULL) {
        return -ENOMEM;
    }

    node->value = x;
    node->next = NULL;

    for ( ;; ) {
        tail = q->tail;
        next = tail->next;

        if (tail != q->tail) {
            continue;
        }

        if (next == NULL) {
            if (__sync_bool_compare_and_swap(&(tail->next), next, node)) {
                __sync_bool_compare_and_swap(&(q->tail), tail, node);
                return 0;
            }

        } else {
            __sync_bool_compare_and_swap(&(q->tail), tail, next);
        }
    }
}
```


### 出队列

出队列的实现会比较简单：

```c
void *dequeue(queue_t *q)
{
    void    *v;
    node_t  *head, *tail, *next;

    for ( ;; ) {
        head = q->head;
        tail = q->tail;
        next = head->next;

        if (head != q->head) {
            continue;
        }

        if (head == tail) {
            if (next == NULL) {
                return NULL;
            }

            __sync_bool_compare_and_swap(&(q->tail), tail, next);

        } else {
            if (next == NULL) {
                continue;
            }

            v = next->value;

            if (__sync_bool_compare_and_swap(&(q->head), head, next)) {
                // FIXME: 释放会引发并发结构经典的 ABA 和内存回收问题
                //free(head);
                return v;
            }
        }
    }
}
```


### ABA 问题

在多线程中，ABA 问题发生在同步期间，当一个位置被读取两次，两次读取的值都是一样的，“值是一样的”被用来表示“没有变化”。然而，另一个线程可以在两次读取之间执行，并改变值，做其他工作，然后把值改回来，从而欺骗第一个线程，使其认为“没有变化”，即使第二个线程所做的工作违反了这个假设：

```
T1 从共享内存中读取 A=Load(A) 后被暂停
T2 被调度执行
T2 修改共享内存 CAS(A, B) 将 A 修改成 B，并在被系统调度前 CAS(B, A） B 再被修改成 A
T1 再次被调度执行，从而看到 A 并没有被改变过
```

这需要保证内存不能立即释放（还有线程饮用它），也不能立即被重用，这就是无锁结构 CAS 最常见的坑，实际项目中，通常配合 128 位 CAS 来避免 ABA 问题，而支持 128 位 CAS 的硬件并不通用，所以需要做指针压缩(TaggedPointer)


#### Tagged Pointer

在 x86_64 机器上，指针高位地址用于在内核层表示，在应用层空间中就能够使用高位地址来作为 tag。

如下是 64 位长度的地址：

```
0000 0000 0000 0000
```

根据 linux mm 文档中描述，应用程序虚拟内存范围是 0000000000000000 - 00007fffffffffff

也就是说高 16 位是可以用来作为 tag.

```
0000 FFFF FFFF FFFF
^^^^
Free Data!
```


### 内存回收问题

在多线程操作中，内存不能直接释放，由于有其他线程在访问它，这样会造成 **释放后访问** 的问题：

> T1 执行到 next = tail->next; 时被调度走
> T2 执行 dequeue，将 tail 指向的内存释放
> T1 再次被调度到，此时访问 tail->next 将造成 内存释放后再访问的问题

这种情况需要保证内存访问的安全性，可以使用 引用计数、hazard pointers 和 epoch based reclamation 等内存延迟回收算法。


### Rust 实现

最后附上一版使用 Rust 实现无锁队列的完整代码，这里使用 **crossbeam_epoch** crate 来解决 ABA 问题和内存回收问题。

lib.rs

```rust
use std::sync::atomic::Ordering::{Acquire, Relaxed, Release};

use crossbeam_epoch::{self as epoch, Atomic, Owned, Shared};

unsafe impl<T: Send> Sync for Queue<T> {}

struct Node<T: Send> {
    next: Atomic<Node<T>>,
    data: Option<T>,
}

impl<T: Send> Node<T> {
    fn new(v: T) -> Self {
        Self {
            next: Default::default(),
            data: Some(v),
        }
    }

    fn sentinel() -> Self {
        Self {
            next: Atomic::null(),
            data: None,
        }
    }
}

pub struct Queue<T: Send> {
    head: Atomic<Node<T>>,
    tail: Atomic<Node<T>>,
}

impl<T: Send> Queue<T> {
    pub fn new() -> Self {
        let q = Queue {
            head: Atomic::null(),
            tail: Atomic::null(),
        };
        let sentinel = Owned::new(Node::sentinel());

        let guard = unsafe { &epoch::unprotected() };

        let sentinel = sentinel.into_shared(guard);
        q.head.store(sentinel, Relaxed);
        q.tail.store(sentinel, Relaxed);
        q
    }

    pub fn enq(&self, v: T) {
        unsafe { self.try_enq(v) }
    }

    unsafe fn try_enq(&self, v: T) {
        let guard = &epoch::pin();
        let node = Owned::new(Node::new(v)).into_shared(guard);

        loop {
            let p = self.tail.load(Acquire, guard);

            if (*p.as_raw())
                .next
                .compare_exchange(Shared::null(), node, Release, Relaxed, guard)
                .is_ok()
            {
                let _ = self.tail.compare_exchange(p, node, Release, Relaxed, guard);
                return;
            } else {
                let _ = self.tail.compare_exchange(
                    p,
                    (*p.as_raw()).next.load(Acquire, guard),
                    Release,
                    Relaxed,
                    guard,
                );
            }
        }
    }

    pub fn deq(&self) -> Option<T> {
        unsafe { self.try_deq() }
    }

    unsafe fn try_deq(&self) -> Option<T> {
        let guard = &epoch::pin();

        loop {
            let p = self.head.load(Acquire, guard);

            if (*p.as_raw()).next.load(Acquire, guard).is_null() {
                return None;
            }

            if self
                .head
                .compare_exchange(
                    p,
                    (*p.as_raw()).next.load(Acquire, guard),
                    Release,
                    Relaxed,
                    guard,
                )
                .is_ok()
            {
                let next = (*p.as_raw()).next.load(Acquire, guard).as_raw() as *mut Node<T>;
                return (*next).data.take();
            }
        }
    }
}
```

### benchmark

lib.rs Queue 与 标准库的 Mutex\<LinkedList\> 类型进行压测对比

#### 压测代码

Queue 压测代码 与 Mutex\<LinkedList\> 实现大同小异，不同的只是 enq 操作对应 push_front，deq 操作对应 pop_back。

这里贴两个 Queue 的压测相关代码，更多详细内容见文末的 queue-rs 仓库链接。

```rust
// n 次压测操作
fn queue_loop_n(n: u32) -> Duration {
    let q = Queue::new();
    let earler = Instant::now();

    for i in 0..n {
        q.enq(i as *mut u8);
    }
    for _ in 0..n {
        q.deq();
    }

    Instant::now().duration_since(earler)
}

// n 线程 + m 次操作
fn queue_thread_n_m(n: u32, m: u32) -> Duration {
    let mut handles = Vec::new();
    let elapsed = Arc::new(AtomicU64::default());

    for _ in 0..n {
        let q = Queue::new();
        let elapsed_clone = elapsed.clone();

        handles.push(thread::spawn(move || {
            let start = Instant::now();

            for i in 0..m {
                q.enq(i as *mut u8);
            }
            for _ in 0..m {
                q.deq();
            }

            let nanos = Instant::now().duration_since(start).as_nanos();
            elapsed_clone.fetch_add(nanos as u64, Ordering::SeqCst);
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }

    Duration::from_nanos(Arc::try_unwrap(elapsed).unwrap().into_inner())
}
```



#### 结果对比

笔记本电脑 CPU 参数如下:

```
machdep.cpu.brand_string: Intel(R) Core(TM) i5-4278U CPU @ 2.60GHz
machdep.cpu.core_count: 2
machdep.cpu.thread_count: 4
```

**备注**: 领先的数据加黑标注

输出结果:

|    压测类型    | 总耗时 | 平均耗时 |
| ------------- | ----- | ------- |
| queue_loop_n(100000) | **17.843828ms** | **178ns** |
| list_loop_n(100000)  | 23.066353ms | 230ns |
| queue_thread_n_m(2, 100000) | **64.018836ms** | **320ns** |
| list_thread_n_m(2, 100000)  | 74.660454ms | 373ns |
| queue_thread_n_m(4, 100000) | **149.736868ms** | **374ns** |
| list_thread_n_m(4, 100000)  | 189.6352ms | 474ns |
| queue_thread_n_m(8, 100000) | **544.476377ms** | **680ns** |
| list_thread_n_m(8, 100000)  | 980.688619ms | 1225ns |


### 参考

[Implementing Lock-Free Queues (1994)](http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.53.8674)

[https://en.wikipedia.org/wiki/ABA_problem](https://en.wikipedia.org/wiki/ABA_problem)

[https://gcc.gnu.org/onlinedocs/gcc/_005f_005fatomic-Builtins.html](https://gcc.gnu.org/onlinedocs/gcc/_005f_005fatomic-Builtins.html)

[Keir Fraser’s epoch-based reclamation](https://www.cl.cam.ac.uk/techreports/UCAM-CL-TR-579.pdf)

[crossbeam-epoch crate](https://lib.rs/crates/crossbeam-epoch)

[https://en.wikipedia.org/wiki/Tagged_pointer](https://en.wikipedia.org/wiki/Tagged_pointer)

[https://www.kernel.org/doc/Documentation/x86/x86_64/mm.txt](https://www.kernel.org/doc/Documentation/x86/x86_64/mm.txt)

[https://github.com/cppcoffee/queue-rs](https://github.com/cppcoffee/queue-rs)

