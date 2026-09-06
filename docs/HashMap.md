# Rust 中的 `HashMap`：从入门到正确使用

`HashMap<K, V>` 是 Rust 标准库提供的哈希表，用来保存“键—值”映射。它适合通过键快速查询、插入和删除数据，例如：

- 用户 ID 到用户信息；
- 单词到出现次数；
- 配置名称到配置值；
- 缓存键到计算结果。

`HashMap` 位于 `std::collections` 中：

```rust
use std::collections::HashMap;
```

## 1. 基本用法

### 1.1 创建与写入

```rust
use std::collections::HashMap;

fn main() {
    // 类型也可以由后续的 insert 推断出来。
    let mut scores = HashMap::new();

    scores.insert("Alice", 95);
    scores.insert("Bob", 88);

    assert_eq!(scores.len(), 2);
}
```

只有声明为 `mut` 的 `HashMap` 才能执行插入、删除等修改操作。

如果事先知道大致会有多少个元素，可以预分配容量，减少扩容和重新哈希的次数：

```rust
use std::collections::HashMap;

let mut cache = HashMap::with_capacity(1_000);
cache.insert(1, "one");
```

### 1.2 从数组或迭代器创建

少量固定数据可以使用 `HashMap::from`：

```rust
use std::collections::HashMap;

let status_codes = HashMap::from([
    (200, "OK"),
    (404, "Not Found"),
    (500, "Internal Server Error"),
]);
```

动态数据通常通过 `collect` 创建：

```rust
use std::collections::HashMap;

let names = ["Alice", "Bob"];
let scores = [95, 88];

let score_map: HashMap<_, _> = names.into_iter().zip(scores).collect();

assert_eq!(score_map.get("Alice"), Some(&95));
```

这里的类型标注 `HashMap<_, _>` 用于告诉 `collect` 要构造哪种集合，键和值的具体类型仍由编译器推断。

### 1.3 `into_iter`、`zip` 和 `collect` 分别做了什么

下面这行代码把两组相互对应的数据组合成一个 `HashMap`：

```rust
use std::collections::HashMap;

let names = ["Alice", "Bob"];
let scores = [95, 88];
let score_map: HashMap<_, _> = names.into_iter().zip(scores).collect();

assert_eq!(score_map.get("Alice"), Some(&95));
```

可以将它理解成一条由三个步骤组成的流水线：

```text
names                         ["Alice", "Bob"]
  │ into_iter()
  ▼
姓名迭代器                     "Alice", "Bob"
  │ zip(scores)
  ▼
键值对迭代器                   ("Alice", 95), ("Bob", 88)
  │ collect::<HashMap<_, _>>()
  ▼
HashMap                       {"Alice": 95, "Bob": 88}
```

#### `into_iter`：把集合转换为迭代器

`into_iter()` 来自 `IntoIterator` trait。它会取得调用者的所有权，并依次产生其中的元素：

```rust
let names = vec![String::from("Alice"), String::from("Bob")];
let mut iterator = names.into_iter();

assert_eq!(iterator.next(), Some(String::from("Alice")));
assert_eq!(iterator.next(), Some(String::from("Bob")));
assert_eq!(iterator.next(), None);

// names 已被移动，不能再使用。
```

对于 `Vec<String>`，迭代器产生的是拥有所有权的 `String`，因此原来的 `Vec` 会被消耗。常见的三种遍历方式是：

- `collection.into_iter()`：取得集合所有权，通常产生拥有所有权的元素 `T`；
- `collection.iter()`：借用集合，产生不可变引用 `&T`；
- `collection.iter_mut()`：可变借用集合，产生可变引用 `&mut T`。

如果后面不再需要原集合，使用 `into_iter()` 可以直接移动元素，避免克隆。如果还要继续使用原集合，则应使用 `iter()`，并决定新的 `HashMap` 是保存引用还是克隆后的独立数据。

#### `zip`：把两个迭代器逐项配对

`zip` 会从左右两个迭代器中各取一个元素，组成二元组：

```rust
let names = ["Alice", "Bob"];
let scores = [95, 88];

let pairs: Vec<_> = names.into_iter().zip(scores).collect();

assert_eq!(pairs, vec![("Alice", 95), ("Bob", 88)]);
```

这里写 `.zip(scores)` 而不是 `.zip(scores.into_iter())` 也可以，因为 `zip` 接受任何实现了 `IntoIterator` 的值，并会自动将 `scores` 转换为迭代器。

