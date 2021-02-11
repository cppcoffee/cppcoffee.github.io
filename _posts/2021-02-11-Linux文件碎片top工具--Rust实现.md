---
layout: post
title:  "Linux 文件碎片 top 工具 -- Rust实现"
subtitle: ""
description: "Linux 文件碎片 top 工具 -- Rust实现"
date: 2021-02-11
author: "Sharp Liu"
categories: [filesystem, rust]
---
{{ page.title }}

## fragtop-rs

上一篇提到 Linux 下的 `filefrag` 工具的实现方式，可以用它来查看文件碎片，它没有提供一个扫描目录进行 top 碎片数量排序的功能，既然这样，那就动手做一个玩。

工具项目名为 `fragtop-rs`，寓意是跟 top 工具一样。`fragtop-rs` 能够根据 glob 匹配的文件进行碎片统计，并进行 top 排序输出。

`fragtop-rs` 采用 Rust 实现，Rust 有一个 `fiemap` 的 crate 可以使用 [https://docs.rs/fiemap/](https://docs.rs/fiemap/)，接口干净整洁，可以拿过来使用。

功能包括：需要指定 glob pattern，遍历匹配的文件进行碎片查询，最后给出 top-n 的列表。


### Cargo.toml

首先创建项目，并进入项目目录中

```rust
% cargo new --bin fragtop-rs
     Created binary (application) `fragtop-rs` package

% cd fragtop-rs
```

需要的 crate 列表如下：

- `clap`： 用于命令行操作
- `glob`： 匹配文件路径
- `anyhow`： 错误处理
- `fiemap`： Linux 文件碎片查找

逐个添加依赖的 crate

```rust
% cargo add clap
    Updating 'https://github.com/rust-lang/crates.io-index' index
      Adding clap v2.33.3 to dependencies

% cargo add glob
% cargo add anyhow
% cargo add fiemap
```

添加完 crate 后，`Cargo.toml` 的 `dependencies` 字段如下所示：

```toml
[dependencies]
clap = "2.33"
glob = "0.3"
anyhow = "1.0"
fiemap = "0.1"
```

### 逻辑实现

增加 clap 命令行处理，该工具需要 `-p` 参数来指定 glob pattern 路径和 `-n` 来指定 top-n 数量。

其中 `-p` 是要求必须指定；`-n` 默认值为 20，如果文件过多，就只显示 top 20 的文件

```rust
#[macro_use]
extern crate clap;

use clap::Arg;

fn main() -> anyhow::Result<()> {
    let matches = clap::app_from_crate!()
        .arg(
            Arg::with_name("path")
                .short("p")
                .help("Set the glob file path")
                .required(true)
                .takes_value(true),
        )
        .arg(
            Arg::with_name("top-n")
                .short("n")
                .help("Top fragment file")
                .default_value("20")
                .takes_value(true),
        )
        .get_matches();

    let path = matches.value_of("path").unwrap();
    let top_n = matches.value_of("top-n").unwrap().parse::<usize>()?;
    
    println!("path: {}, top: {}", path, top_n);
    
    Ok(())
}
```

运行输出：

```
% cargo run -- -p /tmp/
path: /tmp/, top: 20
```

输出结果正常，接下来要添加遍历匹配 glob pattern 的文件，并记录对应文件的碎片数量。

使用 `BTreeSet<Tuple(fragments, path)>` 来记录文件和它对应的碎片数量，BTreeSet 的好处是可以按照从小到大进行遍历，如果 `rev()` 则可以从大到小进行遍历。


```rust
...
use std::collections::BTreeSet;

fn main() -> anyhow::Result<()> {
    ...

    let mut records = BTreeSet::new();

    for entry in glob::glob(path)? {
        let entry = entry?;

        // 输出正在处理的文件
        // \r 开头则使用同一行进行替换输出
        print!("\rIn progress: {}", entry.display());

        // 获取文件碎片，并保存文件碎片数和文件路径
        let count = fiemap::fiemap(&entry)?.count();
        records.insert((count, entry));
    }

    Ok(())
}
```

已经有了文件路径和它对应的碎片数量，最后就是对这些信息的总结输出（遍历）。

```rust
fn main() -> anyhow::Result<()> {
...

    if records.len() == 0 {
        return Err(anyhow!("no files are scanned."));
    }

    println!("\nScan total file: {}", records.len());

    // rev() 倒序（从大到小），只取 take(top_n) 项
    for (count, entry) in records.iter().rev().take(top_n) {
        println!("{:<48}  {}", entry.display(), count);
    }

    Ok(())
}
```

以上，`fragtop-rs` 的代码完成了。

用它来查看 `/var/log/` 目录下面的所有日志文件，并根据碎片大小输出

```shell
$ ./target/debug/fragtop-rs -p '/var/log/**/*'
In progress: /var/log/yum.log-20210101
Scan total file: 657
/var/log/access.log                               266
/var/log/wtmp                                     39
/var/log/messages-20210131                        22
/var/log/messages-20210207                        21
/var/log/messages-20210117                        20
/var/log/audit/audit.log.2                        20
/var/log/audit/audit.log.4                        19
/var/log/messages-20210124                        18
/var/log/audit/audit.log.1                        18
/var/log/nginx/access.log                         17
/var/log/audit/audit.log.3                        17
/var/log/cron-20210207                            14
/var/log/cron-20210131                            14
/var/log/cron-20210124                            14
/var/log/cron-20210117                            14
/var/log/messages                                 13
/var/log/audit/audit.log                          13
/var/log/yum.log-20200511                         10
/var/log/tuned/tuned.log                          10
/var/log/grubby                                   8
```

### 参考

[https://docs.rs/fiemap/0.1.1/fiemap/](https://docs.rs/fiemap/0.1.1/fiemap/)

[https://github.com/cppcoffee/fragtop-rs](https://github.com/cppcoffee/fragtop-rs)

