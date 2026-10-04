# Rust 常用库：`axum`

[`axum`](https://docs.rs/axum/latest/axum/) 是 Tokio 生态中的 Web 框架，围绕路由、Extractor、响应转换和 Tower 中间件构建。

本文是快速入门。更完整的项目结构、中间件、测试、安全和生产实践请阅读：[Axum 完整指南](../web/axum/README.md)。

## 1. 添加依赖

```bash
cargo add axum
cargo add tokio --features macros,rt-multi-thread
cargo add serde --features derive
```

## 2. 最小服务

```rust
use axum::{routing::get, Router};

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let app = Router::new().route("/health", get(|| async { "ok" }));
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await?;
    axum::serve(listener, app).await
}
```

## 3. Extractor 与 JSON

```rust
use axum::{extract::{Path, State}, Json};
use serde::{Deserialize, Serialize};
use std::sync::Arc;

struct AppState {
    service_name: String,
}

#[derive(Deserialize)]
struct CreateUser {
    name: String,
}

#[derive(Serialize)]
struct User {
    id: u64,
    name: String,
}

async fn create_user(
    State(state): State<Arc<AppState>>,
    Path(team_id): Path<u64>,
    Json(input): Json<CreateUser>,
) -> Json<User> {
    let _ = (&state.service_name, team_id);
    Json(User { id: 1, name: input.name })
}
```

Extractor 从请求中提取路径、查询参数、状态、请求头和 JSON。函数参数本身就是接口所需输入的声明。

## 4. 统一错误响应

Handler 返回的错误需要实现 `IntoResponse`：

```rust
use axum::{http::StatusCode, response::{IntoResponse, Response}, Json};
use serde_json::json;

struct ApiError {
    status: StatusCode,
    message: String,
}

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        (self.status, Json(json!({ "error": self.message }))).into_response()
    }
}
```

生产环境应把内部错误映射为稳定的公开错误码，不能把数据库错误或 backtrace 原样返回客户端。

## 5. 状态和中间件

数据库连接池、配置和 HTTP Client 可以放入 `AppState`。使用 `Arc` 共享只读服务对象；可并发资源自身通常已有连接池或内部共享机制。

常见中间件由 `tower-http` 提供，例如请求追踪、CORS、压缩和超时。

## 6. 最佳实践

1. Handler 保持轻薄：提取输入、调用服务、转换响应。
2. 领域逻辑放在 service/domain 层，不依赖 Axum 类型。
3. 为请求体大小、并发数、执行时间设置限制。
4. CORS 使用明确的允许来源，不要在带凭证场景无条件放开。
5. 使用 `TraceLayer` 和 `tracing` 记录请求 ID、状态码和耗时。
6. `AppState` 中复用数据库连接池和 `reqwest::Client`。
7. 使用 `Router` 直接进行集成测试，不必真的绑定 TCP 端口。
8. 实现优雅关闭，让正在处理的请求有机会完成。

## 总结

Axum 的核心是 Router、Extractor、`IntoResponse` 和 Tower 中间件。保持 Handler 薄、领域层独立，并统一处理状态、错误、超时与可观察性，可以得到易测试、易扩展的 Web 服务。
