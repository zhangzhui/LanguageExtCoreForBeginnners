# language-ext 操作符：Functor 与 Applicative

language-ext 使用 **`*` 操作符** 作为语法糖，对应 Haskell 中的 `<$>` 和 `<*>`。

---

## 操作符对照表

| Haskell | language-ext | 含义 |
|---------|--------------|------|
| `<$>` (fmap) | `*` | Functor map |
| `<*>` (apply) | `*` | Applicative apply |
| `*>` / `>>` | `>>>` | Applicative sequence (丢弃左边结果) |

---
## `operator *`不知道为什么报错

---

## 用法示例

### Functor map (`*` = `<$>`)

```csharp
// Haskell: (+1) <$> Just 5
// language-ext:
Func<int, int> addOne = x => x + 1;
//Option<int> result = addOne * Some(5);  // Some(6)
Option<int> result = Some(addOne).Apply(Some(5)); // Some(6)
Option<int> result1 = map(addOne, Some(5)); // Some(6)

```

### Applicative apply (`*` = `<*>`)

```csharp
// Haskell: Just (+) <*> Just 1 <*> Just 2
// language-ext:
Func<int, int, int> add = (a, b) => a + b;
//Option<int> result = add * Some(1) * Some(2);  // Some(3)
Option<int> result = Some(add).Apply(Some(1)).Apply(Some(2));
Option<int> result = map(add, Some(1)).Apply(Some(2));
```

### Applicative sequence (`>>>` = `*>`)

```csharp
// 执行左边，丢弃结果，返回右边
Option<int> result = Some("hello") >>> Some(42);  // Some(42)
```

---

## 为什么都用 `*`？

C# 的操作符重载限制较多，不能自定义 `<$>` 或 `<*>` 这样的符号。`*` 被选中是因为：

1. **可以重载** - C# 允许重载 `*` 操作符
2. **视觉上简洁** - 比方法调用更接近数学表达式
3. **通过类型推断区分**：
   - `Func<A,B> * K<M,A>` → Functor map（函数 + 容器）
   - `K<M, Func<A,B>> * K<M,A>` → Applicative apply（容器里的函数 + 容器）

---

## 源码位置

- Functor 操作符：`LanguageExt.Core/Traits/Functor/Functor.Operators.cs`
- Applicative 操作符：`LanguageExt.Core/Traits/Applicative/Applicative.Operators.cs`


