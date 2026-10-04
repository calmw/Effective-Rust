# egui 完整指南：特点、组件、使用方式与最佳实践

[`egui`](https://github.com/emilk/egui) 是一个纯 Rust、即时模式（immediate mode）的图形界面库，可以运行在 Windows、macOS、Linux 和 WebAssembly 上。它尤其适合开发者工具、数据可视化工具、编辑器、调试面板和游戏内工具。

如果要用 egui 构建一个完整的独立应用，通常还会使用官方应用框架 [`eframe`](https://docs.rs/eframe/)。简单理解：

```text
egui    负责界面、布局、控件、交互和绘制命令
eframe  负责窗口、事件循环、渲染后端、持久化以及 native/Web 启动
```

本文以 **egui/eframe 0.36** 为例。egui 仍会频繁调整 API，升级时应逐版本阅读 changelog，不能直接照搬旧教程。特别是当前 `eframe::App` 的主要回调是 `ui`；旧版本教程中常见的是 `update`。

## 1. egui 适合什么项目

egui 比较适合：

- 内部工具、运维工具和配置工具；
- 调试面板、性能分析器和游戏编辑器；
- 数据查看、图表和实时监控界面；
- 图片、音频、地图或节点编辑器；
- 需要快速迭代 UI 的原型；
- 希望业务逻辑与界面都使用 Rust 的桌面程序；
- 已有 `wgpu`、`winit`、`glow` 或游戏引擎，需要嵌入 GUI 的项目；
- 同时发布 native 和 WebAssembly 的工具型应用。

它不一定适合：

- 必须使用操作系统原生控件和原生外观的应用；
- 非常强调传统 HTML/CSS 布局、SEO 或复杂富文本排版的产品；
- 依赖完整无障碍体验且尚未完成专项验证的应用；
- 团队已经有成熟 Web 前端，希望直接复用现有页面的项目；
- 要求长期稳定 API、几乎不愿承担升级成本的项目。

如果界面主要由 Web 技术构成，可以考虑 Tauri；如果希望全部使用 Rust、界面偏工具型且需要自定义绘制，egui 通常更加直接。

## 2. 即时模式 GUI 是什么

传统保留模式 GUI 通常先创建按钮、文本框等对象，再修改对象属性。egui 的即时模式则在每次需要重绘时，重新执行一遍构建界面的 Rust 代码：

```rust,ignore
if ui.button("保存").clicked() {
    save();
}
```

这段代码同时描述了：

1. 当前帧要显示一个“保存”按钮；
2. 按钮被点击时要执行什么操作。

egui 会在内部根据控件的 `Id` 保存焦点、滚动位置、窗口位置等交互状态。真正的业务数据仍由应用自己持有：

```rust
struct MyApp {
    username: String,
    dark_mode: bool,
    tasks: Vec<String>,
}
```

即时模式的优点：

- UI 代码与业务分支放在一起，阅读和修改都很直接；
- 不需要保存按钮、标签等控件对象；
- 界面由当前状态生成，不容易出现“模型已变但控件忘记更新”；
- 动态工具界面和自定义绘制非常方便。

需要注意：

- 构建 UI 的代码可能每秒执行很多次；
- 不应该在 UI 回调中执行阻塞 I/O 或昂贵计算；
- 稳定 `Id`、重绘策略和大列表虚拟化会直接影响正确性与性能；
- “每帧执行”不等于每帧都要重新读取文件、请求网络或创建大对象。

## 3. egui 生态中的核心组件

### 3.1 `egui`

核心 GUI crate，提供：

- `Context`：一套 egui UI 的全局上下文；
- `Ui`：在某个区域内添加控件和进行布局；
- `Widget`：控件统一接口；
- `Response`：控件产生的点击、悬停、拖动和焦点响应；
- `Panel`、`Window`、`Area`、`ScrollArea` 等容器；
- `Painter`、`Shape`、`Color32` 等自定义绘制能力；
- 输入、字体、主题、图片和内存管理。

### 3.2 `eframe`

构建独立 egui 应用时最常用的官方框架，负责：

- 创建窗口和事件循环；
- native 与 WebAssembly 启动；
- 连接 `winit` 和渲染后端；
- 调用 `eframe::App`；
- 应用状态持久化；
- 多视口和平台集成。

### 3.3 `epaint` 与 `emath`

- `epaint`：形状、文字布局、纹理和绘制数据；
- `emath`：`Pos2`、`Vec2`、`Rect`、范围和 2D 变换等数学类型。

多数应用直接通过 `eframe::egui` 使用这些能力，不必单独添加依赖。

### 3.4 平台与渲染集成

- `egui-winit`：把 `winit` 窗口输入转换给 egui；
- `egui-wgpu`：使用 `wgpu` 渲染；
- `egui_glow`：使用 OpenGL/`glow` 渲染；
- `eframe`：把这些组件组合成开箱即用的应用框架。

只有在已有事件循环、渲染器或引擎中嵌入 egui 时，才通常需要直接操作集成 crate。

### 3.5 `egui_extras`

提供不适合放进核心 crate、依赖更重或仍偏实验性的功能，例如：

- `TableBuilder`；
- 固定尺寸条带布局；
- 图片加载器；
- 日期选择器；
- 代码高亮。

### 3.6 `egui_kittest`

官方 UI 测试工具。它基于 AccessKit 语义树查询控件，可以模拟点击和键盘输入，也支持图片快照测试。

## 4. 创建第一个 eframe 应用

### 4.1 创建项目

```bash
cargo new egui-demo
cd egui-demo
cargo add eframe
```

也可以从官方 [`eframe_template`](https://github.com/emilk/eframe_template) 开始，特别是同时支持 native 和 Web 时。实际项目应提交 `Cargo.lock`，并将 egui 生态 crate 保持在兼容版本。

### 4.2 最小完整示例

将 `src/main.rs` 改为：

```rust,ignore
use eframe::egui;

#[derive(Default)]
struct CounterApp {
    count: i32,
    name: String,
}

impl eframe::App for CounterApp {
    fn ui(&mut self, ui: &mut egui::Ui, _frame: &mut eframe::Frame) {
        egui::CentralPanel::default().show(ui, |ui| {
            ui.heading("egui 示例");

            ui.horizontal(|ui| {
                ui.label("名字：");
                ui.text_edit_singleline(&mut self.name);
            });

            if ui.button("加一").clicked() {
                self.count += 1;
            }

            ui.label(format!("你好，{}；当前计数：{}", self.name, self.count));
        });
    }
}

fn main() -> eframe::Result {
    let options = eframe::NativeOptions {
        viewport: egui::ViewportBuilder::default()
            .with_title("egui demo")
            .with_inner_size([640.0, 420.0]),
        ..Default::default()
    };

    eframe::run_native(
        "egui-demo",
        options,
        Box::new(|_cc| Ok(Box::<CounterApp>::default())),
    )
}
```

运行：

```bash
cargo run
```

正式体验性能时使用 release 构建：

```bash
cargo run --release
```

`run_native` 的第一个参数也是持久化目录选择的重要输入。应用发布后不要随意改变这个标识。

### 4.3 `run_ui_native` 适合快速原型

只需要一个闭包时可以使用：

```rust,ignore
fn main() -> eframe::Result {
    eframe::run_ui_native("quick-demo", Default::default(), |ui, _frame| {
        ui.heading("快速原型");
        ui.label("不需要先定义 App 类型");
    })
}
```

需要自定义持久化、初始化资源或组织复杂状态时，使用 `run_native` 和自己的 `App` 类型更合适。

## 5. 理解 `Context`、`Ui`、`Widget` 和 `Response`

### 5.1 `Context`

`egui::Context` 表示整个 egui 实例，可用于：

- 读取输入；
- 设置字体和主题；
- 请求重绘；
- 管理纹理；
- 打开额外 viewport；
- 访问 egui 的内部状态。

`Context` 克隆成本低，可以交给后台线程，任务完成后调用 `request_repaint()` 唤醒界面。

### 5.2 `Ui`

`Ui` 表示当前可布局、可绘制的区域。常用方法包括：

```rust,ignore
ui.label("普通文本");
ui.heading("标题");
ui.separator();
ui.button("按钮");
ui.horizontal(|ui| { /* 一行 */ });
ui.vertical(|ui| { /* 一列 */ });
ui.group(|ui| { /* 分组 */ });
```

不要长期保存 `&mut Ui`。它只在当前界面构建调用中有效。

### 5.3 `Widget`

所有控件都围绕 `Widget` trait 工作，其核心形式是：

```rust,ignore
pub trait Widget {
    fn ui(self, ui: &mut egui::Ui) -> egui::Response;
}
```

控件本身通常是临时 builder，不负责保存长期业务状态：

```rust,ignore
let response = ui.add(
    egui::Slider::new(&mut self.volume, 0.0..=1.0)
        .text("音量")
        .show_value(true),
);
```

### 5.4 `Response`

`Response` 描述一个控件在当前帧占用的区域和产生的交互：

```rust,ignore
let response = ui.button("执行");

if response.clicked() {
    self.run_job();
}

if response.hovered() {
    // 当前指针位于按钮上
}

response
    .on_hover_text("执行当前任务")
    .context_menu(|ui| {
        if ui.button("复制命令").clicked() {
            ui.ctx().copy_text("cargo run".to_owned());
            ui.close();
        }
    });
```

常用判断包括 `clicked()`、`double_clicked()`、`changed()`、`hovered()`、`dragged()`、`has_focus()` 和 `lost_focus()`。

## 6. 常用容器和布局

### 6.1 Panel

一个典型应用可以由顶部工具栏、左侧导航和中央内容组成：

当前 0.36 使用统一的 `Panel::top/bottom/left/right`。旧教程中的 `TopBottomPanel` 和 `SidePanel` 属于早期 API，阅读示例时要先核对版本。

```rust,ignore
impl eframe::App for MyApp {
    fn ui(&mut self, ui: &mut egui::Ui, _frame: &mut eframe::Frame) {
        egui::Panel::top("top_bar").show(ui, |ui| {
            ui.horizontal(|ui| {
                ui.heading("My App");
                ui.separator();
                ui.label("就绪");
            });
        });

        egui::Panel::left("navigation")
            .resizable(true)
            .default_size(180.0)
            .show(ui, |ui| {
                ui.selectable_value(&mut self.page, Page::Home, "首页");
                ui.selectable_value(&mut self.page, Page::Settings, "设置");
            });

        // CentralPanel 必须最后添加，它占用其他 Panel 剩余的区域。
        egui::CentralPanel::default().show(ui, |ui| {
            self.show_page(ui);
        });
    }
}
```

顶层 Panel 的添加顺序决定嵌套关系。规则是：

- 外层 Panel 先添加；
- `CentralPanel` 最后添加；
- 不要在一个顶层 Panel 的闭包内部再创建另一个顶层 Panel；
- 顶层 Panel 之后再显示浮动 `Window`。

如果应用只使用 eframe 提供的根 `Ui`，可以用 `CentralPanel::show(ui, ...)` 给根区域添加标准边距和背景。旧版 `show_inside` 在 0.36 中已重命名为 `show`。

### 6.2 Window、Area 和弹出内容

```rust,ignore
egui::Window::new("详情")
    .open(&mut self.details_open)
    .resizable(true)
    .show(ui.ctx(), |ui| {
        ui.label("这是应用内浮动窗口，不一定是操作系统窗口。");
    });
```

- `Window`：可拖动、可调整尺寸的 egui 浮动窗口；
- `Area`：自由定位的一块区域；
- viewport：真正的额外操作系统窗口，适合多窗口应用。

不要把 egui `Window` 和 native viewport 混为一谈。

### 6.3 横向、纵向和响应式布局

```rust,ignore
ui.horizontal_wrapped(|ui| {
    for tag in &self.tags {
        ui.label(format!("#{tag}"));
    }
});

if ui.available_width() > 700.0 {
    ui.columns(2, |columns| {
        self.show_editor(&mut columns[0]);
        self.show_preview(&mut columns[1]);
    });
} else {
    self.show_editor(ui);
    ui.separator();
    self.show_preview(ui);
}
```

不要大量依赖固定像素坐标。优先使用 `available_width()`、`available_size()`、布局容器以及可调整尺寸的 Panel。

### 6.4 Grid 与表格

少量规则数据可用 `Grid`：

```rust,ignore
egui::Grid::new("settings_grid")
    .num_columns(2)
    .striped(true)
    .show(ui, |ui| {
        ui.label("用户名");
        ui.text_edit_singleline(&mut self.username);
        ui.end_row();

        ui.label("超时");
        ui.add(egui::DragValue::new(&mut self.timeout_secs).suffix(" s"));
        ui.end_row();
    });
```

需要固定表头、可伸缩列和大量行时，使用 `egui_extras::TableBuilder` 更方便。

### 6.5 滚动区域和大列表

普通列表：

```rust,ignore
egui::ScrollArea::vertical().show(ui, |ui| {
    for item in &self.items {
        ui.label(item);
    }
});
```

数万条等高数据不要全部构建，使用 `show_rows` 只创建可见行：

```rust,ignore
let row_height = ui.spacing().interact_size.y;

egui::ScrollArea::vertical().show_rows(
    ui,
    row_height,
    self.items.len(),
    |ui, visible_range| {
        for index in visible_range {
            ui.label(&self.items[index]);
        }
    },
);
```

`show_rows` 假设行高固定。可变高度内容需要使用其他虚拟化策略或限制数据量。

## 7. 常用控件

### 7.1 文本与按钮

```rust,ignore
ui.heading("设置");
ui.label("普通说明");
ui.colored_label(egui::Color32::YELLOW, "警告");
ui.hyperlink_to("Rust", "https://www.rust-lang.org/");

if ui.button("保存").clicked() {
    self.save_requested = true;
}
```

需要不同样式时使用 `RichText`：

```rust,ignore
ui.label(
    egui::RichText::new("构建成功")
        .strong()
        .color(egui::Color32::GREEN),
);
```

### 7.2 文本输入

```rust,ignore
ui.text_edit_singleline(&mut self.title);

ui.add(
    egui::TextEdit::multiline(&mut self.body)
        .hint_text("请输入内容")
        .desired_rows(12),
);
```

提交表单时可以同时处理按钮与 Enter：

```rust,ignore
let response = ui.text_edit_singleline(&mut self.query);
let enter = ui.input(|input| input.key_pressed(egui::Key::Enter));

if ui.button("搜索").clicked() || (response.lost_focus() && enter) {
    self.start_search();
}
```

不要用每帧都为真的 `key_down` 替代一次性动作的 `key_pressed`。

### 7.3 数值与选择

```rust,ignore
ui.add(egui::Slider::new(&mut self.opacity, 0.0..=1.0).text("透明度"));
ui.add(egui::DragValue::new(&mut self.retries).range(0..=10));
ui.checkbox(&mut self.auto_save, "自动保存");

ui.radio_value(&mut self.mode, Mode::Fast, "快速");
ui.radio_value(&mut self.mode, Mode::Safe, "安全");

egui::ComboBox::from_label("主题")
    .selected_text(self.theme.to_string())
    .show_ui(ui, |ui| {
        ui.selectable_value(&mut self.theme, Theme::System, "跟随系统");
        ui.selectable_value(&mut self.theme, Theme::Dark, "深色");
        ui.selectable_value(&mut self.theme, Theme::Light, "浅色");
    });
```

### 7.4 状态和进度

```rust,ignore
ui.add(egui::ProgressBar::new(self.progress).show_percentage());

if self.loading {
    ui.horizontal(|ui| {
        ui.spinner();
        ui.label("正在加载……");
    });
}
```

进度值通常应规范到 `0.0..=1.0`，并在后台任务完成后请求重绘。

## 8. 状态管理与稳定 Id

### 8.1 业务状态放在应用结构体中

推荐把状态区分为三层：

```rust,ignore
struct MyApp {
    // 领域数据：文档、项目、记录
    document: Document,

    // 可持久化偏好：主题、字体大小
    preferences: Preferences,

    // 临时 UI 状态：筛选词、当前选择、任务接收端
    ui_state: UiState,
}
```

不要把整个应用数据塞进 egui 的 `Memory`。显式字段更容易测试、序列化和重构。

### 8.2 哪些状态由 egui 保存

egui 会使用控件 `Id` 管理部分交互状态，例如：

- 文本编辑光标和选择；
- CollapsingHeader 是否展开；
- Window 的位置和尺寸；
- ScrollArea 的滚动位置；
- 当前焦点和拖动目标。

### 8.3 动态列表必须使用稳定 Id

重复标签或动态增删元素可能产生 Id 冲突或状态错位：

```rust,ignore
for task in &mut self.tasks {
    ui.push_id(task.id, |ui| {
        ui.collapsing("详情", |ui| {
            ui.text_edit_multiline(&mut task.description);
        });
    });
}
```

最佳实践：

- 使用数据库主键、UUID 或不会随排序变化的业务 ID；
- 不要把数组下标当作可重排列表的长期 ID；
- 相同标题的 `CollapsingHeader` 使用 `id_salt` 或 `push_id`；
- 调试构建中的 Id 冲突警告不要忽略。

## 9. 自定义 Widget 与绘制

### 9.1 先从普通函数开始

复用一组控件时，不一定需要自定义 trait：

```rust,ignore
fn status_badge(ui: &mut egui::Ui, ok: bool) -> egui::Response {
    let (text, color) = if ok {
        ("正常", egui::Color32::GREEN)
    } else {
        ("异常", egui::Color32::RED)
    };

    ui.label(egui::RichText::new(text).strong().color(color))
}
```

如果控件需要 builder API 或希望传给 `ui.add(...)`，再实现 `Widget`。

### 9.2 实现 `Widget`

```rust,ignore
struct Toggle<'a> {
    value: &'a mut bool,
    label: &'a str,
}

impl egui::Widget for Toggle<'_> {
    fn ui(self, ui: &mut egui::Ui) -> egui::Response {
        let text = if *self.value { "ON" } else { "OFF" };
        let response = ui.button(format!("{}: {}", self.label, text));

        if response.clicked() {
            *self.value = !*self.value;
        }

        response
    }
}

ui.add(Toggle {
    value: &mut self.enabled,
    label: "服务",
});
```

### 9.3 使用 Painter

`allocate_painter` 同时申请布局空间、声明交互类型并返回画笔：

```rust,ignore
let desired_size = egui::vec2(160.0, 160.0);
let (response, painter) = ui.allocate_painter(
    desired_size,
    egui::Sense::click_and_drag(),
);

let center = response.rect.center();
let radius = response.rect.width().min(response.rect.height()) * 0.35;

painter.circle_filled(center, radius, egui::Color32::from_rgb(70, 120, 220));
painter.circle_stroke(
    center,
    radius,
    egui::Stroke::new(2.0, egui::Color32::WHITE),
);

if response.dragged() {
    self.offset += response.drag_delta();
}
```

自定义控件应先通过 `allocate_*` 申请空间和 `Response`，再绘制。不要只在任意屏幕坐标上画图而完全绕过布局和交互系统。

## 10. 主题、样式和中文字体

### 10.1 设置主题

```rust,ignore
cc.egui_ctx.set_theme(egui::Theme::Dark);
```

局部调整样式时使用 `scope`，避免污染整个应用：

```rust,ignore
ui.scope(|ui| {
    ui.visuals_mut().override_text_color = Some(egui::Color32::LIGHT_RED);
    ui.label("只在这个区域生效");
});
```

### 10.2 安装中文字体

egui 默认字体不覆盖完整中日韩字符集。把经过许可的 `.ttf` 或 `.otf` 放入项目，例如 `assets/fonts/NotoSansSC-Regular.otf`，然后在初始化时安装：

```rust,ignore
fn install_fonts(ctx: &egui::Context) {
    let mut fonts = egui::FontDefinitions::default();

    fonts.font_data.insert(
        "noto_sans_sc".to_owned(),
        std::sync::Arc::new(egui::FontData::from_static(include_bytes!(
            "../assets/fonts/NotoSansSC-Regular.otf"
        ))),
    );

    fonts
        .families
        .get_mut(&egui::FontFamily::Proportional)
        .unwrap()
        .insert(0, "noto_sans_sc".to_owned());

    fonts
        .families
        .get_mut(&egui::FontFamily::Monospace)
        .unwrap()
        .push("noto_sans_sc".to_owned());

    ctx.set_fonts(fonts);
}
```

在 `run_native` 的创建闭包中调用一次：

```rust,ignore
Box::new(|cc| {
    install_fonts(&cc.egui_ctx);
    Ok(Box::new(MyApp::new(cc)))
})
```

字体文件会进入程序体积。发布前检查字体许可证、字形覆盖、字号、Windows 缩放和高 DPI 显示效果。

## 11. 图片和资源

为常见图片格式启用加载器：

```toml
[dependencies]
eframe = "0.36"
egui_extras = { version = "0.36", features = ["image"] }
image = { version = "0.25", default-features = false, features = ["png", "jpeg"] }
```

初始化并显示内嵌图片：

```rust,ignore
fn new(cc: &eframe::CreationContext<'_>) -> Self {
    egui_extras::install_image_loaders(&cc.egui_ctx);
    Self::default()
}

fn show_logo(&self, ui: &mut egui::Ui) {
    ui.image(egui::include_image!("../assets/logo.png"));
}
```

实际支持的来源由 feature 决定：

- `image`：PNG、JPEG 等位图解码；
- `svg`：SVG；
- `file`：`file://`；
- `http`：`http://` 与 `https://`；
- `all_loaders`：一次启用全部加载器。

启用 `image` 仍需直接依赖 `image` crate 并选择需要的格式。不要为了方便启用所有格式和所有网络加载器，最小 feature 集能减少编译时间、体积和攻击面。

## 12. 后台任务与异步操作

eframe 不会替你提供一个通用异步业务运行时。最重要的规则是：**不要阻塞 UI 线程**。

下面用标准线程和 channel 执行任务：

```rust,ignore
use std::sync::mpsc::{self, Receiver};

struct MyApp {
    result_rx: Option<Receiver<Result<String, String>>>,
    result: Option<Result<String, String>>,
}

impl MyApp {
    fn start_job(&mut self, ctx: egui::Context) {
        let (tx, rx) = mpsc::channel();
        self.result_rx = Some(rx);

        std::thread::spawn(move || {
            let result = std::fs::read_to_string("config.toml")
                .map_err(|error| error.to_string());
            let _ = tx.send(result);
            ctx.request_repaint();
        });
    }

    fn poll_job(&mut self) {
        if let Some(rx) = &self.result_rx {
            if let Ok(result) = rx.try_recv() {
                self.result = Some(result);
                self.result_rx = None;
            }
        }
    }
}
```

项目已经使用 Tokio 时，可以在独立 runtime 上执行异步任务，再通过 channel 回传结果。不要在 UI 回调里调用 `block_on`，也不要持有 `MutexGuard` 跨越 `.await`。

后台任务实践：

- UI 状态只能在 UI 线程集中修改，后台线程传递消息或不可变结果；
- 为任务设计取消、超时和重复点击保护；
- 任务完成后调用克隆的 `Context::request_repaint()`；
- 高频进度消息做节流，不要每个字节都唤醒 UI；
- 应用关闭时明确处理线程和资源生命周期；
- 对 native 与 WASM 分别选择可用的任务实现。

## 13. 重绘和动画

在没有输入且没有主动请求时，eframe 可以让界面休眠。因此：

```rust,ignore
// 数据刚从后台到达，需要尽快显示：
ui.ctx().request_repaint();

// 一秒后刷新时钟，不必持续满帧运行：
ui.ctx().request_repaint_after(std::time::Duration::from_secs(1));
```

动画期间需要连续请求后续帧；动画结束后停止请求。若程序空闲时仍持续占用 CPU，可使用 `Context::repaint_causes()` 辅助查找是谁在请求重绘。

## 14. 持久化应用状态

启用依赖：

```toml
[dependencies]
eframe = { version = "0.36", features = ["persistence"] }
serde = { version = "1", features = ["derive"] }
```

只序列化适合长期保存的数据：

```rust,ignore
#[derive(serde::Serialize, serde::Deserialize)]
#[serde(default)]
struct Preferences {
    dark_mode: bool,
    font_size: f32,
}

impl Default for Preferences {
    fn default() -> Self {
        Self {
            dark_mode: true,
            font_size: 16.0,
        }
    }
}

struct MyApp {
    preferences: Preferences,
}

impl MyApp {
    fn new(cc: &eframe::CreationContext<'_>) -> Self {
        let preferences = cc
            .storage
            .and_then(|storage| eframe::get_value(storage, "preferences"))
            .unwrap_or_default();

        Self { preferences }
    }
}

impl eframe::App for MyApp {
    fn ui(&mut self, ui: &mut egui::Ui, _frame: &mut eframe::Frame) {
        ui.checkbox(&mut self.preferences.dark_mode, "深色模式");
    }

    fn save(&mut self, storage: &mut dyn eframe::Storage) {
        eframe::set_value(storage, "preferences", &self.preferences);
    }
}
```

native 上数据保存在文件系统，Web 上由浏览器本地存储承载。持久化格式也需要演进策略：

- 为新增字段提供默认值；
- 不保存 channel、文件句柄、GPU 资源和缓存；
- 关键数据使用显式 schema 版本和迁移；
- 用户文档不要只依赖应用自动状态文件；
- 敏感信息不要明文写入普通持久化存储。

## 15. 错误处理

不要只把错误输出到终端。桌面应用应把可操作的错误反馈给用户：

```rust,ignore
match save_document(&self.document) {
    Ok(()) => self.notice = Some(Notice::success("保存成功")),
    Err(error) => {
        tracing::error!(error = ?error, "保存文档失败");
        self.notice = Some(Notice::error(format!("保存失败：{error}")));
    }
}
```

推荐同时提供：

- 面向用户的简明提示；
- 可重试或修复操作；
- 面向开发者的完整错误链和日志；
- 避免重复弹出的错误状态；
- 后台任务 panic 的捕获和故障状态。

库层用具体错误类型，应用边界可使用 `anyhow` 汇总上下文。

## 16. 推荐的项目结构

中型应用不要把所有 UI 塞进 `main.rs`：

```text
src/
├── main.rs              # native 入口和 NativeOptions
├── lib.rs               # 可复用应用入口，Web 构建也可使用
├── app.rs               # MyApp、顶层页面切换
├── domain/              # 领域模型和业务规则，不依赖 egui
│   ├── mod.rs
│   └── document.rs
├── services/            # 文件、网络、数据库、后台任务
│   ├── mod.rs
│   └── storage.rs
├── ui/
│   ├── mod.rs
│   ├── pages/           # 页面级 UI
│   ├── components/      # 可复用控件
│   └── theme.rs
└── assets/
    ├── fonts/
    └── images/
```

依赖方向建议：

```text
ui -> application/service interface -> domain
                     |
                     v
              filesystem/network/db
```

领域逻辑尽量不依赖 `egui::Ui`、`Response` 或 `Context`。这样可以用普通单元测试覆盖业务规则，也便于以后替换前端。

## 17. 测试策略

### 17.1 业务逻辑使用普通单元测试

把“点击之后做什么”抽成可测试方法：

```rust
#[derive(Default)]
struct Counter {
    value: i32,
}

impl Counter {
    fn increment(&mut self) {
        self.value += 1;
    }
}

#[test]
fn increment_adds_one() {
    let mut counter = Counter::default();
    counter.increment();
    assert_eq!(counter.value, 1);
}
```

### 17.2 用 `egui_kittest` 测交互

```rust,ignore
use egui_kittest::{
    Harness,
    kittest::{NodeT, Queryable},
};

#[test]
fn checkbox_can_be_toggled() {
    let mut checked = false;
    let mut harness = Harness::new_ui(|ui| {
        ui.checkbox(&mut checked, "自动保存");
    });

    harness.get_by_label("自动保存").click();
    harness.run();

    assert!(checked);
}
```

优先断言语义和状态，图片快照只覆盖少数视觉回归场景。快照测试较慢、容易受渲染后端和操作系统影响，应固定窗口尺寸、主题、像素密度和测试 OS 配置。

### 17.3 人工跨平台验证

至少检查：

- Windows、macOS、Linux 中实际支持的平台；
- 100%、125%、150%、200% 缩放；
- 深色和浅色主题；
- 中文输入法、复制粘贴和组合键；
- 键盘导航和焦点顺序；
- 小窗口、超长文本和空数据；
- 睡眠唤醒、窗口关闭和状态恢复。

## 18. 性能最佳实践

1. **不要在 UI 回调中阻塞。** 文件、网络、数据库和大计算放到后台执行。
2. **不要误解即时模式。** 可以每帧重新描述控件，但不要每帧重新加载资源或构造昂贵数据。
3. **缓存计算结果。** 只有输入变化时才重新排序、解析、布局或生成网格数据。
4. **虚拟化大列表。** 等高行使用 `ScrollArea::show_rows`。
5. **控制重绘。** 静态页面不要无条件调用 `request_repaint()`。
6. **减少临时分配。** 热路径避免反复克隆大 `String`、`Vec` 和图片。
7. **纹理只上传一次。** 保存纹理句柄，不要每帧重新解码图片并传给 GPU。
8. **用 release 构建测量。** debug 构建不代表最终交互性能。
9. **先分析再优化。** 可结合 `profiling`、puffin 或系统 profiler 找出真正热点。
10. **复杂绘制才考虑并行 tessellation。** 不要盲目启用 `rayon` feature。

## 19. 可访问性和交互最佳实践

- 按钮使用表达动作的文本，不只依赖颜色或图标；
- 图标按钮增加 hover 提示和无障碍标签；
- 表单标签与输入框建立清晰关系；
- 为常用动作提供标准键盘快捷键；
- 不要使用颜色作为错误、成功或选中状态的唯一线索；
- 保证足够对比度和可点击区域；
- 测试 Tab 焦点、Enter、Escape、复制粘贴和屏幕阅读器语义；
- Web、移动端输入和无障碍支持存在平台差异，发布前必须按目标平台验证。

## 20. 常见错误

### 20.1 在 `ui` 中发起阻塞请求

错误：

```rust,ignore
if ui.button("下载").clicked() {
    self.data = reqwest::blocking::get(url)?.text()?;
}
```

结果是窗口卡住。应启动后台任务，显示 loading 状态，通过 channel 回传结果。

### 20.2 每帧重置状态

错误：

```rust,ignore
fn ui(&mut self, ui: &mut egui::Ui, _frame: &mut eframe::Frame) {
    let mut input = String::new();
    ui.text_edit_singleline(&mut input);
}
```

`input` 每次回调都会变回空字符串。应把它放到 `self` 中。

### 20.3 动态列表使用不稳定 Id

删除或排序后，下标变化会让展开状态、焦点或编辑状态跑到另一项。应使用稳定业务 ID。

### 20.4 Panel 顺序错误

`CentralPanel` 不是最后添加时，布局可能与预期不同。顶层 Panel 也不应互相嵌套。

### 20.5 无条件持续重绘

每帧调用 `request_repaint()` 会让静态界面持续占用 CPU。使用事件唤醒或 `request_repaint_after()`。

### 20.6 忽略 feature 和二进制体积

`all_loaders`、图片格式、持久化和不同渲染后端都会引入额外依赖。只启用实际需要的 feature。

### 20.7 从旧教程复制 API

egui 仍有较多 breaking changes。遇到 `App::update`、已移除方法或签名不一致时，先确认教程和项目版本，再查当前 docs.rs 与 changelog。

## 21. 一套可落地的开发流程

1. 使用 `eframe_template` 或最小 `run_native` 项目启动；
2. 先建立 `App` 状态模型，再让 UI 从状态渲染；
3. 用 Panel 搭出稳定的顶层布局；
4. 将页面拆成 `show_xxx(&mut self, ui: &mut Ui)` 方法；
5. 把领域逻辑和 I/O 从 UI 模块分离；
6. 为后台任务建立消息、取消和错误状态；
7. 尽早安装中文字体并测试 DPI；
8. 为动态控件设计稳定业务 ID；
9. 用普通单元测试覆盖领域逻辑，用 `egui_kittest` 覆盖关键交互；
10. 使用 release 构建在所有目标平台验证；
11. 锁定依赖版本，逐版本升级 egui 并阅读 changelog；
12. 发布前检查许可证、资源体积、崩溃日志和持久化迁移。

## 22. egui、Tauri、Iced 和 Slint 怎么选

- **egui**：即时模式、纯 Rust、工具界面和自定义绘制非常高效；外观不是系统原生控件。
- **Tauri**：使用 Web UI 加 Rust 本地能力，适合已有 Web 技术栈和内容型产品。
- **Iced**：偏 Elm 架构和声明式消息更新，适合重视单向数据流的 Rust GUI。
- **Slint**：有专用声明式 UI 语言，强调设计和嵌入式/桌面产品界面。

选择时先做一个包含真实列表、输入、后台任务、打包和中文显示的小原型，再根据团队能力、目标平台、可访问性、包体、升级成本和设计要求决定。

## 23. 总结

正确使用 egui 的核心不是堆叠控件，而是理解即时模式的状态边界：

- 用 `eframe` 负责应用生命周期，用 `egui` 描述界面；
- 业务状态放在应用结构体，控件通过借用直接编辑状态；
- 使用 `Response` 处理交互，使用稳定 `Id` 保存控件状态；
- UI 线程不做阻塞操作，后台任务完成后主动请求重绘；
- 大列表虚拟化，资源和昂贵计算缓存；
- Panel 顺序、字体、DPI、持久化和跨平台行为要尽早验证；
- 领域逻辑与 egui 解耦，关键交互使用 `egui_kittest` 测试；
- egui API 升级较快，依赖当前官方文档而不是旧代码片段。

## 24. 参考资料

- [egui 官方仓库](https://github.com/emilk/egui)
- [egui API 文档](https://docs.rs/egui/)
- [eframe API 文档](https://docs.rs/eframe/)
- [eframe 官方模板](https://github.com/emilk/eframe_template)
- [egui 在线 Demo](https://www.egui.rs/#demo)
- [egui 架构说明](https://github.com/emilk/egui/blob/main/ARCHITECTURE.md)
- [egui_extras API 文档](https://docs.rs/egui_extras/)
- [egui_kittest API 文档](https://docs.rs/egui_kittest/)
