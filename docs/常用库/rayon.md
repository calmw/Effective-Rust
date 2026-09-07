# Rust 常用库：`rayon`

[`rayon`](https://docs.rs/rayon/latest/rayon/) 是数据并行库，可以把许多顺序迭代计算改为并行执行，适合 CPU 密集型任务。

## 1. 添加依赖

```bash
cargo add rayon
```

## 2. 并行迭代器

```rust
use rayon::prelude::*;

let sum: u64 = (1_u64..=1_000_000)
    .into_par_iter()
    .map(|n| n * n)
    .sum();

assert!(sum > 0);
```

常见转换：

```text
iter()      -> par_iter()
iter_mut()  -> par_iter_mut()
into_iter() -> into_par_iter()
sort()      -> par_sort()
```

## 3. `join` 并行执行两个计算

```rust
use rayon::join;

fn fib(n: u32) -> u64 {
    if n < 2 {
        return n as u64;
    }

    if n < 20 {
        return fib(n - 1) + fib(n - 2);
    }

    let (a, b) = join(|| fib(n - 1), || fib(n - 2));
    a + b
}
```

实际代码需要设置合理的顺序执行阈值；如果每个小任务都并行拆分，调度成本会超过收益。

## 4. `map`、`reduce` 和共享状态

优先让每个线程产生局部结果，再归并：

```rust
use rayon::prelude::*;

let total = (1_u64..=1000)
    .into_par_iter()
    .map(|n| n * n)
    .reduce(|| 0, |left, right| left + right);
```

不要让所有并行任务频繁争抢同一个 `Mutex`，否则会抵消并行收益。

## 5. 与 Tokio 的区别

```text
Tokio  -> 大量等待网络、磁盘、定时器等 I/O 并发
Rayon  -> 图像处理、压缩、解析、计算等 CPU 并行
```

在 Tokio 服务中执行较长的 Rayon 计算时要评估线程池资源和并发上限，避免 CPU 被请求无限占满。

## 6. 最佳实践

1. 先用基准测试确认并行化确实更快。
2. 任务必须足够大；微小任务的调度和同步成本可能更高。
3. 避免并行循环中使用单一全局锁或执行阻塞网络 I/O。
4. 不依赖未声明的执行顺序；并行迭代的完成顺序可能变化。
5. 控制服务端 CPU 并行任务数量，防止多个请求同时耗尽核心。
6. 浮点归约的计算顺序变化可能导致微小结果差异。
7. 特殊场景使用 `ThreadPoolBuilder` 建立独立线程池，而不是随意修改全局池。

## 总结

Rayon 擅长将独立的数据计算分发到多个 CPU 核心。只有任务粒度足够、共享状态较少且经过基准验证时，并行化才会带来稳定收益。
