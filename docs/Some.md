# Rust 中的 `Some`：理解并正确使用 `Option<T>`

`Some` 不是一个独立类型，而是标准库枚举 `Option<T>` 的一个变体。`Option<T>` 用来表示“一个值可能存在，也可能不存在”：

```rust
enum Option<T> {
    None,
    Some(T),
}
```

- `Some(value)` 表示值存在，并在其中保存一个 `T`；
- `None` 表示值不存在。

例如，查找用户时可能找到一个用户，也可能找不到：

```rust
fn find_username(id: u64) -> Option<String> {
    if id == 1 {
        Some(String::from("Alice"))
    } else {
        None
    }
}

assert_eq!(find_username(1), Some(String::from("Alice")));
assert_eq!(find_username(2), None);
```

Rust 没有普通的 `null` 值。通过 `Option<T>`，调用者必须显式考虑值缺失的情况，从而在编译阶段避免许多空指针错误。

## 1. 创建 `Some`

直接用 `Some(value)` 包装一个值：

```rust
let port: Option<u16> = Some(8080);
let username = Some(String::from("Alice"));

assert_eq!(port, Some(8080));
assert_eq!(username.as_deref(), Some("Alice"));
```

多数情况下，编译器可以根据被包装的值或使用位置推断 `T`。对于单独的 `None`，通常需要提供类型信息，因为 `None` 本身没有告诉编译器缺失的是哪种值：

```rust
let missing_port: Option<u16> = None;
let missing_name = None::<String>;

assert!(missing_port.is_none());
assert!(missing_name.is_none());
```

需要特别注意：`Some` 只表示“存在”，不会判断内部值在业务上是否有效。

```rust
let enabled = Some(false);
let count = Some(0);
let name = Some("");

assert!(enabled.is_some());
assert!(count.is_some());
assert!(name.is_some());
```

`Some(false)`、`Some(0)` 和 `Some("")` 都不是 `None`。如果这些值在业务上应视为缺失，需要显式转换或重新设计数据模型。

## 2. 使用模式匹配处理 `Some` 和 `None`

### 2.1 `match`：完整处理所有情况

`match` 是最明确、最完整的处理方式：

```rust
fn describe_score(score: Option<u32>) -> String {
    match score {
        Some(value) => format!("分数是 {value}"),
        None => String::from("暂无分数"),
    }
}

assert_eq!(describe_score(Some(95)), "分数是 95");
assert_eq!(describe_score(None), "暂无分数");
```

`Option<T>` 只有 `Some` 和 `None` 两种情况。`match` 的穷尽性检查会确保两者都被处理。

还可以在模式中加入条件守卫：

```rust
fn classify(value: Option<i32>) -> &'static str {
    match value {
        Some(number) if number > 0 => "正数",
        Some(0) => "零",
        Some(_) => "负数",
        None => "不存在",
    }
}

assert_eq!(classify(Some(3)), "正数");
assert_eq!(classify(None), "不存在");
```

### 2.2 `if let`：只关心值存在的情况

如果只需要处理 `Some`，使用 `if let` 通常比完整的 `match` 更简洁：

```rust
let username = Some("Alice");

if let Some(name) = username {
    println!("欢迎，{name}");
}
```

也可以添加 `else`：

```rust
let username: Option<&str> = None;

if let Some(name) = username {
    println!("欢迎，{name}");
} else {
    println!("请先登录");
}
```

### 2.3 `let ... else`：缺失时尽早退出

当后续逻辑需要直接使用内部值时，`let ... else` 很适合进行前置检查：

```rust
fn send_welcome(username: Option<&str>) -> String {
    let Some(name) = username else {
        return String::from("没有可用的用户名");
    };

    format!("欢迎，{name}")
}

assert_eq!(send_welcome(Some("Alice")), "欢迎，Alice");
assert_eq!(send_welcome(None), "没有可用的用户名");
```

`else` 分支必须离开当前控制流，例如执行 `return`、`break`、`continue` 或 panic。

### 2.4 `while let`：反复处理存在的值

当某个操作不断返回 `Some`，直到返回 `None` 时，可以使用 `while let`：

```rust
let mut stack = vec![1, 2, 3];
let mut result = Vec::new();

while let Some(value) = stack.pop() {
    result.push(value);
}

assert_eq!(result, vec![3, 2, 1]);
```

`Vec::pop()` 在向量非空时返回 `Some(T)`，为空时返回 `None`。

## 3. 判断值是否存在

使用 `is_some()` 和 `is_none()` 进行简单判断：

```rust
let value = Some(42);

assert!(value.is_some());
assert!(!value.is_none());
```

