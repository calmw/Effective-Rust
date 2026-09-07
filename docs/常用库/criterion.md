# Rust 常用库：`criterion`

[`criterion`](https://bheisler.github.io/criterion.rs/book/) 是统计驱动的微基准测试库，通过预热、重复采样和统计分析帮助判断性能变化是否真实。

## 1. 添加开发依赖

```bash
cargo add criterion --dev
```

在 `Cargo.toml` 中声明 benchmark：

```toml
[[bench]]
name = "sorting"
harness = false
```

## 2. 编写基准测试

创建 `benches/sorting.rs`：

```rust
use criterion::{criterion_group, criterion_main, Criterion};
use std::hint::black_box;

fn sort_numbers(c: &mut Criterion) {
    c.bench_function("sort 100 integers", |b| {
        b.iter(|| {
            let mut values: Vec<_> = (0..100).rev().collect();
            values.sort_unstable();
            black_box(values);
        });
    });
}

criterion_group!(benches, sort_numbers);
criterion_main!(benches);
```

运行：

```bash
cargo bench
```

`black_box` 用于降低编译器把待测计算优化掉的可能性，但不能替代合理的基准设计。

## 3. 比较不同输入规模

```rust
use criterion::{BenchmarkId, Criterion};
use std::hint::black_box;

fn bench_sizes(c: &mut Criterion) {
    let mut group = c.benchmark_group("sum");

    for size in [100, 1_000, 10_000] {
        group.bench_with_input(BenchmarkId::from_parameter(size), &size, |b, &size| {
            let values: Vec<u64> = (0..size).collect();
            b.iter(|| black_box(&values).iter().sum::<u64>());
        });
    }

    group.finish();
}
```

多个输入规模能够揭示复杂度变化，而不仅是某个固定样本的速度。

## 4. 区分准备和测量

数据生成、文件创建等准备工作可能掩盖真正要测量的代码。需要每轮重新准备输入时，可以使用 `iter_batched` 或 `iter_batched_ref`，明确哪些操作计入测量。

## 5. 最佳实践

1. 在 release 优化条件下测量，使用 `cargo bench`。
2. 基准只测一个清晰问题，并固定输入数据和环境。
3. 同时测试多个有代表性的输入规模和数据分布。
4. 避免把测试数据创建、日志输出或磁盘波动意外算入核心算法耗时。
5. 关注置信区间和长期趋势，不只比较一次运行的单个数字。
6. 微基准改进必须结合真实应用 profiling，局部更快不一定让系统更快。
7. CI 机器噪声较大，性能门禁应留有合理阈值并使用稳定硬件。

## 6. 常见误区

- 测量 debug 构建，结果通常没有实际意义。
- 输入恒定且结果未使用，可能被编译器优化掉。
- 只测极小输入，最终比较的是函数调用或计时开销。
- 把 Criterion 当 profiler；它回答“这段代码多快”，profiling 回答“时间花在哪里”。

## 总结

Criterion 适合验证性能优化和发现回归。可信的基准需要代表性输入、明确测量边界、稳定环境和统计视角，而不只是调用一次计时函数。
