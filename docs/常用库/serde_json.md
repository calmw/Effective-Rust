# Rust 常用库：`serde_json`

[`serde_json`](https://docs.rs/serde_json/latest/serde_json/) 是基于 Serde 的 JSON 实现，可在 Rust 类型、JSON 文本和动态 JSON 值之间转换。

## 1. 添加依赖

```bash
cargo add serde --features derive
cargo add serde_json
```

## 2. 结构体与 JSON 互转

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize, PartialEq)]
struct User {
    id: u64,
    name: String,
}

fn main() -> Result<(), serde_json::Error> {
    let user = User { id: 1, name: "Alice".into() };

    let json = serde_json::to_string(&user)?;
    let decoded: User = serde_json::from_str(&json)?;

    assert_eq!(decoded, user);
    Ok(())
}
```

常用函数：

- `to_string`、`to_string_pretty`：序列化为字符串；
- `to_writer`、`to_writer_pretty`：直接写入输出流；
- `from_str`、`from_slice`：从内存解析；
- `from_reader`：从文件或网络流读取。

## 3. 动态 JSON：`Value`

```rust
use serde_json::{json, Value};

let value: Value = json!({
    "name": "Alice",
    "roles": ["admin", "user"]
});

assert_eq!(value["name"], "Alice");
assert_eq!(value["roles"][0], "admin");
```

结构固定时优先定义结构体；只有字段动态、只访问少量字段或需要透传 JSON 时才使用 `Value`。

## 4. 安全访问动态字段

索引语法在字段不存在时通常得到 `Value::Null`，容易混淆“字段不存在”和“字段确实为 null”。严格处理输入时使用：

```rust
use serde_json::Value;

fn read_name(value: &Value) -> Option<&str> {
    value.get("name")?.as_str()
}
```

## 5. 大数据和流式处理

处理大文件时不要先读成一个巨大的 `String`：

```rust
use std::{fs::File, io::BufReader};

fn read_users(path: &str) -> anyhow::Result<Vec<serde_json::Value>> {
    let file = File::open(path)?;
    let reader = BufReader::new(file);
    Ok(serde_json::from_reader(reader)?)
}
```

连续 JSON 值可以使用 `Deserializer::from_reader(...).into_iter::<T>()` 逐个处理。

## 6. 最佳实践

1. 固定协议使用强类型结构体，让缺失字段和类型错误尽早暴露。
2. 输出到文件或网络时优先 `to_writer`，减少中间字符串分配。
3. API 响应不应直接暴露内部数据库模型。
4. 不要记录包含令牌、密码或个人信息的完整 JSON。
5. 金额和高精度数字不要直接依赖 `f64`；使用字符串或十进制类型并明确协议。
6. 将解析错误用 `anyhow::Context` 或自定义错误补充数据来源。

## 总结

`serde_json` 是 JSON 格式实现，`serde` 是通用序列化框架。结构已知时使用类型化反序列化；结构未知时才选择 `Value`。
