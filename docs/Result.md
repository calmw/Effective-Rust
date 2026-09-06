# Rust 中的 `Result`：可恢复错误处理与正确使用

`Result<T, E>` 是 Rust 标准库用于表示“操作可能成功，也可能失败”的枚举：

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

- `Ok(value)` 表示操作成功，并携带类型为 `T` 的结果；
- `Err(error)` 表示操作失败，并携带类型为 `E` 的错误。

```rust
fn divide(left: f64, right: f64) -> Result<f64, &'static str> {
    if right == 0.0 {
        Err("除数不能为零")
    } else {
        Ok(left / right)
    }
}

assert_eq!(divide(10.0, 2.0), Ok(5.0));
assert_eq!(divide(10.0, 0.0), Err("除数不能为零"));
```

与 panic 不同，`Result` 表达的是调用者可能处理、恢复或继续传播的错误。

## 1. `Result<T, E>` 的类型含义

在 `Result<T, E>` 中，成功值和错误值可以是完全不同的类型：

```rust
let success: Result<u32, String> = Ok(42);
let failure: Result<u32, String> = Err(String::from("无效数字"));

assert_eq!(success, Ok(42));
assert_eq!(failure, Err(String::from("无效数字")));
```

单独书写 `Ok` 或 `Err` 时，编译器有时无法推断另一侧的类型，需要显式标注：

```rust
let success = Ok::<u32, String>(42);
let failure = Err::<u32, String>(String::from("failed"));

assert!(success.is_ok());
assert!(failure.is_err());
```

`Result` 被标记为 `#[must_use]`。忽略返回的 `Result` 通常会产生编译器警告，因为失败可能被悄悄丢弃。

## 2. 使用 `match` 处理成功与失败

`match` 是最完整的处理方式：

```rust
fn parse_port(text: &str) -> String {
    match text.parse::<u16>() {
        Ok(port) => format!("端口是 {port}"),
        Err(error) => format!("端口无效：{error}"),
    }
}

assert_eq!(parse_port("8080"), "端口是 8080");
assert!(parse_port("invalid").starts_with("端口无效："));
```

`match` 的穷尽性检查会确保 `Ok` 和 `Err` 都被处理。

只关心其中一个分支时，可以使用 `if let`：

```rust
let result = "42".parse::<u32>();

if let Ok(value) = result {
    assert_eq!(value, 42);
}
```

需要成功值才能继续时，`let ... else` 可以尽早返回：

```rust
fn double_number(text: &str) -> Result<u32, String> {
    let Ok(number) = text.parse::<u32>() else {
        return Err(format!("无法解析数字：{text}"));
    };

    Ok(number * 2)
}

assert_eq!(double_number("21"), Ok(42));
assert_eq!(double_number("x"), Err(String::from("无法解析数字：x")));
```

## 3. 判断和查看结果

### 3.1 `is_ok`、`is_err` 与条件判断

```rust
let success = "42".parse::<u32>();
let failure = "x".parse::<u32>();

assert!(success.is_ok());
assert!(failure.is_err());
assert!(success.is_ok_and(|value| value > 40));
```

如果判断后还需要内部值，通常直接使用模式匹配或组合子，避免先判断再提取。

### 3.2 `as_ref` 和 `as_mut`

`as_ref` 把 `Result<T, E>` 借用为 `Result<&T, &E>`，不会消费原值：

```rust
let result: Result<String, String> = Ok(String::from("Alice"));
let length = result.as_ref().map(String::len);

assert_eq!(length, Ok(5));
assert_eq!(result, Ok(String::from("Alice")));
```

`as_mut` 可以取得成功值或错误值的可变引用：

```rust
let mut result: Result<String, String> = Ok(String::from("Alice"));

if let Ok(name) = result.as_mut() {
    name.push_str(" Chen");
}

assert_eq!(result, Ok(String::from("Alice Chen")));
```

## 4. 使用 `?` 传播错误

