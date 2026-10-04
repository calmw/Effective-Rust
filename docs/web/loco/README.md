# Loco 完整指南：特点、使用方式与最佳实践

[`Loco`](https://loco.rs/) 是建立在 Axum 之上的 Rust Web 应用框架。它借鉴 Ruby on Rails 的开发体验，在 Axum、Tower、SeaORM 等组件上预先组装了数据库、配置、日志、认证、后台任务、邮件、缓存、文件存储、测试和部署能力。

可以把 Loco 理解为：

```text
Axum + SeaORM + 约定式项目结构 + 代码生成 + 应用级基础设施
```

本文面向当前 Loco 1.x。当前稳定线使用 Axum 0.8 和 SeaORM 2.0，需要较新的 Rust 工具链。升级框架、ORM 或生成器前，应阅读官方升级说明。

## 1. Loco 的主要特点

### 1.1 Rails 风格，但生成的是普通 Rust 代码

Loco 提供约定式目录、Controller、Model、Migration、Worker、Mailer、Task 和代码生成器。生成代码属于应用自身，可以阅读和修改，不依赖隐藏的运行时元编程。

### 1.2 底层仍然是 Axum

Loco 最终构建真正的 `axum::Router<AppContext>`：

- Axum Extractor 可以直接使用；
- Handler 仍然是普通 `async fn`；
- Tower/Tower HTTP 中间件仍然适用；
- 可以通过 Hooks 修改 Router 和启动流程；
- 已有 Axum 代码和知识能够继续复用。

### 1.3 SeaORM 数据层

数据库项目默认使用 SeaORM，提供异步查询、连接池、Migration、Entity、ActiveModel、关系、分页和代码生成。

### 1.4 Batteries Included

Loco 已经集成或预留了这些应用能力：

- 多环境配置；
- 数据库与 Migration；
- JWT 和 API Token 认证；
- 后台 Worker 与持久队列；
- Scheduler；
- Mailer；
- Cache 和 Storage；
- 健康检查；
- 测试工具；
- Docker、Nginx 和 Lambda 部署生成器。

### 1.5 CLI 与生成器

`cargo loco` 能生成 Model、Migration、Controller、完整 CRUD、Worker、Task、Mailer 和部署文件。它减少样板代码，但不能替代业务建模、权限设计和代码审查。

## 2. 适用场景

Loco 比较适合：

- SaaS、管理后台和数据库驱动的 API；
- 需要用户认证、邮件、后台任务的产品；
- 希望快速交付的个人开发者和小团队；
- 希望多个服务采用统一结构的团队；
- 熟悉 Rails、Django 或 Laravel 的开发者；
- 已经了解 Axum，希望减少基础设施接线工作。

不一定适合：

- 只有少量端点、没有数据库的极小服务；
- 需要完全自定义网络栈或极简依赖的程序；
- 团队已经拥有成熟的 Axum 基础项目；
- 协议网关、流处理或底层网络服务；
- 不希望接受 SeaORM 和约定式目录结构的项目。

```text
需要完全控制组件和架构       -> Axum
需要快速构建完整 Web 产品     -> Loco
现有 Axum 服务只缺少一项能力 -> 优先添加对应 crate
```

## 3. 安装与创建项目

更新 Rust，并安装应用生成器和 SeaORM CLI：

```bash
rustup update
cargo install loco
cargo install sea-orm-cli --version '^2.0'
```

`loco` 是创建应用的独立 CLI，`loco-rs` 是生成项目实际依赖的框架 crate。

官方当前推荐显式指定数据库、后台执行方式和资源类型，使创建过程可重复：

```bash
loco new --name hello_loco --db sqlite --bg async --assets none
cd hello_loco
cargo loco start
```

检查内置端点和路由：

```bash
cargo loco routes
curl http://127.0.0.1:5150/_ping
curl http://127.0.0.1:5150/_health
curl http://127.0.0.1:5150/_readiness
```

端口和监控端点以生成项目的当前配置为准。

## 4. 项目结构

典型数据库项目：

```text
hello_loco/
├── Cargo.toml
├── config/
│   ├── development.yaml
│   ├── test.yaml
│   └── production.yaml
├── migration/
│   └── src/
├── src/
│   ├── app.rs
│   ├── lib.rs
│   ├── controllers/
│   ├── models/
│   │   ├── _entities/
│   │   └── mod.rs
│   ├── workers/
│   ├── tasks/
│   ├── mailers/
│   └── initializers/
└── tests/
    ├── requests/
    ├── models/
    └── fixtures/
```

- `src/app.rs`：通过 `Hooks` 组装路由、Worker、Task 和生命周期；
- `controllers/`：HTTP 输入输出适配；
- `models/_entities/`：SeaORM 自动生成的 Entity；
- `models/*.rs`：开发者维护的模型查询和行为；
- `migration/`：数据库 Schema 变更；
- `workers/`：后台作业；
- `tasks/`：一次性 CLI 任务；
- `config/`：不同环境的配置；
- `tests/`：请求、模型和快照测试。

不要手工修改 `_entities`。自定义逻辑写在外层 Model 文件，Schema 变化通过 Migration 完成后重新生成 Entity。

## 5. Controller 与路由

生成 Controller：

```bash
cargo loco generate controller notes list show
```

生成器会创建 Controller 和请求测试、声明模块，并把路由注册到 `src/app.rs`。不要把 action 命名为 `get`、`post`、`delete` 等 HTTP 方法名，否则可能遮蔽 prelude 中的路由函数。

手写 Controller：

```rust,ignore
use loco_rs::prelude::*;

async fn hello() -> Result<Response> {
    format::text("hello from Loco")
}

async fn echo(Json(body): Json<serde_json::Value>) -> Result<Response> {
    format::json(body)
}

pub fn routes() -> Routes {
    Routes::new()
        .prefix("api/example")
        .add("/", get(hello))
        .add("/echo", post(echo))
}
```

在 `src/app.rs` 注册：

```rust,ignore
fn routes(_ctx: &AppContext) -> AppRoutes {
    AppRoutes::with_default_routes()
        .add_route(controllers::example::routes())
}
```

`AppRoutes::with_default_routes()` 会保留框架提供的监控端点。

Controller 应保持轻薄：

1. 使用 Extractor 读取 HTTP 输入；
2. 校验并转换请求 DTO；
3. 调用 Model 或应用服务；
4. 把结果转换成 HTTP 响应。

复杂业务逻辑应放入 Service/use case，使 HTTP Handler、Worker 和 Task 可以复用。

## 6. 响应与 DTO

`format` 模块提供常用响应：

```rust,ignore
async fn json_response() -> Result<Response> {
    format::json(serde_json::json!({ "status": "ok" }))
}

async fn text_response() -> Result<Response> {
    format::text("ok")
}

async fn empty_response() -> Result<Response> {
    format::empty()
}
```

输入 DTO、领域模型、数据库 Entity 和响应 DTO 应按职责分开。不要把完整 Entity 直接作为公开 API：它可能暴露内部主键、密码哈希、租户字段或将来新增的列。

## 7. Model、Migration 与 Scaffold

生成 Model：

```bash
cargo loco generate model posts title:string! content:text user:references
```

这个命令会创建 Migration、应用开发数据库 Migration、重新生成 Entity，并创建外层 Model 文件和测试。

字段后缀：

```text
name:string    可空字段，Rust 中通常是 Option<String>
name:string!   NOT NULL
name:string^   UNIQUE，并且 NOT NULL
user:references 创建 belongs-to 外键
```

完整 CRUD 脚手架：

```bash
cargo loco generate scaffold posts title:string! content:text
```

Scaffold 是起点，不是最终产品。生成后必须检查：

- 哪些字段允许客户端设置；
- 是否需要认证、授权和资源归属检查；
- 分页、过滤和排序是否有限制；
- DTO 是否暴露内部字段；
- 错误是否符合公开 API 契约；
- 测试是否覆盖权限边界。

简单 SeaORM 查询：

```rust,ignore
use loco_rs::prelude::*;
use sea_orm::{ActiveModelTrait, ActiveValue::Set, EntityTrait};

use crate::models::_entities::posts;

async fn find_post(ctx: &AppContext, id: i32) -> ModelResult<posts::Model> {
    posts::Entity::find_by_id(id)
        .one(&ctx.db)
        .await?
        .ok_or(ModelError::EntityNotFound)
}

async fn create_post(ctx: &AppContext, title: String) -> ModelResult<posts::Model> {
    posts::ActiveModel {
        title: Set(title),
        ..Default::default()
    }
    .insert(&ctx.db)
    .await
    .map_err(Into::into)
}
```

实际主键和字段类型以生成的 Entity 为准。

### Migration 最佳实践

- 所有 Schema 变化创建新的 Migration；
- Migration 与依赖它的代码一起提交；
- 已进入共享环境的 Migration 不随意改写；
- 删除列和大表变更采用兼容性分阶段发布；
- CI 验证 Migration 能从空数据库完整执行；
- 生产 Migration 作为明确发布步骤执行；
- 修改 Schema 后运行 `cargo loco db entities`；
- 使用 `cargo loco db status` 检查状态。

## 8. `AppContext` 与依赖注入

Loco 使用 `AppContext` 传递共享资源，而不是使用可变全局变量：

```rust,ignore
async fn index(State(ctx): State<AppContext>) -> Result<Response> {
    let users = users::Entity::find().all(&ctx.db).await?;
    format::json(users)
}
```

`AppContext` 通常包含环境、数据库连接、Queue Provider、Config、Mailer、Storage、Cache 和 `SharedStore`。Controller、Worker、Task 和 Scheduler 可以获得同一套上下文。

最佳实践：

- 长期资源在启动时创建并复用；
- 不要每个请求新建数据库池或 HTTP Client；
- 自定义客户端或服务通过明确扩展点加入；
- `after_context` 修改 Context 时使用 `into_builder()`，保留已有组件；
- 请求级数据放在 Extractor/Extension，不放全局 Context；
- 不用一个巨大的 Mutex 包住全部状态。

## 9. `Hooks` 生命周期

应用通常在 `src/app.rs` 为 `App` 实现 `Hooks`。主要方法包括：

- `app_name`：应用名称；
- `boot`：创建应用和 Context；
- `routes`：注册路由；
- `connect_workers`：注册 Worker；
- `register_tasks`：注册 Task；
- `truncate`：测试环境清理数据库；
- `seed`：加载种子数据。

还可以覆盖日志、配置加载、Context 构建前后、Router 构建前后和服务启动方式。优先使用默认生命周期；如果大量替换框架组装逻辑，应重新判断直接使用 Axum 是否更简单。

## 10. 配置与秘密

默认配置文件：

```text
config/development.yaml
config/test.yaml
config/production.yaml
```

环境名称解析优先级：

```text
LOCO_ENV -> RAILS_ENV -> NODE_ENV -> development
```

本地覆盖可以写入 `config/development.local.yaml`，并加入 `.gitignore`。

使用环境变量注入秘密：

```yaml
database:
  uri: <%= get_env(name="DATABASE_URL") %>

auth:
  jwt:
    secret: <%= get_env(name="JWT_SECRET") %>
    expiration: 604800
```

最佳实践：

- YAML 只保存结构和非敏感默认值；
- 密码和密钥由环境变量或 Secret Manager 注入；
- 生产环境显式设置 `LOCO_ENV=production`；
- 缺失配置时启动立即失败；
- 业务代码不要到处直接读取环境变量；
- 不在日志中输出解析后的完整配置。

## 11. 错误处理与校验

推荐的错误边界：

```text
数据库/外部服务错误 -> Model/Service 错误 -> 稳定的 HTTP 错误响应
```

Model 层可以用 `ModelError`/`ModelResult` 表达实体不存在、实体重复、校验失败、数据库错误和领域错误。

公开错误建议包含稳定错误码：

```json
{
  "error": {
    "code": "post_not_found",
    "message": "post not found"
  }
}
```

客户端不应看到 SeaORM 错误、SQL、服务器文件路径或 Backtrace。请求 DTO 负责输入格式，Service/Model 负责业务不变量，数据库约束作为最后一道防线。

## 12. JWT 认证与授权

Loco 提供：

- `auth::JWT`：验证 Token 并返回 Claims，不要求数据库；
- `auth::JWTWithUser<T>`：验证 Token 后加载数据库用户。

```rust,ignore
use loco_rs::controller::extractor::auth;
use loco_rs::prelude::*;

async fn current_user(
    auth: auth::JWT,
    State(_ctx): State<AppContext>,
) -> Result<Response> {
    format::json(serde_json::json!({
        "pid": auth.claims.pid
    }))
}
```

无效、缺失或过期 Token 会在 Handler 执行前被拒绝。

生成 Base64 Secret：

```bash
openssl rand -base64 64
```

当前 JWT 配置要求 Secret 是有效 Base64，普通短字符串可能在生成或验证 Token 时才报错。

认证不等于授权：通过 JWT 后，仍需检查用户角色、资源所有者和租户边界。

安全建议：

- API 客户端优先使用 Bearer Header；
- Cookie 使用 `HttpOnly`、`Secure`、`SameSite` 并考虑 CSRF；
- 不把 Token 放查询字符串；
- 登录、验证码、密码重置接口限流；
- 设计 Token 过期、刷新、注销和密钥轮换；
- 密码使用成熟密码哈希算法。

## 13. 后台 Worker 与队列

生成 Worker：

```bash
cargo loco generate worker report_worker
```

Worker 参数需要实现序列化，并且应尽量小：

```rust,ignore
#[derive(Debug, serde::Serialize, serde::Deserialize)]
pub struct WorkerArgs {
    pub report_id: i32,
}
```

Worker 模式：

```text
ForegroundBlocking  当前调用中执行，适合测试
BackgroundAsync     当前进程 tokio::spawn，重启会丢任务
BackgroundQueue     持久队列，由 Worker 进程消费
```

运行持久队列 Worker：

```bash
cargo loco start --worker
```

开发环境可以同时运行：

```bash
cargo loco start --server-and-worker
```

重要作业应使用持久队列，并遵循：

- 作业幂等；
- 外部调用有超时；
- 重试有上限、退避和抖动；
- 区分永久错误与临时错误；
- 记录 Job ID 和业务资源 ID；
- 有死信或人工恢复流程；
- 消息结构兼容滚动发布期间的新旧 Worker。

## 14. Task 与 Scheduler

Task 用于一次性运维或批处理：

```bash
cargo loco generate task cleanup
cargo loco task cleanup
```

Scheduler 用于周期执行 Task 或命令：

```bash
cargo loco generate scheduler
cargo loco scheduler --config config/scheduler.yaml --list
cargo loco scheduler --config config/scheduler.yaml
```

`cargo loco start --all` 可以把 Server、Worker 和 Scheduler 放进同一进程，但生产环境是否合并要根据故障隔离与扩缩容决定。

周期任务应考虑重复触发、多实例竞争、时区、执行超时和幂等。耗时任务最好由 Scheduler 入队，再交给 Worker 执行。

## 15. Mailer、Cache 与 Storage

### Mailer

- 邮件发送放入持久 Worker；
- 模板和业务参数分开；
- 开发测试使用捕获邮箱或测试 Provider；
- 不记录完整正文、验证码和重置链接。

### Cache

- Cache 是优化，不是唯一数据源；
- Key 包含版本和租户；
- 设置 TTL 和失效策略；
- 权限结果不要缓存过久。

### Storage

- 限制上传大小和内容类型；
- 不信任客户端文件名；
- 随机生成对象 Key；
- 私有文件使用鉴权下载或短期签名 URL；
- 必要时扫描恶意文件。

## 16. 测试

Loco 的测试工具可以启动接近真实应用的 Context：

```rust,ignore
use loco_rs::testing::prelude::*;

use myapp::app::App;
use myapp::models::_entities::posts;
use sea_orm::{ActiveModelTrait, ActiveValue::Set};

#[tokio::test]
async fn creates_a_post() -> loco_rs::Result<()> {
    let boot = boot_test::<App>().await?;

    let post = posts::ActiveModel {
        title: Set("hello".to_owned()),
        ..Default::default()
    }
    .insert(&boot.app_context.db)
    .await?;

    assert_eq!(post.title, "hello");
    Ok(())
}
```

推荐分层：

1. 纯业务函数单元测试；
2. Model/Service 集成测试；
3. Request 测试，验证路由、认证和响应；
4. Worker 与 Task 测试；
5. 少量完整端到端测试。

快照测试适合复杂稳定结构，但 UUID、时间戳和密码哈希需要 Redaction。权限、状态码和关键字段仍应使用明确断言。

```bash
cargo test
cargo test --test requests_notes
cargo insta review
```

Fixture 应保持小而清晰，不应依赖测试执行顺序。

## 17. 正确使用生成器

常用命令：

```bash
cargo loco generate model posts title:string!
cargo loco generate migration AddViewsToPosts views:int
cargo loco generate scaffold comments body:text!
cargo loco generate controller metrics index
cargo loco generate worker report_worker
cargo loco generate task cleanup
cargo loco generate mailer welcome
cargo loco generate deployment docker
```

每次生成后都应：

1. 阅读 Git diff；
2. 检查修改了哪些注册文件；
3. 校验 Migration；
4. 检查 DTO 和批量赋值；
5. 添加认证和资源级授权；
6. 补充业务测试；
7. 运行格式化、Clippy 和测试。

不要盲目重新生成并覆盖已经修改过的业务代码。

## 18. 部署

构建 Release：

```bash
cargo build --release
```

目标服务器不需要 Cargo 和 Rust 工具链，但 `config/`、邮件模板、静态资源等运行时文件必须一起部署。

生成 Docker、Nginx 或 Lambda 文件：

```bash
cargo loco generate deployment docker
cargo loco generate deployment nginx
cargo loco generate deployment lambda
```

Docker 示例：

```bash
docker build -t myapp .
docker run -p 5150:5150 --env-file .env myapp
```

Lambda 适合 HTTP 请求，但常驻 Worker 和 Scheduler 应部署在长期运行的环境，或改用云平台的队列与定时触发。

上线前执行：

```bash
myapp-cli doctor --environment production
```

生产检查：

- 设置 `LOCO_ENV=production`；
- 数据库、队列和 SMTP 可连接；
- Secret 来自安全存储；
- 使用结构化 JSON 日志；
- 连接池大小符合数据库容量；
- Migration 由单独发布步骤执行；
- 多实例启动时禁用自动 Migration，避免竞争；
- Server、Worker、Scheduler 按负载独立扩缩容；
- 健康检查接入负载均衡器；
- 支持优雅关闭。

## 19. 推荐架构

业务变复杂后，可以在 Loco 约定内增加应用服务层：

```text
Controller -> Application Service -> Model/Repository -> Database
   HTTP             用例                  数据访问
```

```text
src/
├── controllers/
├── models/
├── services/
│   ├── account.rs
│   ├── billing.rs
│   └── reporting.rs
├── workers/
├── tasks/
└── app.rs
```

关键原则：

- Controller 不承担复杂业务规则；
- Worker 和 HTTP 共用 Service，不复制逻辑；
- 跨多个 Model 的流程放在应用服务；
- 事务边界和业务用例保持一致；
- 不在数据库事务中等待无关网络请求；
- 外部 API 封装成可替换客户端，方便测试。

## 20. 生产最佳实践

### 数据库

- 所有列表分页并限制最大页大小；
- 避免 N+1 查询；
- 唯一性、外键和关键约束落实到数据库；
- 事务短小；
- 监控慢查询和连接池；
- 针对真实负载优化。

### API 与安全

- 使用请求/响应 DTO；
- 公开稳定错误码；
- 限制 Body、上传、分页、并发和超时；
- 认证之后继续执行授权；
- Secret 不进入仓库、日志和错误响应；
- 登录和高成本端点限流；
- 明确 API 版本和兼容策略。

### 后台系统

- 关键任务使用持久队列；
- Worker 独立扩缩容；
- 监控积压、失败率和执行时长；
- Scheduler 避免多实例重复执行；
- Job Payload 保持小且向后兼容。

### 可观察性

- 使用结构化日志；
- 传递 Request ID、User ID 和 Job ID；
- 不记录 Token、密码和完整个人数据；
- 监控请求耗时、错误率、数据库池和队列积压；
- 错误在系统边界记录一次，并保留错误链。

## 21. 常见错误

### 21.1 没理解 Axum 就依赖生成器

Loco 的 Handler、Extractor、Router 和 Middleware 都来自 Axum。至少应先理解 Axum 基础。

### 21.2 修改 `_entities`

重新生成 Entity 会覆盖改动。自定义逻辑必须写在外层 Model 文件。

### 21.3 把 Scaffold 当最终产品

生成器不知道真实权限、业务约束和 API 契约，生成后必须审查。

### 21.4 生产读取 development 配置

显式设置 `LOCO_ENV=production`，并在真实部署环境运行 `doctor`。

### 21.5 JWT Secret 不是 Base64

配置可能成功加载，却在生成或校验 Token 时失败。

### 21.6 使用 `BackgroundAsync` 执行关键任务

进程重启会丢任务。邮件、付款和报表等重要作业使用持久队列。

### 21.7 多实例自动执行 Migration

生产应由 CI/CD 或单独 Release Job 迁移数据库。

### 21.8 Controller 包含所有业务逻辑

这会造成 HTTP、Worker 和 Task 重复代码，并让事务与测试边界混乱。

## 22. Loco 与 Axum 如何选择

选择 Axum：

- 需要精确控制依赖和目录；
- 服务很小或不是典型数据库应用；
- 团队已有基础设施模板；
- 希望深入掌握 Tower 和异步 Web 架构。

选择 Loco：

- 需要数据库、认证、邮件和后台任务；
- 重视开发速度与团队约定；
- 产品属于 SaaS、CRUD 或后台系统；
- 接受 SeaORM 和框架默认结构。

推荐先学习 Axum，再用 Loco 完成真实产品。这样既理解底层，也知道何时遵循框架约定、何时使用扩展点。

## 23. 推荐学习路线

1. Rust 错误处理、模块和 async；
2. Tokio、Axum、Serde 与 tracing；
3. 创建一个 Loco SQLite 项目；
4. 生成 Model 和 Controller；
5. 学习 SeaORM Entity、ActiveModel、关系和 Migration；
6. 加入 DTO、校验、错误响应和分页；
7. 添加 JWT 认证与资源级授权；
8. 添加持久化 Worker；
9. 编写 Model、Request 和 Worker 测试；
10. 切换 PostgreSQL，生成 Docker 配置并运行生产检查。

## 24. 参考资料

- [Loco 官方网站](https://loco.rs/)
- [Loco 官方文档](https://loco.rs/docs/)
- [第一个 Loco 应用](https://loco.rs/docs/tutorials/your-first-app/)
- [从 Axum 到 Loco](https://loco.rs/docs/explanation/coming-from-axum/)
- [Controller 指南](https://loco.rs/docs/how-to/add-controller/)
- [Model 指南](https://loco.rs/docs/how-to/add-model/)
- [配置参考](https://loco.rs/docs/reference/configuration/)
- [后台 Worker](https://loco.rs/docs/how-to/add-worker/)
- [生产部署](https://loco.rs/docs/how-to/deploy/)
- [升级指南](https://loco.rs/docs/extras/upgrades/)

## 总结

Loco 不是 Axum 的底层替代品，而是建立在 Axum 之上的应用框架。它通过约定、生成器和预先组装的基础设施，让开发者把更多时间投入业务功能。

正确使用 Loco 的关键是：

- 理解 Axum 和 SeaORM；
- 把生成代码当成需要审查的起点；
- 保持 Controller 轻薄；
- 分清 DTO、Service、Model 和数据库边界；
- 关键后台任务使用持久队列；
- 正确管理配置、Secret、Migration、授权和测试；
- 在生产环境运行正确的 Server、Worker 和 Scheduler 进程。

如果目标是快速构建具备数据库、认证、邮件和后台任务的 Rust Web 产品，Loco 很值得学习；如果目标是完全定制底层 Web 架构，则应以 Axum 为主。
