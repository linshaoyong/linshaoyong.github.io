---
layout: post
title: "Deno源码解读之deno_core（二）"
date: 2021-04-22 22:12:50 +0800
tags: [rust, deno, javascript, v8]
---

## Deno源码解读之deno_core（二）

从 hello word 的例子中可以看到，deno_core 为我们封装了 JsRuntime ，理解 JsRuntime 背后原理，就基本上掌握了 deno_core。

### core/runtime.rs

```rust
/// A single execution context of JavaScript. Corresponds roughly to the "Web
/// Worker" concept in the DOM. A JsRuntime is a Future that can be used with
/// an event loop (Tokio, async_std).
pub struct JsRuntime {
  // This is an Option<OwnedIsolate> instead of just OwnedIsolate to workaround
  // an safety issue with SnapshotCreator. See JsRuntime::drop.
  v8_isolate: Option<v8::OwnedIsolate>,
  snapshot_creator: Option<v8::SnapshotCreator>,
  has_snapshotted: bool,
  allocations: IsolateAllocations,
}

/// Objects that need to live as long as the isolate
#[derive(Default)]
struct IsolateAllocations {
  near_heap_limit_callback_data:
    Option<(Box<RefCell<dyn Any>>, v8::NearHeapLimitCallback)>,
}
```

JsRuntime 主要包含两个字段，v8::OwnedIsolate 和 IsolateAllocations。v8::OwnedIsolate 来自rusty_v8，相当于 V8 isolate 实例，IsolateAllocations 保存生命周期不短于 isolate 的各种对象。

```rust
impl JsRuntime {

  pub fn new(mut options: RuntimeOptions) -> Self {
    ......
    js_runtime
  }
}
```

JsRuntime 的 new 函数定义了对象的创建过程，大体流程如下

1. 初始化 v8::Platform，一个进程即使创建多个JsRuntime对象，也只会初始化一次v8::Platform
2. 读取快照（snapshot）或新创建快照，创建 isolate 对象
3. 初始化 isolate context
4. 设置 isoalte 关联的状态（状态封装在 JsRuntimeState 对象中）
5. 创建 JsRuntime 对象
6. 按需执行目录下的 core.js/error.js/init.js 这3个JavaScript文件

其中第三步初始化 isolate context 使用快照或调用bindings.rs#initialize_context 在 isolate 中创建了所需的 JavaScript 对象和函数。

### core/bindings.rs

初始化 context，定义全局对象 Deno = { core: {} }，再为 Deno.core 绑定各种属性 print/recv/send 等等。

```rust
pub fn initialize_context<'s>(
  scope: &mut v8::HandleScope<'s, ()>,
) -> v8::Local<'s, v8::Context> {
  let scope = &mut v8::EscapableHandleScope::new(scope);
  let context = v8::Context::new(scope);
  let global = context.global(scope);
  let scope = &mut v8::ContextScope::new(scope, context);

  // global.Deno = { core: {} };
  let deno_key = v8::String::new(scope, "Deno").unwrap();
  let deno_val = v8::Object::new(scope);
  global.set(scope, deno_key.into(), deno_val.into());
  let core_key = v8::String::new(scope, "core").unwrap();
  let core_val = v8::Object::new(scope);
  deno_val.set(scope, core_key.into(), core_val.into());

  // Bind functions to Deno.core.*
  set_func(scope, core_val, "print", print);
  set_func(scope, core_val, "recv", recv);
  set_func(scope, core_val, "send", send);
  ......

  scope.escape(context)
}
```

当V8执行 JavaScript 时，如果遇到不认识的函数，会尝试查找外部引用。所以在 Rust 端定义的 print/recv/send等函数需要作为外部引用注册给 isolate

```rust
lazy_static::lazy_static! {
  pub static ref EXTERNAL_REFERENCES: v8::ExternalReferences =
    v8::ExternalReferences::new(&[
      v8::ExternalReference {
        function: print.map_fn_to()
      },
      v8::ExternalReference {
        function: recv.map_fn_to()
      },
      v8::ExternalReference {
        function: send.map_fn_to()
      },
			......
    ]);
}
```

这样当 V8 执行 Deno.core.print 时，就会转移到 Rust 端执行 print 函数

```rust
fn print(
  scope: &mut v8::HandleScope,
  args: v8::FunctionCallbackArguments,
  _rv: v8::ReturnValue,
) {
  let arg_len = args.length();
  if !(0..=2).contains(&arg_len) {
    return throw_type_error(scope, "Expected a maximum of 2 arguments.");
  }

  let obj = args.get(0);
  let is_err_arg = args.get(1);

  let mut is_err = false;
  if arg_len == 2 {
    let int_val = match is_err_arg.integer_value(scope) {
      Some(v) => v,
      None => return throw_type_error(scope, "Invalid arugment. Argument 2 should indicate wheter or not to print to stderr."),
    };
    is_err = int_val != 0;
  };
  let tc_scope = &mut v8::TryCatch::new(scope);
  let str_ = match obj.to_string(tc_scope) {
    Some(s) => s,
    None => v8::String::new(tc_scope, "").unwrap(),
  };
  if is_err {
    eprint!("{}", str_.to_rust_string_lossy(tc_scope));
    stdout().flush().unwrap();
  } else {
    print!("{}", str_.to_rust_string_lossy(tc_scope));
    stdout().flush().unwrap();
  }
}
```

外部引用函数接收3个参数

1. v8::HandleScope： isolate context
2. v8::FunctionCallbackArguments：JavaScript 调用端传回的参数
3. v8::ReturnValue：Rust 端执行后返回的值

通过这种方式，Rust 端和 JavaScript 端完成了交互。

Deno只定义了10来个外部引用函数，上层的 operation 其实是通过 send 函数转移到 Rust 端来完成的。

下一篇我们再来讲讲什么是 operation ，以及背后的实现细节。