# Rust 常用库：`tracing`

[`tracing`](https://docs.rs/tracing/latest/tracing/) 是结构化诊断框架。它通过事件（Event）记录某个时刻发生的事情，通过跨度（Span）描述一段具有开始、结束和上下文的操作。

## 1. 添加依赖

```bash
cargo add tracing --features attributes
```

`tracing` 负责产生数据；应用程序还需要 `tracing-subscriber` 收集、过滤并输出数据。

## 2. 记录结构化事件

```rust
use tracing::{info, warn};

fn process_order(order_id: u64, amount: u64) {
    info!(order_id, amount, "processing order");

    if amount == 0 {
        warn!(order_id, "order amount is zero");
    }
}
```

字段应作为结构化数据记录，而不是全部拼进字符串：

```rust
tracing::info!(user_id = user.id, elapsed_ms, "request completed");
# struct User { id: u64 }
# let user = User { id: 1 };
# let elapsed_ms = 20;
```

## 3. 使用 Span

```rust
use tracing::{info, info_span};

let span = info_span!("create_user", user_id = 42);
let _guard = span.enter();
info!("validating user");
```

同步代码可以用 `enter()`；异步代码不要让 enter guard 跨越 `.await`，应使用 `Instrument` 或 `#[instrument]`。

## 4. 使用 `#[instrument]`

```rust
use tracing::instrument;

#[instrument(skip(password), fields(user.name = %username))]
async fn login(username: &str, password: &str) -> bool {
    !username.is_empty() && !password.is_empty()
}
```

- `skip(password)` 防止敏感数据或大型对象进入日志；
- `%value` 使用 `Display`；
- `?value` 使用 `Debug`；
- `err`、`ret` 可以记录返回错误或返回值，但要注意敏感信息。

## 5. 最佳实践

1. 事件消息保持稳定，查询条件放在结构化字段中。
2. HTTP 请求、数据库查询和后台任务应各自建立 Span。
3. 使用请求 ID、用户 ID、任务 ID 关联日志，但不要记录密码和令牌。
4. 错误通常在系统边界记录一次，底层函数只负责返回并补充上下文。
5. 高频路径使用合适的级别，避免大量 `info!` 造成成本和噪音。
6. 库产生 tracing 事件，但由最终应用决定过滤器和输出格式。

## 6. 常见误区

- 只添加 `tracing` 不初始化 Subscriber，事件不会被输出。
- 不要把整个请求对象用 `?request` 直接记录，可能泄露请求头和凭证。
- 不要在异步代码里让 `span.enter()` 的 guard 跨过 `.await`。
- `tracing` 是诊断基础设施，不代替业务错误类型和错误处理。

## 总结

`tracing` 的核心价值是让日志拥有上下文和结构。使用稳定事件名、可查询字段和边界清晰的 Span，才能真正改善异步系统的可观察性。
