# Axum 完整指南：特点、使用方式与最佳实践

[`axum`](https://docs.rs/axum/latest/axum/) 是 Tokio 团队维护的 Rust Web 框架。它建立在 `tokio`、`hyper` 和 `tower` 生态之上，重点是类型安全、模块化、低样板代码以及与异步生态的自然组合。

Axum 更准确地说是 HTTP 路由与请求处理库：

```text
Tokio       异步运行时、任务调度、网络 I/O
Hyper       HTTP 协议实现
Tower       Service、Layer 和中间件抽象
Axum        路由、Extractor、Handler、Response
Tower HTTP  CORS、追踪、压缩、超时、请求体限制等 HTTP 中间件
```

本文示例面向 Axum 0.8。升级依赖时应同时阅读官方 changelog 和迁移说明。

## 1. Axum 的主要特点

### 1.1 无宏路由

路由通过普通 Rust API 组合，不依赖属性宏：

```rust,ignore
let app = Router::new()
    .route("/users", get(list_users).post(create_user))
    .route("/users/{id}", get(get_user).delete(delete_user));
```

路由结构直观，可以拆分、嵌套、合并，也容易在测试中直接调用。

### 1.2 Extractor 驱动的请求解析

Handler 的参数声明它需要从请求中提取什么：

```rust,ignore
async fn update_user(
    State(state): State<AppState>,
    Path(id): Path<u64>,
    Query(params): Query<UpdateParams>,
    Json(input): Json<UpdateUser>,
) -> Result<Json<User>, ApiError> {
    // ...
}
```

常见 Extractor：

- `Path<T>`：路径参数；
- `Query<T>`：查询字符串；
- `Json<T>`：JSON 请求体；
- `Form<T>`：表单；
- `State<T>`：应用共享状态；
- `Extension<T>`：请求扩展中的数据；
- `HeaderMap`、`Request`、`Bytes`、`String`：底层请求数据。

类型不匹配、JSON 无效或字段缺失时，Extractor 会拒绝请求并生成响应。

### 1.3 统一的响应转换

任何实现 `IntoResponse` 的值都可以作为 Handler 返回值，例如：

```rust,ignore
"ok"
StatusCode::NO_CONTENT
Json(value)
(StatusCode::CREATED, Json(value))
(headers, body)
Result<Json<T>, ApiError>
```

这使简单接口非常短，同时允许复杂应用定义统一错误响应。

### 1.4 复用 Tower 中间件生态

Axum 的 Router 和 Handler 建立在 `tower::Service` 之上，可以使用 Tower/Tower HTTP 中间件处理：

- 请求追踪和指标；
- CORS；
- 超时；
- 并发限制；
- 压缩；
- 请求 ID；
- 敏感请求头；
- 请求体大小限制。

### 1.5 类型安全但不过度抽象

路径、查询、JSON、共享状态和错误均可使用普通 Rust 类型建模。编译器能够检查 Handler 签名和 Router 状态是否匹配，而领域层仍可保持与 Web 框架无关。

## 2. 适合与不适合的场景

Axum 适合：

- REST API、内部服务和微服务；
- 需要 Tokio、Tower、SQLx、Tonic 等生态集成的项目；
- 希望通过类型系统约束请求、响应和状态的团队；
- WebSocket、SSE、流式响应和反向代理类服务；
- 希望 Handler 易于单元测试和集成测试的项目。

选择前也要考虑：

- Axum 不自带 ORM、模板系统、身份认证、后台任务和管理后台；
- Rust 的类型、生命周期和异步约束有一定学习成本；
- 小型脚本或极快交付的后台系统，成熟的全栈框架可能更省时间；
- CPU 密集任务不能直接长期占用 Tokio 工作线程。

## 3. 创建项目

```bash
cargo new axum-demo
cd axum-demo

cargo add axum
cargo add tokio --features macros,rt-multi-thread,net,signal
cargo add serde --features derive
cargo add serde_json
cargo add thiserror
cargo add anyhow
cargo add reqwest --features json,rustls
cargo add tracing
cargo add tracing-subscriber --features env-filter
cargo add tower --features util
cargo add tower-http --features trace,timeout,limit,cors,request-id
```

学习阶段可以给 Tokio 开启 `full` feature；生产项目建议只开启实际使用的 feature，减少编译时间和依赖面。

## 4. 最小可运行服务

```rust,ignore
use axum::{routing::get, Router};
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let app = Router::new()
        .route("/", get(|| async { "Hello, Axum!" }))
        .route("/health", get(health));

    let listener = TcpListener::bind("127.0.0.1:3000").await?;
    println!("listening on {}", listener.local_addr()?);

    axum::serve(listener, app).await
}

async fn health() -> &'static str {
    "ok"
}
```

运行并访问：

```bash
cargo run
curl http://127.0.0.1:3000/health
```

生产环境通常监听 `0.0.0.0`，本地开发默认监听 `127.0.0.1` 更安全。

## 5. Router 与路由组织

### 5.1 HTTP 方法

```rust,ignore
use axum::{routing::{delete, get, patch, post, put}, Router};

let app = Router::new()
    .route("/users", get(list_users).post(create_user))
    .route(
        "/users/{id}",
        get(get_user)
            .put(replace_user)
            .patch(update_user)
            .delete(delete_user),
    );
```

Axum 0.8 的路径参数写作 `{id}`。不要照搬旧版本示例中的 `:id`。

### 5.2 嵌套路由

按业务模块创建 Router，再由应用入口组装：

```rust,ignore
fn user_routes() -> Router<AppState> {
    Router::new()
        .route("/", get(list_users).post(create_user))
        .route("/{id}", get(get_user).delete(delete_user))
}

fn app(state: AppState) -> Router {
    Router::new()
        .nest("/api/v1/users", user_routes())
        .route("/health", get(health))
        .with_state(state)
}
```

推荐按业务能力拆分路由，而不是建立一个包含所有 Handler 的巨大 `routes.rs`。

### 5.3 `layer` 的作用范围

`Router::layer` 只应用于调用它之前已经存在的路由：

```rust,ignore
let app = Router::new()
    .route("/private", get(private_handler))
    .layer(auth_layer)
    .route("/public", get(public_handler));
```

这里 `auth_layer` 只作用于 `/private`。中间件顺序和作用范围必须通过路由结构清楚表达。

## 6. Extractor 的正确使用

### 6.1 路径参数

```rust,ignore
use axum::extract::Path;

async fn get_user(Path(id): Path<u64>) -> String {
    format!("user id: {id}")
}
```

多个路径参数可以反序列化成元组或结构体：

```rust,ignore
use axum::extract::Path;
use serde::Deserialize;

#[derive(Deserialize)]
struct UserPath {
    team_id: u64,
    user_id: u64,
}

async fn handler(Path(path): Path<UserPath>) {
    let _ = (path.team_id, path.user_id);
}
```

对应路由可以是 `/teams/{team_id}/users/{user_id}`。

### 6.2 查询参数

```rust,ignore
use axum::extract::Query;
use serde::Deserialize;

#[derive(Deserialize)]
struct Pagination {
    #[serde(default = "default_page")]
    page: u32,
    #[serde(default = "default_page_size")]
    page_size: u32,
}

fn default_page() -> u32 { 1 }
fn default_page_size() -> u32 { 20 }

async fn list(Query(mut pagination): Query<Pagination>) {
    pagination.page_size = pagination.page_size.clamp(1, 100);
    // ...
}
```

即使提供默认值，也应限制分页大小，避免一次请求读取无限数据。

### 6.3 JSON 请求与响应

```rust,ignore
use axum::{http::StatusCode, Json};
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]
struct CreateUser {
    name: String,
    email: String,
}

#[derive(Serialize)]
struct UserResponse {
    id: u64,
    name: String,
}

async fn create_user(Json(input): Json<CreateUser>)
    -> (StatusCode, Json<UserResponse>)
{
    let user = UserResponse {
        id: 1,
        name: input.name,
    };

    (StatusCode::CREATED, Json(user))
}
```

输入 DTO、领域模型和响应 DTO 最好分开。这样不会意外接收只应由服务端设置的字段，也不会直接暴露数据库内部字段。

### 6.4 Extractor 顺序

请求体只能被消费一次，因此 `Json`、`String`、`Bytes` 等消费 Body 的 Extractor 必须放在参数列表最后：

```rust,ignore
async fn handler(
    State(state): State<AppState>,
    Path(id): Path<u64>,
    Json(input): Json<UpdateUser>,
) {
    // Json 在最后
}
```

`Json` 还要求请求具有合适的 `Content-Type: application/json`。

### 6.5 自定义拒绝响应

如果需要统一 JSON 错误格式，可以接收 `Result<Json<T>, JsonRejection>`：

```rust,ignore
use axum::{extract::rejection::JsonRejection, Json};

async fn handler(
    payload: Result<Json<CreateUser>, JsonRejection>,
) -> Result<Json<UserResponse>, ApiError> {
    let Json(input) = payload.map_err(ApiError::invalid_json)?;
    // ...
}
```

不要把内部解析细节直接暴露给公网客户端；公开错误消息应稳定、安全且便于调用者处理。

## 7. 共享状态 `State`

应用状态通常包含：

- 数据库连接池；
- 配置；
- 可复用的 `reqwest::Client`；
- 缓存、消息客户端和服务对象；
- 只读密钥或验证器。

```rust,ignore
use std::sync::Arc;

#[derive(Clone)]
struct AppState {
    user_service: Arc<UserService>,
    http_client: reqwest::Client,
    config: Arc<Config>,
}

async fn list_users(
    State(state): State<AppState>,
) -> Result<Json<Vec<UserResponse>>, ApiError> {
    let users = state.user_service.list().await?;
    Ok(Json(users))
}
```

最佳实践：

1. `AppState` 应廉价克隆；大型对象放进 `Arc`。
2. `sqlx::Pool`、`reqwest::Client` 等通常已在内部共享资源，可以直接克隆。
3. 不要把每个请求临时产生的数据放进全局 State。
4. 不要用一个大 `Mutex` 包住整个应用状态。
5. 需要锁时缩小锁的作用域，不要跨 `.await` 持有同步锁 guard。

## 8. 错误处理

Axum Handler 的错误最终也必须转换成 HTTP 响应。推荐使用 `thiserror` 定义内部错误，并集中实现 `IntoResponse`：

```rust,ignore
use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
    Json,
};
use serde::Serialize;
use thiserror::Error;

#[derive(Debug, Error)]
enum ApiError {
    #[error("resource not found")]
    NotFound,

    #[error("invalid input: {0}")]
    Validation(String),

    #[error("internal server error")]
    Internal(#[source] anyhow::Error),
}

#[derive(Serialize)]
struct ErrorBody {
    code: &'static str,
    message: String,
}

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        let (status, code, message) = match &self {
            Self::NotFound => (
                StatusCode::NOT_FOUND,
                "not_found",
                self.to_string(),
            ),
            Self::Validation(message) => (
                StatusCode::BAD_REQUEST,
                "validation_error",
                message.clone(),
            ),
            Self::Internal(error) => {
                tracing::error!(error = ?error, "request failed");
                (
                    StatusCode::INTERNAL_SERVER_ERROR,
                    "internal_error",
                    "internal server error".to_owned(),
                )
            }
        };

        (status, Json(ErrorBody { code, message })).into_response()
    }
}
```

注意：

- 公开响应提供稳定的 `code`，调用方不应依赖自然语言消息；
- 数据库错误、文件路径、堆栈和密钥不得返回客户端；
- 内部错误在系统边界记录一次，避免每层重复打印；
- `anyhow` 适合应用内部汇总错误，但不应直接作为公开 HTTP 协议。

## 9. 中间件

### 9.1 常用 Tower HTTP Layer

```rust,ignore
use axum::{extract::DefaultBodyLimit, http::StatusCode, Router};
use std::time::Duration;
use tower::ServiceBuilder;
use tower_http::{
    request_id::{MakeRequestUuid, PropagateRequestIdLayer, SetRequestIdLayer},
    timeout::TimeoutLayer,
    trace::TraceLayer,
};

let request_id_header = axum::http::HeaderName::from_static("x-request-id");

let middleware = ServiceBuilder::new()
    .layer(SetRequestIdLayer::new(
        request_id_header.clone(),
        MakeRequestUuid,
    ))
    .layer(TraceLayer::new_for_http())
    .layer(PropagateRequestIdLayer::new(request_id_header))
    .layer(TimeoutLayer::with_status_code(
        StatusCode::REQUEST_TIMEOUT,
        Duration::from_secs(10),
    ));

let app = Router::new()
    // 先添加路由
    .route("/health", get(health))
    // 再添加作用于这些路由的 Layer
    .layer(middleware)
    .layer(DefaultBodyLimit::max(1024 * 1024));
```

Axum 的 `Json`、`String`、`Bytes` 等 Extractor 默认有请求体大小限制。可以用 `DefaultBodyLimit` 按路由调整；第三方 Extractor 或直接读取 Body 时，应评估是否需要 `RequestBodyLimitLayer` 做全局限制。

### 9.2 自定义中间件

```rust,ignore
use axum::{
    extract::Request,
    middleware::Next,
    response::Response,
};

async fn log_method(request: Request, next: Next) -> Response {
    let method = request.method().clone();
    let path = request.uri().path().to_owned();

    let response = next.run(request).await;

    tracing::info!(%method, %path, status = %response.status());
    response
}

let app = app.layer(axum::middleware::from_fn(log_method));
```

需要访问 State 时使用 `from_fn_with_state`。通用、可配置或准备发布成 crate 的中间件，更适合直接实现 Tower `Layer` 和 `Service`。

### 9.3 中间件错误

Axum 希望最终服务错误为 `Infallible`：失败应被转换成响应。如果某个 Tower Layer 返回实际错误，应通过适当的错误处理 Layer 将其映射成 HTTP 响应，否则连接可能被关闭而没有响应体。

## 10. CORS 的安全配置

开发环境可以临时放宽 CORS，生产环境应明确允许来源、方法和请求头：

```rust,ignore
use axum::http::{header, HeaderValue, Method};
use tower_http::cors::CorsLayer;

let cors = CorsLayer::new()
    .allow_origin("https://app.example.com".parse::<HeaderValue>()?)
    .allow_methods([Method::GET, Method::POST])
    .allow_headers([header::CONTENT_TYPE, header::AUTHORIZATION]);
```

不要在允许 Cookie 或 Authorization 等凭证时，同时对任意来源使用宽松配置。

## 11. 优雅关闭

优雅关闭会停止接受新连接，并等待正在处理的连接结束：

```rust,ignore
use tokio::signal;

async fn shutdown_signal() {
    let ctrl_c = async {
        signal::ctrl_c()
            .await
            .expect("failed to install Ctrl+C handler");
    };

    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("failed to install SIGTERM handler")
            .recv()
            .await;
    };

    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => {},
        _ = terminate => {},
    }
}

axum::serve(listener, app)
    .with_graceful_shutdown(shutdown_signal())
    .await?;
```

优雅关闭不代表无限等待。SSE、WebSocket 或卡住的请求可能长期不结束，因此请求本身、后台任务以及部署平台还需要合理的关闭期限和取消机制。

## 12. 测试 Router

Router 实现 Tower `Service`，测试通常不需要监听真实端口：

```rust,ignore
use axum::{
    body::Body,
    http::{Request, StatusCode},
    routing::get,
    Router,
};
use tower::ServiceExt;

fn app() -> Router {
    Router::new().route("/health", get(|| async { "ok" }))
}

#[tokio::test]
async fn health_check_returns_ok() {
    let response = app()
        .oneshot(
            Request::builder()
                .uri("/health")
                .body(Body::empty())
                .unwrap(),
        )
        .await
        .unwrap();

    assert_eq!(response.status(), StatusCode::OK);
}
```

建议测试三个层次：

1. 领域和 service 层单元测试，不依赖 Axum；
2. Router 集成测试，验证路由、Extractor、中间件和错误映射；
3. 少量端到端测试，使用真实数据库或外部依赖的测试替身。

## 13. 推荐的项目结构

中小型服务可以按业务模块组织：

```text
src/
├── lib.rs
├── main.rs
├── app.rs                 # 组装 Router、State 和中间件
├── config.rs
├── error.rs               # ApiError 与公开错误格式
├── state.rs               # AppState
├── infrastructure/
│   ├── mod.rs
│   ├── database.rs
│   └── http_client.rs
└── user/
    ├── mod.rs
    ├── model.rs            # 领域模型
    ├── dto.rs              # 请求/响应 DTO
    ├── repository.rs
    ├── service.rs
    └── routes.rs           # Handler 与 Router
```

依赖方向建议：

```text
HTTP Handler -> Service/Use Case -> Repository trait -> Infrastructure
     Axum          业务逻辑             抽象              SQLx/外部 API
```

不要让核心业务逻辑直接依赖 `Json`、`Path`、`StatusCode` 等 Axum 类型。这样可以在 CLI、后台任务和测试中复用业务代码。

## 14. 一个推荐的应用入口

```rust,ignore
use axum::{extract::DefaultBodyLimit, http::StatusCode, routing::get, Router};
use std::time::Duration;
use tokio::net::TcpListener;
use tower_http::{timeout::TimeoutLayer, trace::TraceLayer};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| "info,tower_http=info".into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let state = build_state().await?;

    let app = Router::new()
        .route("/health", get(health))
        .nest("/api/v1/users", user_routes())
        .layer(DefaultBodyLimit::max(1024 * 1024))
        .layer(TimeoutLayer::with_status_code(
            StatusCode::REQUEST_TIMEOUT,
            Duration::from_secs(10),
        ))
        .layer(TraceLayer::new_for_http())
        .with_state(state);

    let listener = TcpListener::bind("0.0.0.0:3000").await?;
    tracing::info!(address = %listener.local_addr()?, "server listening");

    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await?;

    Ok(())
}
```

程序入口负责组装和生命周期管理，Handler 负责协议适配，Service 负责业务逻辑。

## 15. 生产环境最佳实践清单

### API 与类型设计

- 请求 DTO、领域模型、数据库模型和响应 DTO 按职责分开；
- 对分页、字符串长度、枚举值和数值范围做显式校验；
- 公开稳定错误码，不把内部错误消息当 API 契约；
- 为 API 版本和兼容策略建立清晰边界。

### 性能与资源

- 复用数据库连接池和 HTTP Client；
- 为请求、数据库查询和外部 HTTP 调用设置超时；
- 限制请求体、分页大小、并发数和上传大小；
- 流式处理大响应，不要把所有数据一次装入内存；
- CPU 密集工作放入受控线程池或任务队列，避免阻塞 Tokio；
- 通过 profiling 和指标定位瓶颈，不凭感觉优化。

### 安全

- 生产 CORS 使用明确 allowlist；
- 不记录 Authorization、Cookie、密码、密钥和完整个人数据；
- 身份认证与授权分开：认证回答“是谁”，授权回答“能做什么”；
- 不信任 `X-Forwarded-For` 等代理头，除非请求来自可信代理；
- TLS 通常由可信反向代理、Ingress 或 Rust TLS 层终止；
- 错误响应不得暴露 SQL、文件路径、backtrace 或内部服务地址；
- 对登录、短信、搜索等高成本接口实施限流和防滥用策略。

### 可观察性

- 使用 `TraceLayer` 和 `tracing`；
- 为每个请求生成或传递 Request ID；
- 记录方法、路径模板、状态码、耗时和关联 ID；
- 指标按路径模板聚合，不按原始 URL 聚合，避免高基数；
- 错误在边界记录一次，并保留内部错误链。

### 生命周期

- 支持 SIGTERM/Ctrl+C 优雅关闭；
- 后台任务必须有明确所有者、取消信号和 `JoinHandle`；
- 健康检查区分进程存活和服务是否准备好接收流量；
- 启动失败应快速退出，而不是带着不可用依赖继续运行；
- 数据库 Migration 应由明确的部署步骤管理。

## 16. 常见错误

### 16.1 把 `Json` 放在其他 Extractor 前面

`Json` 会消费请求体，应该放在 Handler 参数最后。

### 16.2 每个请求新建数据库池或 HTTP Client

连接池和 Client 应在启动时创建并放入 State 复用。

### 16.3 Handler 包含全部业务逻辑

这会让测试、复用和事务边界越来越困难。Handler 应保持薄。

### 16.4 直接把 `anyhow::Error` 返回给客户端

内部错误必须映射为稳定、安全的公开响应。

### 16.5 在异步 Handler 中执行阻塞操作

`std::fs`、同步数据库驱动、复杂压缩和 CPU 密集计算会阻塞 Tokio 工作线程。使用异步 API、`spawn_blocking` 或独立任务系统。

### 16.6 无限制接收请求体或返回结果

请求体、分页、并发、超时和上传都应有上限。

### 16.7 中间件顺序错误

Layer 的包裹顺序会影响追踪、超时和错误处理行为。先画清楚请求进入与响应返回的顺序，再组合 `ServiceBuilder`。

## 17. 学习路线

建议按以下顺序掌握：

1. Tokio 的 `async`、任务、超时和取消；
2. `Router`、Handler 和 `IntoResponse`；
3. `Path`、`Query`、`Json`、`State`；
4. `serde` 请求/响应类型；
5. `thiserror` 与统一 `ApiError`；
6. Tower/Tower HTTP 中间件；
7. Router 集成测试；
8. SQLx、认证授权、指标和优雅关闭；
9. 再根据业务需要学习 WebSocket、SSE、multipart 和 OpenAPI。

## 18. 参考资料

- [Axum 官方文档](https://docs.rs/axum/latest/axum/)
- [Axum 官方仓库与示例](https://github.com/tokio-rs/axum)
- [Axum Middleware 文档](https://docs.rs/axum/latest/axum/middleware/)
- [Tower HTTP 文档](https://docs.rs/tower-http/latest/tower_http/)
- [Tokio 官方文档](https://tokio.rs/)

## 总结

Axum 的核心并不复杂：使用 Router 匹配请求，使用 Extractor 把请求转换成类型，使用 Handler 调用业务逻辑，再通过 `IntoResponse` 生成响应。

真正决定项目质量的是外围边界：

- Handler 是否足够薄；
- 错误是否统一且不会泄露内部信息；
- State 是否只保存可安全共享的长期资源；
- 超时、请求体、并发和分页是否有限制；
- 中间件顺序是否正确；
- 日志、指标、测试和优雅关闭是否从一开始就纳入设计。

把 Axum 当作 HTTP 适配层，而不是整个业务架构，通常能够得到更易测试、易维护、也更适合长期演进的 Rust Web 服务。
