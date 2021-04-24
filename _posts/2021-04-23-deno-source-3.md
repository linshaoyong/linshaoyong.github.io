---
layout: post
title: "Deno源码解读之deno_core（三）"
date: 2021-04-23 22:12:50 +0800
tags: [rust, deno, javascript, v8]
---

## Deno源码解读之deno_core（三）

如果我们回顾下 hello world 那个例子，会看到 JsRuntime 提供了一个 register_op 函数，在 Rust 端注册的 op 可以在 JavaScript 调用。

### core/examples/hello_world.rs

```rust
runtime.register_op(
  "op_print",
  // The op_fn callback takes a state object OpState,
  // a structured arg of type `T` and an optional ZeroCopyBuf,
  // a mutable reference to a JavaScript ArrayBuffer
  op_sync(|_state, msg: Option<String>, zero_copy| {
    let mut out = std::io::stdout();

    // Write msg to stdout
    if let Some(msg) = msg {
      out.write_all(msg.as_bytes()).unwrap();
    }

    // Write the contents of every buffer to stdout
    if let Some(buf) = zero_copy {
      out.write_all(&buf).unwrap();
    }

    Ok(()) // No meaningful result
  }),
);
```

```rust
runtime
  .execute(
    "<init>",
    r#"
Deno.core.ops();
Deno.core.registerErrorClass('Error', Error);

Deno.core.dispatchByName('op_print', 0, "Hello World!", new Uint8Array([10]));
}
"#,).unwrap();
```

首先，调用 register_op 注册 op_print 这个 op，然后在 JavaScript 调用 Deno.core.dispatchByName('op_print', 0, "Hello World!", new Uint8Array([10])) 就可以在 stdout打印出 Hello World!

那 Deno.core.dispatchByName 这个函数从哪来的呢？创建 JsRuntime 对象时，最后会执行 core.js 这个文件，我们看下这个文件。

### core/core.js

```javascript
"use strict";

((window) => {
  // Available on start due to bindings.
  const core = window.Deno.core;
  const { recv, send } = core;
  
  function ops() {
    // op id 0 is a special value to retrieve the map of registered ops.
    const newOpsCache = Object.fromEntries(send(0));
    opsCache = Object.freeze(newOpsCache);
    return opsCache;
  }
  
  function dispatch(opName, promiseId, control, zeroCopy) {
    return send(opsCache[opName], promiseId, control, zeroCopy);
  }
  
  Object.assign(window.Deno.core, {
    opAsync,
    opSync,
    dispatch: send,
    dispatchByName: dispatch,
    ops,
    close,
    resources,
    registerErrorClass,
    init,
  });
})(this);
```

所以，Rust 注册的 op_print 在 JavaScript 最终是通过 send(opsCache[opName], promiseId, control, zeroCopy) 调用的。

在 JavaScript 代码最开始，我们需要先执行 Deno.core.ops()，这个函数调用了 send(0)，获取所有已经注册的 ops，之后 send 函数就可以通过 opName 来查找 opId。

### core/bindings.rs

```rust
// send(0) returns obj of all ops, handle as special case
if op_id == 0 {
  let ops = OpTable::op_entries(state.op_state.clone());
  rv.set(to_v8(scope, ops).unwrap());
  return;
}
```

在 JavaScript 端调用 send 会转移到 Rust 端调用 bindings.rs#send 函数，该函数的主要逻辑如下

1.  获取跟 isoalte 关联的 JsRuntimeState 对象
2. 从参数中提取 OpId（如果 OpId 等于 0，从 JsRuntimeState 的 OpTable 中获取所有 ops 作为返回值，函数结束）
3. 从参数中提取 PromiseId 等数据
4. 根据 OpId 从 OpTable 查找 op，找到了就调用 OpTable::route_op 执行并设置返回值，找不到报错

```rust
let op = OpTable::route_op(op_id, state.op_state.clone(), payload, buf);
match op {
  Op::Sync(resp) => match resp {
    OpResponse::Value(v) => {
      rv.set(v.to_v8(scope).unwrap());
    }
    OpResponse::Buffer(buf) => {
      rv.set(boxed_slice_to_uint8array(scope, buf).into());
    }
  },
  Op::Async(fut) => {
    state.pending_ops.push(fut);
    state.have_unpolled_ops = true;
  }
  Op::AsyncUnref(fut) => {
    state.pending_unref_ops.push(fut);
    state.have_unpolled_ops = true;
  }
  Op::NotFound => {
    throw_type_error(scope, format!("Unknown op id: {}", op_id));
  }
}
```



接下来我们再来看看注册 op 是怎么注册的

### core/runtime.rs

```rust
  /// Registers an op that can be called from JavaScript.
  pub fn register_op<F>(&mut self, name: &str, op_fn: F) -> OpId
  where
    F: Fn(Rc<RefCell<OpState>>, OpPayload, Option<ZeroCopyBuf>) -> Op + 'static,
  {
    Self::state(self.v8_isolate())
      .borrow_mut()
      .op_state
      .borrow_mut()
      .op_table
      .register_op(name, op_fn)
  }
```

JsRuntime 内部获取跟 isolate 关联的 op_state，再从 op_state 取出 op_table，调用 op_table.register_op(name, op_fn)

### core/ops.rs

```rust
pub type OpFn =
  dyn Fn(Rc<RefCell<OpState>>, OpPayload, Option<ZeroCopyBuf>) -> Op + 'static;

pub struct OpTable(IndexMap<String, Rc<OpFn>>);

impl OpTable {
  pub fn register_op<F>(&mut self, name: &str, op_fn: F) -> OpId
  where
    F: Fn(Rc<RefCell<OpState>>, OpPayload, Option<ZeroCopyBuf>) -> Op + 'static,
  {
    let (op_id, prev) = self.0.insert_full(name.to_owned(), Rc::new(op_fn));
    assert!(prev.is_none());
    op_id
  }
}
```

OpTable 就是一个 IndexMap，保存注册的 ops。

```rust
pub enum Op {
  Sync(OpResponse),
  Async(OpAsyncFuture),
  /// AsyncUnref is the variation of Async, which doesn't block the program
  /// exiting.
  AsyncUnref(OpAsyncFuture),
  NotFound,
}
```

Op 是一个 enum 类型，共 4 种 variation。OpTable::route_op 执行用户注册的 op 并返回一个 Op variation。

```rust
pub fn route_op(
  op_id: OpId,
  state: Rc<RefCell<OpState>>,
  payload: OpPayload,
  buf: Option<ZeroCopyBuf>,
) -> Op {
  let op_fn = state
    .borrow()
    .op_table
    .0
    .get_index(op_id)
    .map(|(_, op_fn)| op_fn.clone());
  match op_fn {
    Some(f) => (f)(state, payload, buf),
    None => Op::NotFound,
  }
}
```

deno 把 Rust 跟 V8 之间的交互做了很好的抽象封装 ，只需要在 Rust 端定义好 Op 名称和 OpFn， 调用 register_op 注册，然后就可以在 V8 执行Deno.core.dispatchByName(op_name, op_id, ...) 来调用 Rust 的代码。

下一篇我们将一起看看之前提到的 isolate 关联的状态 JsRuntimeState。