需要注意：如果两个迭代器长度不同，`zip` 会在较短的一方结束时停止，多余元素会被忽略，并且不会报错：

```rust
let names = ["Alice", "Bob", "Carol"];
let scores = [95, 88];

let pairs: Vec<_> = names.into_iter().zip(scores).collect();

assert_eq!(pairs, vec![("Alice", 95), ("Bob", 88)]);
// "Carol" 没有对应分数，因此没有出现在结果中。
```

如果业务要求两个输入长度必须一致，应在 `zip` 前显式检查：

```rust
let names = ["Alice", "Bob"];
let scores = [95, 88];

assert_eq!(names.len(), scores.len(), "姓名与分数数量不一致");
```

#### `collect`：把迭代器收集为目标容器

`collect()` 会消费迭代器，并根据上下文构造目标类型。构造 `HashMap<K, V>` 时，迭代器的每一项必须是 `(K, V)`：

```rust
use std::collections::HashMap;

let pairs = [("Alice", 95), ("Bob", 88)];
let scores: HashMap<&str, i32> = pairs.into_iter().collect();
```

同一个迭代器也可以被收集成其他容器，因此编译器通常需要目标类型提示。以下两种写法等价：

```rust
use std::collections::HashMap;

let pairs = [("Alice", 95), ("Bob", 88)];

let first: HashMap<_, _> = pairs.into_iter().collect();
let second = pairs.into_iter().collect::<HashMap<_, _>>();

assert_eq!(first, second);
```

如果迭代器中出现重复键，后出现的值会覆盖先出现的值：

```rust
use std::collections::HashMap;

let scores: HashMap<_, _> = [
    ("Alice", 90),
    ("Bob", 88),
    ("Alice", 95),
]
.into_iter()
.collect();

assert_eq!(scores.get("Alice"), Some(&95));
assert_eq!(scores.len(), 2);
```

如果重复键代表非法输入，应在收集前检查，或逐项调用 `insert` 并检查其返回值，不要无意中接受覆盖。

#### 等价的 `for` 循环

这条迭代器流水线大致等价于下面的代码：

```rust
use std::collections::HashMap;

let names = ["Alice", "Bob"];
let scores = [95, 88];
let mut score_map = HashMap::new();

for (name, score) in names.into_iter().zip(scores) {
    score_map.insert(name, score);
}

assert_eq!(score_map.get("Alice"), Some(&95));
```

迭代器写法更紧凑，`for` 循环则便于加入数据校验、日志记录或重复键处理。两者没有绝对优劣，应根据逻辑复杂度选择。

## 2. 查询数据

### 2.1 使用 `get`

`get` 返回 `Option<&V>`，因为目标键可能不存在：

```rust
use std::collections::HashMap;

let scores = HashMap::from([("Alice", 95)]);

match scores.get("Alice") {
    Some(score) => println!("Alice 的分数是 {score}"),
    None => println!("没有 Alice 的成绩"),
}
```

在只需要一个默认值时，可以写得更紧凑：

```rust
use std::collections::HashMap;

let scores = HashMap::from([("Alice", 95)]);
let score = scores.get("Bob").copied().unwrap_or(0);
assert_eq!(score, 0);
```

`copied()` 将 `Option<&i32>` 转换为 `Option<i32>`。对于不能直接复制的值，可以根据需要使用 `cloned()`，或者继续持有引用以避免不必要的克隆。

### 2.2 不要随意使用索引语法

`map[&key]` 在键不存在时会 panic，而且只能取得不可变引用：

```rust
use std::collections::HashMap;

let scores = HashMap::from([("Alice", 95)]);

assert_eq!(scores["Alice"], 95);
// scores["Bob"] 会 panic。
```

除非能够严格保证键一定存在，否则应优先使用 `get`，显式处理 `None`。

### 2.3 使用借用形式查询

当键是 `String` 时，通常不需要为了查询而创建或克隆一个新的 `String`。可以直接使用 `&str`：

```rust
use std::collections::HashMap;

let mut users: HashMap<String, u64> = HashMap::new();
users.insert("alice".to_owned(), 1);

assert_eq!(users.get("alice"), Some(&1));
assert!(users.contains_key("alice"));
```

这是因为 `String` 可以借用为 `str`，并且两者具有一致的哈希和相等语义。

## 3. 插入、覆盖与条件更新

### 3.1 `insert` 会返回旧值

如果键已存在，`insert` 会覆盖旧值，并通过 `Option<V>` 返回被替换的值：

