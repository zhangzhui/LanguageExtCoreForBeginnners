# 函数式编程（FP）与 LINQ 的对应关系

在函数式编程（Functional Programming, FP）和 C# 的 LINQ 之间，有着非常直接和紧密的对应关系。LINQ 实际上就是函数式编程中对集合（Monad/Functor）进行操作的 C# 实现。

以下是常见的函数式编程概念与 LINQ 方法的对照表：

---

### 核心转换与过滤

| 函数式编程 (FP) | LINQ 方法 | 作用说明 |
| :--- | :--- | :--- |
| **Map** / fmap | `Select` | 转换集合中的每一个元素，返回一个同样大小的新集合。<br>`[1, 2].Select(x => x * 2) // [2, 4]` |
| **Filter** | `Where` | 根据条件过滤集合，保留满足条件的元素。<br>`[1, 2].Where(x => x > 1) // [2]` |
| **Bind** / FlatMap / Chain | `SelectMany` | 映射每一个元素使其返回一个集合，然后将所有的集合"压平"（展平）成一个单一的集合。这是 Monad 的核心操作。<br>`[[1,2], [3,4]].SelectMany(x => x) // [1, 2, 3, 4]` |
| **Fold** / Reduce / Inject | `Aggregate` | 提供一个初始值，遍历集合将其累加/折叠为一个单一的结果。<br>`[1, 2, 3].Aggregate(0, (acc, x) => acc + x) // 6` |

---

### 逻辑与量词

| 函数式编程 (FP) | LINQ 方法 | 作用说明 |
| :--- | :--- | :--- |
| **All** / Every | `All` | 检查集合中的*所有*元素是否都满足指定条件。<br>`[2, 4].All(x => x % 2 == 0) // true` |
| **Any** / Some | `Any` | 检查集合中是否*有任意一个*元素满足指定条件。<br>`[1, 2].Any(x => x > 1) // true` |

---

### 结构与组合

| 函数式编程 (FP) | LINQ 方法 | 作用说明 |
| :--- | :--- | :--- |
| **Zip** | `Zip` | 将两个集合像拉链一样成对合并起来。<br>`[1, 2].Zip(["a", "b"]) // [(1, "a"), (2, "b")]` |
| **Concat** | `Concat` | 连接两个集合。<br>`[1].Concat([2]) // [1, 2]` |
| **Head** / First | `First` / `FirstOrDefault` | 获取集合的第一个元素。 |
| **Tail** / Rest | `Skip(1)` | 获取除第一个元素外的所有元素。 |
| **Take** | `Take` | 获取集合前 n 个元素。 |
| **Drop** / Skip | `Skip` | 跳过集合前 n 个元素，获取剩下的元素。 |

---

### 分组与去重

| 函数式编程 (FP) | LINQ 方法 | 作用说明 |
| :--- | :--- | :--- |
| **GroupBy** | `GroupBy` | 根据指定的键选择器函数对集合元素进行分组。 |
| **Distinct** / Unique | `Distinct` | 移除集合中的重复元素。 |

---

### 总结

你当前所在的目录是 `language-ext` 项目，这正是 C# 中最著名的函数式编程库。在 C# 中，原生的 LINQ 主要是针对 `IEnumerable<T>`（序列）设计的，而像 `language-ext` 这样的库则将这些函数式概念（Map, Bind, Filter, Fold）扩展到了其他类型上，比如 `Option<T>`、`Either<L, R>` 和 `Try<T>` 等，使得 C# 能够支持更完整的函数式编程范式。