如果判断存在的同时还要检查内部值，可以使用 `is_some_and`：

```rust
let score = Some(95);

assert!(score.is_some_and(|value| value >= 60));
assert!(!None::<u32>.is_some_and(|value| value >= 60));
```

但如果判断之后还要再次取值，通常直接使用 `if let`、`match` 或组合子更合适，避免把一次处理拆成多个步骤。

## 4. 安全地取得内部值

### 4.1 优先通过匹配处理缺失情况

最通用的方式仍然是 `match`、`if let` 或 `let ... else`，因为代码会明确说明 `None` 应如何处理。

### 4.2 提供默认值

`unwrap_or` 在 `None` 时返回给定的默认值：

```rust
let configured_port = Some(8080);
let missing_port: Option<u16> = None;

assert_eq!(configured_port.unwrap_or(3000), 8080);
assert_eq!(missing_port.unwrap_or(3000), 3000);
```

如果默认值需要计算，使用 `unwrap_or_else` 延迟执行：

```rust
fn default_name() -> String {
    String::from("guest")
}

let username: Option<String> = None;
let name = username.unwrap_or_else(default_name);

assert_eq!(name, "guest");
```

区别在于：传给 `unwrap_or` 的表达式总会先被求值，而 `unwrap_or_else` 的闭包只在值为 `None` 时调用。默认值构造昂贵或有副作用时，应使用 `unwrap_or_else`。

对于实现了 `Default` 的类型，还可以使用 `unwrap_or_default`：

```rust
let tags: Option<Vec<String>> = None;
assert_eq!(tags.unwrap_or_default(), Vec::<String>::new());
```

### 4.3 谨慎使用 `unwrap` 和 `expect`

`unwrap()` 在 `Some` 时返回内部值，在 `None` 时 panic：

```rust
let value = Some(42);
assert_eq!(value.unwrap(), 42);
```

生产代码中不要仅仅为了省事而使用 `unwrap`。当 `None` 代表正常输入、外部错误或可恢复情况时，应显式处理或传播。

只有在程序逻辑能够保证值存在时才适合使用 `expect`，并在消息中说明为什么这个不变量理应成立：

```rust
let values = [10, 20, 30];
let first = values.first().expect("固定数组一定包含首个元素");

assert_eq!(*first, 10);
```

好的 `expect` 消息描述应当成立的不变量，而不只是写“获取失败”。测试、原型和不可恢复的程序内部错误中，使用 `unwrap` 或 `expect` 通常更合理。

## 5. 使用组合子转换 `Some`

`Option` 的组合子可以把“存在时执行，缺失时跳过”的逻辑写得紧凑而清晰。

### 5.1 `map`：转换内部值

`map` 只在值为 `Some` 时调用闭包，并把结果重新包装为 `Some`：

```rust
let name = Some(String::from("Alice"));
let length = name.map(|value| value.len());

assert_eq!(length, Some(5));
```

如果原值为 `None`，闭包不会运行，结果仍然是 `None`。

可以使用 `map_or` 同时提供缺失时的默认结果：

```rust
let name = Some("Alice");
let length = name.map_or(0, str::len);

assert_eq!(length, 5);
```

默认值计算昂贵时，使用惰性的 `map_or_else`。

### 5.2 `and_then`：串联可能失败的操作

如果转换函数本身也返回 `Option`，应使用 `and_then`，避免得到嵌套的 `Option<Option<T>>`：

```rust
fn parse_positive(text: &str) -> Option<u32> {
    text.parse::<u32>().ok().filter(|value| *value > 0)
}

let port = Some("8080").and_then(parse_positive);
let invalid = Some("not-a-number").and_then(parse_positive);

assert_eq!(port, Some(8080));
assert_eq!(invalid, None);
```

可以把 `map` 和 `and_then` 的区别记成：

- `map: Option<T> + (T -> U) -> Option<U>`；
- `and_then: Option<T> + (T -> Option<U>) -> Option<U>`。

### 5.3 `filter`：保留满足条件的值

```rust
let score = Some(95).filter(|value| *value >= 60);
let failed = Some(40).filter(|value| *value >= 60);

assert_eq!(score, Some(95));
assert_eq!(failed, None);
```

`filter` 不会改变内部值，只会决定保留 `Some` 还是变成 `None`。

### 5.4 `or` 和 `or_else`：提供备用选项

```rust
let command_line = None;
let environment = Some("8080");
let port = command_line.or(environment);

assert_eq!(port, Some("8080"));
```

`or` 的备用 `Option` 会立即求值；需要延迟计算时使用 `or_else`：

