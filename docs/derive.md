# Rust 中的 `derive`：自动派生 trait 与正确使用

`derive` 是 Rust 的属性语法，用来为结构体或枚举自动生成某些 trait 的实现：

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
struct User {
    id: u64,
    name: String,
}

let first = User {
    id: 1,
    name: String::from("Alice"),
};
let second = first.clone();

assert_eq!(first, second);
println!("{first:?}");
```

`#[derive(...)]` 会在编译期展开为 trait 实现。它不是运行时反射，也不会给类型自动添加所有可能的方法。

## 1. 基本语法

`derive` 写在类型定义上方：

```rust
#[derive(Debug, Clone)]
struct Point {
    x: i32,
    y: i32,
}
```

可以在一个属性中列出多个 trait，也可以写成多个属性：

```rust
#[derive(Debug)]
#[derive(Clone, PartialEq)]
struct Label(String);

let first = Label(String::from("rust"));
let second = first.clone();

assert_eq!(first, second);
```

通常集中写在一个 `derive` 中更容易浏览。

标准库最常用的可派生 trait 包括：

- `Debug`
- `Clone`
- `Copy`
- `PartialEq` 和 `Eq`
- `PartialOrd` 和 `Ord`
- `Hash`
- `Default`

第三方库还可以提供自定义派生宏，例如序列化、数据库映射和命令行参数解析。

## 2. 派生的基本条件

对于包含字段的结构体，只有字段类型满足相应 trait 约束时，派生实现才能使用：

```rust
#[derive(Debug, Clone, PartialEq)]
struct Document {
    title: String,
    page_count: usize,
}

let document = Document {
    title: String::from("Rust Guide"),
    page_count: 100,
};

let copy = document.clone();
assert_eq!(document, copy);
```

这里 `String` 和 `usize` 都实现了 `Debug`、`Clone` 与 `PartialEq`，所以 `Document` 可以派生它们。

如果某个字段不支持所需 trait，派生会编译失败：

```rust,compile_fail
#[derive(Clone)]
struct Task {
    callback: Box<dyn FnOnce()>,
}
```

`dyn FnOnce()` 不能按这种方式克隆，因此包含它的 `Task` 也不能自动派生 `Clone`。

## 3. `Debug`

`Debug` 允许通过 `{:?}` 或 `{:#?}` 输出面向开发者的表示：

```rust
#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}

let point = Point { x: 10, y: 20 };

assert_eq!(format!("{point:?}"), "Point { x: 10, y: 20 }");
assert!(format!("{point:#?}").contains("x: 10"));
```

`Debug` 主要用于日志、诊断和测试，不等同于面向最终用户的 `Display`。标准库不提供通用的 `Display` 派生，因为如何向用户展示类型通常需要人工设计。

### 3.1 注意敏感信息

派生的 `Debug` 默认会打印所有字段。包含密码、访问令牌或个人信息的类型不应被随意完整记录：

```rust
use std::fmt;

struct Credentials {
    username: String,
    token: String,
}

impl fmt::Debug for Credentials {
    fn fmt(&self, formatter: &mut fmt::Formatter<'_>) -> fmt::Result {
        formatter
            .debug_struct("Credentials")
            .field("username", &self.username)
            .field("token", &"[REDACTED]")
            .finish()
    }
}

let credentials = Credentials {
    username: String::from("alice"),
    token: String::from("secret"),
};

let output = format!("{credentials:?}");
assert!(output.contains("alice"));
assert!(!output.contains("secret"));
```

当派生语义可能泄露数据时，应手写安全的实现，或者避免输出整个值。

## 4. `Clone` 与 `Copy`

### 4.1 派生 `Clone`

`Clone` 提供显式的 `.clone()`：

```rust
#[derive(Debug, Clone, PartialEq)]
struct User {
    name: String,
}

let first = User {
    name: String::from("Alice"),
};
let second = first.clone();

assert_eq!(first, second);
```

