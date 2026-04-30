# 函数式编程中跟产生新数据有关的常见操作

以 `C#` / `LINQ` 为主，但概念本身是通用函数式编程里的常见套路。

核心思路只有一句话：

**不改旧数据，而是从旧数据推导出新数据，或者直接生成新数据。**

所以除了你提到的 `filter` 和 `fold`，其实还有一大批操作都属于这个范畴。

---

## 一、先给一个大图景

如果只看“会产生新数据”的操作，大致可以分成这几类：

1. **一对一变换**
   - `map` / `Select`

2. **一对多再展平**
   - `flatMap` / `bind` / `SelectMany`

3. **筛选与裁剪**
   - `filter` / `Where`
   - `Take`, `Skip`, `takeWhile`, `skipWhile`

4. **重组与切片**
   - `GroupBy`
   - `partition`
   - `Chunk`
   - `window`
   - `pairwise`

5. **组合多个序列**
   - `Zip`
   - `Concat`
   - `Append` / `Prepend`
   - `interleave`

6. **去重、集合运算、排序**
   - `Distinct`
   - `DistinctBy`
   - `Union`, `Intersect`, `Except`
   - `OrderBy`, `ThenBy`, `Reverse`

7. **从无到有生成序列**
   - `Enumerable.Range`
   - `Enumerable.Repeat`
   - `unfold`
   - `replicate`
   - `iterate`

8. **保留中间结果的渐进构造**
   - `scan`

你如果问：**哪些最像“生产新数据”的核心操作？**
答案通常是：
- `map`
- `flatMap`
- `groupBy`
- `partition`
- `zip`
- `chunk`
- `window`
- `distinct`
- `sort`
- `concat`
- `unfold`
- `scan`

---

## 二、最基础也最重要：映射类

### 1. `map` / `Select`
把每个元素映射成另一个元素，得到一个**新序列**。

```csharp
var numbers = new[] { 1, 2, 3 };
var squares = numbers.Select(x => x * x);
// [1, 4, 9]
```

这是最典型的“产生新数据”。

它不是删掉元素，也不是汇总成一个值，而是：
**把旧集合中的每个元素变形成新集合中的每个元素。**

---

### 2. `flatMap` / `bind` / `SelectMany`
先把每个元素映射成一个子序列，再把所有子序列压平。

```csharp
var words = new[] { "ab", "cd" };
var chars = words.SelectMany(x => x.ToCharArray());
// ['a', 'b', 'c', 'd']
```

这个比 `map` 更强，因为它能做：
- 一变多
- 多层结构展开
- 组合式查询

如果说 `map` 是“一对一生产”，`flatMap` 就是“一对多生产再合并”。

---

## 三、筛选与裁剪类

### 3. `filter` / `Where`
从原集合中保留符合条件的元素，得到新集合。

```csharp
var numbers = new[] { 1, 2, 3, 4, 5 };
var evens = numbers.Where(x => x % 2 == 0);
// [2, 4]
```

这是你已经提到的。

严格说，它不是“创造全新值”，但它确实**创建了一个新序列**。

---

### 4. `Take` / `Skip`
按位置裁剪序列。

```csharp
var numbers = Enumerable.Range(1, 10);
var first3 = numbers.Take(3);   // [1, 2, 3]
var rest = numbers.Skip(3);     // [4, 5, 6, 7, 8, 9, 10]
```

变体还有：
- `TakeWhile`
- `SkipWhile`

这些操作本质上也是：
**从原始序列切出一个新的序列。**

---

## 四、重组类：不是简单变换，而是重新组织数据

### 5. `GroupBy`
按照 key 对数据重新组织，产出“分组后的新结构”。

```csharp
var words = new[] { "apple", "ape", "banana", "book" };
var grouped = words.GroupBy(x => x[0]);
```

结果不再是普通线性列表，而是：
- key -> 对应元素集合

这是一种很典型的数据重构造。

---

### 6. `partition`
把一个集合按条件分成两个集合。

在很多 FP 语言里，`partition` 很常见；`LINQ` 没有直接内置同名函数，但概念非常重要。

```csharp
var numbers = new[] { 1, 2, 3, 4, 5, 6 };
var evens = numbers.Where(x => x % 2 == 0);
var odds = numbers.Where(x => x % 2 != 0);
```

`partition` 比 `filter` 更像一次性生产两个新集合：
- 满足条件的一组
- 不满足条件的一组

所以它是“二路拆分”。

---

### 7. `Chunk`
把一个序列按固定大小切成多个小块。

```csharp
var numbers = Enumerable.Range(1, 10);
var chunks = numbers.Chunk(3);
// [ [1,2,3], [4,5,6], [7,8,9], [10] ]
```

这个非常像“重新切片生产新结构”。

---