```rust
fn read_fallback() -> Option<String> {
    Some(String::from("fallback"))
}

let primary: Option<String> = None;
let value = primary.or_else(read_fallback);

assert_eq!(value.as_deref(), Some("fallback"));
```

### 5.5 `zip`：两个值都存在时配对

```rust
let host = Some("localhost");
let port = Some(8080);

assert_eq!(host.zip(port), Some(("localhost", 8080)));
assert_eq!(host.zip(None::<u16>), None);
```

只有两个 `Option` 都是 `Some` 时，结果才是 `Some((A, B))`。

## 6. 使用 `?` 提前传播 `None`

在返回 `Option` 的函数中，`?` 可以解开 `Some`；遇到 `None` 时立即从当前函数返回 `None`：

```rust
fn divide_text(numerator: &str, denominator: &str) -> Option<f64> {
    let left = numerator.parse::<f64>().ok()?;
    let right = denominator.parse::<f64>().ok()?;

    if right == 0.0 {
        return None;
    }

    Some(left / right)
}

assert_eq!(divide_text("10", "2"), Some(5.0));
assert_eq!(divide_text("ten", "2"), None);
assert_eq!(divide_text("10", "0"), None);
```

上面的第一个 `?` 大致相当于：

```rust
fn parse_number(text: &str) -> Option<f64> {
    let value = match text.parse::<f64>().ok() {
        Some(value) => value,
        None => return None,
    };

    Some(value)
}

assert_eq!(parse_number("10"), Some(10.0));
assert_eq!(parse_number("ten"), None);
```

`?` 不会记录值为何缺失。如果调用者需要知道具体错误原因，例如“格式错误”和“除数为零”，应返回 `Result<T, E>`，而不是 `Option<T>`。

## 7. 借用 `Some` 中的值

### 7.1 `as_ref`：从 `Option<T>` 得到 `Option<&T>`

直接匹配拥有所有权的 `Option<String>` 可能移动其中的字符串。只想读取时，可以使用 `as_ref`：

```rust
let username = Some(String::from("Alice"));

let length = username.as_ref().map(|name| name.len());

assert_eq!(length, Some(5));
assert_eq!(username, Some(String::from("Alice"))); // 仍可使用。
```

等价的模式匹配方式是在模式中借用：

```rust
let username = Some(String::from("Alice"));

if let Some(ref name) = username {
    assert_eq!(name.len(), 5);
}

assert!(username.is_some());
```

现代 Rust 代码中，直接匹配 `&username` 或使用 `as_ref()` 往往更直观。

### 7.2 `as_mut`：取得内部值的可变引用

```rust
let mut username = Some(String::from("alice"));

if let Some(name) = username.as_mut() {
    name.make_ascii_uppercase();
}

assert_eq!(username.as_deref(), Some("ALICE"));
```

### 7.3 `as_deref` 与 `as_deref_mut`

`as_deref` 在借用的同时执行 `Deref` 转换。最常见的用途是把 `Option<String>` 转成 `Option<&str>`：

```rust
fn greet(name: Option<&str>) -> String {
    format!("Hello, {}!", name.unwrap_or("guest"))
}

let owned_name = Some(String::from("Alice"));
assert_eq!(greet(owned_name.as_deref()), "Hello, Alice!");
```

这可以避免为了调用接收 `Option<&str>` 的函数而克隆字符串。

### 7.4 `copied` 与 `cloned`

对于 `Option<&T>`：

- `copied()` 在 `T: Copy` 时得到 `Option<T>`；
- `cloned()` 在 `T: Clone` 时克隆出 `Option<T>`。

```rust
let number = 42;
let borrowed_number = Some(&number);
assert_eq!(borrowed_number.copied(), Some(42));

let name = String::from("Alice");
let borrowed_name = Some(&name);
assert_eq!(borrowed_name.cloned(), Some(String::from("Alice")));
```

优先保留引用；只有确实需要独立所有权时才复制或克隆。

## 8. 原地插入、取出与替换

### 8.1 `get_or_insert` 和 `get_or_insert_with`

对于一个可变 `Option<T>`，可以在它为 `None` 时插入默认值，并取得内部值的可变引用：

```rust
let mut timeout: Option<u64> = None;

let value = timeout.get_or_insert(30);
*value += 5;

assert_eq!(timeout, Some(35));
```

默认值构造昂贵时使用 `get_or_insert_with`；类型实现了 `Default` 时还可以使用 `get_or_insert_default`。

### 8.2 `insert`：无条件设置新值

`Option::insert` 会替换原有值并返回新值的可变引用：

```rust
let mut current = Some(String::from("old"));

let value = current.insert(String::from("new"));
value.push_str(" value");

assert_eq!(current.as_deref(), Some("new value"));
```

