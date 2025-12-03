# Monad Laws（Monad 三定律）

## 核心定律

```csharp
// 假设：
// M<A>     - Monad 类型
// Pure(a)  - 将值 a 提升到 Monad（也叫 Return/Unit）
// Bind     - 也叫 FlatMap/SelectMany，签名：M<A> → (A → M<B>) → M<B>
```

| 定律 | 名称 | 表达式 |
|-----|------|--------|
| **1** | 左单位元 (Left Identity) | `Pure(a).Bind(f) == f(a)` |
| **2** | 右单位元 (Right Identity) | `m.Bind(Pure) == m` |
| **3** | 结合律 (Associativity) | `m.Bind(f).Bind(g) == m.Bind(x => f(x).Bind(g))` |

---

## 定律 1：左单位元（Left Identity）

```csharp
Pure(a).Bind(f) == f(a)
```

**含义**：把值放进 Monad 后立即 Bind，等于直接调用函数

```csharp
// 示例
Option.Some(5).Bind(x => Some(x * 2))  ==  Some(5 * 2)  ==  Some(10)

// LINQ 写法
(from x in Pure(5)
 from y in f(x)
 select y)  ==  f(5)
```

**记忆**：`Pure` 是"透明"的，不会添加额外效果

---

## 定律 2：右单位元（Right Identity）

```csharp
m.Bind(Pure) == m
```

**含义**：Bind 到 Pure 不改变 Monad

```csharp
// 示例
Some(5).Bind(x => Some(x))  ==  Some(5)
None.Bind(x => Some(x))     ==  None

// LINQ 写法
(from x in m
 select x)  ==  m
```

**记忆**：`Pure` 是 Bind 的"单位元"，像乘法中的 1

---

## 定律 3：结合律（Associativity）

```csharp
m.Bind(f).Bind(g) == m.Bind(x => f(x).Bind(g))
```

**含义**：嵌套顺序无关紧要，可以重新组合

```csharp
// 示例
Some(5)
    .Bind(x => Some(x + 1))    // Some(6)
    .Bind(y => Some(y * 2))    // Some(12)
==
Some(5)
    .Bind(x => Some(x + 1).Bind(y => Some(y * 2)))  // Some(12)

// LINQ 写法 - 这两个等价：
(from x in m
 from y in f(x)
 from z in g(y)
 select z)
==
(from y in (from x in m from y in f(x) select y)
 from z in g(y)
 select z)
```

**记忆**：类似数学的 `(a + b) + c == a + (b + c)`

---

## 为什么这些定律重要？

```
┌────────────────────────────────────────────────────────┐
│  1. 保证重构安全 - 可以自由重组 Bind 链                  │
│  2. 保证 LINQ 语法正确工作                              │
│  3. 保证 Monad 组合（Transformer）的正确性               │
│  4. 是类型系统无法检查的"契约"                           │
└────────────────────────────────────────────────────────┘
```

---

## 直观理解

```
把 Monad 想象成"盒子"：

定律 1：把东西放进盒子再立即取出操作 = 直接操作
        ┌───┐
    a → │ M │ → f → ...  等于  a → f → ...
        └───┘

定律 2：取出东西又原样放回 = 什么都没做
        ┌───┐        ┌───┐
        │ a │ → Pure │ a │
        └───┘        └───┘
        
定律 3：连续操作可以重新分组
        m → f → g  等于  m → (f · g)
```

---

## 用 Kleisli 组合表达（更简洁）

```csharp
// Kleisli 组合：(>=>) 
// f >=> g = x => f(x).Bind(g)

// 三定律变成：
Pure >=> f      == f           // 左单位元
f >=> Pure      == f           // 右单位元
(f >=> g) >=> h == f >=> (g >=> h)  // 结合律
```

这就是说 **Kleisli 箭头形成一个 Category（范畴）**，Pure 是恒等态射。

---

## 违反定律的后果

```csharp
// 假设 Option 违反右单位元：
Some(5).Bind(Pure) == None  // 错误实现！

// 那么这段代码会出问题：
var result = from x in GetUser(id)
             select x;  // 预期返回 user，但实际返回 None！
```

---

## 快速记忆口诀

```
左单位：Pure 进去就出来，等于没进过
右单位：Bind 到 Pure，原封不动
结合律：链式操作随便括，结果一样
```