`?` 是 Rust 错误处理的核心语法。对 `Result` 使用 `?` 时：

- 遇到 `Ok(value)`：取出 `value` 并继续执行；
- 遇到 `Err(error)`：立即从当前函数返回相应错误。

```rust
fn parse_and_add(left: &str, right: &str) -> Result<i32, std::num::ParseIntError> {
    let left_number = left.parse::<i32>()?;
    let right_number = right.parse::<i32>()?;

    Ok(left_number + right_number)
}

assert_eq!(parse_and_add("20", "22"), Ok(42));
assert!(parse_and_add("twenty", "22").is_err());
```

第一处 `?` 大致等价于：

```rust
fn parse_number(text: &str) -> Result<i32, std::num::ParseIntError> {
    let number = match text.parse::<i32>() {
        Ok(value) => value,
        Err(error) => return Err(error),
    };

    Ok(number)
}

assert_eq!(parse_number("42"), Ok(42));
```

### 4.1 `?` 只能在兼容的返回环境中使用

通常，处理 `Result` 的 `?` 所在函数也需要返回 `Result`：

```rust,compile_fail
fn invalid() -> i32 {
    let number = "42".parse::<i32>()?;
    number
}
```

`main` 也可以返回 `Result`：

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let number = "42".parse::<u32>()?;
    assert_eq!(number, 42);
    Ok(())
}
```

### 4.2 `?` 可以通过 `From` 转换错误

错误类型不完全相同时，只要目标错误实现了相应的 `From` 转换，`?` 就会自动转换。这也是自定义错误通常要实现 `From` 的原因。

`?` 不会自动添加业务上下文。如果底层错误不足以定位问题，应先使用 `map_err` 转换或包装错误，再传播。

## 5. 提取成功值

### 5.1 `unwrap` 与 `expect`

`unwrap()` 在 `Ok` 时返回成功值，在 `Err` 时 panic：

```rust
let value = "42".parse::<u32>().unwrap();
assert_eq!(value, 42);
```

`expect()` 行为相同，但可以提供说明程序不变量的消息：

```rust
let port = "8080"
    .parse::<u16>()
    .expect("硬编码端口应当是合法的 u16");

assert_eq!(port, 8080);
```

不要对用户输入、文件内容、网络响应等正常可能失败的数据随意使用 `unwrap`。它更适合测试、快速原型以及逻辑上不可能失败的内部不变量。

### 5.2 `unwrap_or` 与 `unwrap_or_else`

错误时使用简单默认值：

```rust
let port = "invalid".parse::<u16>().unwrap_or(8080);
assert_eq!(port, 8080);
```

默认值需要计算时使用 `unwrap_or_else`。与 `Option::unwrap_or_else` 不同，`Result::unwrap_or_else` 的闭包会接收到错误值：

```rust
let port = "invalid".parse::<u16>().unwrap_or_else(|error| {
    eprintln!("端口解析失败：{error}");
    8080
});

assert_eq!(port, 8080);
```

`unwrap_or` 的默认值表达式总会先求值；只有在发生 `Err` 时才需要计算默认值，应使用惰性的 `unwrap_or_else`。

## 6. 转换成功值与错误值

### 6.1 `map` 转换 `Ok`

`map` 只在结果为 `Ok` 时执行闭包：

```rust
let length = Ok::<String, &str>(String::from("Alice"))
    .map(|name| name.len());

assert_eq!(length, Ok(5));
```

遇到 `Err` 时，错误原样保留，闭包不会执行：

```rust
let result: Result<String, &str> = Err("missing");
let length = result.map(|name| name.len());

assert_eq!(length, Err("missing"));
```

### 6.2 `map_err` 转换 `Err`

```rust
#[derive(Debug, PartialEq, Eq)]
struct InvalidPort(String);

fn parse_port(text: &str) -> Result<u16, InvalidPort> {
    text.parse::<u16>()
        .map_err(|_error| InvalidPort(text.to_owned()))
}