### 8.3 `take`：取出值并留下 `None`

`take` 非常适合从结构体字段中安全地移动值：

```rust
#[derive(Debug)]
struct Job {
    result: Option<String>,
}

let mut job = Job {
    result: Some(String::from("done")),
};

let result = job.result.take();

assert_eq!(result.as_deref(), Some("done"));
assert_eq!(job.result, None);
```

如果直接从一个借用中的结构体字段移动 `String`，编译器会拒绝；`take` 用 `None` 留在原位置，因此满足所有权规则。

### 8.4 `replace`：替换并返回旧值

```rust
let mut value = Some(10);

let old = value.replace(20);

assert_eq!(old, Some(10));
assert_eq!(value, Some(20));
```

## 9. `Option` 与迭代器

`Option<T>` 可以看作包含零个或一个元素的集合，并实现了 `IntoIterator`：

```rust
let value = Some(42);
let values: Vec<_> = value.into_iter().collect();

assert_eq!(values, vec![42]);
```

这使 `Option` 很容易与迭代器组合。`filter_map` 可以在遍历时同时过滤失败项并提取 `Some` 中的值：

```rust
let inputs = ["10", "invalid", "20"];

let numbers: Vec<i32> = inputs
    .into_iter()
    .filter_map(|text| text.parse().ok())
    .collect();

assert_eq!(numbers, vec![10, 20]);
```

只有当忽略失败项符合业务语义时才应使用 `filter_map`。如果任何解析失败都应使整个操作失败，可以直接收集为 `Result<Vec<_>, _>`。

嵌套的 `Option<Option<T>>` 可以使用 `flatten` 压平：

```rust
let nested = Some(Some(42));
assert_eq!(nested.flatten(), Some(42));

let missing: Option<Option<i32>> = Some(None);
assert_eq!(missing.flatten(), None);
```

## 10. `Option` 与 `Result` 的转换

### 10.1 `ok_or` 和 `ok_or_else`

当缺失值需要变成一个明确错误时，可以把 `Option<T>` 转换为 `Result<T, E>`：

```rust
fn require_username(name: Option<&str>) -> Result<&str, &'static str> {
    name.ok_or("缺少用户名")
}

assert_eq!(require_username(Some("Alice")), Ok("Alice"));
assert_eq!(require_username(None), Err("缺少用户名"));
```

错误值构造昂贵时使用惰性的 `ok_or_else`：

```rust
let value: Option<u32> = None;
let result = value.ok_or_else(|| String::from("没有可用的数字"));

assert_eq!(result, Err(String::from("没有可用的数字")));
```

### 10.2 `Result::ok` 会丢弃错误信息

`result.ok()` 可以将 `Result<T, E>` 转为 `Option<T>`，但错误内容会被丢弃：

```rust
let parsed = "42".parse::<u32>().ok();
let invalid = "forty-two".parse::<u32>().ok();

assert_eq!(parsed, Some(42));
assert_eq!(invalid, None);
```

只有确实不关心失败原因时才使用 `.ok()`。

### 10.3 `transpose` 调换嵌套结构

`Option<Result<T, E>>` 可以通过 `transpose` 转换为 `Result<Option<T>, E>`：

```rust
fn parse_optional_port(text: Option<&str>) -> Result<Option<u16>, std::num::ParseIntError> {
    text.map(str::parse::<u16>).transpose()
}

assert_eq!(parse_optional_port(Some("8080")), Ok(Some(8080)));
assert_eq!(parse_optional_port(None), Ok(None));
assert!(parse_optional_port(Some("invalid")).is_err());
```

这适用于“输入本身可选，但只要提供了输入，就必须正确解析”的场景。

## 11. 在结构体中使用 `Option`

可选字段应明确使用 `Option<T>`：

```rust
#[derive(Debug)]
struct User {
    id: u64,
    nickname: Option<String>,
}

impl User {
    fn display_name(&self) -> &str {
        self.nickname.as_deref().unwrap_or("anonymous")
    }
}

let user = User {
    id: 1,
    nickname: Some(String::from("Alice")),
};

assert_eq!(user.id, 1);
assert_eq!(user.display_name(), "Alice");
```

这样比用空字符串、`0` 或其他特殊值表示缺失更清楚，因为类型本身记录了字段可能不存在。

`Option` 也常用于暂时从字段中移动所有权、延迟初始化，或表示状态机中的可选数据。但如果不同状态拥有完全不同的字段组合，使用多个结构体配合枚举通常比堆叠大量 `Option` 更可靠。

## 12. `Option` 与内存布局

