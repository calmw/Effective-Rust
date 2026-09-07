# Rust 常用库：`clap`

[`clap`](https://docs.rs/clap/latest/clap/) 是 Rust 常用的命令行参数解析库，能够生成帮助信息、校验参数、解析子命令，并把参数转换成强类型结构体。

## 1. 添加依赖

```bash
cargo add clap --features derive
```

## 2. Derive 风格

```rust
use clap::{Parser, ValueEnum};
use std::path::PathBuf;

#[derive(Debug, Clone, ValueEnum)]
enum Format {
    Json,
    Text,
}

#[derive(Debug, Parser)]
#[command(version, about = "Process an input file")]
struct Cli {
    #[arg(short, long)]
    input: PathBuf,

    #[arg(short, long, value_enum, default_value = "text")]
    format: Format,

    #[arg(long, default_value_t = 3)]
    retries: u8,
}

fn main() {
    let cli = Cli::parse();
    println!("{cli:?}");
}
```

优先使用 `PathBuf`、数字、枚举等目标类型，让 Clap 在业务逻辑执行前完成解析和校验。

## 3. 子命令

```rust
use clap::{Parser, Subcommand};

#[derive(Parser)]
struct Cli {
    #[command(subcommand)]
    command: Command,
}

#[derive(Subcommand)]
enum Command {
    Add { name: String },
    Remove { id: u64 },
}
```

子命令枚举比手工判断字符串更清晰，也能获得自动生成的帮助信息。

## 4. 可测试的解析

```rust
use clap::Parser;

#[derive(Debug, Parser)]
struct Cli {
    #[arg(long)]
    port: u16,
}

let cli = Cli::try_parse_from(["app", "--port", "8080"])?;
assert_eq!(cli.port, 8080);
# Ok::<(), clap::Error>(())
```

测试使用 `try_parse_from`，避免 `parse()` 在参数错误时直接退出进程。

## 5. 最佳实践

1. 参数解析结构和业务配置结构分开，通过显式转换合并配置。
2. 使用 `ValueEnum` 表达有限选项，不要在业务层匹配字符串。
3. 密码和令牌优先从环境变量、标准输入或密钥服务读取，避免出现在 shell 历史和进程列表。
4. 合理提供默认值，但不要给安全敏感选项设置危险默认值。
5. 错误输出到 stderr，正常结果输出到 stdout，方便管道组合。
6. CLI 入口只负责解析和调用 `run(cli)`，核心逻辑保持可测试。

## 总结

Clap 的最佳用法是把命令行接口建模成强类型结构体和枚举。解析层越明确，业务代码中手工校验字符串和处理缺失参数的代码就越少。