assert_eq!(parse_port("8080"), Ok(8080));
assert_eq!(parse_port("x"), Err(InvalidPort(String::from("x"))));
```

`map_err` 只转换错误，不改变成功值。它适合将底层错误变成领域错误，或者补充与当前操作有关的上下文。

### 6.3 `map_or` 与 `map_or_else`

`map_or` 在成功时转换值，在失败时返回默认值，最终不再返回 `Result`：

```rust
let valid_length = "42".parse::<u32>().map_or(0, |value| value.to_string().len());
let invalid_length = "x".parse::<u32>().map_or(0, |value| value.to_string().len());

assert_eq!(valid_length, 2);
assert_eq!(invalid_length, 0);
```

需要根据错误惰性生成默认值时使用 `map_or_else`：

```rust
let value = "x".parse::<u32>().map_or_else(
    |_error| 0,
    |number| number * 2,
);

assert_eq!(value, 0);
```

## 7. 串联多个可能失败的操作

### 7.1 优先使用 `?`

过程式业务逻辑中，`?` 通常最容易阅读：

```rust
fn positive_number(text: &str) -> Result<u32, &'static str> {
    let number = text.parse::<u32>().map_err(|_error| "不是有效数字")?;

    if number == 0 {
        return Err("数字必须大于零");
    }

    Ok(number)
}

assert_eq!(positive_number("42"), Ok(42));
assert_eq!(positive_number("0"), Err("数字必须大于零"));
```

### 7.2 `and_then` 组合返回 `Result` 的函数

```rust
fn ensure_positive(number: i32) -> Result<i32, &'static str> {
    if number > 0 {
        Ok(number)
    } else {
        Err("必须为正数")
    }
}

let valid = "42"
    .parse::<i32>()
    .map_err(|_error| "解析失败")
    .and_then(ensure_positive);

assert_eq!(valid, Ok(42));
```

`map` 的闭包返回普通值，`and_then` 的闭包返回另一个 `Result`：

```text
map:      T -> U             得到 Result<U, E>
and_then: T -> Result<U, E>  得到 Result<U, E>
```

### 7.3 `or_else` 恢复或转换错误

```rust
fn fallback(error: &str) -> Result<u32, String> {
    Ok(if error == "not configured" { 8080 } else { 0 })
}

let result = Err::<u32, &str>("not configured").or_else(fallback);
assert_eq!(result, Ok(8080));
```

复杂链条如果难以阅读，应改用中间变量、`match` 和 `?`，而不是追求最长的方法调用链。

## 8. `Result` 与 `Option` 的转换

### 8.1 `ok()` 与 `err()`

```rust
let success: Result<u32, &str> = Ok(42);
let failure: Result<u32, &str> = Err("failed");

assert_eq!(success.ok(), Some(42));
assert_eq!(failure.err(), Some("failed"));
```

`ok()` 会丢弃错误信息，`err()` 会丢弃成功值。只有确实不需要被丢弃的信息时才使用。

### 8.2 `Option::ok_or`

当缺失需要变成明确错误时：

```rust
fn require_name(name: Option<&str>) -> Result<&str, &'static str> {
    name.ok_or("缺少名称")
}

assert_eq!(require_name(Some("Alice")), Ok("Alice"));
assert_eq!(require_name(None), Err("缺少名称"));
```

### 8.3 `transpose`

`Option<Result<T, E>>` 可以转换成 `Result<Option<T>, E>`：

```rust
fn parse_optional(text: Option<&str>) -> Result<Option<u32>, std::num::ParseIntError> {
    text.map(str::parse::<u32>).transpose()
}

