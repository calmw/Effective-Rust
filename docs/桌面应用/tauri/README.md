# Tauri 完整指南：特点、组件、使用方式与最佳实践

[`Tauri`](https://tauri.app/) 是用于构建桌面和移动应用的跨平台框架。它通常使用 HTML、CSS、JavaScript/TypeScript 构建界面，使用 Rust 实现本地能力，通过受控 IPC 连接 WebView 前端和 Rust Core。

与捆绑完整浏览器运行时的方案不同，Tauri 默认使用操作系统提供的 WebView，因此安装包通常更小、内存基线更低。不过不同平台使用不同 WebView 实现，兼容性测试也更重要。

本文面向 Tauri 2.x。

## 1. Tauri 适合做什么

Tauri 适合：

- 跨平台桌面工具；
- 文件管理、数据库客户端和开发者工具；
- 系统托盘、菜单栏和常驻小工具；
- 已有 Web 前端，希望增加本地能力的应用；
- 需要文件、通知、剪贴板、窗口、数据库等系统集成的应用；
- 希望复用 React、Vue、Svelte、Solid 或原生 Web 技术的团队；
- 使用 Leptos、Yew 等 Rust/WASM 前端的应用；
- 部分 Android、iOS 跨平台应用。

不一定适合：

- 强依赖完全一致浏览器渲染结果的应用；
- 大型 3D 游戏或专业实时渲染；
- 必须使用大量 Node.js 原生模块的 Electron 项目；
- 要求所有控件都是真正原生控件的桌面应用；
- 团队完全不熟悉 Web 安全、Rust 和跨平台打包。

## 2. 核心架构

```text
┌───────────────────────────────────────┐
│ WebView 前端                          │
│ HTML / CSS / JS / TS / WASM          │
│ React / Vue / Svelte / Leptos ...    │
└──────────────────┬────────────────────┘
                   │ IPC
                   │ Command / Event / Channel
┌──────────────────▼────────────────────┐
│ Tauri Core + Rust 应用逻辑             │
│ Window / Plugin / State / Permission  │
└──────────────────┬────────────────────┘
                   │
┌──────────────────▼────────────────────┐
│ 操作系统                              │
│ 文件、窗口、通知、菜单、进程、网络      │
└───────────────────────────────────────┘
```

安全边界非常重要：WebView 中的前端代码不应该自动拥有全部系统权限。前端只能调用已注册的 Command 或 Capability 明确允许的 API。

## 3. Tauri 的主要特点

### 3.1 前端框架无关

Tauri 只需要一份可以输出静态资源的前端。常见选择包括：

- Vanilla HTML/TypeScript；
- React；
- Vue；
- Svelte；
- Solid；
- Angular；
- Leptos、Yew、Sycamore。

Tauri 更适合 SPA、静态生成或传统多页面应用。它不是服务器端 SSR 运行时，Next.js、Nuxt 等框架需要使用静态或纯客户端模式。

### 3.2 Rust 本地后端

Rust 侧适合处理：

- 文件和目录；
- SQLite 或本地数据库；
- 加密与密钥管理；
- 大数据解析；
- 系统 API；
- 后台任务；
- 本地 HTTP、WebSocket 和进程通信；
- 对性能和可靠性要求较高的逻辑。

### 3.3 系统 WebView

Tauri 通过 WRY 抽象系统 WebView，通过 TAO 管理窗口和事件：

```text
Windows  通常使用 WebView2
macOS    使用 WKWebView
Linux    通常使用 WebKitGTK
```

优点是无需为每个应用捆绑完整浏览器；代价是不同平台的 WebView 版本、CSS 和系统行为可能不同。

### 3.4 权限与 Capability

Tauri 2 引入更明确的权限模型，可以限制：

- 哪个窗口或 WebView；
- 可以调用哪个插件命令；
- 可以访问哪些路径；
- 可以执行哪些 Sidecar；
- 可以使用哪些参数。

遵循最小权限原则是 Tauri 开发的核心，而不是发布前才补的配置。

### 3.5 插件生态

官方插件覆盖文件、Dialog、Shell、通知、日志、SQL、Store、Updater、Stronghold、深链接、单实例和窗口状态等能力。

## 4. 核心组件

### 4.1 `tauri`

核心 Rust crate，负责 Builder、窗口、WebView、IPC、State、事件、菜单、托盘和插件集成。

### 4.2 Tauri CLI

负责：

- 初始化项目；
- 开发运行；
- 编译 Rust 和前端；
- 生成图标；
- 添加插件；
- 构建安装包；
- 初始化移动平台。

### 4.3 TAO

跨平台窗口和事件循环抽象，负责窗口创建、菜单、托盘等原生桌面能力。

### 4.4 WRY

跨平台 WebView 抽象，负责选择和操作系统 WebView。

### 4.5 IPC

前端和 Rust 之间的消息通道，主要包括：

```text
Command  请求/响应，适合明确操作
Event    单向通知，松耦合但类型安全较弱
Channel  有序流式传输，适合进度和数据流
```

### 4.6 Plugins

插件通常同时包含：

- Rust 实现；
- JavaScript/TypeScript Guest API；
- 权限声明；
- 平台特定实现；
- 初始化代码。

### 4.7 `tauri.conf.json`

主要配置文件，包括：

- 应用名、版本和 Bundle Identifier；
- 开发服务器和前端构建目录；
- 窗口；
- CSP；
- 打包格式；
- 图标；
- Updater；
- Sidecar；
- 平台相关配置。

### 4.8 Capabilities 和 Permissions

`src-tauri/capabilities/` 中的文件描述哪些窗口可以使用哪些能力。插件命令即使安装并初始化，也可能因为没有权限或 Scope 而被拒绝。

## 5. 安装前提

先安装 Rust：

```bash
rustup update
```

使用 JavaScript 前端时还需要 Node.js LTS 或其他受支持的运行时和包管理器。

不同操作系统还需要平台依赖：

- Windows：MSVC 工具链与 WebView2；
- macOS：Xcode Command Line Tools；
- Linux：WebKitGTK 及相关开发库；
- Android：Android SDK、NDK 等；
- iOS：macOS、Xcode 和对应 SDK。

具体包名随发行版变化，应使用官方 Prerequisites 页面中的当前命令。

## 6. 创建第一个项目

使用 Cargo：

```bash
cargo install create-tauri-app --locked
cargo create-tauri-app
```

或者使用 Node 包管理器：

```bash
npm create tauri-app@latest
```

创建时会选择：

- 项目名称；
- 唯一 Bundle Identifier；
- 前端语言；
- 包管理器；
- UI 框架。

如果第一次学习，建议使用 Vanilla TypeScript，先理解 IPC 和安全边界，再引入大型前端框架。

启动开发环境：

```bash
cd tauri-app
npm install
npm run tauri dev
```

只使用 Cargo CLI 时：

```bash
cargo install tauri-cli --version '^2.0.0' --locked
cargo tauri dev
```

## 7. 项目结构

```text
tauri-app/
├── package.json
├── index.html
├── src/                         # Web 前端
│   ├── main.ts
│   └── ...
└── src-tauri/                   # Rust 应用
    ├── Cargo.toml
    ├── Cargo.lock
    ├── build.rs
    ├── tauri.conf.json
    ├── capabilities/
    │   └── default.json
    ├── icons/
    └── src/
        ├── lib.rs
        └── main.rs
```

Tauri 2 通常把应用组装写在 `src-tauri/src/lib.rs`。桌面端的 `main.rs` 调用库入口；移动端会把 Rust 项目作为库加载，因此不要把全部逻辑堆在 `main.rs`。

推荐继续拆分：

```text
src-tauri/src/
├── lib.rs
├── commands/
│   ├── mod.rs
│   ├── files.rs
│   └── settings.rs
├── services/
├── state.rs
├── error.rs
└── domain/
```

## 8. Rust Command

Command 是前端调用 Rust 的主要方式。

### 8.1 定义并注册 Command

```rust,ignore
use serde::Serialize;

#[derive(Serialize)]
#[serde(rename_all = "camelCase")]
struct Greeting {
    message: String,
}

#[tauri::command]
fn greet(name: String) -> Greeting {
    Greeting {
        message: format!("Hello, {name}!"),
    }
}

#[cfg_attr(mobile, tauri::mobile_entry_point)]
pub fn run() {
    tauri::Builder::default()
        .invoke_handler(tauri::generate_handler![greet])
        .run(tauri::generate_context!())
        .expect("error while running Tauri application");
}
```

所有 Command 必须一次性注册到同一个 `generate_handler!`。多次调用 `invoke_handler` 时，只有最后一次配置会生效。

### 8.2 前端调用

```ts
import { invoke } from '@tauri-apps/api/core';

type Greeting = {
  message: string;
};

const greeting = await invoke<Greeting>('greet', {
  name: 'Alice',
});

console.log(greeting.message);
```

默认参数对象使用 camelCase；可以通过 Command 属性改变重命名规则。输入需要实现 `Deserialize`，输出需要实现 `Serialize`。

### 8.3 异步 Command

```rust,ignore
#[tauri::command]
async fn load_document(path: String) -> Result<String, CommandError> {
    let contents = tokio::fs::read_to_string(path).await?;
    Ok(contents)
}
```

异步 Command 优先使用拥有所有权的参数，例如 `String`，而不是借用的 `&str`。

不要把 CPU 密集计算直接放入异步任务。使用受控线程池、`spawn_blocking` 或后台 Worker，避免阻塞运行时。

## 9. Command 错误设计

前端只应收到稳定、可序列化的错误：

```rust,ignore
use serde::Serialize;
use thiserror::Error;

#[derive(Debug, Error)]
enum AppError {
    #[error("file not found")]
    NotFound,

    #[error("I/O operation failed")]
    Io(#[from] std::io::Error),
}

#[derive(Debug, Serialize)]
#[serde(tag = "code", content = "message", rename_all = "snake_case")]
enum CommandError {
    FileNotFound(String),
    Internal(String),
}

impl From<AppError> for CommandError {
    fn from(error: AppError) -> Self {
        match error {
            AppError::NotFound => Self::FileNotFound("file not found".into()),
            AppError::Io(source) => {
                tracing::error!(error = ?source, "file operation failed");
                Self::Internal("operation failed".into())
            }
        }
    }
}
```

不要把文件路径、数据库错误、Secret、Token 或 Backtrace 直接返回前端。WebView 不是可信的秘密存储边界。

## 10. State 管理

通过 `manage` 注册 Rust 共享状态：

```rust,ignore
use std::sync::Mutex;
use tauri::State;

#[derive(Default)]
struct AppState {
    counter: u64,
}

#[tauri::command]
fn increment(state: State<'_, Mutex<AppState>>) -> Result<u64, String> {
    let mut state = state.lock().map_err(|_| "state lock poisoned")?;
    state.counter += 1;
    Ok(state.counter)
}

tauri::Builder::default()
    .manage(Mutex::new(AppState::default()))
    .invoke_handler(tauri::generate_handler![increment]);
```

最佳实践：

- State 保存长期共享资源，而不是所有业务数据；
- 不把一个巨大对象放进单个 Mutex；
- 缩小锁作用域；
- 不跨 `.await` 持有同步锁；
- 数据库连接池和 HTTP Client 通常可以直接廉价克隆；
- UI 临时状态留在前端状态管理中；
- Rust State 应通过 Service API 访问，而不是任意修改字段。

## 11. IPC 选择：Command、Event 与 Channel

### Command

适合请求/响应：

- 打开文档；
- 保存设置；
- 查询数据库；
- 执行明确业务操作。

Command 是首选，因为调用关系、参数、结果和错误更清晰。

### Event

适合生命周期和状态变化通知：

- 窗口状态变化；
- 文件被外部修改；
- 同步完成；
- 后台任务开始或结束。

事件是单向、动态、基于 JSON 的通信，不返回结果，类型安全弱于 Command。

前端监听后应在组件卸载时清理：

```ts
import { listen } from '@tauri-apps/api/event';

const unlisten = await listen<number>('download-progress', event => {
  console.log(event.payload);
});

// 页面或组件销毁时
unlisten();
```

### Channel

适合有顺序的数据流：

- 下载进度；
- 大文件分块；
- 流式 HTTP 响应；
- 子进程输出；
- 高频且需要保持顺序的更新。

不要用大量全局 Event 模拟流式协议。官方建议高吞吐、有序数据使用 Channel。

## 12. 窗口、WebView、菜单与托盘

Tauri 可以创建：

- 多窗口；
- 多 WebView；
- 原生应用菜单；
- 右键菜单；
- 系统托盘；
- 无边框和透明窗口；
- 启动画面和辅助窗口。

最佳实践：

- 每个窗口使用稳定且唯一的 label；
- 不让所有窗口拥有相同 Capability；
- 设置页只授予设置相关权限；
- 关闭与隐藏到托盘的行为明确告知用户；
- 多窗口通信优先使用定向 Event，而不是全局广播；
- 窗口创建、销毁和监听器解除应形成清晰生命周期。

## 13. 官方插件

推荐使用 `tauri add` 安装官方插件，它会更新 Rust、前端依赖和基础配置：

```bash
npm run tauri add dialog
npm run tauri add fs
npm run tauri add store
npm run tauri add updater
```

常用插件：

- `dialog`：文件选择和原生消息框；
- `fs`：受 Scope 控制的文件访问；
- `shell`：打开外部程序或执行 Sidecar；
- `store`：持久键值设置；
- `sql`：本地 SQL 数据库；
- `stronghold`：加密安全存储；
- `notification`：系统通知；
- `log`：日志；
- `single-instance`：单实例；
- `deep-link`：协议链接；
- `window-state`：保存窗口位置和大小；
- `updater`：应用内更新。

只安装真正需要的插件。每个插件都会扩大依赖、权限面、构建时间和需要测试的平台组合。

## 14. 文件访问

Rust 侧可以直接使用 `std::fs` 或 `tokio::fs`。前端使用文件插件时，需要同时满足：

1. 插件命令 Permission 已允许；
2. 路径位于允许 Scope；
3. 路径通过安全校验。

示例：

```ts
import { BaseDirectory, readTextFile } from '@tauri-apps/plugin-fs';

const text = await readTextFile('settings.json', {
  baseDir: BaseDirectory.AppConfig,
});
```

最佳实践：

- 优先使用 AppConfig、AppData、AppCache 等应用专属目录；
- 用户文件通过 Dialog 选择；
- 不授予整个 Home 目录读写权限；
- 不接受未经验证的绝对路径；
- 防止路径穿越和符号链接逃逸；
- 文件句柄使用完及时关闭；
- 写重要文件采用临时文件、刷新并原子替换；
- 不在资源目录写运行时数据。

## 15. Capability 与最小权限

典型 Capability：

```json
{
  "$schema": "../gen/schemas/desktop-schema.json",
  "identifier": "main-capability",
  "description": "Permissions for the main window",
  "windows": ["main"],
  "permissions": [
    "core:default",
    "dialog:allow-open",
    "store:default"
  ]
}
```

原则：

- 按窗口拆分 Capability；
- 只允许实际调用的命令；
- Permission 之外继续限制 Scope；
- Shell/Sidecar 参数使用明确 allowlist；
- 不用 `allow-all` 解决开发期报错；
- 定期删除不再使用的权限；
- CI 检查 Capability 变更。

Runtime Authority 会在 Command 执行前检查来源、Capability、Permission 和 Scope。不满足条件时，命令不会被调用。

## 16. CSP 与前端安全

Tauri UI 本质上仍是 Web 内容，因此仍可能发生 XSS。XSS 在桌面应用中更危险，因为攻击代码可能通过已授权 IPC 访问本地能力。

推荐：

- 启用严格 CSP；
- 优先打包本地脚本、字体和样式；
- 避免从 CDN 加载可执行脚本；
- 不使用不必要的 `unsafe-eval`；
- 用户 HTML 必须净化；
- 不通过 `innerHTML` 渲染不可信内容；
- IPC 输入在 Rust 侧再次校验；
- Capability 假设前端可能被攻破，仍保持最小权限。

示意配置：

```json
{
  "app": {
    "security": {
      "csp": {
        "default-src": "'self'",
        "connect-src": "ipc: http://ipc.localhost https://api.example.com",
        "img-src": "'self' asset: http://asset.localhost blob: data:",
        "style-src": "'self' 'unsafe-inline'"
      }
    }
  }
}
```

实际 CSP 必须根据前端构建方式调整。Rust/WASM 前端可能需要专门的 WASM CSP 来源。

## 17. Sidecar 与子进程

Sidecar 用于捆绑额外可执行文件，例如已有 CLI、Python 打包程序或本地服务。

必须同时配置：

- `tauri.conf.json` 中的 `bundle.externalBin`；
- Shell 插件；
- Capability 中的执行或 Spawn Permission；
- 允许的 Sidecar 名称和参数。

最佳实践：

- 能用 Rust 库实现时优先直接集成；
- 不把用户输入直接拼接为 Shell 命令；
- 使用参数数组，不通过 shell 字符串执行；
- 对动态参数使用严格模式或正则 Scope；
- 验证 Sidecar 来源和版本；
- 管理子进程退出、超时和异常重启；
- 使用 Channel 处理持续 stdout/stderr；
- 打包时为每个目标提供正确架构的二进制。

## 18. 数据存储与 Secret

不同数据选择不同存储：

```text
简单偏好设置        Store 插件
结构化本地业务数据   SQLite / SQL 插件 / Rust 数据库库
缓存                AppCache
普通应用数据         AppData / AppLocalData
密钥或敏感令牌       Stronghold 或系统密钥链方案
```

不要把长期 Secret 明文放在 `localStorage`、前端状态、日志或普通 JSON 设置文件中。前端 Bundle 和 WebView 代码不能保存真正的秘密。

数据库最佳实践：

- Schema 使用 Migration；
- 事务与业务用例一致；
- 数据库调用不要阻塞 UI 主线程；
- 对敏感数据考虑加密和系统密钥保护；
- 备份与恢复策略明确；
- 升级前兼容旧数据库版本。

## 19. 网络请求

可以选择：

- 前端 `fetch`；
- 官方 HTTP 插件；
- Rust `reqwest`；
- 自定义 Command 封装 API。

选择建议：

- 普通公开 API、无需本地 Secret：前端请求简单直接；
- 需要统一证书、代理、重试或本地凭据：Rust Client 更合适；
- 大文件或流式响应：Rust + Channel；
- 多个平台权限差异明显：使用官方插件。

不要把 API Secret 打包到前端或 Rust 二进制中。编译进程序的 Secret 仍能被逆向提取。

## 20. 后台任务与性能

### 避免阻塞主线程

同步 Command 默认可能在主线程执行。文件扫描、压缩、哈希和大型解析应使用异步 Command 或后台线程。

### 控制 IPC 成本

- 小型结构使用 Serde JSON；
- 大型二进制使用 `tauri::ipc::Response`；
- 流式数据使用 Channel；
- 不在高频循环中发送巨大 Event；
- 对进度更新节流。

### 减少前端负担

- 对大列表使用虚拟滚动；
- 不把整个数据库加载到 WebView；
- Rust 侧完成重计算，返回必要结果；
- 使用 Release 构建衡量性能；
- 分别分析前端渲染、IPC 和 Rust 执行时间。

## 21. 日志与可观察性

建议统一前端和 Rust 日志：

- Rust 使用 `tracing` 或官方日志插件；
- Release 环境设置合理级别；
- 日志写入 AppLog 并进行轮转；
- 记录 Command 名称、耗时和错误类别；
- 不记录 Token、密码、用户文档内容和完整路径；
- 崩溃报告遵守隐私和用户同意要求。

日志应该帮助定位问题，但不能成为敏感数据副本。

## 22. 测试策略

### Rust 单元测试

把业务逻辑放入普通模块，不依赖 Tauri Handle，即可使用常规 `cargo test`。

### Command 测试

Command 尽量只做参数转换和 Service 调用。核心 Service 使用 mock 或临时目录测试。

### 前端测试

组件和状态逻辑使用前端测试框架；将 `invoke` 封装成客户端接口，测试时替换为 mock。

```ts
export interface NativeApi {
  readDocument(path: string): Promise<string>;
}
```

### 端到端测试

在各目标平台测试：

- 首次启动；
- 文件 Dialog；
- 权限拒绝；
- 窗口和托盘；
- 安装、升级和卸载；
- 离线行为；
- 异常关闭后的恢复；
- 不同 DPI、主题和语言。

系统 WebView 的差异决定了不能只在开发者电脑验证。

## 23. 构建与发布

```bash
npm run tauri build
```

或：

```bash
cargo tauri build
```

Tauri 可以生成平台安装包，例如 DMG、App Bundle、MSI、NSIS、Deb/RPM/AppImage，以及移动平台产物。

发布前需要：

- 维护稳定 Bundle Identifier；
- 在 `tauri.conf.json` 管理版本；
- 准备各平台图标；
- macOS 签名和公证；
- Windows 代码签名；
- Android/iOS 签名；
- 在目标系统构建和验证安装包；
- 保存签名密钥但不提交仓库。

大多数平台要求或强烈建议代码签名。签名身份和密钥应由 CI Secret 或专门签名基础设施管理。

## 24. Updater

官方 Updater 插件可以检查、下载并安装更新。需要配置：

- 更新产物生成；
- 公钥；
- HTTPS Endpoint；
- 平台与架构对应产物；
- 签名。

最佳实践：

- 私钥离线或放在受保护 CI Secret；
- 应用只包含公钥；
- 生产 Endpoint 使用 HTTPS；
- 更新元数据和二进制均需要可信签名；
- 分阶段发布并提供回滚策略；
- 在真实旧版本上测试升级；
- 数据库和配置 Migration 向前兼容；
- 不假设用户一定按顺序安装每个版本。

## 25. 推荐架构

```text
Frontend Components
        │
Frontend Native API Client
        │ invoke / channel / event
Tauri Commands
        │
Application Services
        │
Domain / Repository / Platform Adapters
```

职责：

- 前端组件：展示和用户交互；
- Native API Client：集中封装 `invoke` 和 TypeScript 类型；
- Command：IPC 边界、权限与输入转换；
- Service：业务用例；
- Domain：不依赖 Tauri 的核心规则；
- Adapter：文件、数据库、HTTP 和系统 API。

Command 不应成为几百行的万能函数。保持核心逻辑与 Tauri 解耦，测试和未来迁移都会更容易。

## 26. 生产最佳实践清单

### 权限与安全

- 每个窗口单独定义最小 Capability；
- 文件和 Shell 权限配置精确 Scope；
- Rust 侧验证所有 IPC 输入；
- 启用严格 CSP；
- 不加载不可信远程脚本；
- Secret 不进入前端、Bundle 或日志；
- Sidecar 参数使用 allowlist；
- 定期审计依赖和权限。

### 稳定性

- Command 返回结构化错误；
- 后台任务支持取消和超时；
- 写文件采用安全替换策略；
- 数据库执行 Migration；
- 监听器在组件卸载时解除；
- 子进程和窗口有明确生命周期；
- 崩溃后能够恢复未完成状态。

### 性能

- 避免阻塞 UI 和 async runtime；
- 减少大型 JSON IPC；
- 流式数据使用 Channel；
- 复用数据库池和 HTTP Client；
- 大列表分页或虚拟化；
- 使用真实 Release 包进行性能测试。

### 跨平台

- 不手工拼接路径；
- 使用 Tauri Path/BaseDirectory API；
- 不假设大小写、路径分隔符和权限一致；
- 在目标系统测试 WebView、安装器和升级；
- 对平台特有能力使用 `cfg` 和清晰抽象。

## 27. 常见错误

### 27.1 把 WebView 当可信环境

一旦前端发生 XSS，攻击代码可能调用已授权的本地能力。Capability 必须遵循最小权限。

### 27.2 给文件插件整个磁盘权限

只允许应用目录或用户明确选择的路径。

### 27.3 把所有逻辑写在 Command

这会造成难测试、难复用和 IPC 层与业务层耦合。

### 27.4 用 Event 实现所有通信

请求/响应使用 Command，有序流使用 Channel，生命周期通知才使用 Event。

### 27.5 忘记清理监听器

SPA 页面切换可能累积监听器，导致重复处理和内存泄漏。

### 27.6 在前端保存长期 Token 或 Secret

`localStorage` 和前端 Bundle 都不是安全存储。使用 Stronghold 或系统凭据方案。

### 27.7 只在一个平台测试

系统 WebView、路径、权限、签名和安装器都有平台差异。

### 27.8 直接执行拼接后的 Shell 命令

这可能导致命令注入。优先调用 Rust API，Sidecar 使用固定程序和受限参数。

## 28. Tauri 与其他方案如何选择

### Tauri 与 Electron

选择 Tauri：

- 重视安装包、内存基线和 Rust 本地能力；
- 接受系统 WebView 差异；
- 愿意维护权限和 Rust 代码。

选择 Electron：

- 需要统一 Chromium 行为；
- 强依赖 Node.js 生态；
- 团队主要是 Web/Node 开发者；
- 包体和资源占用不是主要约束。

### Tauri 与 egui/iced

选择 Tauri：复用 Web UI、需要复杂排版或成熟前端生态。

选择 egui：即时模式工具、调试界面和数据可视化。

选择 iced：希望使用纯 Rust 和 Elm 风格状态模型。

## 29. 推荐学习路线

1. Rust 所有权、错误处理、Serde 与 async；
2. HTML/CSS/TypeScript 或一种前端框架；
3. 创建 Vanilla Tauri 项目；
4. 学习 Command 和结构化错误；
5. 学习 State、Event 和 Channel；
6. 添加 Dialog、FS、Store 插件；
7. 学习 Capability、Permission、Scope 和 CSP；
8. 完成本地 SQLite 应用；
9. 添加日志、自动更新和签名；
10. 在 Windows、macOS、Linux 真实构建测试。

## 30. 参考资料

- [Tauri 官方网站](https://tauri.app/)
- [创建项目](https://v2.tauri.app/start/create-project/)
- [项目结构](https://tauri.app/start/project-structure/)
- [Tauri 架构](https://v2.tauri.app/concept/architecture/)
- [前端调用 Rust](https://v2.tauri.app/develop/calling-rust/)
- [IPC](https://v2.tauri.app/concept/inter-process-communication/)
- [安全与 Runtime Authority](https://v2.tauri.app/security/runtime-authority/)
- [CSP](https://v2.tauri.app/security/csp/)
- [官方插件](https://v2.tauri.app/plugin/)
- [打包与发布](https://v2.tauri.app/distribute/)
- [Updater](https://v2.tauri.app/plugin/updater/)

## 总结

Tauri 的核心价值不是“用 Rust 写前端”，而是把成熟 Web UI 与 Rust 本地能力组合起来，同时通过 IPC 和 Capability 建立安全边界。

正确使用 Tauri 的关键是：

- 把前端视为潜在不可信输入源；
- 使用 Command、Event、Channel 各自适合的通信模型；
- Command 保持轻薄，业务逻辑放入普通 Rust Service；
- Capability、Permission 和 Scope 坚持最小权限；
- CSP、文件访问、Shell 和 Sidecar 从设计阶段考虑安全；
- 在真实平台验证 WebView、打包、签名和升级；
- 不把 Secret 编译进前端或二进制。

如果团队已有 Web 前端能力，同时希望获得跨平台桌面打包和 Rust 的安全、高性能本地能力，Tauri 是非常值得学习的方案。
