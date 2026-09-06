# Rust 中的 `HashSet`：从入门到正确使用

`HashSet<T>` 是 Rust 标准库提供的哈希集合，用于保存一组**互不重复**的值。它适合表达“某个值是否存在”，例如：

- 已访问过的页面；
- 用户拥有的权限；
- 不重复的标签；
- 图搜索中已经处理过的节点；
- 两组数据的交集、并集和差集。

`HashSet` 位于 `std::collections` 中：

```rust
use std::collections::HashSet;
```

从实现角度看，`HashSet<T>` 与只保存键、不关心值的哈希表类似。如果需要为每个键关联额外数据，应使用 `HashMap<K, V>`。

## 1. 创建集合

### 1.1 创建空集合

```rust
use std::collections::HashSet;

fn main() {
    let mut languages = HashSet::new();

    languages.insert("Rust");
    languages.insert("Go");

    assert_eq!(languages.len(), 2);
}
```

集合必须声明为 `mut`，才能插入或删除元素。

如果事先知道大致的元素数量，可以预分配容量，减少扩容和重新哈希：

```rust
use std::collections::HashSet;

let mut ids = HashSet::with_capacity(1_000);
ids.insert(42_u64);
```

### 1.2 从数组或迭代器创建

少量固定数据可以使用 `HashSet::from`：

```rust
use std::collections::HashSet;

let permissions = HashSet::from(["read", "write", "delete"]);

assert!(permissions.contains("read"));
```

动态数据通常通过 `collect` 创建：

```rust
use std::collections::HashSet;

let numbers = vec![1, 2, 2, 3, 3, 3];
let unique: HashSet<_> = numbers.into_iter().collect();

assert_eq!(unique.len(), 3);
```

`collect` 会自动去除重复值。类型标注 `HashSet<_>` 用来告诉编译器目标集合的类型。

## 2. 插入与查询

### 2.1 `insert` 的返回值很有用

`insert` 返回一个 `bool`：

- `true` 表示该值原本不存在，本次成功加入；
- `false` 表示集合中已有相等的值，集合没有发生变化。

```rust
use std::collections::HashSet;

let mut names = HashSet::new();

assert!(names.insert("Alice"));
assert!(!names.insert("Alice"));
assert_eq!(names.len(), 1);
```

因此不必先调用 `contains` 再调用 `insert`：

```rust
if names.insert("Bob") {
    println!("首次看到 Bob");
}
```

这样只需进行一次集合查找。

### 2.2 使用 `contains` 判断是否存在

```rust
use std::collections::HashSet;

let online = HashSet::from(["Alice", "Bob"]);

assert!(online.contains("Alice"));
assert!(!online.contains("Carol"));
```

当集合存储 `String` 时，通常可以直接使用 `&str` 查询，不需要临时创建 `String`：

```rust
use std::collections::HashSet;

let mut names: HashSet<String> = HashSet::new();
names.insert("Alice".to_owned());

assert!(names.contains("Alice"));
```

这是因为 `String` 可以借用为 `str`，而且两者具有一致的哈希和相等语义。

### 2.3 使用 `get` 取得集合中保存的值

`contains` 只能回答值是否存在；`get` 会返回集合中实际保存的元素引用：

```rust
use std::collections::HashSet;

let names = HashSet::from([String::from("Alice")]);

if let Some(stored_name) = names.get("Alice") {
    println!("集合中保存的是：{stored_name}");
}
```

当“用于查询的值”与“集合中保存的规范值”可以相等但表现形式不同时，`get` 尤其有用。

## 3. 删除、取出与替换

### 3.1 使用 `remove` 删除

`remove` 返回是否确实删除了元素：

```rust
use std::collections::HashSet;

let mut names = HashSet::from(["Alice", "Bob"]);

assert!(names.remove("Bob"));
assert!(!names.remove("Bob"));
```

### 3.2 使用 `take` 取回所有权

`remove` 只返回 `bool`。如果希望删除元素并取得集合中原值的所有权，应使用 `take`：