assert_eq!(parse_optional(Some("42")), Ok(Some(42)));
assert_eq!(parse_optional(None), Ok(None));
assert!(parse_optional(Some("x")).is_err());
```

适用语义是：输入可以缺失，但只要提供了就必须正确。

## 9. 批量收集 `Result`

迭代器产生 `Result<T, E>` 时，可以收集为 `Result<Vec<T>, E>`。所有元素成功才得到 `Ok(Vec<T>)`，遇到第一个错误就停止：

```rust
fn parse_all(inputs: &[&str]) -> Result<Vec<u32>, std::num::ParseIntError> {
    inputs.iter().map(|text| text.parse::<u32>()).collect()
}

assert_eq!(parse_all(&["1", "2", "3"]), Ok(vec![1, 2, 3]));
assert!(parse_all(&["1", "x", "3"]).is_err());
```

如果业务需要收集全部错误，标准的 `collect::<Result<_, _>>()` 不够，需要显式遍历并分别积累成功值和错误。

## 10. 设计自定义错误类型

库和较大型应用通常应定义有语义的错误类型，而不只是返回字符串：

```rust
use std::fmt;

#[derive(Debug, PartialEq, Eq)]
enum UserError {
    EmptyName,
    InvalidAge(String),
    UnderAge(u8),
}

impl fmt::Display for UserError {
    fn fmt(&self, formatter: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            UserError::EmptyName => write!(formatter, "用户名不能为空"),
            UserError::InvalidAge(value) => write!(formatter, "年龄无效：{value}"),
            UserError::UnderAge(age) => write!(formatter, "年龄 {age} 小于最低要求"),
        }
    }
}

impl std::error::Error for UserError {}

assert_eq!(UserError::EmptyName.to_string(), "用户名不能为空");
```

错误设计建议：

- 用枚举变体区分调用者可能采取不同措施的错误；
- 保存诊断和恢复所需的信息；
- 实现 `Debug`、`Display` 和 `std::error::Error`；
- 包装底层错误时保留错误来源；
- 不要把密码、令牌等敏感数据放入错误消息；
- 公共 API 优先返回稳定的具体错误类型。

### 10.1 使用 `From` 配合 `?`

```rust
use std::num::ParseIntError;

#[derive(Debug, PartialEq, Eq)]
enum ConfigError {
    InvalidNumber,
    ZeroNotAllowed,
}

impl From<ParseIntError> for ConfigError {
    fn from(_error: ParseIntError) -> Self {
        ConfigError::InvalidNumber
    }
}

fn parse_limit(text: &str) -> Result<u32, ConfigError> {
    let value = text.parse::<u32>()?;
    if value == 0 {
        Err(ConfigError::ZeroNotAllowed)
    } else {
        Ok(value)
    }
}

assert_eq!(parse_limit("10"), Ok(10));
assert_eq!(parse_limit("x"), Err(ConfigError::InvalidNumber));
```

`?` 遇到 `ParseIntError` 时，会调用 `ConfigError::from` 转换成函数声明的错误类型。

## 11. `Box<dyn Error>` 与具体错误类型

在小型应用、示例或程序入口中，`Box<dyn std::error::Error>` 可以方便地统一多种错误：

```rust
fn calculate(text: &str) -> Result<u32, Box<dyn std::error::Error>> {
    let number = text.parse::<u32>()?;
    Ok(number * 2)
}

