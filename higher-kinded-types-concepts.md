# 高阶类型 (Higher-Kinded Types, HKT)

## 什么是高阶类型？

**高阶类型**是指"类型的类型构造器"——即接受`一个`或`多个`类型参数并返回新类型的类型。

用类比来理解：
- **普通值**：`5`, `"hello"` — 具体的值
- **函数**：`int → string` — 接受值，返回值
- **高阶函数**：`(int → int) → int` — 接受函数，返回值
- **普通类型**：`int`, `string` — 具体的类型 (kind: `*`)
- **类型构造器**：`List<_>`, `Option<_>` — 接受类型，返回类型 (kind: `* → *`)
- **高阶类型**：`Functor<F<_>>` — 接受类型构造器，返回类型 (kind: `(* → *) → *`)

## Kind 系统

| Kind | 含义 | 示例 |
|------|------|------|
| `*` | 具体类型 | `int`, `string`, `List<int>` |
| `* → *` | 一元类型构造器 | `List<_>`, `Option<_>` |
| `* → * → *` | 二元类型构造器 | `Either<_, _>`, `Map<_, _>` |
| `(* → *) → *` | 高阶类型 | `Functor<F>`, `Monad<M>` |

## 相关术语

| 术语 | 英文 | 含义 |
|------|------|------|
| 类型构造器 | Type Constructor | 接受类型参数产生新类型，如 `List<T>` |
| Kind | Kind | 类型的"类型"，描述类型的结构 |
| 类型类 | Type Class | 定义一组类型共享的行为（如 `Functor`, `Monad`） |
| 多态 | Polymorphism | 代码可以操作多种类型 |
| 参数多态 | Parametric Polymorphism | 泛型，如 `List<T>` |
| Ad-hoc 多态 | Ad-hoc Polymorphism | 通过类型类/接口实现的多态 |
| 关联类型 | Associated Types | 类型类中定义的类型成员 |
| 类型族 | Type Families | Haskell 中的类型级函数 |

## 为什么需要 HKT？

HKT 允许我们抽象`容器本身`，而不仅仅是容器中的元素：

```haskell
-- Haskell 中可以直接表达
class Functor f where
  fmap :: (a -> b) -> f a -> f b
  
-- f 是一个类型构造器 (kind: * → *)
-- 可以是 List, Maybe, Either e, IO 等
```

```csharp
// C# 没有原生 HKT 支持，需要模拟
// language-ext 使用特殊技术来实现类似功能
public interface Functor<F> {
    // 无法直接写 F<B> Map<A, B>(F<A> fa, Func<A, B> f)
}
```

## 实际应用

HKT 是实现以下抽象的基础：
- **Functor** — 可映射的容器
- **Applicative** — 可应用函数的容器  
- **Monad** — 可链式组合的计算
- **Traverse** — 可遍历并交换效果的结构
- **Free Monad** — 将程序描述与执行分离

这些抽象让我们可以编写与具体容器类型无关的通用代码。