```rust
use std::collections::HashSet;

let mut names = HashSet::from([String::from("Alice")]);

let removed = names.take("Alice");
assert_eq!(removed, Some(String::from("Alice")));
assert!(names.is_empty());
```

### 3.3 使用 `replace` 替换相等的旧值

`replace` 会将新值放入集合，并返回原来与它相等的旧值。对于“部分字段定义身份，其他字段是可更新数据”的类型，这个方法很实用。

```rust
use std::collections::HashSet;
use std::hash::{Hash, Hasher};

#[derive(Debug)]
struct User {
    id: u64,
    display_name: String,
}

// 用户身份只由 id 决定。
impl PartialEq for User {
    fn eq(&self, other: &Self) -> bool {
        self.id == other.id
    }
}

impl Eq for User {}

impl Hash for User {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.id.hash(state);
    }
}

let mut users = HashSet::new();
users.insert(User {
    id: 1,
    display_name: "Alice".to_owned(),
});

let old = users.replace(User {
    id: 1,
    display_name: "Alice Chen".to_owned(),
});

assert_eq!(old.unwrap().display_name, "Alice");
assert_eq!(users.iter().next().unwrap().display_name, "Alice Chen");
```

手写 `Eq` 和 `Hash` 时，两者必须使用一致的身份字段。后文会进一步说明这一约束。

## 4. 集合运算

`HashSet` 提供了常见的数学集合运算。假设有两个集合：

```rust
use std::collections::HashSet;

let a = HashSet::from([1, 2, 3]);
let b = HashSet::from([3, 4, 5]);
```

### 4.1 差集 `difference`

取得只在 `a` 中、不在 `b` 中的元素：

```rust
let values: HashSet<_> = a.difference(&b).copied().collect();
assert_eq!(values, HashSet::from([1, 2]));
```

### 4.2 交集 `intersection`

取得同时存在于两个集合中的元素：

```rust
let values: HashSet<_> = a.intersection(&b).copied().collect();
assert_eq!(values, HashSet::from([3]));
```

### 4.3 并集 `union`

取得至少存在于一个集合中的元素：

```rust
let values: HashSet<_> = a.union(&b).copied().collect();
assert_eq!(values, HashSet::from([1, 2, 3, 4, 5]));
```

### 4.4 对称差集 `symmetric_difference`

取得只存在于其中一个集合、但不同时存在于两者中的元素：

```rust
let values: HashSet<_> = a.symmetric_difference(&b).copied().collect();
assert_eq!(values, HashSet::from([1, 2, 4, 5]));
```

这些方法返回惰性迭代器，并不会立即创建新集合。如果只需要遍历结果，可以避免 `collect`：

```rust
for value in a.intersection(&b) {
    println!("共同元素：{value}");
}
```

对于可克隆的元素，也可以使用运算符创建新集合：

```rust
let difference = &a - &b;
let intersection = &a & &b;
let union = &a | &b;
let symmetric_difference = &a ^ &b;

assert_eq!(difference, HashSet::from([1, 2]));
assert_eq!(intersection, HashSet::from([3]));
assert_eq!(union, HashSet::from([1, 2, 3, 4, 5]));
assert_eq!(symmetric_difference, HashSet::from([1, 2, 4, 5]));
```

方法形式返回引用迭代器，更适合只读遍历；运算符形式会构造新的 `HashSet`。应根据是否真正需要拥有结果来选择。

## 5. 集合关系判断

可以判断两个集合是否互不相交，以及是否存在子集或超集关系：

```rust
use std::collections::HashSet;

let all_permissions = HashSet::from(["read", "write", "delete"]);
let editor_permissions = HashSet::from(["read", "write"]);
let billing_permissions = HashSet::from(["billing"]);

assert!(editor_permissions.is_subset(&all_permissions));
assert!(all_permissions.is_superset(&editor_permissions));
assert!(editor_permissions.is_disjoint(&billing_permissions));
```

权限检查是这类 API 的典型应用：

