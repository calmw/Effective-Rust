# Effective Rust

一份面向 Rust 学习者和实践者的中文笔记，持续整理编写安全、清晰、高效 Rust 代码时值得掌握的知识与方法。

本项目不只罗列 API，还会解释：

- 一段代码为什么能够工作；
- 类型、所有权和借用在其中如何变化；
- 哪些写法虽然能编译，却容易产生错误或额外开销；
- 不同方法和数据结构应该如何选择；
- 如何通过完整示例把零散语法应用到实际问题中。

## 文档目录

### 集合类型

- [`HashMap`](docs/HashMap.md)：键值映射、查询与更新、`entry` API、集合构造、所有权、性能和并发使用。
- [`HashSet`](docs/HashSet.md)：唯一值集合、去重、集合运算、`take`、`replace`、哈希约束和数据结构选择。

### 错误与可选值

- [`Some` 与 `Option<T>`](docs/Some.md)：可选值、模式匹配、组合子、借用、`?` 以及与 `Result` 的转换。
- [`Result<T, E>`](docs/Result.md)：可恢复错误、错误传播、`map_err`、自定义错误、批量处理和 panic 的使用边界。

### 字符串

- [字符串的常用方法](docs/字符串的常用方法.md)：`String` 与 `&str`、UTF-8、查找、切分、修改、拼接、解析和性能建议。

### 常用语法与方法

- [常用函数](docs/常用函数.md)：`copied()`、`cloned()`、`clone()`、`Option::map()` 和 `map_or()` 的区别与正确使用。
- [闭包语法解释](docs/语法解释.md)：`|| 30`、`|key| key.len()`、环境捕获、`Fn`/`FnMut`/`FnOnce` 和 `HashMap::entry`。
- [`derive`](docs/derive.md)：自动派生 trait、常用内置派生、泛型约束、自定义派生及派生与手写实现的选择。

### 工程组织

- [Rust 模块如何组织](docs/模块如何组织.md)：package、crate 与 module 的关系，`lib.rs`、`main.rs`、`mod`、`use`、`pub use`、可见性、工作空间及分层最佳实践。

### 常用第三方库

#### 数据转换

- [`serde`](docs/常用库/serde.md)：通用序列化与反序列化框架，派生宏、字段属性、借用数据和传输模型设计。
- [`serde_json`](docs/常用库/serde_json.md)：Rust 类型、JSON 文本与动态 `Value` 的转换，以及大数据和流式处理建议。

#### 错误处理

- [`anyhow`](docs/常用库/anyhow.md)：应用层错误传播、上下文、错误链、Backtrace，以及和具体错误类型的使用边界。
- [`thiserror`](docs/常用库/thiserror.md)：通过派生宏定义结构化错误，并与 `anyhow` 配合建立清晰的错误分层。

#### 异步与可观察性

- [`tokio`](docs/常用库/tokio.md)：异步运行时、任务、并发执行、阻塞操作、超时、取消和背压。
- [`tracing`](docs/常用库/tracing.md)：结构化事件、Span、`#[instrument]` 和异步程序的诊断埋点。
- [`tracing-subscriber`](docs/常用库/tracing-subscriber.md)：日志过滤、格式化、Layer、JSON 输出和应用入口初始化。

#### 网络、Web 与数据库

- [`reqwest`](docs/常用库/reqwest.md)：异步 HTTP 客户端、连接池、超时、状态码处理和安全重试。
- [`axum`](docs/常用库/axum.md)：路由、Extractor、共享状态、统一错误响应和 Tower 中间件。
- [Axum 完整指南](docs/web/axum/README.md)：Axum 的特点、请求提取、状态、错误、中间件、测试、项目结构和生产最佳实践。
- [Loco 完整指南](docs/web/loco/README.md)：Rails 风格的 Axum 应用框架，涵盖 SeaORM、生成器、认证、后台任务、配置、测试和部署。
- [`sqlx`](docs/常用库/sqlx.md)：异步 SQL、连接池、参数绑定、事务、Migration 和编译期查询检查。

#### 命令行与数据处理

- [`clap`](docs/常用库/clap.md)：使用结构体和枚举构建强类型命令行接口、子命令及可测试的参数解析。
- [`itertools`](docs/常用库/itertools.md)：标准迭代器的扩展适配器，以及惰性执行、分组、去重和组合操作。
- [`rayon`](docs/常用库/rayon.md)：面向 CPU 密集型任务的数据并行、并行迭代器和线程池使用边界。

#### 测试与性能

- [`proptest`](docs/常用库/proptest.md)：属性测试、输入策略、失败用例缩减和业务不变量设计。
- [`criterion`](docs/常用库/criterion.md)：统计驱动的微基准测试、输入规模设计和可信性能测量。

### 桌面应用

- [Tauri 完整指南](docs/桌面应用/tauri/README.md)：Tauri 2 的架构、组件、IPC、权限、插件、状态管理、打包更新、安全和生产最佳实践。
- [egui 完整指南](docs/桌面应用/egui/README.md)：即时模式 GUI、eframe、核心组件、布局、状态、资源、自定义控件、后台任务、测试和性能最佳实践。

