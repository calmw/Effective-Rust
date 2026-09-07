# Rust 常用库：`itertools`

[`itertools`](https://docs.rs/itertools/latest/itertools/) 在标准 `Iterator` 之上提供额外的迭代器适配器、工具函数和宏，适合表达分组、去重、组合、批处理等数据转换。

## 1. 添加依赖

```bash
cargo add itertools
```

导入扩展 trait：

```rust
use itertools::Itertools;
```

## 2. 常用方法

### `join`：连接显示内容

```rust
use itertools::Itertools;

let text = ["red", "green", "blue"].iter().join(", ");
assert_eq!(text, "red, green, blue");
```

### `unique`：保持顺序去重

```rust
use itertools::Itertools;

let values = vec![3, 1, 3, 2, 1];
let unique: Vec<_> = values.into_iter().unique().collect();
assert_eq!(unique, vec![3, 1, 2]);
```

### `chunks`：按块处理

```rust
use itertools::Itertools;

let chunks = &(1..=5).chunks(2);
let result: Vec<Vec<_>> = chunks.into_iter()
    .map(|chunk| chunk.collect())
    .collect();

assert_eq!(result, vec![vec![1, 2], vec![3, 4], vec![5]]);
```

### `zip_longest`：合并不同长度的迭代器

```rust
use itertools::{EitherOrBoth, Itertools};

let pairs: Vec<_> = [1, 2].into_iter()
    .zip_longest(["a"])
    .collect();

assert_eq!(pairs[0], EitherOrBoth::Both(1, "a"));
assert_eq!(pairs[1], EitherOrBoth::Left(2));
```

其他常用方法包括 `sorted`、`group_by`/`chunk_by`、`tuple_windows`、`cartesian_product`、`collect_tuple` 和 `exactly_one`。

## 3. 惰性执行

大多数适配器都是惰性的，只有被 `collect`、`for_each`、`sum` 等操作消费时才执行：

```rust
let iter = (1..).map(|n| n * 2).take(3);
let values: Vec<_> = iter.collect();
assert_eq!(values, vec![2, 4, 6]);
```

## 4. 最佳实践

1. 标准库能清晰完成时优先标准 `Iterator`，减少不必要依赖。
2. 保持迭代器链易读；链过长时拆成具名变量或函数。
3. 能借用就不要过早 `cloned()`，能惰性处理就不要提前 `collect()`。
4. `unique`、排序、分组通常需要额外内存或比较成本，要关注大数据集。
5. 笛卡尔积、排列和组合的结果可能指数增长，使用前估算规模。
6. 返回公共 API 时可使用 `impl Iterator`，避免暴露复杂适配器具体类型。

## 总结

Itertools 让复杂迭代操作更接近问题本身的表达。最佳实践不是把所有循环改成一条超长链，而是在可读性、惰性执行和内存成本之间取得平衡。