```rust
fn has_required_permissions(
    owned: &HashSet<&str>,
    required: &HashSet<&str>,
) -> bool {
    required.is_subset(owned)
}
```

## 6. 遍历与顺序

使用不可变引用遍历不会消耗集合：

```rust
use std::collections::HashSet;

let tags = HashSet::from(["rust", "systems", "safe"]);

for tag in &tags {
    println!("{tag}");
}
```

也可以调用 `iter()`。如果直接对集合调用 `into_iter()`，集合会被消耗，元素的所有权会被移出：

```rust
let tags = HashSet::from([String::from("rust"), String::from("safe")]);
let owned_tags: Vec<String> = tags.into_iter().collect();
// 此后 tags 已不能使用。
```

`HashSet` **不保证遍历顺序**。顺序可能在不同运行、不同版本或插入删除后发生变化。因此：

- 不要依赖遍历顺序编写业务逻辑；
- 测试时直接比较集合，而不是比较未经排序的向量；
- 需要稳定输出时，先收集到 `Vec` 并排序；
- 需要始终有序时，考虑 `BTreeSet`。

```rust
let tags = HashSet::from(["rust", "systems", "safe"]);
let mut sorted_tags: Vec<_> = tags.iter().copied().collect();
sorted_tags.sort_unstable();

assert_eq!(sorted_tags, vec!["rust", "safe", "systems"]);
```

## 7. 批量修改集合

### 7.1 使用 `retain` 按条件保留

```rust
use std::collections::HashSet;

let mut numbers = HashSet::from([1, 2, 3, 4, 5, 6]);
numbers.retain(|number| number % 2 == 0);

assert_eq!(numbers, HashSet::from([2, 4, 6]));
```

### 7.2 使用 `drain` 取出全部元素

`drain` 会清空集合，同时返回拥有元素所有权的迭代器，并保留集合已经分配的容量以供复用：

```rust
use std::collections::HashSet;

let mut jobs = HashSet::from([String::from("build"), String::from("test")]);
let drained: Vec<_> = jobs.drain().collect();

assert_eq!(drained.len(), 2);
assert!(jobs.is_empty());
```

只想清空且不需要元素时，使用 `clear()` 更直接：

```rust
jobs.clear();
```

## 8. 所有权与借用

### 8.1 插入非 `Copy` 值会转移所有权

```rust
use std::collections::HashSet;

let name = String::from("Alice");
let mut names = HashSet::new();

names.insert(name);
// name 的所有权已移动到 names，不能继续使用。
```

如果插入后仍需要独立拥有原值，可以显式 `clone`，但克隆字符串和大型结构体可能产生额外开销：

```rust
names.insert(name.clone());
println!("{name}");
```

对于整数等实现了 `Copy` 的类型，插入的是副本。

### 8.2 集合可以保存引用，但受生命周期约束

```rust
use std::collections::HashSet;

let first = String::from("Alice");
let second = String::from("Bob");

let names: HashSet<&str> = HashSet::from([first.as_str(), second.as_str()]);
assert!(names.contains("Alice"));
```

集合不能比它引用的 `first` 和 `second` 存活更久。如果集合需要被返回、长期保存或跨线程传递，通常让它拥有 `String` 会更简单。

### 8.3 不提供普通的元素可变引用

`HashSet` 没有类似 `HashMap::values_mut` 的 API，也不能通过迭代直接修改元素。这是因为修改元素可能改变其哈希值或相等性，从而破坏哈希表内部结构。

要更新元素，应采用以下方式之一：

- 使用 `take` 取出、修改后重新插入；
- 构造新值并使用 `replace`；
- 如果元素包含与哈希、相等性无关的内部可变字段，谨慎使用内部可变性。

最清晰的做法通常是 `take` 后修改：

```rust
use std::collections::HashSet;

let mut values = HashSet::from([String::from("rust")]);

if let Some(mut value) = values.take("rust") {
    value.make_ascii_uppercase();
    values.insert(value);
}

assert!(values.contains("RUST"));
```

