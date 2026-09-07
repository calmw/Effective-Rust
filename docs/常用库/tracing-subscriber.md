# Rust 常用库：`tracing-subscriber`

[`tracing-subscriber`](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/) 用于收集 `tracing` 产生的 Span 和 Event，并负责过滤、格式化和输出。

## 1. 添加依赖

```bash
cargo add tracing-subscriber --features env-filter,json
```

## 2. 最小初始化

```rust
fn main() {
    tracing_subscriber::fmt::init();
    tracing::info!("application started");
}
```

应用中通常只能设置一次全局默认 Subscriber，因此应在程序入口集中初始化。

## 3. 使用环境变量过滤日志

```rust
use tracing_subscriber::EnvFilter;

fn init_tracing() {
    let filter = EnvFilter::try_from_default_env()
        .unwrap_or_else(|_| EnvFilter::new("info,my_app=debug"));

    tracing_subscriber::fmt()
        .with_env_filter(filter)
        .with_target(true)
        .init();
}
```

运行时设置：

```bash
RUST_LOG=info,my_app::database=debug cargo run
```

## 4. 分层组合

复杂应用使用 `registry().with(...)` 组合多个 Layer：

```rust
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt, EnvFilter};

tracing_subscriber::registry()
    .with(EnvFilter::new("info"))
    .with(tracing_subscriber::fmt::layer())
    .init();
```

之后可以继续加入文件输出、OpenTelemetry、错误信息或自定义 Layer。

## 5. JSON 日志

生产环境常用结构化 JSON，便于日志平台解析：

```rust
tracing_subscriber::fmt()
    .json()
    .with_current_span(true)
    .flatten_event(true)
    .init();
```

本地开发通常使用易读文本，生产环境使用 JSON；可通过配置切换。

## 6. 最佳实践

1. 在 `main` 最开始初始化，避免丢失启动阶段事件。
2. 过滤器应支持环境变量，同时提供安全的默认级别。
3. 不要默认开启所有依赖的 `trace`，否则可能产生海量日志。
4. 输出到文件时处理日志轮转，不要让单个文件无限增长。
5. 测试中优先使用 `.with_test_writer()`，避免干扰测试框架捕获输出。
6. 可复用的库不要安装全局 Subscriber，把最终选择权交给应用。

## 总结

`tracing` 负责埋点，`tracing-subscriber` 负责消费。把初始化、过滤和输出策略集中在应用入口，可以保持业务代码与日志后端解耦。