派生实现会依次克隆各字段。克隆包含 `String`、`Vec<T>` 等数据的结构体可能涉及堆分配，不能因为是自动派生就假设没有成本。

### 4.2 派生 `Copy`

`Copy` 表示赋值和传参可以进行隐式值复制。派生 `Copy` 时还必须实现 `Clone`，并且所有字段都必须是 `Copy`：

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
struct Coordinate {
    x: i32,
    y: i32,
}

let first = Coordinate { x: 10, y: 20 };
let second = first;

assert_eq!(first, second); // first 没有被移动。
```

包含 `String` 的类型不能派生 `Copy`：

```rust,compile_fail
#[derive(Clone, Copy)]
struct Name {
    value: String,
}
```

是否实现 `Copy` 是 API 语义的一部分。小型、简单、没有资源所有权语义的值类型通常适合 `Copy`；大型对象或需要明确所有权转移的类型即使技术上可行，也应谨慎决定。

## 5. `PartialEq` 与 `Eq`

### 5.1 `PartialEq` 自动比较字段

结构体派生的 `PartialEq` 会按字段进行相等比较：

```rust
#[derive(Debug, PartialEq)]
struct Version {
    major: u32,
    minor: u32,
}

assert_eq!(
    Version { major: 1, minor: 2 },
    Version { major: 1, minor: 2 }
);
assert_ne!(
    Version { major: 1, minor: 2 },
    Version { major: 1, minor: 3 }
);
```

枚举值只有变体相同且对应字段相等时才相等：

```rust
#[derive(Debug, PartialEq)]
enum Status {
    Pending,
    Completed(u64),
}

assert_eq!(Status::Completed(42), Status::Completed(42));
assert_ne!(Status::Pending, Status::Completed(42));
```

### 5.2 `Eq` 表示完全的等价关系

`Eq` 没有额外方法，它表示 `PartialEq` 的相等关系满足自反性，即 `value == value` 总为真。通常一起派生：

```rust
#[derive(Debug, PartialEq, Eq)]
struct UserId(u64);
```

`f32` 和 `f64` 由于 `NaN != NaN`，只实现了 `PartialEq`，没有实现 `Eq`：

```rust,compile_fail
#[derive(PartialEq, Eq)]
struct Measurement {
    value: f64,
}
```

需要将类型用作 `HashMap` 的键或 `HashSet` 的元素时，一般需要 `Eq` 和 `Hash`。

### 5.3 领域身份不一定等于所有字段

派生 `PartialEq` 会比较所有字段。但有时实体身份只由 ID 决定，名称等字段不应参与相等判断：

```rust
#[derive(Debug)]
struct User {
    id: u64,
    display_name: String,
}

impl PartialEq for User {
    fn eq(&self, other: &Self) -> bool {
        self.id == other.id
    }
}

impl Eq for User {}

let first = User { id: 1, display_name: String::from("Alice") };
let second = User { id: 1, display_name: String::from("Alice Chen") };

assert_eq!(first, second);
assert_ne!(first.display_name, second.display_name);
```

自动派生前应先确认“所有字段相等”是否真的符合业务语义。

## 6. `Hash`

`Hash` 允许值被哈希，常与 `PartialEq`、`Eq` 一起派生：

```rust
use std::collections::HashSet;

#[derive(Debug, Hash, PartialEq, Eq)]
struct UserId {
    tenant: u32,
    id: u64,
}

let mut users = HashSet::new();
users.insert(UserId { tenant: 1, id: 42 });

assert!(users.contains(&UserId { tenant: 1, id: 42 }));
```

派生的 `Hash` 会把字段依次写入哈希器。必须满足：

```text
a == b  =>  hash(a) == hash(b)
```

如果手写 `PartialEq` 只比较部分字段，就不能继续派生一个使用全部字段的 `Hash`，否则两者可能不一致。应让二者使用相同的身份字段：

```rust
use std::hash::{Hash, Hasher};

#[derive(Debug)]
struct User {
    id: u64,
    display_name: String,
}