## 9. 元素必须满足什么条件

`HashSet<T>` 中的元素需要实现 `Eq` 和 `Hash`。常用的整数、布尔值、字符、`String`、`&str`、元组等通常已经实现了它们。

自定义类型可以派生这些 trait：

```rust
use std::collections::HashSet;

#[derive(Debug, Hash, PartialEq, Eq)]
struct Coordinate {
    x: i32,
    y: i32,
}

let mut visited = HashSet::new();
visited.insert(Coordinate { x: 10, y: 20 });
assert!(visited.contains(&Coordinate { x: 10, y: 20 }));
```

元素必须遵守两个重要规则：

1. 如果两个值通过 `Eq` 比较相等，它们的哈希值也必须相等；
2. 元素放入集合后，参与相等比较和哈希计算的内容不能改变。

不要为同一个类型实现相互矛盾的 `Eq` 与 `Hash`。也不要通过 `Cell`、`RefCell` 等内部可变性改变已插入元素的身份字段，否则可能导致查找失败、重复元素、panic 等不可预测的局部结果。

浮点数 `f32` 和 `f64` 没有实现 `Eq`，主要原因是 `NaN` 不等于自身，所以不能直接作为标准 `HashSet` 的元素。如果确实需要存储浮点数，应先定义明确的相等语义，或者使用提供全序包装类型的第三方库。

## 10. 使用 `HashSet` 去重

如果不关心结果顺序，可以直接收集到 `HashSet`：

```rust
use std::collections::HashSet;

let values = vec![3, 1, 2, 3, 2];
let unique: HashSet<_> = values.into_iter().collect();

assert_eq!(unique, HashSet::from([1, 2, 3]));
```

如果希望保留首次出现的顺序，可以让 `Vec::retain` 配合辅助集合：

```rust
use std::collections::HashSet;

let mut values = vec![3, 1, 2, 3, 2];
let mut seen = HashSet::new();

values.retain(|value| seen.insert(*value));

assert_eq!(values, vec![3, 1, 2]);
```

对于非 `Copy` 元素，这种写法可能需要克隆用于判重的键。此时应根据数据规模和所有权需求决定是接受克隆、保存较小的身份字段，还是使用其他算法。

## 11. 性能与容量管理

平均情况下，`contains`、`insert`、`remove` 和 `take` 的时间复杂度接近 `O(1)`。实际性能还会受到哈希函数、冲突数量、扩容、元素大小和缓存局部性的影响。

常用的容量方法包括：

```rust
use std::collections::HashSet;

let mut set = HashSet::with_capacity(100);
set.reserve(50);      // 确保至少还能容纳一定数量的元素。
set.shrink_to_fit();  // 尝试释放多余容量。
```

实践建议：

- 已知大致规模时使用 `with_capacity`；
- 批量插入前使用 `reserve`；
- 不要频繁调用 `shrink_to_fit`，重新分配本身也有成本；
- 遍历集合的成本与内部容量有关，清空少量元素后保留巨大容量未必合算；
- 先通过性能分析确定瓶颈，再考虑自定义哈希器。

标准 `HashSet` 默认使用带随机种子的哈希器，以抵御针对哈希冲突的拒绝服务攻击。更换为更快的自定义哈希器可能改变安全特性，不应只因为微基准更快，就用于处理不可信输入。

## 12. 多线程使用

`HashSet` 本身不支持多个线程同时修改。多个线程只读共享时，可以使用 `Arc<HashSet<T>>`；需要共享修改时，通常使用 `Arc<Mutex<HashSet<T>>>` 或 `Arc<RwLock<HashSet<T>>>`：

```rust
use std::collections::HashSet;
use std::sync::{Arc, Mutex};

let visited = Arc::new(Mutex::new(HashSet::<String>::new()));
let worker_set = Arc::clone(&visited);

let worker = std::thread::spawn(move || {
    let mut set = worker_set.lock().expect("mutex poisoned");
    set.insert("task-1".to_owned());
});

worker.join().expect("worker panicked");
assert!(visited.lock().unwrap().contains("task-1"));
```

