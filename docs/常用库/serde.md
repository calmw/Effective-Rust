# Rust 常用库：`serde`

[`serde`](https://serde.rs/) 是 Rust 最常用的序列化框架。它通过 `Serialize` 和 `Deserialize` trait 描述 Rust 类型如何转换为外部数据，以及如何从外部数据恢复。

`serde` 本身不负责 JSON、YAML 等具体格式；格式由 `serde_json`、`toml`、`serde_yaml` 等 crate 提供。

## 1. 添加依赖

```bash
cargo add serde --features derive
```

`derive` feature 允许使用 `#[derive(Serialize, Deserialize)]`。

## 2. 基本用法

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize, PartialEq)]
struct User {
    id: u64,
    name: String,
    active: bool,
}
```

此时 `User` 可以交给任何兼容 Serde 的数据格式处理。

## 3. 常用字段属性

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
struct UserProfile {
    user_id: u64,

    #[serde(default)]
    display_name: String,

    #[serde(skip_serializing_if = "Option::is_none")]
    avatar_url: Option<String>,

    #[serde(rename = "type")]
    kind: String,
}
```

- `rename`：修改单个字段的外部名称；
- `rename_all`：统一转换全部字段名；
- `default`：字段缺失时使用默认值；
- `skip_serializing_if`：满足条件时不输出字段；
- `flatten`：把嵌套结构的字段展开到当前层；
- `deny_unknown_fields`：遇到未知字段时拒绝反序列化。

## 4. 借用数据，减少分配

只在输入文本的生命周期足够长时使用借用字段：

```rust
use serde::Deserialize;

#[derive(Deserialize)]
struct Message<'a> {
    topic: &'a str,
    body: &'a str,
}
```

长期保存、跨线程传递或返回解析结果时，通常使用 `String` 更简单安全。

## 5. 自定义序列化

当外部格式与领域类型不同，可以使用 `serialize_with`、`deserialize_with` 或实现 `Serialize`、`Deserialize`。优先把转换逻辑放在独立函数中，不要为了接口格式破坏领域模型。

## 6. 最佳实践

1. 业务类型和 API 传输类型差异明显时，分别定义 Domain Model 与 DTO。
2. 对外部输入使用专门的请求结构体，不要直接反序列化进数据库实体。
3. 新增可选字段时使用 `Option<T>` 或 `#[serde(default)]` 保持兼容。
4. 对安全敏感配置谨慎使用 `Debug`，避免密钥出现在日志中。
5. 库 crate 通常只依赖 `serde`，是否选择 JSON 应交给调用者。
6. 只启用需要的 feature，降低编译时间和依赖数量。

## 7. 常见误区

- `serde` 不等于 JSON；JSON 由 `serde_json` 处理。
- `#[serde(default)]` 可能掩盖调用方漏传字段，应只用于确实可缺省的数据。
- 不要随意给公共协议字段改名，序列化名称也是接口契约的一部分。
- `flatten` 很方便，但大量使用会使字段冲突和版本演进更难理解。

## 总结

`serde` 负责类型与数据模型之间的转换规则。正确使用的关键是区分领域模型和传输模型，并把字段名称、默认值和兼容策略视为稳定的外部协议。