`Option<T>` 在语义上增加了 `None` 状态，但它不一定增加内存占用。对于引用、`Box<T>`、`NonZeroUsize` 等具有无效位模式的类型，Rust 可以利用“空位优化”表示 `None`。

```rust
use std::mem::size_of;

assert_eq!(size_of::<Option<&u8>>(), size_of::<&u8>());
assert_eq!(size_of::<Option<Box<u8>>>(), size_of::<Box<u8>>());
```

不要笼统地认为所有 `Option<T>` 都与 `T` 一样大；这取决于具体类型和 Rust 对该类型提供的布局保证。业务代码也不应依赖未被语言或标准库文档保证的内部表示。

## 13. 什么时候使用 `Option`，什么时候使用 `Result`

适合使用 `Option<T>` 的情况：

- 值缺失是正常状态；
- 调用者只关心“有或没有”；
- 没有必要说明缺失原因；
- 例如集合查找、迭代器搜索、可选配置字段。

适合使用 `Result<T, E>` 的情况：

- 操作可能失败，并且调用者需要知道原因；
- 不同错误需要不同恢复策略；
- 需要记录、展示或传播错误上下文；
- 例如文件读取、网络请求、数据解析和权限检查。

不要为了让签名更短而把有意义的错误压缩成 `None`。反过来，如果缺失本来就是预期分支，也不必人为构造一个错误。

## 14. 常见错误清单

1. **把 `Some(false)` 或 `Some(0)` 当成 `None`**：`Some` 只表示值存在。
2. **到处使用 `unwrap()`**：外部输入或正常缺失应通过匹配、默认值或错误传播处理。
3. **使用含糊的 `expect("出错了")`**：消息应说明哪个程序不变量被破坏。
4. **用 `map` 返回另一个 `Option`**：这会产生嵌套，通常应该使用 `and_then`。
5. **默认值昂贵却使用 `unwrap_or`**：需要惰性计算时使用 `unwrap_or_else`。
6. **只想读取却移动了 `String`**：使用 `as_ref`、`as_deref` 或匹配引用。
7. **为了调用函数而克隆可选字符串**：优先考虑从 `Option<String>` 借用为 `Option<&str>`。
8. **使用 `.ok()` 丢失重要错误**：需要错误原因时保留 `Result`。
9. **用特殊值代替缺失状态**：优先使用 `Option<T>` 表达真实的数据模型。
10. **认为 `?` 会记录错误**：对 `Option` 使用 `?` 只会提前返回 `None`。

## 15. 一个完整示例

下面的示例从可选文本中读取端口号。未提供端口时使用默认值；提供了无效端口时返回具体错误：

```rust
#[derive(Debug, PartialEq)]
enum ConfigError {
    InvalidPort(String),
    PortCannotBeZero,
}

fn parse_port(input: Option<&str>) -> Result<u16, ConfigError> {
    let Some(text) = input else {
        return Ok(8080);
    };

    let port = text
        .parse::<u16>()
        .map_err(|_error| ConfigError::InvalidPort(text.to_owned()))?;

    if port == 0 {
        return Err(ConfigError::PortCannotBeZero);
    }

    Ok(port)
}

fn main() {
    assert_eq!(parse_port(None), Ok(8080));
    assert_eq!(parse_port(Some("3000")), Ok(3000));
    assert_eq!(
        parse_port(Some("invalid")),
        Err(ConfigError::InvalidPort(String::from("invalid")))
    );
    assert_eq!(parse_port(Some("0")), Err(ConfigError::PortCannotBeZero));
}
```

这个示例体现了几个重要原则：

- `None` 表达正常的“未配置”状态，而不是错误；
- `let Some(...) = ... else` 清晰处理缺失分支；
- 文本存在但无法解析时使用 `Result` 保留错误语义；
- `?` 传播解析错误；
- 不使用 `unwrap` 处理外部输入。

## 总结

正确使用 `Some` 的本质，是正确设计和处理 `Option<T>`：

- `Some(value)` 表示值存在，`None` 表示不存在；
- 使用 `match`、`if let`、`let ... else` 或 `?` 显式处理缺失；
- 使用 `map` 转换值，使用 `and_then` 串联可能缺失的操作；
- 使用 `as_ref`、`as_mut` 和 `as_deref` 避免不必要的移动与克隆；
- 使用 `take` 安全地从借用的结构体字段中移出值；
- 只有缺失是正常状态时才使用 `Option`，需要错误原因时使用 `Result`；
- 不要用 `unwrap` 掩盖本应认真处理的 `None`。

把“值可能不存在”放进类型系统，是 Rust 代码安全性和可读性的一个重要来源。