impl PartialEq for User {
    fn eq(&self, other: &Self) -> bool {
        self.id == other.id
    }
}

impl Eq for User {}

impl Hash for User {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.id.hash(state);
    }
}

let first = User { id: 1, display_name: String::from("Alice") };
let second = User { id: 1, display_name: String::from("Alice Chen") };

assert_eq!(first, second);
assert_ne!(first.display_name, second.display_name);
```

插入哈希集合或哈希表后，参与 `Eq` 和 `Hash` 的内容不应通过内部可变性改变。

## 7. `PartialOrd` 与 `Ord`

结构体派生排序 trait 时，会按照字段声明顺序进行字典序比较：

```rust
#[derive(Debug, PartialEq, Eq, PartialOrd, Ord)]
struct Version {
    major: u32,
    minor: u32,
    patch: u32,
}

let stable = Version { major: 1, minor: 9, patch: 0 };
let next = Version { major: 2, minor: 0, patch: 0 };

assert!(stable < next);
```

字段顺序会影响排序语义：先比较 `major`，相等时比较 `minor`，最后比较 `patch`。

枚举派生 `Ord` 时，默认按变体在源码中的声明顺序排序，再比较变体字段：

```rust
#[derive(Debug, PartialEq, Eq, PartialOrd, Ord)]
enum Priority {
    Low,
    Medium,
    High,
}

assert!(Priority::Low < Priority::Medium);
assert!(Priority::Medium < Priority::High);
```

如果声明顺序不等于业务顺序，不应盲目派生，应调整定义或手写实现。

`PartialOrd` 允许某些值无法比较，`Ord` 要求全序。包含浮点数字段的类型不能直接派生 `Eq` 和 `Ord`，需要先定义清楚 `NaN` 等值的业务语义。

## 8. `Default`

结构体派生 `Default` 时，每个字段都使用自己的默认值：

```rust
#[derive(Debug, Default, PartialEq, Eq)]
struct Config {
    host: String,
    port: u16,
    verbose: bool,
}

let config = Config::default();

assert_eq!(config.host, "");
assert_eq!(config.port, 0);
assert!(!config.verbose);
```

派生成功不代表默认值在业务上有效。例如端口 `0` 可能不允许，空主机名可能没有意义。此时应手写 `Default`：

```rust
#[derive(Debug, PartialEq, Eq)]
struct Config {
    host: String,
    port: u16,
}

impl Default for Config {
    fn default() -> Self {
        Self {
            host: String::from("127.0.0.1"),
            port: 8080,
        }
    }
}

let config = Config::default();
assert_eq!(config.host, "127.0.0.1");
assert_eq!(config.port, 8080);
```

枚举派生 `Default` 时，需要用 `#[default]` 标记默认的无字段变体：

```rust
#[derive(Debug, Default, PartialEq, Eq)]
enum LogLevel {
    Trace,
    Debug,
    #[default]
    Info,
    Warning,
    Error,
}

assert_eq!(LogLevel::default(), LogLevel::Info);
```

常见的结构体更新语法：

```rust
#[derive(Debug, Default, PartialEq, Eq)]
struct Options {
    verbose: bool,
    retries: u32,
}

let options = Options {
    verbose: true,
    ..Options::default()
};

assert_eq!(options, Options { verbose: true, retries: 0 });
```

## 9. 泛型类型上的派生

泛型类型派生 trait 时，生成的实现通常会带上必要的 trait 约束：

```rust
#[derive(Debug, Clone, PartialEq)]
struct Wrapper<T> {
    value: T,
}

let first = Wrapper { value: String::from("hello") };
let second = first.clone();

assert_eq!(first, second);
```

可以将派生的 `Clone` 粗略理解成生成了类似实现：

```text
impl<T: Clone> Clone for Wrapper<T> { ... }
```

有时自动派生会为泛型参数生成比实际字段行为更严格的约束。例如智能指针自身可以在不克隆内部 `T` 的情况下克隆，但自动派生策略未必能表达最宽松的公共边界。遇到不必要的泛型约束时，可以手写实现：

