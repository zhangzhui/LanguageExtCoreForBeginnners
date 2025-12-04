# Map / Bind / Select / SelectMany 概念关系

这几个概念来源于函数式编程的两个核心抽象：**Functor** 和 **Monad**。

Map/Bind : **Functional Programming**

Select/SelectMany : **LINQ**

## 1. Map 与 Select（Functor）

```
Map:    F<A> → (A → B) → F<B>
Select: F<A> → (A → B) → F<B>    // C# LINQ 名称
```

它们是**完全相同的操作**，只是名字不同：
- **Map** - 函数式编程（Haskell/Scala）的标准名称
- **Select** - C# LINQ 的名称，用于支持 `from x in m select f(x)` 语法

在 language-ext 中 `Option.cs`:
```csharp
public Option<B> Select<B>(Func<A, B> f) =>
    isSome ? Option.Some(f(Value!)) : default;

public Option<B> Map<B>(Func<A, B> f) =>
    isSome ? Option.Some(f(Value!)) : default;
```

## 2. Bind 与 SelectMany（Monad）

```
Bind:       M<A> → (A → M<B>) → M<B>
SelectMany: M<A> → (A → M<B>) → ((A, B) → C) → M<C>   // 带 project 参数
```

- **Bind** - 函数式编程的标准名称（Haskell 中是 `>>=`）
- **SelectMany** - C# LINQ 的名称

**关键区别**：C# 的 `SelectMany` 带有一个额外的 `project` 参数，这是为了支持多 `from` 的 LINQ 语法：

```csharp
// 这段 LINQ 语法
from a in optA
from b in GetB(a)
select Combine(a, b)

// 编译器翻译成
optA.SelectMany(
    a => GetB(a),           // bind 函数
    (a, b) => Combine(a, b) // project 函数
)
```

在 `Option.cs`:
```csharp
public Option<C> SelectMany<B, C>(
    Func<A, Option<B>> bind,    // 第一个参数
    Func<A, B, C> project)      // 第二个参数（project）
{
    if (IsNone) return default;
    var mb = bind(Value!);
    if (mb.IsNone) return default;
    return project(Value!, mb.Value!);  // 同时使用 a 和 b
}
```

## 3. 关系图

```
┌─────────────────────────────────────────────────────────┐
│                     Functor                             │
│  Map / Select:  容器内的值转换                           │
│  Option<int> → (int → string) → Option<string>          │
└─────────────────────────────────────────────────────────┘
                        ↓ 扩展
┌─────────────────────────────────────────────────────────┐
│                      Monad                              │
│  Bind / SelectMany:  容器内的值产生新容器，然后扁平化      │
│  Option<int> → (int → Option<string>) → Option<string>  │
└─────────────────────────────────────────────────────────┘
```

## 4. 为什么 SelectMany 需要 project 参数？

纯 `Bind` 只能访问第二个操作的结果，但 LINQ 需要同时访问多个 `from` 绑定的变量：

```csharp
from x in optX      // 绑定 x
from y in optY      // 绑定 y  
from z in optZ      // 绑定 z
select x + y + z    // 需要同时访问 x, y, z
```

`project` 参数让我们保留前面步骤的值，实现链式组合。

## 5. 总结表格

| FP 名称 | C# LINQ 名称 | 类型签名 | 用途 |
|--------|-------------|---------|------|
| Map | Select | `F<A> → (A→B) → F<B>` | 转换容器内的值 |
| Bind | SelectMany | `M<A> → (A→M<B>) → M<B>` | 链式组合产生新容器的操作 |
| - | SelectMany (带 project) | `M<A> → (A→M<B>) → ((A,B)→C) → M<C>` | LINQ 多 from 语法支持 |

## 6. language-ext 中的运算符

language-ext 还提供了运算符形式：
- `*` 运算符 = Map（Functor map）
- `>>` 运算符 = Bind（Monad bind）

```csharp
// Option.Operators.Functor.cs
public static Option<B> operator *(Func<A, B> f, K<Option, A> ma) =>
    +ma.Map(f);

// Option.Operators.Monad.cs  
public static Option<B> operator >> (K<Option, A> ma, Func<A, K<Option, B>> f) =>
    +ma.Bind(f);
```

## 7. Summary1 (Use an example to explain the concepts)
```csharp
var o1 = Option<int>.Some(1);
var o2 = Option<int>.Some(2);
var o3 = Option<int>.Some(3);


// syntactic sugar: SelectMany
var res = from a1 in o1
          from a2 in o2
          from a3 in o3
          select a1 + a2 + a3;
Console.WriteLine($"LINQ: {res}");



// use SelectMany, better than Bind
var res1 = o1.SelectMany(a1 => o2, (a1, a2) => a1 + a2)
    .SelectMany(prev => o3, (prev, a3) => prev + a3);
Console.WriteLine($"SelectMany: {res1}");

// use bind like a hell
var res2 = o1.Bind(a1 =>
{
    return o2.Bind(a2 =>
    {
        return o3.Bind(a3 =>
        {
            return Some(a1 + a2 + a3);
        });
    });
});
Console.WriteLine($"Bind: {res2}");

```

## 8. Summary2 (Use different monad)
```csharp
var o1 = Option<int>.Some(1);
var e2 = Either<string, int>.Right(2);
var o3 = Option<int>.Some(3);


// syntactic sugar: SelectMany
var res1 = from a1 in o1
           from a2 in e2
           from a3 in o3
           select a1 + a2 + a3;
res1.AsIterable().Iter(v => Console.WriteLine(v));

// use SelectMany, better than Bind
var res2 = o1.SelectMany(a1 => e2.ToOption(), (a1, a2) => a1 + a2)
             .SelectMany(prev => o3, (prev, a3) => prev + a3);
Console.WriteLine(res2);

// all lift to Iterable
// think of use Bind version
// 
var res22 = o1.SelectMany(a1 => e2.ToIterable(), (a1, a2) => a1 + a2)
              .SelectMany(prev => o3.ToIterable(), (prev, a3) => prev + a3);
res22.AsIterable().Iter(v => Console.WriteLine(v));


// use bind like a hell
var res3 = o1.Bind(a1 =>
{
    return e2.Bind(a2 =>
    {
        return o3.Bind(a3 =>
        {
            // here returns an Iterable
            // otherwise it must use ToEither
            // and outside must use ToOption
            // 
            // I'm a lazy person.
            // I don't like so much work to do
            //
            return Seq(a1 + a2 + a3);
        });
    });
});
res3.AsIterable().Iter(v => Console.WriteLine(v));
```

