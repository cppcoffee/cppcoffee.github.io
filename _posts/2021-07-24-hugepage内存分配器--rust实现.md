---
layout: post
title:  "Hugepage 内存分配器 -- Rust实现"
subtitle: ""
description: "Hugepage 内存分配器 -- Rust分配"
date: 2021-07-24
author: "Sharp Liu"
categories: system program
---

{{ page.title }}

### HugePage简介

Linux 默认内存页大小是 4KB（x86和x86_64），hugepage 的特性允许内核管理比默认内存页还要大的内存页（Huge Page）。

在 Linux 虚拟内存系统中维护一张 TLB（Translation Lookaside Buffer）的表，该表用于虚拟内存地址映射到物理内存地址。当系统需要访问一个虚拟内存位置时，需要进行 TLB 查找并进行地址转换。

启用 HugePages 后，系统使用更少的页表，减少了维护和访问页表的开销。Hugepages 保持在内存中，不被 swap，所以内核 swap 守护程序没有管理它们的工作，内核也不需要为它们执行页表查找。较少的页面数量减少了执行内存操作的开销，同时也减少了访问页表时出现瓶颈的可能性。

HugePage 在 x86 上是 4MB，x86_64 是 2MB。

关于 Hugepages 更详细的内容可以参考本文末尾的 References。


### HugePage API

Linux 提供 mmap(MAP_HUGETLB) 来分配 hugepages，如下调用分配 len 长度的 hugepages。flags 参数传递 MAP_HUGETLB：

```c
int flags = MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB;

void *p = mmap(null_ptr, len, PROT_READ | PROT_WRITE, flags, -1, 0);
```


### Allocator

接下来使用 rust 实现一个 hugepage 分配器

```rust
const MEMINFO_PATH: &str = "/proc/meminfo";
const TOKEN: &str = "Hugepagesize:";

lazy_static! {
    // 从 '/proc/meminfo' 中解析出 'Hugepagesize' 来初始化全局变量 HUGEPAGE_SIZE
    // HUGEPAGE_SIZE 用于 Allocator 分配内存时做对齐用。
    static ref HUGEPAGE_SIZE: isize = {
        let buf = File::open(MEMINFO_PATH).map_or("".to_owned(), |mut f| {
            let mut s = String::new();
            let _ = f.read_to_string(&mut s);
            s
        });
        parse_hugepage_size(&buf)
    };
}

// 解析 Hugepagesize
// meminfo 内容存在多行，需一行行找到 TOKEN='Hugepagesize:' 并对值进行解析
fn parse_hugepage_size(s: &str) -> isize {
    for line in s.lines() {
        // 找到 ‘Hugepagesize:’ 前缀
        if line.starts_with(TOKEN) {
            let mut parts = line[TOKEN.len()..].split_whitespace();

            // parse size
            let p = parts.next().unwrap_or("0");
            let mut hugepage_size = p.parse::<isize>().unwrap_or(-1);

            // parse unit
            hugepage_size *= parts.next().map_or(1, |x| match x {
                // 当前支持 kB 解析
                "kB" => 1024,
                _ => 1,
            });

            return hugepage_size;
        }
    }

    return -1;
}
```

定义 Allocator 结构体，采用空结构体类型（不需要内部数据，所以无任何结构字段）

```rust
pub(crate) struct HugePageAllocator;
```

使用 libc crate 提供的接口来调用 **libc::mmap**。那么接下来实现 std::alloc::GlobalAlloc trait：

```rust
// 实现 GlobalAlloc trait
unsafe impl GlobalAlloc for HugePageAllocator {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        // 分配的内存大小需对齐 HUGEPAGE_SIZE，调用辅助函数 align_to
        let len = align_to(layout.size(), *HUGEPAGE_SIZE as usize);
        let p = libc::mmap(
            null_mut(),
            len,
            PROT_READ | PROT_WRITE,
            MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB,
            -1,
            0,
        );

        // 无法分配 hugepage 则返回 null.
        if p == MAP_FAILED {
            return null_mut();
        }

        p as *mut u8
    }

    // 删除时候也需要 layout 参数.
    unsafe fn dealloc(&self, p: *mut u8, layout: Layout) {
        libc::munmap(p as *mut c_void, layout.size());
    }
}

// 辅助函数，用于对其字节
fn align_to(size: usize, align: usize) -> usize {
    (size + align - 1) & !(align - 1)
}
```

以上就完成了简单的 Hugepage 分配器。


### Boxed

实现 Allocator 后，导出一个全局的 Allocator 给 Box 使用。

```rust
// lib.rs 定义一个全局的 default_allocator() 接口，给整个 crate 使用。
lazy_static! {
    static ref HUGEPAGE_ALLOCATOR: HugePageAllocator = HugePageAllocator;
}

// 只暴露给自身 crate 调用
pub(crate) fn default_allocator() -> &'static HugePageAllocator {
    &HUGEPAGE_ALLOCATOR
}
```

实现一个简单的 Box，支持 deref 操作，过了 Box scope 后，自动释放，具体实现如下：

```rust
pub struct Box<T> {
    data: NonNull<T>,
}

impl<T> Box<T> {
    pub fn new(data: T) -> Box<T> {
        let layout = Layout::new::<T>();
        unsafe {
            let mut p = NonNull::new(default_allocator().alloc(layout) as *mut T).unwrap();
            *(p.as_mut()) = data;
            Self { data: p }
        }
    }

    pub unsafe fn from_raw(raw: *mut T) -> Self {
        Self {
            data: NonNull::new(raw).unwrap(),
        }
    }
}

impl<T> Drop for Box<T> {
    fn drop(&mut self) {
        unsafe {
            default_allocator().dealloc(self.data.as_ptr() as *mut u8, Layout::new::<T>());
        }
    }
}

impl<T> Deref for Box<T> {
    type Target = T;
    fn deref(&self) -> &T {
        unsafe { self.data.as_ref() }
    }
}

impl<T> DerefMut for Box<T> {
    fn deref_mut(&mut self) -> &mut T {
        unsafe { self.data.as_mut() }
    }
}
```

更详细的代码在文末给的 github 仓库中。


### Reference

[Huge pages part 1 (Introduction)](https://lwn.net/Articles/374424/)

[Huge pages part 2: Interfaces](https://lwn.net/Articles/375096/)

[Huge pages part 3: Administration](https://lwn.net/Articles/376606/)

[Huge pages part 4: benchmarking with huge pages](https://lwn.net/Articles/378641/)

[Huge pages part 5: A deeper look at TLBs and costs](https://lwn.net/Articles/379748/)

[https://www.kernel.org/doc/Documentation/vm/hugetlbpage.txt](https://www.kernel.org/doc/Documentation/vm/hugetlbpage.txt)

[https://man7.org/linux/man-pages/man2/mmap.2.html](https://man7.org/linux/man-pages/man2/mmap.2.html)

[https://github.com/cppcoffee/hugepage-rs](https://github.com/cppcoffee/hugepage-rs)