### 8. `window`
`window` 和 `Chunk` 很像，但它通常是**滑动窗口**，相邻窗口会重叠。

例如窗口大小为 3：
- `[1,2,3]`
- `[2,3,4]`
- `[3,4,5]`

`LINQ` 没有直接内置标准 `Window`，但这是函数式数据处理中非常常见的操作，尤其用于：
- 时间序列
- 邻项分析
- 模式检测

---

### 9. `pairwise`
把相邻元素两两组合成新序列。

例如：
- 输入: `[10, 20, 30, 40]`
- 输出: `[(10,20), (20,30), (30,40)]`

它相当于窗口大小为 2 的特殊情况。

这个操作对于：
- 求相邻差值
- 轨迹分析
- 趋势判断
特别常见。

---

## 五、组合类：把多个序列拼出新序列

### 10. `Zip`
把两个序列按位置合并。

```csharp
var names = new[] { "A", "B", "C" };
var scores = new[] { 90, 80, 70 };
var result = names.Zip(scores, (name, score) => new { name, score });
```

这不是修改任一原序列，而是生成了一个新的组合序列。

---

### 11. `Concat`
把两个序列首尾连接起来。

```csharp
var a = new[] { 1, 2 };
var b = new[] { 3, 4 };
var c = a.Concat(b);
// [1, 2, 3, 4]
```

---

### 12. `Append` / `Prepend`
在序列尾部或头部新增元素，得到新序列。

```csharp
var xs = new[] { 2, 3 };
var ys = xs.Prepend(1).Append(4);
// [1, 2, 3, 4]
```

它们看似简单，但本质也是不可变思路：
**不是修改原序列，而是返回一个新序列。**

---

### 13. `interleave`
交错合并两个序列。

例如：
- `[1,2,3]`
- `[10,20,30]`

得到：
- `[1,10,2,20,3,30]`

`LINQ` 没有直接内置，但在很多数据流处理中很常见。

---

## 六、去重、集合运算、排序：这些也都在生产新数据

### 14. `Distinct`
去重，得到新序列。

```csharp
var xs = new[] { 1, 2, 2, 3, 3, 3 };
var ys = xs.Distinct();
// [1, 2, 3]
```

---

### 15. `DistinctBy`
按某个 key 去重。

```csharp
var people = new[]
{
    new { Name = "A", Age = 20 },
    new { Name = "B", Age = 20 },
    new { Name = "C", Age = 30 }
};

var result = people.DistinctBy(x => x.Age);
```

这类操作非常符合函数式风格：
- 输入一个集合
- 返回一个结构更“规整”的新集合

---

### 16. `Union`, `Intersect`, `Except`
集合运算：
- `Union`: 并集
- `Intersect`: 交集
- `Except`: 差集

```csharp
var a = new[] { 1, 2, 3 };
var b = new[] { 3, 4, 5 };

var union = a.Union(b);         // [1, 2, 3, 4, 5]
var intersect = a.Intersect(b); // [3]
var except = a.Except(b);       // [1, 2]
```

这类特别明显是在“生成新的结果集”。

---

### 17. `OrderBy`, `ThenBy`, `Reverse`
排序和反转同样返回新序列。

```csharp
var xs = new[] { 3, 1, 4, 2 };
var sorted = xs.OrderBy(x => x);
var reversed = xs.Reverse();
```

注意这里的思想仍然是：
**原数据不变，得到一个新的排列结果。**

---

## 七、从无到有：真正的“生成器”类

### 18. `Enumerable.Range`
生成一段连续整数。

```csharp
var xs = Enumerable.Range(1, 5);
// [1, 2, 3, 4, 5]
```

---

### 19. `Enumerable.Repeat`
重复生成同一个值。

```csharp
var xs = Enumerable.Repeat("A", 3);
// ["A", "A", "A"]
```

---

### 20. `iterate`
从一个初始值开始，不断套用同一个函数：

- 初始值 `x`
- 下一个 `f(x)`
- 再下一个 `f(f(x))`
- 继续下去

例如：
- `1`
- `2`
- `4`
- `8`
- `16`

这个在很多 FP 语言中是标准思路，`LINQ` 没直接内置，但概念上很重要。

---

### 21. `replicate`
本质上接近 `Enumerable.Repeat`。

比如：
- 把某个元素重复 N 次
- 得到一个新列表

---

### 22. `unfold`
这个非常值得单独记住，因为它经常被拿来和 `fold` 对照。

- `fold`: 多个值 -> 一个值
- `unfold`: 一个种子状态 -> 多个值

也就是：
**`fold` 是收缩，`unfold` 是展开。**

比如从状态不断演化生成序列：
- 斐波那契
- 分页结果流
- 树遍历展开
- 状态机输出流

`LINQ` 没内置标准名为 `Unfold` 的 API，但这个思想非常函数式。

