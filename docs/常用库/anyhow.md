# Rust 常用库：`anyhow` 的特点与正确使用

[`anyhow`](https://docs.rs/anyhow/latest/anyhow/) 是 Rust 生态中常用的应用层错误处理库。它提供：

- `anyhow::Error`：可以封装多种具体错误的动态错误类型；
- `anyhow::Result<T>`：`Result<T, anyhow::Error>` 的简写；
- `Context`：为底层错误添加“正在做什么”的上下文；
- `anyhow!`、`bail!`、`ensure!`：方便构造和提前返回错误；
- 错误链遍历、根因查询、downcast 和 Backtrace 支持。

最重要的使用边界是：

```text
应用程序、命令行工具、服务内部组装层  -> 适合 anyhow
公共库、需要调用者匹配错误的 API      -> 优先具体错误类型
```

`anyhow` 的目标是让“传播和诊断错误”更方便，而不是取代所有自定义错误设计。

## 1. 添加依赖

在项目中添加依赖：

```bash
cargo add anyhow
```

或者在 `Cargo.toml` 中添加：

```toml
[dependencies]
anyhow = "1"
```

导入最常用的类型和 trait：

```rust
use anyhow::{Context, Result};

fn parse_number(text: &str) -> Result<u32> {
    let number = text.parse::<u32>()?;
    Ok(number)
}

assert_eq!(parse_number("42")?, 42);
# Ok::<(), anyhow::Error>(())
```

`Context` 是 trait。只有把它引入作用域，才能对 `Result` 和 `Option` 调用 `.context(...)` 或 `.with_context(...)`。

## 2. `anyhow::Result<T>` 是什么

`anyhow::Result<T>` 等价于：

```text
std::result::Result<T, anyhow::Error>
```

因此下面两个函数的返回类型表达相同含义：

```rust
use anyhow::Error;

fn first() -> anyhow::Result<u32> {
    Ok(42)
}

fn second() -> Result<u32, Error> {
    Ok(42)
}

assert_eq!(first()?, second()?);
# Ok::<(), anyhow::Error>(())
```

成功但没有额外返回数据时，使用 `Result<()>`：

```rust
use anyhow::Result;

fn validate_ready(ready: bool) -> Result<()> {
    if ready {
        Ok(())
    } else {
        Err(anyhow::anyhow!("not ready"))
    }
}

assert!(validate_ready(true).is_ok());
assert!(validate_ready(false).is_err());
```

### 2.1 `anyhow::Error` 的主要特点

`anyhow::Error` 是一个围绕动态错误的具体包装类型，行为类似线程安全的错误 trait object：

- 可以封装实现 `std::error::Error + Send + Sync + 'static` 的错误；
- 调用者不需要为函数涉及的每种底层错误建立大型枚举；
- 可以保留错误来源链；
- 可以附加多层上下文；
- 可以在运行时 downcast 回具体错误类型；
- 可以提供 Backtrace。

代价是函数签名只显示“可能返回某种错误”，无法像具体错误枚举一样让调用者通过类型系统穷尽匹配全部失败情况。

## 3. 使用 `?` 自动转换并传播错误

在返回 `anyhow::Result<T>` 的函数中，可以直接对不同错误类型使用 `?`：

```rust
use anyhow::Result;
use std::net::IpAddr;

fn parse_server(ip: &str, port: &str) -> Result<(IpAddr, u16)> {
    let ip = ip.parse::<IpAddr>()?;
    let port = port.parse::<u16>()?;
    Ok((ip, port))
}

let (ip, port) = parse_server("127.0.0.1", "8080")?;
assert_eq!(ip.to_string(), "127.0.0.1");
assert_eq!(port, 8080);
# Ok::<(), anyhow::Error>(())
```

这里两个 `parse` 分别可能产生不同的错误类型，但它们都可以被转换成 `anyhow::Error`。

`?` 的行为仍然与标准 `Result` 相同：

```text
Ok(value)?  -> 取出 value，继续执行
Err(error)? -> 将错误转换为 anyhow::Error，并立即返回
```

### 3.1 `main` 可以直接返回 `anyhow::Result<()>`

```rust
use anyhow::Result;

fn run() -> Result<()> {
    let number = "42".parse::<u32>()?;
    assert_eq!(number, 42);
    Ok(())
}

fn main() -> Result<()> {
    run()?;
    Ok(())
}
```

这非常适合命令行程序和服务启动入口。错误到达 `main` 时会以 Debug 形式输出，并导致非零退出状态。

## 4. 最重要的能力：添加上下文

只传播底层错误通常不够：

```text
No such file or directory (os error 2)
```

这条信息没有说明正在读取哪个文件，也没有说明读取文件的目的。`Context` 可以在保留底层错误的同时添加高层语义。

### 4.1 `context()`：添加固定上下文

```rust
use anyhow::{Context, Result};

fn read_missing_file() -> Result<String> {
    std::fs::read_to_string("file-that-does-not-exist.txt")
        .context("failed to read application configuration")
}

let error = read_missing_file().unwrap_err();
assert_eq!(error.to_string(), "failed to read application configuration");
assert!(format!("{error:#}").contains("No such file"));
```

外层上下文回答“哪一步失败了”，底层错误回答“为什么失败”。

### 4.2 `with_context()`：惰性构造上下文

需要在错误消息中包含变量时，优先使用 `with_context`：

```rust
use anyhow::{Context, Result};
use std::path::Path;

fn read_config(path: &Path) -> Result<String> {
    std::fs::read_to_string(path)
        .with_context(|| format!("failed to read config from {}", path.display()))
}

let path = Path::new("missing-config.toml");
let error = read_config(path).unwrap_err();

assert!(error.to_string().contains("missing-config.toml"));
```

闭包只在原结果为 `Err` 时执行。成功路径不会分配这条格式化字符串。

选择原则：

```text
固定、便宜的上下文                 -> context("...")
包含变量或构造成本较高的上下文       -> with_context(|| format!(...))
```

### 4.3 `Option` 也可以添加上下文

`Context` 也为 `Option<T>` 提供实现，可以把 `None` 变成 `anyhow::Error`：

```rust
use anyhow::{Context, Result};

fn first_name(names: &[String]) -> Result<&str> {
    names.first()
        .map(String::as_str)
        .context("expected at least one name")
}

let names = vec![String::from("Alice")];
assert_eq!(first_name(&names)?, "Alice");

let empty: Vec<String> = Vec::new();
assert_eq!(
    first_name(&empty).unwrap_err().to_string(),
    "expected at least one name"
);
# Ok::<(), anyhow::Error>(())
```

这相当于把“缺失值”提升为可传播且带诊断信息的应用错误。

### 4.4 在合适的抽象边界添加上下文

推荐在能够回答下列问题的位置添加上下文：

- 正在执行哪个业务步骤？
- 哪个文件、用户、请求或资源失败？
- 哪个输入值有问题？
- 操作者下一步需要检查什么？

避免每经过一层函数都机械添加重复信息：

```text
failed to run
caused by: failed to execute
caused by: operation failed
caused by: request failed
```

好的上下文应增加新信息，而不是换一种说法重复“失败”。

## 5. `anyhow!`：构造临时错误

`anyhow!` 可以从消息、格式化参数或已有错误构造 `anyhow::Error`：

```rust
use anyhow::{anyhow, Result};

fn find_user(id: u64) -> Result<String> {
    if id == 1 {
        Ok(String::from("Alice"))
    } else {
        Err(anyhow!("user {id} was not found"))
    }
}

assert_eq!(find_user(1)?, "Alice");
assert_eq!(find_user(2).unwrap_err().to_string(), "user 2 was not found");
# Ok::<(), anyhow::Error>(())
```

也可以把标准错误包装为 `anyhow::Error`：

```rust
use anyhow::anyhow;

let parse_error = "invalid".parse::<u32>().unwrap_err();
let error = anyhow!(parse_error);

assert!(error.to_string().contains("invalid digit"));
```

已有错误实现 `std::error::Error` 时，`anyhow!(error)` 会保留它的错误来源和类型信息。

### 5.1 `Error::msg`

需要函数而不是宏的场景可以使用 `anyhow::Error::msg`：

```rust
use anyhow::{Error, Result};

fn convert_error(result: Result<u32, &'static str>) -> Result<u32> {
    result.map_err(Error::msg)
}

assert_eq!(convert_error(Ok(42))?, 42);
assert_eq!(convert_error(Err("failed")).unwrap_err().to_string(), "failed");
# Ok::<(), anyhow::Error>(())
```

如果已有值实现标准错误 trait，优先通过 `?`、`Error::new` 或 `anyhow!(error)` 保留原错误，而不是先转成字符串。

## 6. `bail!`：立即返回错误

`bail!` 是 `return Err(anyhow!(...))` 的简写：

```rust
use anyhow::{bail, Result};

fn validate_age(age: u8) -> Result<()> {
    if age < 18 {
        bail!("age must be at least 18, got {age}");
    }

    Ok(())
}

assert!(validate_age(20).is_ok());
assert_eq!(
    validate_age(16).unwrap_err().to_string(),
    "age must be at least 18, got 16"
);
```

下面两种写法语义相同：

```rust
use anyhow::{anyhow, bail, Result};

fn first(valid: bool) -> Result<()> {
    if !valid {
        bail!("invalid input");
    }
    Ok(())
}

fn second(valid: bool) -> Result<()> {
    if !valid {
        return Err(anyhow!("invalid input"));
    }
    Ok(())
}

assert_eq!(first(false).unwrap_err().to_string(), second(false).unwrap_err().to_string());
```

## 7. `ensure!`：校验条件并返回错误

`ensure!` 类似不会 panic 的 `assert!`。条件为假时，它返回 `anyhow::Error`：

```rust
use anyhow::{ensure, Result};

fn calculate_percentage(value: u32, total: u32) -> Result<u32> {
    ensure!(total > 0, "total must be greater than zero");
    ensure!(value <= total, "value {value} exceeds total {total}");

    Ok(value * 100 / total)
}

assert_eq!(calculate_percentage(2, 4)?, 50);
assert!(calculate_percentage(1, 0).is_err());
assert!(calculate_percentage(5, 4).is_err());
# Ok::<(), anyhow::Error>(())
```

可以把它理解成：

```text
ensure!(condition, "message")

等价于

if !condition {
    return Err(anyhow!("message"));
}
```

区别于 `assert!`：

- `assert!` 条件失败时 panic，通常表示程序错误或不变量被破坏；
- `ensure!` 条件失败时返回 `Err`，适合无效输入等可恢复情况。

不要用 `ensure!` 代替所有分支逻辑。它适合简洁的前置条件检查。

## 8. 错误的显示方式

`anyhow::Error` 提供几种有不同用途的格式：

```rust
use anyhow::{Context, Result};

fn operation() -> Result<()> {
    "not-a-number"
        .parse::<u32>()
        .context("failed to parse worker count")?;
    Ok(())
}

let error = operation().unwrap_err();

let outer = format!("{error}");
let full_chain = format!("{error:#}");
let debug = format!("{error:?}");

assert_eq!(outer, "failed to parse worker count");
assert!(full_chain.contains("invalid digit"));
assert!(debug.contains("Caused by"));
```

常用格式的含义：

```text
{error}      只显示最外层错误或上下文
{error:#}    在一行中显示完整错误链
{error:?}    多行 Debug 输出，包含原因链和已捕获的 Backtrace
{error:#?}   结构化的漂亮 Debug 输出
```

返回 `Err` 给 `main` 时，Rust 使用 Debug 风格输出。

### 8.1 错误通常只记录一次

底层函数应该添加上下文并传播错误，最外层边界再决定如何记录或返回。每一层都记录同一个错误会产生重复日志。

Web 服务还应注意：详细的内部错误链适合服务端日志，不应原样返回客户端。对外返回稳定、经过脱敏的状态码和错误响应。

## 9. 检查错误链和根因

### 9.1 `chain()`

`chain()` 从最外层错误开始遍历到最底层原因：

```rust
use anyhow::{Context, Result};

fn operation() -> Result<()> {
    "invalid"
        .parse::<u32>()
        .context("unable to parse item count")?;
    Ok(())
}

let error = operation().unwrap_err();
let messages: Vec<String> = error.chain().map(ToString::to_string).collect();

assert_eq!(messages[0], "unable to parse item count");
assert!(messages.last().unwrap().contains("invalid digit"));
```

### 9.2 `root_cause()`

```rust
use anyhow::{Context, Result};

fn operation() -> Result<()> {
    "invalid".parse::<u32>().context("invalid configuration")?;
    Ok(())
}

let error = operation().unwrap_err();
assert!(error.root_cause().to_string().contains("invalid digit"));
```

`root_cause` 是错误链最底层的原因。诊断时有用，但业务决策通常不应依赖错误消息字符串。

## 10. Downcast 到具体错误类型

`anyhow::Error` 擦除了函数签名中的具体错误类型，但仍可在运行时检查或取回底层类型：

```rust
use anyhow::Result;
use std::num::ParseIntError;

fn parse_count(text: &str) -> Result<u32> {
    Ok(text.parse::<u32>()?)
}

let error = parse_count("invalid").unwrap_err();

assert!(error.is::<ParseIntError>());
assert!(error.downcast_ref::<ParseIntError>().is_some());
```

添加字符串上下文后，底层错误仍然可以 downcast：

```rust
use anyhow::{Context, Result};
use std::num::ParseIntError;

fn parse_count(text: &str) -> Result<u32> {
    text.parse::<u32>().context("failed to parse count")
}

let error = parse_count("invalid").unwrap_err();

assert!(error.downcast_ref::<ParseIntError>().is_some());
```

可用方式包括：

- `error.is::<E>()`：判断错误链中是否包含类型 `E`；
- `error.downcast_ref::<E>()`：借用具体错误；
- `error.downcast_mut::<E>()`：可变借用具体错误；
- `error.downcast::<E>()`：消费 `anyhow::Error` 并取得具体错误所有权。

### 10.1 不要让 downcast 变成主要控制流

如果调用者经常必须 downcast 并根据多个错误类型做业务决策，说明函数可能更应该返回一个具体错误枚举。Downcast 更适合：

- 应用最外层的少量特殊恢复；
- 兼容已有动态错误边界；
- 诊断、遥测或退出码选择。

## 11. Backtrace

当底层错误没有自己的 Backtrace 时，`anyhow::Error` 可以捕获 Backtrace。为了看到有意义的堆栈，需要设置环境变量：

```bash
# panic 和 anyhow 错误都显示 Backtrace
RUST_BACKTRACE=1 cargo run

# 只为错误启用 Backtrace
RUST_LIB_BACKTRACE=1 cargo run

# panic 启用，但错误禁用
RUST_BACKTRACE=1 RUST_LIB_BACKTRACE=0 cargo run
```

可以直接访问：

```rust
use anyhow::anyhow;

let error = anyhow!("operation failed");
let _backtrace = error.backtrace();
```

Backtrace 捕获和格式化有成本，生产环境是否启用应根据诊断需求和性能测量决定。Backtrace 是辅助诊断信息，不能替代清晰的上下文。

## 12. 与自定义错误和 `thiserror` 配合

常见实践是：

```text
可复用库 / 领域边界   -> 具体错误枚举，可用 thiserror 减少样板代码
应用组装 / main       -> anyhow::Result，统一传播并添加上下文
```

例如库代码返回稳定的具体错误：

```rust
use std::fmt;

#[derive(Debug, PartialEq, Eq)]
pub enum ParseUserError {
    EmptyName,
}

impl fmt::Display for ParseUserError {
    fn fmt(&self, formatter: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Self::EmptyName => write!(formatter, "name must not be empty"),
        }
    }
}

impl std::error::Error for ParseUserError {}

pub fn parse_user(name: &str) -> Result<String, ParseUserError> {
    if name.trim().is_empty() {
        Err(ParseUserError::EmptyName)
    } else {
        Ok(name.trim().to_owned())
    }
}

assert_eq!(parse_user(" "), Err(ParseUserError::EmptyName));
```

应用层可以直接通过 `?` 接收它并添加场景上下文：

```rust
use anyhow::{Context, Result};
use std::fmt;

#[derive(Debug)]
struct EmptyName;

impl fmt::Display for EmptyName {
    fn fmt(&self, formatter: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(formatter, "name must not be empty")
    }
}

impl std::error::Error for EmptyName {}

fn library_operation(name: &str) -> Result<String, EmptyName> {
    if name.trim().is_empty() {
        Err(EmptyName)
    } else {
        Ok(name.trim().to_owned())
    }
}

fn import_user(name: &str) -> Result<String> {
    library_operation(name).context("failed to import user")
}

let error = import_user(" ").unwrap_err();
assert_eq!(error.to_string(), "failed to import user");
assert!(error.downcast_ref::<EmptyName>().is_some());
```

`anyhow` 本身不提供 `derive(Error)`。如果希望自动实现具体错误类型的 `Display` 和 `Error`，可以单独选择 `thiserror`；两者用途互补，不是互相替代。

## 13. 在 Web 服务中的使用方式

Web 服务中可以在应用内部使用 `anyhow::Result`，但不建议让 HTTP handler 直接把内部错误字符串暴露给客户端。

推荐边界：

```text
repository / 外部客户端  -> 返回底层或具体错误
service / 应用流程       -> 使用 anyhow 添加业务上下文
HTTP handler             -> 记录完整错误，映射为稳定的 HTTP 响应
客户端                    -> 只收到安全、可预期的错误码和消息
```

示意代码：

```rust
use anyhow::{bail, Context, Result};

fn load_user(id: u64) -> Result<String> {
    if id == 0 {
        bail!("user id must not be zero");
    }

    lookup_user(id).with_context(|| format!("failed to load user {id}"))
}

fn lookup_user(id: u64) -> Result<String> {
    if id == 1 {
        Ok(String::from("Alice"))
    } else {
        bail!("user was not found")
    }
}

assert_eq!(load_user(1)?, "Alice");
assert!(format!("{:#}", load_user(2).unwrap_err()).contains("user 2"));
# Ok::<(), anyhow::Error>(())
```

真实 handler 应将不同业务情况映射为 400、404、409、500 等状态。如果这种映射依赖稳定的错误分类，应在领域边界使用具体错误枚举，而不是匹配 anyhow 的显示字符串。

## 14. 异步和多线程代码

`anyhow::Error` 要求被包装的错误满足 `Send + Sync + 'static`，因此通常适合 Tokio 等多线程异步运行时中跨任务传播。

```rust
use anyhow::{Context, Result};

async fn parse_async(text: String) -> Result<u32> {
    text.parse::<u32>()
        .with_context(|| format!("failed to parse {text:?}"))
}

# async fn test() -> Result<()> {
assert_eq!(parse_async(String::from("42")).await?, 42);
# Ok(())
# }
```

如果第三方错误包含非 `Send`、非 `Sync` 或非 `'static` 数据，不能直接转换为 `anyhow::Error`。需要在边界将其转换为拥有所有权、线程安全的错误信息或自定义错误类型。

## 15. `anyhow` 不适合解决什么

### 15.1 不替代输入建模

如果“缺失”是正常情况，应返回 `Option<T>`，而不是每次都构造错误。

### 15.2 不替代领域错误

调用者需要区分 `NotFound`、`Conflict`、`PermissionDenied` 时，具体错误枚举比错误字符串或广泛 downcast 更可靠。

### 15.3 不替代日志和遥测

`anyhow` 保存错误链，但不会自动记录请求 ID、trace、用户动作或指标。应在应用边界与 `tracing` 等工具配合。

### 15.4 不替代安全的错误响应

数据库语句、文件路径、内部主机名和令牌可能出现在错误上下文中。内部诊断信息不能直接展示给不可信客户端。

## 16. 常见错误清单

1. **在公共库所有 API 中都返回 `anyhow::Result`**：调用者无法可靠穷尽匹配错误，优先设计具体错误类型。
2. **只使用 `?`，从不添加上下文**：底层错误往往无法说明失败的业务步骤和资源。
3. **每层都添加“operation failed”**：上下文应提供新信息，而不是重复失败事实。
4. **用 `anyhow!(error.to_string())` 包装已有错误**：会丢失具体类型和来源链，应直接 `anyhow!(error)` 或使用 `?`。
5. **根据错误显示字符串做业务判断**：消息会变化，使用具体错误类型或稳定错误码。
6. **把 `bail!` 当成 panic**：它只是提前返回 `Err`，调用者仍可处理。
7. **把 `ensure!` 用于不可恢复的不变量**：程序内部 bug 通常应使用断言或重新设计类型。
8. **在多个层级重复记录同一错误**：底层添加上下文，最外层统一记录。
9. **把完整错误链返回 Web 客户端**：可能泄露内部实现和敏感数据。
10. **滥用 downcast 代替错误建模**：大量业务分支依赖 downcast 时，应改用具体错误枚举。
11. **认为 `anyhow::Error` 可以轻松克隆**：错误通常不应靠克隆传播，应通过所有权移动、引用或重新建模。
12. **认为 Backtrace 会自动解决诊断问题**：没有清晰上下文的堆栈通常仍难以理解。

## 17. 一个完整示例

下面的示例解析服务配置，综合使用 `Result`、`Context`、`ensure!`、`bail!` 和错误链：

```rust
use anyhow::{bail, ensure, Context, Result};

#[derive(Debug, PartialEq, Eq)]
struct ServerConfig {
    host: String,
    port: u16,
    workers: usize,
}

fn parse_config(input: &str) -> Result<ServerConfig> {
    let mut host = None;
    let mut port = None;
    let mut workers = None;

    for (line_number, raw_line) in input.lines().enumerate() {
        let line = raw_line.trim();
        if line.is_empty() || line.starts_with('#') {
            continue;
        }

        let (key, value) = line
            .split_once('=')
            .with_context(|| format!("invalid config syntax on line {}", line_number + 1))?;

        let key = key.trim();
        let value = value.trim();

        match key {
            "host" => host = Some(value.to_owned()),
            "port" => {
                port = Some(
                    value
                        .parse::<u16>()
                        .with_context(|| format!("invalid port {value:?}"))?,
                );
            }
            "workers" => {
                workers = Some(
                    value
                        .parse::<usize>()
                        .with_context(|| format!("invalid worker count {value:?}"))?,
                );
            }
            unknown => bail!("unknown config key {unknown:?}"),
        }
    }

    let host = host.context("missing required key `host`")?;
    let port = port.context("missing required key `port`")?;
    let workers = workers.unwrap_or(1);

    ensure!(!host.is_empty(), "host must not be empty");
    ensure!(port > 0, "port must be greater than zero");
    ensure!(workers > 0, "workers must be greater than zero");

    Ok(ServerConfig {
        host,
        port,
        workers,
    })
}

fn main() -> Result<()> {
    let config = parse_config(
        r#"
        host = 127.0.0.1
        port = 8080
        workers = 4
        "#,
    )?;

    assert_eq!(
        config,
        ServerConfig {
            host: String::from("127.0.0.1"),
            port: 8080,
            workers: 4,
        }
    );

    let invalid_port = parse_config("host=localhost\nport=abc").unwrap_err();
    assert_eq!(invalid_port.to_string(), "invalid port \"abc\"");
    assert!(format!("{invalid_port:#}").contains("invalid digit"));

    let missing_host = parse_config("port=8080").unwrap_err();
    assert_eq!(missing_host.to_string(), "missing required key `host`");

    let unknown_key = parse_config("host=localhost\nport=8080\nmode=dev").unwrap_err();
    assert_eq!(unknown_key.to_string(), "unknown config key \"mode\"");

    Ok(())
}
```

这个示例体现了推荐实践：

- 用 `?` 传播解析错误；
- 在知道行号、字段名和输入值的位置添加上下文；
- 用 `Option::context` 把缺失字段转换为错误；
- 用 `ensure!` 表达可恢复的输入约束；
- 用 `bail!` 处理未知字段并提前返回；
- 外层消息保持简洁，底层原因仍保留在错误链中。

## 18. 推荐的应用分层方式

一个实际项目可以这样划分：

```text
domain / reusable library
    返回具体 Error 枚举
           │
           ▼
application service
    用 anyhow::Context 添加业务上下文
           │
           ▼
main / CLI / HTTP boundary
    统一记录、选择退出码或映射 HTTP 响应
```

示意：

```rust
use anyhow::{Context, Result};
use std::fmt;

#[derive(Debug)]
struct DomainError;

impl fmt::Display for DomainError {
    fn fmt(&self, formatter: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(formatter, "domain rule rejected the operation")
    }
}

impl std::error::Error for DomainError {}

fn domain_operation() -> Result<(), DomainError> {
    Err(DomainError)
}

fn application_operation() -> Result<()> {
    domain_operation().context("failed to create order")
}

let error = application_operation().unwrap_err();
assert_eq!(error.to_string(), "failed to create order");
assert!(error.downcast_ref::<DomainError>().is_some());
```

## 19. 快速选择指南

```text
函数属于应用内部，可能产生多种错误       -> anyhow::Result<T>
只需创建一条临时错误                    -> anyhow!(...)
条件不满足时立即返回                    -> bail!(...)
前置条件不满足时返回                    -> ensure!(condition, ...)
给固定错误添加说明                      -> .context("...")
上下文依赖变量或构造昂贵                -> .with_context(|| ...)
需要检查底层具体类型                    -> downcast_ref::<E>()
调用者需要稳定匹配所有错误              -> 具体错误枚举，不以 anyhow 为公共边界
```

## 20. 参考资料

- [`anyhow` 官方 API 文档](https://docs.rs/anyhow/latest/anyhow/)
- [`anyhow::Error` 文档](https://docs.rs/anyhow/latest/anyhow/struct.Error.html)
- [`Context` trait 文档](https://docs.rs/anyhow/latest/anyhow/trait.Context.html)
- [`anyhow` 官方仓库](https://github.com/dtolnay/anyhow)

## 总结

正确使用 `anyhow` 的关键原则是：

- 在应用层使用 `anyhow::Result<T>` 统一传播不同来源的错误；
- 不要只写 `?`，还要在有业务信息的边界添加 `context`；
- 固定上下文用 `context`，动态或昂贵上下文用 `with_context`；
- `anyhow!` 构造错误，`bail!` 提前返回，`ensure!` 校验可恢复条件；
- 保留错误链和具体错误类型，不要过早转成字符串；
- 最外层负责记录，内部层主要负责添加上下文并传播；
- 公共库和需要精确恢复的领域边界优先返回具体错误类型；
- Web 服务不能把内部错误链原样泄露给客户端；
- Backtrace 和 downcast 是诊断与兼容工具，不应替代良好的错误建模。

`anyhow` 最擅长的事情，是让应用代码用很少的样板代码保留一条有意义的错误因果链。
