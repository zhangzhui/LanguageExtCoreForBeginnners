# OptionT<M, A> 设计哲学详解

## 1. 核心数据结构

```csharp
public record OptionT<M, A>(K<M, Option<A>> runOption) : K<OptionT<M>, A>
    where M : Monad<M>
```

**关键洞察**：`OptionT<M, A>` 本质上是 `M<Option<A>>` 的包装。

对于 `OptionT<IO, int>`，其内部结构是：
```
IO<Option<int>>
```

这意味着：先执行 IO 副作用，然后得到一个 `Option<int>` 结果。

---

## 2. 如何同时保留两个 Monad 的语义

### Option 的语义（可能有值/无值）

通过 `Match`、`IfNone`、`IfSome` 等方法保留：

```csharp
public K<M, B> Match<B>(Func<A, B> Some, Func<B> None) =>
    M.Map(mx => mx.Match(Some, None), runOption);  // 委托给内部 Option
```

### IO 的语义（延迟执行的副作用）

通过 `Run()` 方法暴露：

```csharp
public K<M, Option<A>> Run() => runOption;  // 返回 IO<Option<A>>
```

### Bind 的实现 - 关键！

```csharp
public OptionT<M, B> Bind<B>(Func<A, OptionT<M, B>> f) =>
    new(M.Bind(runOption,                           // 外层: 用 M (IO) 的 Bind
               ox => ox.Match(                      // 中间: 拆开 Option
                   Some: x => f(x).runOption,       // Some: 继续计算
                   None: () => M.Pure(Option<B>.None))));  // None: 短路
```

**语义组合的关键**：
1. **IO 的 Bind**：保证副作用按顺序执行
2. **Option 的短路**：None 时不执行后续计算，直接返回 `IO.Pure(None)`

---

## 3. Trait 实现 - 让 OptionT 成为合法 Monad

```csharp
public partial class OptionT<M> : 
    MonadT<OptionT<M>, M>,      // Monad Transformer trait
    Alternative<OptionT<M>>,    // 支持 choice/empty
    Fallible<Unit, OptionT<M>>, // 支持 fail
    MonadIO<OptionT<M>>         // 支持 liftIO
    where M : Monad<M>
```

关键实现：

```csharp
// Lift: 将底层 Monad 提升到 Transformer
static K<OptionT<M>, A> MonadT<OptionT<M>, M>.Lift<A>(K<M, A> ma) => 
    OptionT.lift(ma);  // 把 IO<A> 变成 OptionT<IO, A> (即 IO<Some<A>>)

// LiftIO: 将 IO 提升到任意支持 IO 的 Transformer
static K<OptionT<M>, A> MonadIO<OptionT<M>>.LiftIO<A>(IO<A> ma) => 
    OptionT.lift(M.LiftIOMaybe(ma));
```

---

## 4. 设计 Monad Transformer 的规范

如果你要设计一个 `FooT<M, A>` transformer，需要遵循：

### 规范 1：内部表示必须是 `M<Foo<A>>`

```csharp
// 正确
public record FooT<M, A>(K<M, Foo<A>> runFoo) : K<FooT<M>, A>

// 错误: Foo<M<A>> - 这会丢失底层 Monad 的控制权
```

**原因**：Transformer 让底层 Monad M 成为"外层控制者"，内层 Monad Foo 被"包装"在里面。

### 规范 2：实现 `MonadT<FooT<M>, M>` trait

```csharp
public interface MonadT<T, out M> : Monad<T> 
    where T : MonadT<T, M>
    where M : Monad<M>
{
    public static abstract K<T, A> Lift<A>(K<M, A> ma);
}
```

`Lift` 的实现模式：
```csharp
// 将 M<A> 提升为 M<Foo<A>>
static K<FooT<M>, A> Lift<A>(K<M, A> ma) => 
    new FooT<M, A>(ma.Map(a => Foo.Pure(a)));
```

### 规范 3：Bind 必须正确组合两个 Monad 的语义

```csharp
public FooT<M, B> Bind<B>(Func<A, FooT<M, B>> f) =>
    new(M.Bind(runFoo,                    // 用 M 的 Bind
               fx => fx.Bind(             // 用 Foo 的 Bind (或等效逻辑)
                   x => f(x).runFoo)));
```

### 规范 4：提供 `Run()` 方法暴露内部结构

```csharp
public K<M, Foo<A>> Run() => runFoo;
```

这允许用户在需要时"拆开" transformer。

### 规范 5：满足 Monad Laws

必须通过以下定律检验：
- **Left Identity**: `Pure(a).Bind(f) == f(a)`
- **Right Identity**: `m.Bind(Pure) == m`  
- **Associativity**: `m.Bind(f).Bind(g) == m.Bind(a => f(a).Bind(g))`

---

## 5. 图解总结

```
┌─────────────────────────────────────────────────────────┐
│                  OptionT<IO, int>                       │
├─────────────────────────────────────────────────────────┤
│  内部结构:  IO<Option<int>>                             │
│                                                         │
│  IO 语义:   延迟执行，可以有副作用                        │
│  Option 语义: 计算可能成功(Some)或失败(None)             │
│                                                         │
│  Map:  f 只作用于 Some 内的值                            │
│  Bind: None 时短路，Some 时继续                          │
│  Run:  暴露 IO<Option<int>>，执行 IO                    │
└─────────────────────────────────────────────────────────┘

流程图:
OptionT<IO,A>.Bind(f) =
    IO.Bind ──> Option.Match ──> Some: f(x).runOption
                              └─> None: IO.Pure(None)
```

---

## 6. 实际使用示例

```csharp
// 组合 IO 和 Option 的效果
OptionT<IO, int> computation =
    from user   in getUser(userId)           // OptionT<IO, User>  
    from config in getConfig()               // Option<Config> 被自动 lift
    from result in saveData(user, config)    // IO<int> 被自动 lift
    select result;

// 运行时:
// 1. 执行 getUser 的 IO，如果 None 则短路
// 2. 检查 config，如果 None 则短路  
// 3. 执行 saveData 的 IO
// 4. 最终得到 IO<Option<int>>
```

---

## 7. 为什么是 M<Option< A >>而不是 Option<M< A >>？

| 结构 | 含义 | 问题 |
|-----|------|-----|
| `IO<Option<A>>` | 执行 IO，然后可能有值 | ✅ 正确 |
| `Option<IO<A>>` | 可能有一个 IO 操作 | ❌ None 时无法执行任何 IO |

**核心原则**：底层 Monad (M) 必须是外层（副作用一定是外围的，因为要根外界交互），才能保证其效果（如 IO 的副作用控制）始终生效。

---

## 8. SelectMany 重载的设计意图

OptionT 提供多个 SelectMany 重载，支持在 LINQ 中混用不同类型：

```csharp
// 1. OptionT -> OptionT (标准)
SelectMany<B, C>(Func<A, OptionT<M, B>> bind, ...)

// 2. OptionT -> M (自动 lift 底层 Monad)
SelectMany<B, C>(Func<A, K<M, B>> bind, ...)

// 3. OptionT -> Option (自动 lift Option)
SelectMany<B, C>(Func<A, Option<B>> bind, ...)

// 4. OptionT -> Pure (纯值)
SelectMany<B, C>(Func<A, Pure<B>> bind, ...)
```

这使得 LINQ 语法可以无缝混合不同的 Monad 类型。