---

## 八、很容易忽略，但也非常重要：保留中间结果

### 23. `scan`
`scan` 和 `fold` 很像，但区别是：

- `fold` 只返回最后一个累积结果
- `scan` 返回每一步的累积结果序列

例如求前缀和：
- 输入: `[1,2,3,4]`
- `scan` 输出: `[1,3,6,10]`

这特别像“边累计边生产新数据”。

所以 `scan` 在很多场景里比 `fold` 更符合你说的“产生新数据”。

用途很多：
- 前缀和
- 运行统计
- 状态演化轨迹
- UI 状态流
- 事件流累积

---

## 九、如果只关心“返回新集合”，还可以继续补一批

下面这些也都常见：

### 24. `collect`
很多 FP 语言里会把“映射并展平”叫 `collect`，本质接近 `flatMap`。

### 25. `choose`
把 `None` / 空结果去掉，只保留 `Some` / 有值结果。

这个常见于 `F#` 风格。

### 26. `slice`
从序列中切出一个区间。

### 27. `permute`
重排元素顺序。

### 28. `cartesian product`
笛卡尔积，两个集合组合出所有可能对。

例如：
- `[1,2]`
- `['A','B']`

生成：
- `(1,'A')`
- `(1,'B')`
- `(2,'A')`
- `(2,'B')`

这其实可以通过 `SelectMany` 做出来。

### 29. `buffer`
按批收集元素，和 `chunk` 很接近。

### 30. `shuffle`
随机重排后生成新序列。

---

## 十、把这些操作按“感觉”再归纳一遍

### A. 最典型的“变形生产”
- `map`
- `flatMap`
- `scan`

### B. 最典型的“筛出新集合”
- `filter`
- `take`
- `skip`
- `distinct`

### C. 最典型的“重组新结构”
- `groupBy`
- `partition`
- `chunk`
- `window`
- `pairwise`

### D. 最典型的“合成新集合”
- `zip`
- `concat`
- `append`
- `prepend`
- `interleave`
- `union`
- `intersect`
- `except`

### E. 最典型的“直接生成”
- `Enumerable.Range`
- `Enumerable.Repeat`
- `iterate`
- `unfold`

### F. 最典型的“重新排列”
- `sort`
- `reverse`
- `permute`
- `shuffle`

---

## 十一、和你原问题直接对应的一个简短答案

如果你问：
**除了 `fold`、`filter`，函数式编程里还有哪些跟产生新数据有关的？**

最应该立刻想到的是：

- `map`
- `flatMap`
- `groupBy`
- `partition`
- `zip`
- `concat`
- `take` / `skip`
- `distinct`
- `sort`
- `reverse`
- `chunk`
- `window`
- `pairwise`
- `scan`
- `range`
- `repeat`
- `unfold`

其中：
- **最基础**: `map`
- **最强大**: `flatMap`
- **最像生成器**: `unfold`
- **最像中间态生产器**: `scan`
- **最像结构重组器**: `groupBy` / `chunk` / `window`

---

## 十二、和 `LINQ` 的对应关系速查表

| FP 概念 | C#/LINQ 中常见对应 |
|---|---|
| `map` | `Select` |
| `flatMap` / `bind` | `SelectMany` |
| `filter` | `Where` |
| `groupBy` | `GroupBy` |
| `zip` | `Zip` |
| `concat` | `Concat` |
| `prepend` | `Prepend` |
| `append` | `Append` |
| `distinct` | `Distinct` |
| `distinctBy` | `DistinctBy` |
| `union` | `Union` |
| `intersect` | `Intersect` |
| `except` | `Except` |
| `sort` | `OrderBy` / `ThenBy` |
| `reverse` | `Reverse` |
| `take` | `Take` |
| `skip` | `Skip` |
| `chunk` | `Chunk` |
| `range` | `Enumerable.Range` |
| `repeat` | `Enumerable.Repeat` |
| `fold` | `Aggregate` |
| `scan` | 原生 `LINQ` 无标准同名 API |
| `partition` | 原生 `LINQ` 无标准同名 API |
| `window` | 原生 `LINQ` 无标准同名 API |
| `pairwise` | 原生 `LINQ` 无标准同名 API |
| `unfold` | 原生 `LINQ` 无标准同名 API |

---

## 十三、一个最短结论

如果你只想记一句：

> `filter` 是筛，`fold` 是收，`map` 是变，`flatMap` 是展开，`groupBy` 是重组，`zip` 是合并，`unfold` 是生成，`scan` 是一边累积一边生产。

所以，**跟“产生新数据”最直接相关的，不止 `filter`，更核心的其实是 `map`、`flatMap`、`groupBy`、`zip`、`chunk`、`window`、`scan`、`unfold` 这一批。**