```rust
use std::collections::HashMap;

let mut scores = HashMap::new();

assert_eq!(scores.insert("Alice", 90), None);
assert_eq!(scores.insert("Alice", 95), Some(90));
assert_eq!(scores.get("Alice"), Some(&95));
```

不要忽略这个返回值，除非“无条件覆盖”确实是预期行为。

### 3.2 使用 `entry` 避免重复查询

只在键不存在时插入默认值：

```rust
use std::collections::HashMap;

let mut scores = HashMap::from([("Alice", 95)]);

scores.entry("Alice").or_insert(0);
scores.entry("Bob").or_insert(0);

assert_eq!(scores["Alice"], 95);
assert_eq!(scores["Bob"], 0);
```

`or_insert` 返回值的可变引用，因此可以直接原地更新。下面是经典的词频统计：

```rust
use std::collections::HashMap;

fn word_counts(text: &str) -> HashMap<&str, usize> {
    let mut counts = HashMap::new();

    for word in text.split_whitespace() {
        *counts.entry(word).or_insert(0) += 1;
    }

    counts
}

fn main() {
    let counts = word_counts("rust is fast and rust is safe");
    assert_eq!(counts.get("rust"), Some(&2));
    assert_eq!(counts.get("is"), Some(&2));
}
```

如果默认值的构造成本较高，使用 `or_insert_with` 延迟计算；闭包只会在键不存在时执行：

```rust
use std::collections::HashMap;

let mut map = HashMap::new();
let key = "timeout";
let value = map.entry(key).or_insert_with(|| 30);

assert_eq!(*value, 30);
```

当默认值依赖于键本身时，可以使用 `or_insert_with_key`：

```rust
use std::collections::HashMap;

let mut lengths = HashMap::new();
lengths
    .entry(String::from("hello"))
    .or_insert_with_key(|key| key.len());
```

对已有值更新、对缺失值初始化时，`and_modify` 很合适：

```rust
use std::collections::HashMap;

let mut counts = HashMap::new();

for word in ["rust", "safe", "rust"] {
    counts
        .entry(word)
        .and_modify(|count| *count += 1)
        .or_insert(1);
}

assert_eq!(counts["rust"], 2);
```

相比先调用 `contains_key` 再调用 `get_mut` 或 `insert`，`entry` 通常更清晰，也避免了对同一个键进行重复查找。

## 4. 修改与删除

### 4.1 修改已有值

使用 `get_mut` 获取可变引用：

```rust
use std::collections::HashMap;

let mut scores = HashMap::from([("Alice", 95)]);

if let Some(score) = scores.get_mut("Alice") {
    *score += 1;
}

assert_eq!(scores["Alice"], 96);
```

### 4.2 删除数据

`remove` 返回被删除的值；键不存在时返回 `None`：

```rust
use std::collections::HashMap;

let mut scores = HashMap::from([("Alice", 95), ("Bob", 88)]);

assert_eq!(scores.remove("Bob"), Some(88));
assert_eq!(scores.remove("Bob"), None);
```

如果还需要取回原来的键，可以使用 `remove_entry`：

```rust
use std::collections::HashMap;

let mut scores = HashMap::from([("Alice", 95)]);
let removed = scores.remove_entry("Alice");
assert_eq!(removed, Some(("Alice", 95)));
```

批量保留满足条件的元素可使用 `retain`：

```rust
use std::collections::HashMap;

let mut scores = HashMap::from([("Alice", 95), ("Bob", 58)]);
scores.retain(|_name, score| *score >= 60);
assert!(!scores.contains_key("Bob"));
```

## 5. 所有权与借用

这是 Rust `HashMap` 最容易出错、也最值得理解的部分。

### 5.1 插入拥有所有权的值会发生移动

对于 `String` 等非 `Copy` 类型，插入后所有权会转移到 `HashMap`：

```rust
use std::collections::HashMap;

let name = String::from("Alice");
let city = String::from("Shanghai");

let mut users = HashMap::new();
users.insert(name, city);

// name 和 city 的所有权已经移动，之后不能再使用。
```

如果插入后仍需独立拥有原值，可以显式调用 `clone`，但应意识到它可能产生堆分配和数据复制：

```rust
use std::collections::HashMap;

let name = String::from("Alice");
let city = String::from("Shanghai");
let mut users = HashMap::new();

users.insert(name.clone(), city.clone());

assert_eq!(users.get("Alice"), Some(&city));
```

对于整数等实现了 `Copy` 的类型，插入的是副本，原变量仍可继续使用。