```rust
use std::sync::Arc;

struct Shared<T> {
    value: Arc<T>,
}

impl<T> Clone for Shared<T> {
    fn clone(&self) -> Self {
        Self {
            value: Arc::clone(&self.value),
        }
    }
}

struct NotClone;

let first = Shared { value: Arc::new(NotClone) };
let second = first.clone();

assert!(Arc::ptr_eq(&first.value, &second.value));
```

手写实现可以放宽约束，但也增加维护负担。只有自动派生确实不符合 API 需求时才这样做。

## 10. 为枚举派生 trait

`derive` 同样适用于枚举：

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
enum Message {
    Quit,
    Text(String),
    Move { x: i32, y: i32 },
}

let message = Message::Text(String::from("hello"));
assert_eq!(message.clone(), message);
```

所有变体的所有字段都必须满足相应约束。即使代码很少构造某个变体，该变体中不支持 `Clone` 的字段也会阻止整个枚举派生 `Clone`。

## 11. 自定义派生宏

`derive` 不限于标准库内置 trait。过程宏 crate 可以提供自定义派生：

```text
#[derive(Serialize, Deserialize)]
struct User {
    id: u64,
    name: String,
}
```

上面的 `Serialize` 和 `Deserialize` 并非标准库内置，需要在 `Cargo.toml` 中添加提供它们的依赖并启用相应功能。

自定义派生宏还经常配合辅助属性：

```text
#[derive(Serialize)]
struct User {
    #[serde(rename = "userId")]
    id: u64,
}
```

正确使用第三方派生宏时应：

- 阅读该 crate 当前版本的官方文档；
- 明确宏生成哪些实现和 trait 约束；
- 检查字段重命名、默认值和跳过规则；
- 注意生成代码对公共 API、数据兼容性和编译时间的影响；
- 不要假设不同库中名字相似的辅助属性具有相同语义。

可以使用 `cargo expand` 等开发工具查看宏展开后的代码，但展开结果更适合调试和学习，不应依赖其不稳定的内部细节。

## 12. `derive` 属性与其他属性

`derive` 使用 Rust 通用的属性语法 `#[...]`。属性可以作用于类型、字段、函数、模块等项目，但不同属性有不同适用位置。

可以条件化地派生：

```rust
#[cfg_attr(debug_assertions, derive(Debug))]
struct InternalState {
    value: u32,
}

let state = InternalState { value: 42 };
assert_eq!(state.value, 42);
```

这里仅在启用调试断言的构建中派生 `Debug`。条件派生会让不同构建配置下的 API 能力不同，应谨慎使用。

## 13. 派生还是手写实现

适合派生的情况：

- 逐字段实现正好符合业务语义；
- 类型只是普通值对象或数据传输对象；
- 希望减少样板代码并跟随字段变更；
- 标准行为比自定义格式更重要。

适合手写的情况：

- 只有部分字段定义身份、哈希或排序；
- `Debug` 必须隐藏敏感字段；
- `Default` 需要有效的领域默认值；
- `Clone` 需要共享资源、重置缓存或采用特殊策略；
- 派生给泛型参数添加了不必要约束；
- `Display` 等面向用户的输出需要人工设计。

不要为了减少几行代码而接受错误的领域语义。也不要在派生已经完全正确时手写冗长实现，因为手写代码更容易遗漏新增字段。

## 14. trait 之间的一致性

多个派生或手写 trait 之间必须保持一致：

- `Eq` 依赖 `PartialEq`；
- `Ord` 依赖 `Eq` 和 `PartialOrd`；
- `Copy` 依赖 `Clone`；
- `Hash` 必须与 `Eq` 使用一致的相等语义；
- `PartialOrd` 与 `PartialEq`、`Ord` 的结果不应相互矛盾。

最安全的做法是对相关 trait 一起派生：

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
struct ProductId(u64);

let first = ProductId(1);
let second = first;

