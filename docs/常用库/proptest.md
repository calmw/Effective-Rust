# Rust 常用库：`proptest`

[`proptest`](https://proptest-rs.github.io/proptest/) 是属性测试框架。开发者描述“所有有效输入都应满足的性质”，框架自动生成大量输入；发现失败后，它还会尝试缩小为更简单的反例。

## 1. 添加开发依赖

```bash
cargo add proptest --dev
```

## 2. 基本属性测试

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn reverse_twice_returns_original(values in proptest::collection::vec(any::<i32>(), 0..100)) {
        let mut reversed = values.clone();
        reversed.reverse();
        reversed.reverse();

        prop_assert_eq!(reversed, values);
    }
}
```

这里测试的不是几个固定例子，而是“任意符合策略的向量反转两次都等于原值”。

## 3. 使用策略约束输入

```rust
use proptest::prelude::*;

fn username() -> impl Strategy<Value = String> {
    "[a-z][a-z0-9_]{2,15}"
}

proptest! {
    #[test]
    fn generated_username_is_not_empty(name in username()) {
        prop_assert!(!name.is_empty());
        prop_assert!(name.len() <= 16);
    }
}
```

策略应尽量生成有效且有意义的数据，而不是大量使用 `prop_assume!` 丢弃输入。

## 4. 适合测试的性质

- 往返：`decode(encode(value)) == value`；
- 幂等：`normalize(normalize(value)) == normalize(value)`；
- 不变量：排序后长度不变且元素集合不变；
- 等价：优化实现与简单参考实现结果一致；
- 边界安全：任意输入都不 panic；
- 状态机：一系列操作后数据结构仍保持约束。

## 5. 最佳实践

1. 示例测试与属性测试结合：示例负责明确业务案例，属性负责探索输入空间。
2. 属性要描述业务不变量，不要复制被测实现的算法。
3. 为复杂类型组合可复用 Strategy。
4. 尽量通过策略生成有效数据，减少 `prop_assume!` 导致的高拒绝率。
5. CI 中保存失败种子，使反例能够稳定复现。
6. 对时间、线程和随机性敏感的代码先隔离不确定依赖。
7. 属性测试不能证明程序绝对正确，关键边界值仍应写明确的示例测试。

## 总结

Proptest 的价值在于自动探索开发者没有想到的输入组合，并把失败缩小为易理解的反例。最重要的是找到稳定、独立于实现细节的业务性质。