### 5.2 存储引用时要满足生命周期要求

`HashMap` 可以保存引用，但它不能比所引用的数据存活得更久：

```rust
use std::collections::HashMap;

let name = String::from("Alice");
let city = String::from("Shanghai");

let mut users = HashMap::new();
users.insert(name.as_str(), city.as_str());

assert_eq!(users.get("Alice"), Some(&"Shanghai"));
```

如果映射需要被返回、长期保存或跨线程传递，拥有自己的 `String` 往往比保存临时引用更简单。

### 5.3 引用存在时不能随意修改映射

从映射中取得引用后，该引用的有效期内不能再对映射执行可能改变其内部存储的操作：

```rust,compile_fail
use std::collections::HashMap;

let mut scores = HashMap::from([("Alice", 95)]);
let alice_score = scores.get("Alice").unwrap();

scores.insert("Bob", 88); // 编译错误：此时仍持有 alice_score。
println!("{alice_score}");
```

原因是插入操作可能触发扩容并移动元素，导致旧引用失效。通常可以缩短引用的作用域，或先复制出所需的小值：

```rust
use std::collections::HashMap;

let mut scores = HashMap::from([("Alice", 95)]);
let alice_score = scores.get("Alice").copied();
scores.insert("Bob", 88);
println!("{alice_score:?}");
```

## 6. 遍历与顺序

可以分别遍历键值对、键和值：

```rust
use std::collections::HashMap;

let mut scores = HashMap::from([("Alice", 95), ("Bob", 88)]);

for (name, score) in &scores {
    println!("{name}: {score}");
}

for score in scores.values_mut() {
    *score += 1;
}

for name in scores.keys() {
    println!("{name}");
}
```

`HashMap` **不保证遍历顺序**，顺序可能在不同运行、不同版本或数据变化后改变。因此：

- 不要编写依赖遍历顺序的业务逻辑；
- 测试时不要直接假定输出顺序；
- 需要稳定输出时，先收集并排序；
- 需要始终按键排序时，考虑使用 `BTreeMap`。

例如，稳定输出字符串键：

```rust
use std::collections::HashMap;

let scores = HashMap::from([("Alice", 95), ("Bob", 88)]);
let mut entries: Vec<_> = scores.iter().collect();
entries.sort_unstable_by_key(|(name, _score)| *name);

for (name, score) in entries {
    println!("{name}: {score}");
}
```

## 7. 键必须满足什么条件

`HashMap` 的键需要实现 `Eq` 和 `Hash`。常用的整数、布尔值、字符、`String`、`&str`、元组等通常已经实现了它们。

自定义类型可以派生这些 trait：

```rust
use std::collections::HashMap;

#[derive(Debug, Hash, PartialEq, Eq)]
struct UserId {
    tenant: u32,
    id: u64,
}

let mut users = HashMap::new();
users.insert(UserId { tenant: 1, id: 42 }, "Alice");
```

键必须遵守以下一致性规则：

1. 如果两个键通过 `Eq` 比较相等，它们的哈希值也必须相等；
2. 键放入映射后，参与相等比较和哈希计算的内容不应改变。

不要为同一类型手写相互矛盾的 `Eq` 和 `Hash` 实现。也不要通过 `Cell`、`RefCell` 等内部可变性修改已插入键的哈希或相等语义，否则可能出现错误查询、panic 等不可预测的局部结果。

浮点数 `f32` 和 `f64` 没有实现 `Eq`，主要原因是 `NaN` 不等于自身，因此不能直接作为标准 `HashMap` 的键。如果业务确实需要，应先定义明确的浮点相等语义，或使用提供全序包装类型的第三方库。

## 8. 性能与容量管理

在平均情况下，`get`、`insert` 和 `remove` 的时间复杂度接近 `O(1)`。不过，实际性能还会受到哈希函数、冲突数量、扩容、键大小和缓存局部性等因素影响。

正确的优化顺序通常是：先写清晰正确的代码，再测量，再针对热点优化。

常用的容量方法包括：

```rust
use std::collections::HashMap;

let mut map: HashMap<u64, String> = HashMap::with_capacity(100);
map.reserve(50);      // 确保至少还能容纳一定数量的元素。
map.shrink_to_fit();  // 尝试释放多余容量。
```

实践建议：

- 已知数据规模时使用 `with_capacity`；
- 批量插入前使用 `reserve`；
- 不要频繁调用 `shrink_to_fit`，重新分配也有成本；
- 不要为了避免移动而盲目克隆大键或大值；
- 只有性能分析证明哈希函数是瓶颈时，才考虑更换哈希器。

