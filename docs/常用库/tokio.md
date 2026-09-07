# Rust 常用库：`tokio`

[`tokio`](https://tokio.rs/) 是 Rust 主流异步运行时，为异步任务提供调度器、网络 I/O、定时器、通道和异步同步原语。

`async fn` 只会创建 `Future`；Future 必须由 Tokio 这样的运行时驱动才会执行。

## 1. 添加依赖

学习或应用项目可以从完整 feature 开始：

```bash
cargo add tokio --features full
```

库和生产项目建议只开启需要的 feature，例如：

```bash
cargo add tokio --features macros,rt-multi-thread,time,sync
```

## 2. 创建异步入口

```rust
use std::time::Duration;

#[tokio::main]
async fn main() {
    tokio::time::sleep(Duration::from_millis(100)).await;
    println!("done");
}
```

`#[tokio::main]` 会创建运行时，并在其中执行异步 `main`。

## 3. 创建并等待任务

```rust
#[tokio::main]
async fn main() -> Result<(), tokio::task::JoinError> {
    let handle = tokio::spawn(async {
        20 + 22
    });

    let answer = handle.await?;
    assert_eq!(answer, 42);
    Ok(())
}
```

传给 `tokio::spawn` 的 Future 通常必须满足 `Send + 'static`。可以使用 `async move` 把所有权移入任务。

## 4. 并发执行

```rust
async fn fetch_user() -> &'static str { "Alice" }
async fn fetch_score() -> u32 { 100 }

#[tokio::main]
async fn main() {
    let (user, score) = tokio::join!(fetch_user(), fetch_score());
    assert_eq!((user, score), ("Alice", 100));
}
```

- `join!`：并发等待多个任务，全部完成后返回；
- `try_join!`：任一任务失败时提前返回错误；
- `select!`：等待多个分支中的第一个事件。

## 5. 不要阻塞异步线程

下面的操作会阻塞运行时工作线程：

```rust
// 不要在 async 代码中这样等待
std::thread::sleep(std::time::Duration::from_secs(1));
```

应使用 `tokio::time::sleep`。CPU 密集计算或无法替换的阻塞调用使用：

```rust
let result = tokio::task::spawn_blocking(|| {
    expensive_blocking_operation()
}).await?;
# fn expensive_blocking_operation() -> u64 { 42 }
# Ok::<(), tokio::task::JoinError>(())
```

## 6. 超时和取消

```rust
use std::time::Duration;

let result = tokio::time::timeout(
    Duration::from_secs(2),
    perform_request(),
).await;
# async fn perform_request() {}
```

任务被取消时 Future 会被丢弃，因此持有锁、临时状态和外部操作时要考虑取消安全性。

## 7. 最佳实践

1. 不要在异步代码中执行阻塞 I/O 或长时间 CPU 计算。
2. 为网络请求、锁等待和外部服务调用设置超时。
3. 保存并等待重要任务的 `JoinHandle`，不要默默丢弃任务错误。
4. 使用有界 `mpsc::channel` 提供背压，避免生产者无限占用内存。
5. 避免跨越 `.await` 持有 `std::sync::MutexGuard`；根据场景缩小锁作用域或使用 Tokio 锁。
6. 使用 `tracing` 观察并发任务，不依赖无上下文的 `println!`。
7. 库通常暴露 `async fn`，不要擅自为调用者创建嵌套运行时。

## 总结

Tokio 适合 I/O 密集型并发。正确使用的关键是避免阻塞运行时线程、明确处理任务生命周期，并为并发量、背压、超时和取消建立边界。
