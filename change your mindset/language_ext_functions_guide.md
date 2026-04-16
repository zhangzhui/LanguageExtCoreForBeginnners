# language-ext 函数式编程常用函数详解

## 概览

本文介绍 `language-ext` 中最常用的函数式编程函数：`Map`、`Bind`、`Match`、`IfNone`、`Iter`、`Fold`、`Filter`、`Sequence`。

---

## 1. `Map` — 变换值

**作用：** 在`不拆开`容器的情况下，对内部的值进行变换。`（旧瓶装新酒）`

```csharp
Option<int> opt = Some(5);
Option<string> result = opt.Map(x => x.ToString()); // Some("5")

Option<int> none = None;
Option<string> result2 = none.Map(x => x.ToString()); // None
```

> 类比：就像 `List.Select()`，但适用于 `Option`、`Either`、`Try` 等容器。

---

## 2. `Bind` — 链式操作`（flatMap）`，想象下`Railway`

**作用：** 和 `Map` 类似，但函数本身返回的是一个容器（`Option<T>`），`Bind` 会自动"展平"，避免嵌套。

```csharp
Option<int> ParseInt(string s) =>
    int.TryParse(s, out var n) ? Some(n) : None;

Option<string> input = Some("42");
Option<int> result = input.Bind(ParseInt); // Some(42)

Option<string> bad = Some("abc");
Option<int> result2 = bad.Bind(ParseInt); // None
```

> `Map` + `Bind` 对比：
> - `Map(x => x + 1)` → 函数返回普通值
> - `Bind(x => Some(x + 1))` → 函数返回Monad `Option<T>`

---

## 3. `Match` — 模式匹配

**作用：** 对容器的两种状态（有值/无值，成功/失败）分别处理，强制你处理所有情况。

```csharp
Option<int> opt = Some(42);

string result = opt.Match(
    Some: v => $"值是 {v}",
    None: () => "没有值"
);
// result = "值是 42"
```

`Either` 的例子：

```csharp
Either<string, int> either = Right(100);

string result = either.Match(
    Right: v => $"成功: {v}",
    Left: err => $"错误: {err}"
);
```

> 类比 C# `switch expression`，但编译器强制你不能漏掉分支。

---

## 4. `IfNone` — 提供默认值

**作用：** 如果是 `None`，返回默认值；如果有值，直接返回该值。

```csharp
Option<int> opt = None;

int result = opt.IfNone(0);       // 0
int result2 = opt.IfNone(() => ComputeDefault()); // 惰性求值

Option<int> some = Some(99);
int result3 = some.IfNone(0);    // 99
```

> 类比 C# `?? 默认值`，但更函数式、更安全。

---

## 5. `Iter` — 执行副作用

**作用：** 如果有值，执行一个操作（副作用），但不改变容器本身。相当于"只读 `foreach`"。

```csharp
Option<string> name = Some("Alice");

name.Iter(n => Console.WriteLine($"Hello, {n}!")); // 打印 "Hello, Alice!"

Option<string> none = None;
none.Iter(n => Console.WriteLine("这行不会执行"));
```

> 注意：`Iter` 返回 `Unit`（类似 `void`），不用于变换，只用于副作用（日志、写入等）。

---

## 6. `Fold` — 聚合/折叠

**作用：** 将`容器`中的值和一个`初始状态`合并，得到`一个最终结果`。

```csharp
Option<int> opt = Some(10);

int result = opt.Fold(100, (state, value) => state + value);
// result = 110（100 + 10）

Option<int> none = None;
int result2 = none.Fold(100, (state, value) => state + value);
// result2 = 100（None 时直接返回初始值）
```

> 类比 `IEnumerable.Aggregate()`，常用于将可选值累积到某个状态中。

---

## 7. `Filter` — 条件过滤

**作用：** 对 `Option` 的值应用条件，不满足条件则变成 `None`。

```csharp
Option<int> opt = Some(42);

Option<int> result = opt.Filter(x => x > 10);  // Some(42)
Option<int> result2 = opt.Filter(x => x > 100); // None

Option<int> none = None;
Option<int> result3 = none.Filter(x => x > 10); // None（本来就是 None）
```

> 类比 `IEnumerable.Where()`，但作用在单个可选值上。

---

## 8. `Sequence` — 容器翻转

**作用：** 将"容器的容器"翻转结构。最常见的用法是把 `IEnumerable<Option<T>>` 转换为 `Option<IEnumerable<T>>`。

```csharp
var list1 = new List<Option<int>> { Some(1), Some(2), Some(3) };
Option<IEnumerable<int>> result1 = list1.Sequence();
// Some([1, 2, 3])（全部有值则成功）

var list2 = new List<Option<int>> { Some(1), None, Some(3) };
Option<IEnumerable<int>> result2 = list2.Sequence();
// None（只要有一个 None，整体就是 None）
```

> 常见场景：你有一组可能失败的操作结果，想要"全部成功才继续，有一个失败就整体失败"。

---

## 快速参考记忆卡

| 函数 | 一句话记忆 | 输入 → 输出 | 类比 |
|------|-----------|------------|------|
| `Map` | 变换内部值 | `Option<A>` + `A→B` → `Option<B>` | `Select()` |
| `Bind` | 链式操作，自动展平 | `Option<A>` + `A→Option<B>` → `Option<B>` | `SelectMany()` |
| `Match` | 强制处理所有分支 | `Option<A>` + 两个处理函数 → `B` | `switch` |
| `IfNone` | 提供默认值 | `Option<A>` + 默认值 → `A` | `??` |
| `Iter` | 执行副作用，不改变值 | `Option<A>` + `A→void` → `Unit` | `foreach` |
| `Fold` | 聚合到初始状态 | `Option<A>` + 初始值 + 函数 → `B` | `Aggregate()` |
| `Filter` | 条件不满足变 None | `Option<A>` + 条件 → `Option<A>` | `Where()` |
| `Sequence` | 翻转容器结构 | `IEnumerable<Option<A>>` → `Option<IEnumerable<A>>` | 无直接类比 |

---

## 常见使用模式

```csharp
// 链式组合：从字符串解析、验证、转换
Option<string> input = Some("  42  ");

Option<string> result = input
    .Map(s => s.Trim())           // 去空格
    .Bind(ParseInt)               // 解析为 int（可能失败）
    .Filter(n => n > 0)          // 只接受正数
    .Map(n => $"结果是 {n}");    // 转为字符串

string final = result.IfNone("无效输入"); // 提供默认值
```

---


# 记忆卡片

| 你的需求 | 应该用的函数 | 类比理解 |
|---------|-------------|---------|
| A -> B | Map | 换个包装纸（内容变了，盒子没变） |
| A -> 盒子 | Bind | 俄罗斯套娃扁平化（防止盒子里套盒子） |
| 必须拿到最终结果 | Match | 开盲盒并处理所有可能 |
| 没有就用默认的 | IfNone / IfLeft | 备胎计划 |
| 满足条件才留下 | Filter | 安检过滤 |
| 只打日志/写数据库 | Iter / Do | 旁观者（只干活，不改变原流向） |
| 列表聚合为一个值 | Fold | 滚雪球 |
| List 变 Option | Sequence | 袜子翻面 |


> 📝 本文档基于 [`language-ext`](https://github.com/louthy/language-ext) 库，适用于 C# 函数式编程学习参考。
