# Alternative Value Type vs Bound Value Type

## 定义

### Bound Value Type (绑定值类型)
- 指类型构造器中被"绑定"的主要类型参数
- 通常是容器/计算中携带的**成功值**或**核心数据**
- 在 `K<F, A>` 中，`A` 就是 Bound Value Type

### Alternative Value Type (替代值类型)
- 指类型构造器中表示**替代路径**的类型参数
- 通常用于表示**错误**、**失败**或**左分支**的值
- 在 `Either<L, R>` 中，`L` 就是 Alternative Value Type

## 对比

| 特性 | Bound Value Type | Alternative Value Type |
|------|------------------|------------------------|
| 语义 | 主路径/成功值 | 替代路径/失败值 |
| 位置 | 通常是最后一个类型参数 | 通常是倒数第二个类型参数 |
| Functor 作用 | `Map`/`Select` 操作此类型 | 通常不受 `Map` 影响 |
| 示例 | `Either<L, R>` 中的 `R` | `Either<L, R>` 中的 `L` |

## 具体示例

```
类型                    Bound Value    Alternative Value
─────────────────────────────────────────────────────────
Option<A>               A              (无，用 None 表示)
Either<L, R>            R              L
Validation<F, S>        S (Success)    F (Failure)
Fin<A>                  A              Error
IO<A>                   A              (隐式异常)
EitherT<L, M, R>        R              L
```

## 在 Functor/Monad 中的行为

```csharp
// Map 只作用于 Bound Value Type (R)
Either<string, int> either = Right(42);
var mapped = either.Map(x => x * 2);  // Right(84)

// L (Alternative Value) 不受影响
Either<string, int> left = Left("error");
var mapped2 = left.Map(x => x * 2);   // Left("error") - 不变
```

## Bifunctor —— 同时操作两种类型

当需要同时操作两个类型参数时，使用 **Bifunctor**：

```csharp
// BiMap 可以同时转换 L 和 R
Either<string, int> either = Left("error");
var result = either.BiMap(
    Left:  err => err.ToUpper(),      // 转换 Alternative Value
    Right: val => val * 2              // 转换 Bound Value
);
// 结果: Left("ERROR")
```

## Kind 表示

```
Kind 表达式              含义
────────────────────────────────────────────
* → *                   一个 Bound Value (如 Option<_>)
* → * → *               Alternative + Bound (如 Either<_, _>)
```

在 language-ext 的 `K<F, A>` 系统中：
- `K<Option, A>` — `A` 是 Bound Value
- `K<Either<L>, R>` — `R` 是 Bound Value，`L` 是 Alternative Value（被"部分应用"到 `Either` 上）

## 为什么这个区分重要？

1. **Monad 绑定链**只传播 Bound Value，遇到 Alternative 时短路
2. **错误处理**逻辑依赖这个区分（Right-biased）
3. **类型类实例**通常只对 Bound Value 定义（如 `Functor<Either<L>>`）
4. **组合性**：可以固定 Alternative Type 来组合不同的计算
