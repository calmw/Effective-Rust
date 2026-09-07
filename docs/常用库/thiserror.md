# Rust 常用库：`thiserror`

[`thiserror`](https://docs.rs/thiserror/latest/thiserror/) 为实现 `std::error::Error` 提供派生宏，适合公共库、领域层以及需要调用者匹配错误种类的 API。

## 1. 添加依赖

```bash
cargo add thiserror
```

## 2. 定义错误枚举

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum ConfigError {
    #[error("failed to read config from {path}")]
    Read {
        path: String,
        #[source]
        source: std::io::Error,
    },

    #[error("invalid port: {0}")]
    InvalidPort(#[from] std::num::ParseIntError),

    #[error("missing field `{0}`")]
    MissingField(&'static str),
}
```

- `#[error("...")]` 实现 `Display`；
- `#[from]` 自动实现 `From<底层错误>`，同时将字段作为错误来源；
- `#[source]` 显式标记底层错误；
- 名为 `source` 的字段会自动作为错误来源。

## 3. 使用 `?` 转换错误

```rust
use thiserror::Error;

#[derive(Debug, Error)]
enum AppError {
    #[error("invalid number")]
    Parse(#[from] std::num::ParseIntError),
}

fn parse_port(text: &str) -> Result<u16, AppError> {
    Ok(text.parse()?)
}
```

因为派生了 `From<ParseIntError>`，`?` 可以自动把底层错误转换成 `AppError`。

## 4. 与 `anyhow` 的分工

```text
公共库、领域层、调用者需要匹配错误  -> thiserror
应用入口、任务编排、只需报告失败    -> anyhow
```

二者可以一起使用：底层模块返回 `thiserror` 错误，应用层用 `anyhow::Context` 添加执行上下文。

## 5. 最佳实践

1. 错误变体表达调用者能够采取的不同处理方式，而不是每个失败位置都建一个变体。
2. 保存底层错误为 `source`，不要只保存格式化后的字符串。
3. 公共错误类型使用 `pub`，内部字段是否公开则按 API 需要决定。
4. 错误消息应说明发生了什么；日志上下文说明当时正在执行什么。
5. 不要在错误的 `Display` 中泄露密码、令牌或 SQL 参数。
6. 对可能演进的库错误枚举，可考虑 `#[non_exhaustive]`。

## 6. 常见误区

- `thiserror` 不负责记录日志，也不会自动捕获所有上下文。
- 不要同时在底层记录错误又向上返回，避免同一个错误被重复记录。
- `#[from]` 适用于同一种底层错误只对应一个变体的情况；需要额外字段时应手动 `map_err`。
- 不要为了省事把所有变体都写成 `String`，这会丢失错误链和类型信息。

## 总结

`thiserror` 让自定义错误保持标准、类型安全且易于维护。它负责定义稳定的错误契约，`anyhow` 则更适合在应用边界汇总和报告这些错误。
