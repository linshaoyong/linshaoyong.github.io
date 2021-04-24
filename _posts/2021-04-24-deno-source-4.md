---
layout: post
title: "Deno源码解读之deno_core（四）"
date: 2021-04-24 22:12:50 +0800
tags: [rust, deno, javascript, v8]
---

## Deno源码解读之deno_core（四）

### core/runtime.rs

```rust
/// Internal state for JsRuntime which is stored in one of v8::Isolate's
/// embedder slots.
pub(crate) struct JsRuntimeState {
  pub global_context: Option<v8::Global<v8::Context>>,
  pub(crate) op_state: Rc<RefCell<OpState>>,
  ......
}

impl JsRuntime {
  /// Only constructor, configuration is done through `options`.
  pub fn new(mut options: RuntimeOptions) -> Self {
    ......
    
    isolate.set_slot(Rc::new(RefCell::new(JsRuntimeState {
      global_context: Some(global_context),
      op_state: Rc::new(RefCell::new(op_state)),
      ......
    })));
    ......
  }
}
```

JsRuntimeState 的字段较多，我们聚焦在 op_state: Rc<RefCell<OpState>> 上。创建 JsRuntime 时要同时创建一个 JsRuntimeState 并通过 isolate.set_slot 把状态跟 isolate 关联上。OpState 的定义在 core/ops.rs 中。

### core/ops.rs

```rust
/// Maintains the resources and ops inside a JS runtime.
pub struct OpState {
  pub resource_table: ResourceTable,
  pub op_table: OpTable,
  pub get_error_class_fn: GetErrorClassFn,
  gotham_state: GothamState,
}

impl OpState {
  pub(crate) fn new() -> OpState {
    OpState {
      resource_table: Default::default(),
      op_table: OpTable::default(),
      get_error_class_fn: &|_| "Error",
      gotham_state: Default::default(),
    }
  }
}

impl Deref for OpState {
  type Target = GothamState;

  fn deref(&self) -> &Self::Target {
    &self.gotham_state
  }
}

impl DerefMut for OpState {
  fn deref_mut(&mut self) -> &mut Self::Target {
    &mut self.gotham_state
  }
}
```

我们注册的 ops 都保存在 op_table 字段，gotham_state 字段用来保存值，这样 Rust 和 JavaScript 之间就可以传递数据了。

### gotham_state.rs

```rust
// Forked from Gotham:
// https://github.com/gotham-rs/gotham/blob/bcbbf8923789e341b7a0e62c59909428ca4e22e2/gotham/src/state/mod.rs
// Copyright 2017 Gotham Project Developers. MIT license.

#[derive(Default)]
pub struct GothamState {
  data: BTreeMap<TypeId, Box<dyn Any>>,
}

impl GothamState {
  /// Puts a value into the `GothamState` storage. One value of each type is retained.
  /// Successive calls to `put` will overwrite the existing value of the same
  /// type.
  pub fn put<T: 'static>(&mut self, t: T) {
    let type_id = TypeId::of::<T>();
    trace!(" inserting record to state for type_id `{:?}`", type_id);
    self.data.insert(type_id, Box::new(t));
  }
  
  pub fn take<T: 'static>(&mut self) -> T {
    self.try_take().unwrap_or_else(|| missing::<T>())
  }
}
```

有意思的是 GothamState的实现是来自  [Gotham](https://github.com/gotham-rs/gotham) 项目，GothamState 内部用一个 BTreeMap 数据结构来保存数据，同一种类型只保存一个值。

我们用一个例子来演示下整个的过程。

```rust
use deno_core::op_sync;
use deno_core::JsRuntime;
use deno_core::OpState;

fn main() {
  let mut runtime = JsRuntime::new(Default::default());

  runtime.register_op(
    "op_get_input",
    op_sync(|state: &mut OpState, _: Option<u32>, _| {
      let input = state.borrow::<u32>();
      Ok(input.clone())
    }),
  );

  runtime.register_op(
    "op_add_one",
    op_sync(|state: &mut OpState, num: u32, _| {
      let res = num + 1;
      state.put::<u32>(res);
      Ok(res)
    }),
  );

  let op_state = runtime.op_state();
  op_state.borrow_mut().put(100_u32);

  runtime
    .execute(
      "<init>",
      r#"
Deno.core.ops();
Deno.core.registerErrorClass('Error', Error);

let input = Deno.core.opSync('op_get_input');
Deno.core.opSync('op_add_one', input);
"#,
    )
    .unwrap();

  let res = op_state.borrow_mut().take::<u32>();
  println!("{}", res);
}
```

这个例子要做的事情是在 Rust 端有个数据 100_u32，要传递给 V8，在 V8 执行加 1 操作，再从 Rust 获取结果并打印。

通过 register_op 注册函数供 V8 调用，再通过 op_state 把数据传入 V8 或传回 Rust，这样就完成了一个完整交互。