assert_eq!(first, second);
```

如果必须手写其中之一，应重新审视所有关联 trait，而不是只修改单个实现。

## 15. 常见错误清单

1. **认为 `derive` 会生成任意功能**：它只生成列出的、可派生 trait 实现。
2. **为方便打印敏感类型派生 `Debug`**：日志可能泄露令牌、密码和个人数据。
3. **看到类型较小就盲目派生 `Copy`**：还应考虑资源和所有权语义。
4. **派生 `PartialEq` 却忽略领域身份**：默认会比较所有字段。
5. **手写 `Eq` 后继续使用不一致的 `Hash`**：相等值必须产生相等哈希。
6. **认为派生 `Ord` 会按业务优先级排序**：它使用字段或变体声明顺序。
7. **认为派生 `Default` 一定产生有效配置**：零值和空字符串可能违反业务规则。
8. **忽略泛型上的自动约束**：派生实现可能要求类型参数也实现相应 trait。
9. **假设派生 `Clone` 没有性能成本**：包含堆数据时可能执行分配和复制。
10. **忘记第三方 derive 需要依赖和功能开关**：它们不是编译器自动提供的。

## 16. 一个完整示例

下面的例子展示如何根据领域语义选择派生和手写实现：

```rust
use std::collections::HashSet;
use std::fmt;
use std::hash::{Hash, Hasher};

#[derive(Clone)]
struct Account {
    // 账号身份只由 id 决定。
    id: u64,
    display_name: String,
    access_token: String,
}

impl PartialEq for Account {
    fn eq(&self, other: &Self) -> bool {
        self.id == other.id
    }
}

impl Eq for Account {}

impl Hash for Account {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.id.hash(state);
    }
}

impl fmt::Debug for Account {
    fn fmt(&self, formatter: &mut fmt::Formatter<'_>) -> fmt::Result {
        formatter
            .debug_struct("Account")
            .field("id", &self.id)
            .field("display_name", &self.display_name)
            .field("access_token", &"[REDACTED]")
            .finish()
    }
}

fn main() {
    let first = Account {
        id: 1,
        display_name: String::from("Alice"),
        access_token: String::from("secret-1"),
    };

    let renamed = Account {
        id: 1,
        display_name: String::from("Alice Chen"),
        access_token: String::from("secret-2"),
    };

    // ID 相同，因此在领域语义中是同一个账号。
    assert_eq!(first, renamed);

    let mut accounts = HashSet::new();
    accounts.insert(first.clone());
    accounts.insert(renamed);
    assert_eq!(accounts.len(), 1);

    // 自定义 Debug 不会泄露令牌。
    let debug_output = format!("{first:?}");
    assert!(debug_output.contains("Alice"));
    assert!(!debug_output.contains("secret-1"));
}
```

这个例子中：

- `Clone` 的逐字段行为符合需求，因此直接派生；
- `PartialEq`、`Eq` 和 `Hash` 必须只使用 `id`，因此手写并保持一致；
- `Debug` 必须隐藏访问令牌，因此手写；
- 没有盲目派生 `Default`、`Ord` 或 `Copy` 等不需要的 trait。

## 总结

正确使用 `derive` 的关键不是“能派生多少”，而是“自动生成的语义是否正确”：

- `derive` 在编译期生成指定 trait 的实现；
- 只有字段和变体满足约束时，派生实现才可用；
- `Debug` 注意敏感信息，`Clone` 注意成本，`Copy` 注意所有权语义；
- `PartialEq`、`Hash` 和 `Ord` 必须符合领域中的身份与顺序；
- `Default` 应产生真正合理的默认状态；
- 泛型派生可能引入额外 trait 边界；
- 第三方派生宏应依据对应版本的文档使用；
- 当逐字段行为正确时优先派生，不正确时应手写实现并保持相关 trait 一致。

派生宏能显著减少样板代码，但它生成的是行为而不仅是语法。先设计语义，再选择派生，是最稳妥的使用方式。
