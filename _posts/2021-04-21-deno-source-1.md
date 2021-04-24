---
layout: post
title: "Deno源码解读之deno_core（一）"
date: 2021-04-21 22:12:50 +0800
tags: [rust, deno, javascript, v8]
---

## Deno源码解读之deno_core（一）


首先，下载源代码

```shell
git clone git@github.com:denoland/deno.git
```

然后，进入 deno 目录，看下主要的目录结构

```shell
.
├── core
├── runtime
├── op_crates
├── cli
├── serde_v8
├── test_plugin
├── test_util
├── third_party
├── tools
├── Cargo.lock
└── Cargo.toml
```

主要目录共4个，cli/core/runtime/op_crates，最底层的是 core 目录，我们可以先集中研究该目录下的代码。core 目录包含 Cargo.toml 文件，是一个名为 deno_core 的 Crate。

```toml
[package]
name = "deno_core"
version = "0.83.0"
edition = "2018"
description = "A secure JavaScript/TypeScript runtime built with V8, Rust, and Tokio"
authors = ["the Deno authors"]
license = "MIT"
readme = "README.md"
repository = "https://github.com/denoland/deno"
```

因此，我们也可以只依赖 deno_core 来实现跟 V8 交互的需求。因为 core 本身是一个 Crate，我们进入目录后，可以使用执行一些 cargo 命令。先试试 cargo build，结果会出现下面的错误。

```shell
error: failed to run custom build command for `rusty_v8 v0.22.0`
```

因为 deno 需要依赖 V8，但是 deno 项目重构后并不直接依赖 V8，而是通过 rusty_v8 项目来集成 V8。V8 的代码量非常庞大，每次编译要很久，rusty_v8 采用预先编译的方式，每次版本发布都会提供相应的[静态 V8 libs](https://github.com/denoland/rusty_v8/releases)下载，这样用户就不需要自己编译 V8 了。

在 rusty_v8 的 github 页面提供了下载脚本，按说明操作就行。

```shell
# you might want to add this to your .bashrc
$ export RUSTY_V8_MIRROR=$HOME/.cache/rusty_v8
```

保存下面的脚本到文件，版本升级时改下版本号，下载新的文件。

```bash
#!/bin/bash

# see https://github.com/denoland/rusty_v8/releases

# rusty_v8的版本号
for REL in v0.21.0 v0.22.0; do
  mkdir -p $RUSTY_V8_MIRROR/$REL
  for FILE in \
    librusty_v8_debug_x86_64-unknown-linux-gnu.a \
    librusty_v8_release_x86_64-unknown-linux-gnu.a \
  ; do
    if [ ! -f $RUSTY_V8_MIRROR/$REL/$FILE ]; then
      wget -O $RUSTY_V8_MIRROR/$REL/$FILE \
        https://github.com/denoland/rusty_v8/releases/download/$REL/$FILE
    fi
  done
done
```

设置好环境变量并下载好文件后，我们的 deno_core 就可以依赖 rusty_v8 跟 V8 交互了。执行 cargo test 跑下所有测试。

```shell
   Compiling deno_core v0.83.0 (/Users/lin/work/src/opensource/deno/core)
    Finished test [unoptimized + debuginfo] target(s) in 28.36s
     Running /Users/lin/work/src/opensource/deno/target/debug/deps/deno_core-a626e63b567ed9fe

running 58 tests
test async_cancel::tests::cancel_handle_pinning ... ok
test runtime::tests::test_poll_async_delayed_ops ... ignored
test runtime::tests::test_from_boxed_snapshot ... ok
test runtime::tests::will_snapshot ... ok
......

test result: ok. 56 passed; 0 failed; 2 ignored; 0 measured; 0 filtered out

   Doc-tests deno_core

running 4 tests
test ops_json.rs - ops_json::op_async (line 64) ... ignored
test ops_json.rs - ops_json::op_sync (line 26) ... ignored
test resources.rs - resources::ResourceTable::names (line 153) ... ok
test async_cell.rs - async_cell::RcRef (line 125) ... ok

test result: ok. 2 passed; 0 failed; 2 ignored; 0 measured; 0 filtered out
```
 
再跑下 hello world 例子，

```shell
$ cargo run --example hello_world
    Finished dev [unoptimized + debuginfo] target(s) in 0.42s
     Running `/Users/lin/work/src/opensource/deno/target/debug/examples/hello_world`
The sum of
1,2,3
is
6
Exception:
Error: Error parsing args: serde_v8 error: ExpectedArray
```

代码在core/examples/hello_world.rs，

```rust
let mut runtime = JsRuntime::new(Default::default());

runtime.register_op("op_print", ...);

runtime.register_op("op_sum", ...);

runtime.execute("<init>", r#" ...javascript code... "#).unwrap();
runtime.execute("<usage>", r#" ...javascript code... "#).unwrap();
```

deno_core 为我们封装了 JsRuntime 类型，通过 JsRuntime 创建一个 runtime，再注册两个操作

1. op_print: 在 JavaScript 端调用 Deno.core.dispatchByName('op_print', ...)，将在 Rust 端调用回调函数，把数据写入 stdout。
2. op_sum:  在 JavaScript 端调用 Deno.core.opSync('op_sum', ...)，将在 Rust 端调用回调函数，对数组求和，返回结果给 JavaScript。

从这个例子大致可以看出，deno_core 为我们提供了比较方便的方式让 V8 执行 JavaScript。下一步我们将看看 JsRuntime 的实现代码，以此了解更底层的逻辑。