标准 `HashMap` 默认使用带随机种子的哈希器，以抵御针对哈希冲突的拒绝服务攻击。自定义更快的哈希器可能改变安全特性，不应只因微基准更快就在不可信输入场景中替换默认实现。

## 9. 多线程使用

`HashMap` 本身不提供并发写入能力。多个线程只读共享时，可以使用 `Arc<HashMap<K, V>>`；需要多个线程修改时，通常使用 `Arc<Mutex<HashMap<K, V>>>` 或 `Arc<RwLock<HashMap<K, V>>>`：

```rust
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

let shared = Arc::new(Mutex::new(HashMap::<String, usize>::new()));
let worker_map = Arc::clone(&shared);

let worker = std::thread::spawn(move || {
    let mut map = worker_map.lock().expect("mutex poisoned");
    *map.entry("jobs".to_owned()).or_insert(0) += 1;
});

worker.join().expect("worker panicked");
assert_eq!(shared.lock().unwrap().get("jobs"), Some(&1));
```

不要在持有锁时执行耗时 I/O 或复杂计算。高并发场景还可以根据实际访问模式选择分片或专门的并发映射实现。

## 10. 什么时候不该使用 `HashMap`

以下情况可以考虑其他数据结构：

- 需要按键有序遍历或范围查询：使用 `BTreeMap`；
- 键是连续且范围较小的整数：`Vec` 往往更简单、更快；
- 只关心元素是否存在，不需要关联值：使用 `HashSet`；
- 数据量极小且主要顺序扫描：简单的 `Vec<(K, V)>` 有时已经足够；
- 需要固定的插入顺序：标准库 `HashMap` 不提供该保证，需要调整设计或选择合适的第三方集合。

## 11. 常见错误清单

1. **把 `map[&key]` 当作安全查询**：键缺失会 panic，优先使用 `get`。
2. **先 `contains_key` 再 `insert`**：条件插入或更新时优先使用 `entry`，避免重复查找。
3. **误以为插入后仍拥有 `String`**：非 `Copy` 的键和值会被移动。
4. **为查询创建多余的 `String`**：`HashMap<String, V>` 通常可直接用 `&str` 查询。
5. **依赖遍历顺序**：`HashMap` 不保证顺序，需要稳定顺序时排序或使用 `BTreeMap`。
6. **用可变内容作为键并在插入后修改**：键的哈希和相等语义必须保持稳定。
7. **为了方便而到处 `clone`**：先考虑借用、缩短生命周期或重新设计所有权。
8. **用普通 `HashMap` 直接并发写入**：根据读写模式选择锁或并发集合。

## 12. 一个较完整的示例

下面的函数统计单词，并按出现次数从高到低、单词字典序从小到大返回结果：

```rust
use std::collections::HashMap;

fn sorted_word_counts(text: &str) -> Vec<(&str, usize)> {
    let mut counts: HashMap<&str, usize> = HashMap::new();

    for word in text.split_whitespace() {
        *counts.entry(word).or_default() += 1;
    }

    let mut result: Vec<_> = counts.into_iter().collect();
    result.sort_unstable_by(|(word_a, count_a), (word_b, count_b)| {
        count_b.cmp(count_a).then_with(|| word_a.cmp(word_b))
    });
    result
}

fn main() {
    let result = sorted_word_counts("safe fast productive fast safe safe");

    assert_eq!(
        result,
        vec![("safe", 3), ("fast", 2), ("productive", 1)]
    );
}
```

这个示例体现了几条推荐实践：

- 使用 `entry(...).or_default()` 完成单次查询和原地计数；
- 函数内部只保存输入字符串的切片，避免为每个单词分配新字符串；
- 不依赖 `HashMap` 的遍历顺序，而是在返回前显式排序；
- 使用 `into_iter()` 消耗映射，将其中的数据移动到结果中。

## 总结

正确使用 `HashMap` 的关键不只是记住 API，而是明确以下几点：

- 键和值由谁拥有，什么时候借用，什么时候发生移动；
- 查询失败是正常情况，应通过 `Option` 处理；
- 条件插入和原地更新优先考虑 `entry` API；
- 不依赖哈希表的遍历顺序；
- 键的 `Eq` 与 `Hash` 语义必须一致且稳定；
- 根据数据规模、顺序要求和并发模式选择真正合适的集合。

掌握这些原则后，`HashMap` 不仅易用，而且可以写得安全、清晰且高效。