## 推荐阅读顺序

如果刚开始学习 Rust，可以按照下面的顺序阅读：

1. [字符串的常用方法](docs/字符串的常用方法.md)，熟悉 `String`、`&str` 和 UTF-8；
2. [`Some` 与 `Option<T>`](docs/Some.md)，理解 Rust 如何表达可能缺失的值；
3. [`Result<T, E>`](docs/Result.md)，学习可恢复错误和 `?`；
4. [`HashMap`](docs/HashMap.md) 与 [`HashSet`](docs/HashSet.md)，掌握常用集合及所有权问题；
5. [常用函数](docs/常用函数.md)，进一步理解复制、克隆和组合子；
6. [闭包语法解释](docs/语法解释.md)，理解迭代器和 `entry` API 中常见的闭包；
7. [`derive`](docs/derive.md)，学习如何为自己的类型生成或设计 trait 实现；
8. [Rust 模块如何组织](docs/模块如何组织.md)，把单文件示例组织成可维护的 crate；
9. [`serde`](docs/常用库/serde.md)、[`serde_json`](docs/常用库/serde_json.md)，掌握实际项目中的数据转换；
10. [`anyhow`](docs/常用库/anyhow.md)、[`thiserror`](docs/常用库/thiserror.md)，建立应用层和库层的错误边界；
11. [`tokio`](docs/常用库/tokio.md)、[`tracing`](docs/常用库/tracing.md)，学习异步程序及其可观察性；
12. 根据项目方向选择后续内容：CLI 阅读 [`clap`](docs/常用库/clap.md)，Web 后端阅读 [`axum`](docs/常用库/axum.md)、[`reqwest`](docs/常用库/reqwest.md) 和 [`sqlx`](docs/常用库/sqlx.md)。

已经有 Rust 基础时，可以直接从具体问题对应的文档开始阅读。

## 阅读代码示例

文档中的大部分代码块都是可以独立编译的完整示例。建议不要只阅读代码，还可以：

1. 先预测代码的返回类型和所有权变化；
2. 在本地运行示例；
3. 修改输入，观察 `Some`、`None`、`Ok` 和 `Err` 分支；
4. 故意触发编译错误，再结合编译器提示理解限制；
5. 将示例改写成 `match`、组合子或普通循环，对比可读性。

单个 Markdown 文档可以使用 `rustdoc` 检查其中标记为 Rust 的代码块：

```bash
rustdoc --edition=2021 --test docs/Result.md
```

代码块标记为 `compile_fail` 时，测试通过表示该示例确实无法编译，用于演示 Rust 阻止的错误写法。

第三方库文档中的部分代码使用 `ignore`，因为它们需要对应依赖、数据库或网络环境。建议在临时 Cargo 项目中添加文档列出的依赖后运行这些示例。

## 内容原则

本项目中的文档遵循以下原则：

- 使用简体中文解释，保留必要的 Rust 类型和 API 名称；
- 先说明语义和使用场景，再介绍具体方法；
- 明确指出值是被借用、复制、克隆还是移动；
- 对可能 panic、丢失错误信息或产生额外分配的写法给出提醒；
- 优先使用标准库示例，涉及第三方库时明确说明依赖关系；
- 示例尽量自包含，并使用断言表达预期结果；
- 不把“代码更短”作为唯一目标，优先保证正确性和可维护性。

## 如何选择学习重点

遇到以下问题时，可以从对应主题开始：

- 不理解 `Some(...)`、`None` 或 `?`：阅读 `Option` 和 `Result`；
- 不清楚 `.map(...)` 中的竖线：阅读闭包语法解释；
- 分不清 `copied()`、`cloned()` 和 `clone()`：阅读常用函数；
- 字符串不能通过整数索引：阅读字符串中的 UTF-8 与安全切片章节；
- 不知道如何统计、去重或快速查询：阅读 `HashMap` 和 `HashSet`；
- 不确定是否应该派生 `Copy`、`Eq`、`Hash` 或 `Ord`：阅读 `derive`。
- 不理解 `lib.rs`、`mod`、`use` 或模块文件放在哪里：阅读 Rust 模块如何组织；
- 需要读写 JSON：阅读 `serde` 和 `serde_json`；
- 不知道应用错误和公共库错误怎样设计：阅读 `anyhow` 和 `thiserror`；
- 需要编写异步程序或排查异步调用链：阅读 `tokio`、`tracing` 和 `tracing-subscriber`；
- 准备开发 Web API：阅读 `axum`、`reqwest` 和 `sqlx`；
- 希望验证算法性质或性能优化：阅读 `proptest` 和 `criterion`。

## 参与完善

欢迎继续补充新的 Rust 实践主题，或改进已有文档中的解释和示例。新增内容时建议：

1. 为概念提供一个最小示例；
2. 说明方法的输入、输出及所有权影响；
3. 补充至少一个常见错误或使用边界；
4. 使用断言验证关键结果；
5. 运行相关示例和 `git diff --check` 后再提交。

这个仓库会随着学习和实践持续完善。
