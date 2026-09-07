# Rust 常用库：`sqlx`

[`sqlx`](https://docs.rs/sqlx/latest/sqlx/) 是异步 SQL 工具，支持 PostgreSQL、MySQL 和 SQLite。它不是传统 ORM：开发者直接编写 SQL，同时获得连接池、事务、类型映射和可选的编译期 SQL 检查。

## 1. 添加依赖

以 Tokio + PostgreSQL 为例：

```bash
cargo add sqlx --features runtime-tokio,postgres,macros,migrate
```

SQLite 或 MySQL 项目将 `postgres` 换成相应数据库 feature。具体 TLS feature 应根据部署环境选择。

## 2. 创建连接池

```rust
use sqlx::postgres::PgPoolOptions;
use std::time::Duration;

async fn create_pool(database_url: &str) -> Result<sqlx::PgPool, sqlx::Error> {
    PgPoolOptions::new()
        .max_connections(20)
        .acquire_timeout(Duration::from_secs(3))
        .connect(database_url)
        .await
}
```

应用通常创建一个 Pool 并共享其克隆。Pool 克隆成本低，不应为每次查询建立新连接池。

## 3. 查询数据

编译期检查的查询宏：

```rust,ignore
let user = sqlx::query!(
    "SELECT id, name FROM users WHERE id = $1",
    user_id
)
.fetch_optional(&pool)
.await?;
```

映射到结构体：

```rust,ignore
#[derive(sqlx::FromRow)]
struct User {
    id: i64,
    name: String,
}

let users = sqlx::query_as::<_, User>(
    "SELECT id, name FROM users ORDER BY id LIMIT $1"
)
.bind(limit)
.fetch_all(&pool)
.await?;
```

所有外部值都应通过 `.bind(...)` 或查询宏参数绑定，不能用字符串拼接 SQL。

## 4. 事务

```rust,ignore
let mut tx = pool.begin().await?;

sqlx::query("UPDATE accounts SET balance = balance - $1 WHERE id = $2")
    .bind(amount)
    .bind(from_id)
    .execute(&mut *tx)
    .await?;

sqlx::query("UPDATE accounts SET balance = balance + $1 WHERE id = $2")
    .bind(amount)
    .bind(to_id)
    .execute(&mut *tx)
    .await?;

tx.commit().await?;
```

事务应尽量短，不要在事务中等待无关的 HTTP 请求或执行耗时计算。

## 5. Migration 和离线检查

Migration 文件应进入版本控制，并在 CI 或部署流程中显式执行。使用 `query!` 系列宏时，可以通过 SQLx 的离线准备机制保存查询元数据，使 CI 不必连接真实数据库也能编译检查。

## 6. 最佳实践

1. 使用连接池并根据数据库容量设置上限，不要盲目增加连接数。
2. 参数全部绑定，禁止拼接不可信输入。
3. 列名明确写出，长期维护的查询避免 `SELECT *`。
4. 对列表接口使用分页，避免 `fetch_all` 读取无限结果。
5. 保持事务短小，并为锁等待和查询设置合理超时。
6. Migration 一旦进入共享环境，优先新增修复 migration，不随意改写历史。
7. 数据库错误在存储层转换为领域可理解的错误，API 不暴露内部 SQL。
8. 记录慢查询和耗时，但不要把敏感参数直接写入日志。

## 7. `query!` 还是 `query_as`/`FromRow`

- SQL 固定且希望编译期检查：优先 `query!`、`query_as!`；
- SQL 动态拼装：使用 QueryBuilder，并严格绑定参数；
- 希望减少构建时数据库依赖：使用离线模式或运行时检查的 `query_as`。

## 总结

SQLx 适合愿意直接掌控 SQL、同时希望保留类型安全和异步连接池能力的项目。正确使用的重点是参数绑定、连接池边界、短事务、迁移管理和查询结果限制。