assert_eq!(calculate("21").unwrap(), 42);
assert!(calculate("x").is_err());
```

它的优点是组合方便，缺点是调用者难以通过类型穷尽匹配具体错误。在公共库 API 或需要精确恢复策略时，通常应使用具体错误枚举。

## 12. panic 还是 `Result`

适合返回 `Result`：

- 文件不存在、网络超时、输入格式错误等可预期失败；
- 调用者可能重试、使用默认值或向用户报告；
- 库无法替最终应用决定如何恢复。

可能适合 panic：

- 数组越界等程序错误；
- 内部不变量被破坏，继续执行可能不安全；
- 测试中需要快速暴露错误。

不要用 panic 代替正常的输入校验，也不要在本应不可失败的不变量上层层传播无意义的错误。边界需要根据 API 契约明确决定。

## 13. 常见错误清单

1. **忽略 `Result`**：显式处理、传播，或者明确写 `let _ = ...` 表达有意忽略。
2. **对外部输入随意 `unwrap()`**：使用 `?`、`match` 或合适的默认策略。
3. **错误类型只有模糊字符串**：调用者需要区分情况时使用错误枚举。
4. **使用 `.ok()` 丢弃关键错误**：只在错误原因确实无关时转换为 `Option`。
5. **用 `map` 串联返回 `Result` 的函数**：会产生嵌套，使用 `and_then` 或 `?`。
6. **认为 `?` 会自动补充上下文**：需要时先用 `map_err` 包装错误。
7. **错误中泄露敏感信息**：输出和日志中的错误内容应经过设计。
8. **为了函数签名简单而统一返回 `Box<dyn Error>`**：公共 API 可能更适合具体错误类型。
9. **用错误控制普通循环流程**：正常缺失通常更适合 `Option`，普通分支直接使用控制流。

## 14. 一个完整示例

下面的例子解析用户记录，展示输入校验、错误转换、`?` 和自定义错误：

```rust
use std::fmt;
use std::num::ParseIntError;

#[derive(Debug, PartialEq, Eq)]
struct User {
    name: String,
    age: u8,
}

#[derive(Debug, PartialEq, Eq)]
enum ParseUserError {
    MissingSeparator,
    EmptyName,
    InvalidAge,
    UnderAge(u8),
}

impl From<ParseIntError> for ParseUserError {
    fn from(_error: ParseIntError) -> Self {
        ParseUserError::InvalidAge
    }
}

impl fmt::Display for ParseUserError {
    fn fmt(&self, formatter: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ParseUserError::MissingSeparator => write!(formatter, "缺少逗号"),
            ParseUserError::EmptyName => write!(formatter, "姓名不能为空"),
            ParseUserError::InvalidAge => write!(formatter, "年龄格式无效"),
            ParseUserError::UnderAge(age) => write!(formatter, "年龄 {age} 未满 18 岁"),
        }
    }
}

impl std::error::Error for ParseUserError {}

fn parse_user(input: &str) -> Result<User, ParseUserError> {
    let (name_text, age_text) = input
        .split_once(',')
        .ok_or(ParseUserError::MissingSeparator)?;

    let name = name_text.trim();
    if name.is_empty() {
        return Err(ParseUserError::EmptyName);
    }

    let age = age_text.trim().parse::<u8>()?;
    if age < 18 {
        return Err(ParseUserError::UnderAge(age));
    }

    Ok(User {
        name: name.to_owned(),
        age,
    })
}

fn main() {
    assert_eq!(
        parse_user(" Alice, 20 "),
        Ok(User {
            name: String::from("Alice"),
            age: 20,
        })
    );
    assert_eq!(parse_user("Alice"), Err(ParseUserError::MissingSeparator));
    assert_eq!(parse_user(",20"), Err(ParseUserError::EmptyName));
    assert_eq!(parse_user("Alice,x"), Err(ParseUserError::InvalidAge));
    assert_eq!(parse_user("Alice,16"), Err(ParseUserError::UnderAge(16)));
}
```

## 总结

正确使用 `Result` 的核心原则是：

- 用 `Ok` 表示成功，用 `Err` 表示可恢复失败；
- 调用者需要恢复或记录失败时返回 `Result`，程序不变量被破坏时才考虑 panic；
- 使用 `?` 简洁传播错误，使用 `map_err` 转换并补充语义；
- 使用 `map` 转换成功值，使用 `and_then` 串联可能失败的操作；
- 谨慎使用 `unwrap` 和 `expect`，不要让正常错误变成崩溃；
- 设计包含有效上下文、可供调用者匹配且不会泄露敏感信息的错误类型；
- 批量操作可以收集成 `Result<Vec<T>, E>`，但要明确是快速失败还是收集全部错误。

把失败放进返回类型中，可以让错误路径与成功路径一样明确、可组合并受到编译器检查。
