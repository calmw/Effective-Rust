# Rust 常用库：`reqwest`

[`reqwest`](https://docs.rs/reqwest/latest/reqwest/) 是高层 HTTP 客户端，支持异步和阻塞请求、JSON、表单、代理、TLS、重定向等常见能力。

## 1. 添加依赖

异步 JSON 客户端：

```bash
cargo add reqwest --features json,rustls
cargo add tokio --features macros,rt-multi-thread
```

具体 TLS feature 会随版本和平台不同，应以项目当前版本文档为准。

## 2. 发送 GET 请求

```rust
use serde::Deserialize;

#[derive(Deserialize)]
struct User {
    id: u64,
    name: String,
}

async fn fetch_user(client: &reqwest::Client) -> anyhow::Result<User> {
    let user = client
        .get("https://api.example.com/users/1")
        .send()
        .await?
        .error_for_status()?
        .json::<User>()
        .await?;

    Ok(user)
}
```

`send()` 对 404、500 等 HTTP 状态默认仍返回 `Ok(Response)`；需要用 `error_for_status()` 将失败状态转换为错误。

## 3. 复用 `Client`

```rust
use std::time::Duration;

let client = reqwest::Client::builder()
    .connect_timeout(Duration::from_secs(3))
    .timeout(Duration::from_secs(10))
    .user_agent("effective-rust/1.0")
    .build()?;
# Ok::<(), reqwest::Error>(())
```

`Client` 内部维护连接池，并且可以廉价克隆。不要为每个请求创建新 Client。

## 4. 发送 JSON

```rust
use serde::Serialize;

#[derive(Serialize)]
struct CreateUser<'a> {
    name: &'a str,
}

async fn create(client: &reqwest::Client) -> reqwest::Result<()> {
    client
        .post("https://api.example.com/users")
        .json(&CreateUser { name: "Alice" })
        .send()
        .await?
        .error_for_status()?;
    Ok(())
}
```

## 5. 最佳实践

1. 全局或按上游服务复用 `Client`，利用连接池。
2. 同时设置连接超时和整个请求超时。
3. 调用 `error_for_status()`，不要把 HTTP 500 当成功响应解析。
4. 只对幂等操作或明确允许的操作重试，并使用指数退避和随机抖动。
5. 限制响应体大小，特别是处理不可信服务的响应时。
6. Token 放在请求头中，不要放 URL 或日志字段中。
7. 为上游调用增加 tracing Span，并记录状态码和耗时。
8. 测试使用 WireMock 等本地模拟服务，不依赖真实公网接口。

## 6. 阻塞客户端

`reqwest::blocking` 适合简单同步程序，但不能直接在 Tokio 异步任务中调用。异步应用若必须执行阻塞客户端，应使用 `spawn_blocking`，更好的方式是直接使用异步 Client。

## 总结

Reqwest 把 HTTP 常见能力封装成易用接口。可靠使用的关键是复用 Client、设置超时、检查状态码，并谨慎设计重试和响应大小限制。
