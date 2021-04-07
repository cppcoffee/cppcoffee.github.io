---
layout: post
title:  "Lock-Free Queues Implement"
subtitle: ""
description: "Lock-Free Queues Implement"
date: 2021-03-25
author: "Sharp Liu"
categories: datastructure
---

{{ page.title }}


## 无锁队列

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
    node_t  *node, *p;

    node = (node_t *) malloc(sizeof(node_t));
    if (node == NULL) {
        return -ENOMEM;
    }

    node->value = x;
    node->next = NULL;

    for ( ;; ) {
        p = q->tail;

        // tail->next == NULL, 加入新 node.
        if (__sync_bool_compare_and_swap(&(p->next), NULL, node)) {
            break;

        } else {
            // tail 指针被修改了，步进 tail.
            __sync_bool_compare_and_swap(&(q->tail), p, p->next);
        }
    }

    // 操作完毕，设置 tail 指针.
    __sync_bool_compare_and_swap(&(q->tail), p, node);

    return 0;
}
```


### 出队列

出队列的实现会比较简单：

```c
void *dequeue(queue_t *q)
{
    node_t  *p;
    void    *v;

    for ( ;; ) {
        p = q->head;

        // 队列空.
        if (p->next == NULL) {
            return NULL;
        }

        // 设置 head 指向下一个 node
        if (__sync_bool_compare_and_swap(&(q->head), p, p->next)) {
            break;
        }
    }

    v = p->next->value;

    free(p);

    return v;
}
```


### Rust 实现

最后附上一版使用 Rust 实现无锁队列的完整代码

lib.rs

```rust
use std::ptr;
use std::sync::atomic::{AtomicPtr, Ordering};

unsafe impl<T: Send> Sync for Queue<T> {}

struct Node<T: Send> {
    next: AtomicPtr<Node<T>>,
    value: Option<T>,
}

impl<T: Send> Node<T> {
    fn new(v: T) -> Self {
        Self {
            next: Default::default(),
            value: Some(v),
        }
    }

    fn sentinel() -> Self {
        Self {
            next: Default::default(),
            value: None,
        }
    }
}

pub struct Queue<T: Send> {
    head: AtomicPtr<Node<T>>,
    tail: AtomicPtr<Node<T>>,
}

impl<T: Send> Drop for Queue<T> {
    fn drop(&mut self) {
        let mut p = self.head.load(Ordering::Relaxed);

        while !p.is_null() {
            unsafe {
                let next = (*p).next.load(Ordering::Relaxed);
                Box::from_raw(p);
                p = next;
            }
        }
    }
}

impl<T: Send> Queue<T> {
    pub fn new() -> Self {
        let dummy_ptr = Box::into_raw(Box::new(Node::sentinel()));

        Self {
            head: AtomicPtr::new(dummy_ptr),
            tail: AtomicPtr::new(dummy_ptr),
        }
    }

    pub fn enq(&self, v: T) {
        unsafe { self.try_enq(v) }
    }

    unsafe fn try_enq(&self, v: T) {
        let node = Box::into_raw(Box::new(Node::new(v)));

        loop {
            let p = self.tail.load(Ordering::Acquire);

            if (*p)
                .next
                .compare_exchange(ptr::null_mut(), node, Ordering::Release, Ordering::Relaxed)
                .is_ok()
            {
                let _ = self
                    .tail
                    .compare_exchange(p, node, Ordering::Acquire, Ordering::Relaxed);
                return;
            } else {
                let _ = self.tail.compare_exchange(
                    p,
                    (*p).next.load(Ordering::Acquire),
                    Ordering::Release,
                    Ordering::Relaxed,
                );
            }
        }
    }

    pub fn deq(&self) -> Option<T> {
        unsafe { self.try_deq() }
    }

    unsafe fn try_deq(&self) -> Option<T> {
        let mut p;

        loop {
            p = self.head.load(Ordering::Acquire);

            if (*p).next.load(Ordering::Acquire).is_null() {
                return None;
            }

            if self
                .head
                .compare_exchange(
                    p,
                    (*p).next.load(Ordering::Acquire),
                    Ordering::Release,
                    Ordering::Relaxed,
                )
                .is_ok()
            {
                break;
            }
        }

        (*(*p).next.load(Ordering::Acquire)).value.take()
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

[https://gcc.gnu.org/onlinedocs/gcc/_005f_005fatomic-Builtins.html](https://gcc.gnu.org/onlinedocs/gcc/_005f_005fatomic-Builtins.html)

[https://github.com/cppcoffee/queue-rs](https://github.com/cppcoffee/queue-rs)