不要在持有锁时执行耗时 I/O 或复杂计算。高并发场景可以根据实际访问模式考虑分片，或者选择专门的并发集合。

## 13. 什么时候不该使用 `HashSet`

以下情况可以考虑其他数据结构：

- 需要排序遍历、最小值、最大值或范围查询：使用 `BTreeSet`；
- 值是连续且范围较小的非负整数：位图或 `Vec<bool>` 可能更紧凑；
- 需要保留插入顺序：标准库 `HashSet` 不提供该保证；
- 元素数量很少且主要进行顺序扫描：`Vec<T>` 可能更简单；
- 每个元素还需要关联数据：使用 `HashMap<K, V>`；
- 只需对已排序数据临时去除相邻重复项：可以考虑 `Vec::dedup`。

## 14. 常见错误清单

1. **先 `contains` 再 `insert`**：直接检查 `insert` 返回的 `bool`，避免重复查找。
2. **忽略 `insert` 返回值**：需要检测重复输入时，这个返回值正是所需信息。
3. **误以为插入后仍拥有 `String`**：非 `Copy` 元素会移动到集合中。
4. **为查询创建多余的 `String`**：`HashSet<String>` 通常可以直接用 `&str` 查询。
5. **依赖遍历顺序**：需要稳定顺序时显式排序，或改用 `BTreeSet`。
6. **直接修改元素的身份字段**：这会破坏集合所依赖的哈希与相等关系。
7. **用 `remove` 后又想取得原值**：需要所有权时使用 `take`。
8. **无条件收集集合运算结果**：只需遍历时直接使用返回的迭代器。
9. **用普通 `HashSet` 并发写入**：根据读写模式选择锁或并发集合。

## 15. 一个较完整的示例

下面的函数计算用户最终获得的权限：先合并角色权限和额外权限，再删除被明确禁用的权限，最后按字典序输出稳定结果。

```rust
use std::collections::HashSet;

fn effective_permissions<'a>(
    role_permissions: &'a HashSet<&'a str>,
    extra_permissions: &'a HashSet<&'a str>,
    denied_permissions: &HashSet<&str>,
) -> Vec<&'a str> {
    let mut effective: HashSet<&str> = role_permissions
        .union(extra_permissions)
        .copied()
        .collect();

    effective.retain(|permission| !denied_permissions.contains(permission));

    let mut result: Vec<_> = effective.into_iter().collect();
    result.sort_unstable();
    result
}

fn main() {
    let role = HashSet::from(["read", "write"]);
    let extra = HashSet::from(["export", "write"]);
    let denied = HashSet::from(["write"]);

    let permissions = effective_permissions(&role, &extra, &denied);
    assert_eq!(permissions, vec!["export", "read"]);
}
```

这个示例体现了几条推荐实践：

- 使用集合表达权限，自动消除重复项；
- 使用 `union` 清晰表达合并语义；
- 使用 `retain` 批量删除被拒绝的权限；
- 不依赖 `HashSet` 的遍历顺序，输出前显式排序；
- 使用字符串切片避免不必要的字符串分配。

## 总结

正确使用 `HashSet` 的关键是理解它表达的是“唯一值的集合”，并遵守以下原则：

- 通过 `insert` 和 `remove` 的返回值避免重复查询；
- 需要取回元素所有权时使用 `take`，需要替换等价值时使用 `replace`；
- 使用交集、并集、差集和子集等 API 直接表达集合逻辑；
- 明确元素的所有权和生命周期，避免无意义的克隆；
- 不依赖哈希集合的遍历顺序；
- 保证元素的 `Eq` 与 `Hash` 语义一致，并且入集合后保持稳定；
- 根据顺序、范围、内存和并发需求选择真正合适的数据结构。

掌握这些规则后，`HashSet` 可以让去重、成员判断和集合运算代码保持安全、简洁且高效。
