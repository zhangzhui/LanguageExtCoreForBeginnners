# OptionT 目录结构与文件命名规范详解

## 背景与设计哲学

language-ext 库遵循函数式编程的设计原则，特别是 Haskell/Scala 的类型类（Type Class）模式。OptionT 是一个 **Monad Transformer**（单子变换器），用于在另一个 Monad（如 IO、Eff 等）中嵌套 Option 类型。

命名和目录结构遵循以下原则：
1. **关注点分离**：不同功能放在不同的文件和目录中
2. **类型类驱动**：按照函数式编程的类型类（Functor、Applicative、Monad 等）组织代码
3. **C# partial class 特性**：利用 partial class 将大类拆分到多个文件中

---

## 目录结构总览

```
OptionT/
├── OptionT.cs                    # 核心类型定义（record 类型）
├── OptionT.Module.cs             # 静态工厂方法（模块）
├── Extensions/                   # 扩展方法（作用于实例）
│   ├── OptionT.Extensions.cs
│   └── OptionT.Extensions.MapApply.cs
├── Operators/                    # 运算符重载（使用 C# extension 语法）
│   ├── OptionT.Operators.cs
│   ├── OptionT.Operators.Functor.cs
│   ├── OptionT.Operators.Applicative.cs
│   ├── OptionT.Operators.Monad.cs
│   ├── OptionT.Operators.Choice.cs
│   ├── OptionT.Operators.Fallible.cs
│   ├── OptionT.Operators.Final.cs
│   └── OptionT.Operators.SemigroupK.cs
├── Prelude/                      # 全局辅助函数
│   └── OptionT.Prelude.mapapply.cs
└── Trait/                        # Trait 实现（类型类接口实现）
    └── OptionT.TraitImpl.cs
```

---

## 详细说明

### 1. 根目录文件

#### `OptionT.cs` - 核心类型定义

**职责**：定义 `OptionT<M, A>` 的核心结构和实例方法。

**包含内容**：
- **类型定义**：`public record OptionT<M, A>(K<M, Option<A>> runOption)`
- **实现的接口**：`K<OptionT<M>, A>` - higher-kinded type 模拟
- **静态常量**：如 `None` 
- **实例方法**：
  - Match 操作（模式匹配）
  - Map / Select（Functor 操作）
  - Bind / SelectMany（Monad 操作）
  - IfSome / IfNone（条件操作）
  - Run（获取内部值）
  - MapM / MapT（Monad Transformer 特定操作）
- **隐式转换**：从 `Option<A>`、`Pure<A>`、`IO<A>` 等转换到 `OptionT`
- **转换方法**：如 `ToEither<L>()`

**命名原因**：这是类型的主文件，名称与类型名相同，是最直接的入口点。

---

#### `OptionT.Module.cs` - 静态工厂模块

**职责**：提供静态工厂方法，用于创建 `OptionT` 实例。

**包含内容**：
```csharp
public partial class OptionT
{
    public static OptionT<M, A> Some<M, A>(A value);   // 创建 Some 值
    public static OptionT<M, A> None<M, A>();          // 创建 None 值
    public static OptionT<M, A> lift<M, A>(...);       // 各种 lift 方法
    public static OptionT<M, A> liftIO<M, A>(...);     // 从 IO 提升
}
```

**命名原因**：
- "Module" 来源于 Haskell/F# 的概念，表示与类型相关的静态函数集合
- 类似于 Scala 的 companion object
- 允许用户以 `OptionT.Some<M, A>(value)` 的方式调用

---

### 2. `Extensions/` 目录 - 扩展方法

**职责**：提供作用于 `K<OptionT<M>, A>` 或 `OptionT<M, A>` 的扩展方法。

#### `OptionT.Extensions.cs` - 核心扩展

**包含内容**：
- `As()` - 向下转型：`K<OptionT<M>, A>` → `OptionT<M, A>`
- `Run()` - 运行变换器获取结果
- `Bind()` - 与 IO 等类型组合的 Bind 重载
- `Flatten()` - 扁平化嵌套的 OptionT
- `SelectMany()` - LINQ 查询支持

**命名原因**：`Extensions` 是 C# 标准命名约定，表示这是扩展方法的集合。

#### `OptionT.Extensions.MapApply.cs` - Map 和 Apply 扩展

**包含内容**：
- 将函数作为主体的 `Map` 扩展（`f.Map(ma)` 而非 `ma.Map(f)`）
- `Apply` 扩展方法
- `Action` 扩展方法

**命名原因**：文件名反映其内容，专注于 Functor(Map) 和 Applicative(Apply) 操作。

---

### 3. `Operators/` 目录 - 运算符重载

**职责**：定义 C# 运算符重载，使函数式操作更简洁。

**背景说明**：language-ext 使用 C# 14 的 `extension` 语法（目前是预览特性），允许为泛型类型定义运算符。

#### 运算符文件分类

| 文件名 | 运算符 | 对应类型类 | 功能 |
|--------|--------|------------|------|
| `Operators.cs` | `+`（downcast） | - | 向下转型辅助 |
| `Operators.Functor.cs` | `*`（map） | Functor | `f * ma` = `ma.Map(f)` |
| `Operators.Applicative.cs` | `*`（apply）, `>>>` | Applicative | `mf * ma` = `mf.Apply(ma)` |
| `Operators.Monad.cs` | `>>`（bind） | Monad | `ma >> f` = `ma.Bind(f)` |
| `Operators.Choice.cs` | `\|`（choice） | Choice/Alternative | `ma \| mb` = `ma.Choose(mb)` |
| `Operators.Fallible.cs` | `\|`（catch） | Fallible | 错误恢复 |
| `Operators.Final.cs` | `\|`（finally） | Final | finally 操作 |
| `Operators.SemigroupK.cs` | `+`（combine） | SemigroupK | 半群组合 |

**命名原因**：
- 每个文件对应一个函数式编程的类型类（Type Class）
- 文件名格式：`TypeName.Operators.TraitName.cs`
- 这种命名方式使得查找特定功能变得容易

#### 运算符符号含义

```
* (星号)：
  - Func<A,B> * K<F,A>  → Functor map
  - K<F, Func<A,B>> * K<F,A> → Applicative apply

>> (右移)：
  - K<F,A> >> Func<A, K<F,B>> → Monad bind
  - K<F,A> >> K<F,B> → 序列化（丢弃第一个结果）

>>> (三重右移)：
  - Applicative action（序列操作）

| (管道)：
  - Choice/Alternative 选择
  - Fallible 错误处理
  - Final 最终操作

+ (加号)：
  - Downcast（单目）
  - SemigroupK combine（双目）
```

---

### 4. `Prelude/` 目录 - 全局辅助函数

**职责**：提供可通过 `using static LanguageExt.Prelude;` 导入的全局函数。

#### `OptionT.Prelude.mapapply.cs`

**包含内容**：
```csharp
public static partial class Prelude
{
    public static OptionT<M, B> map<M, A, B>(Func<A, B> f, K<OptionT<M>, A> ma);
    public static OptionT<M, B> action<M, A, B>(K<OptionT<M>, A> ma, OptionT<M, B> mb);
    public static OptionT<M, B> apply<M, A, B>(K<OptionT<M>, Func<A, B>> mf, K<OptionT<M>, A> ma);
}
```

**命名原因**：
- "Prelude" 来源于 Haskell，是默认导入的基础函数模块
- 允许用户以 `map(f, ma)` 的前缀函数风格调用

---

### 5. `Trait/` 目录 - 类型类实现

**职责**：实现 language-ext 定义的各种 Trait（类型类接口）。

#### `OptionT.TraitImpl.cs`

**包含内容**：
```csharp
public partial class OptionT<M> : 
    MonadT<OptionT<M>, M>,      // Monad Transformer
    Alternative<OptionT<M>>,    // Alternative（Choice + MonoidK）
    Fallible<Unit, OptionT<M>>, // 错误处理
    MonadIO<OptionT<M>>         // IO 操作
    where M : Monad<M>
{
    // 实现各个接口的静态方法
    static K<OptionT<M>, B> Monad<OptionT<M>>.Bind<A, B>(...);
    static K<OptionT<M>, B> Functor<OptionT<M>>.Map<A, B>(...);
    static K<OptionT<M>, A> Applicative<OptionT<M>>.Pure<A>(...);
    // ... 等等
}
```

**命名原因**：
- "Trait" 是 language-ext 对类型类的称呼（借鉴 Rust/Scala）
- "TraitImpl" 表示这是 Trait 的实现文件
- 注意：这里的类是 `OptionT<M>`（不是 `OptionT<M, A>`），因为 higher-kinded type 需要 partial application

---

## 文件命名规范总结

### 格式模板
```
TypeName.Category.SubCategory.cs
```

### 具体规则

| 模式 | 示例 | 用途 |
|------|------|------|
| `TypeName.cs` | `OptionT.cs` | 核心类型定义 |
| `TypeName.Module.cs` | `OptionT.Module.cs` | 静态工厂方法 |
| `TypeName.Extensions.cs` | `OptionT.Extensions.cs` | 核心扩展方法 |
| `TypeName.Extensions.xxx.cs` | `OptionT.Extensions.MapApply.cs` | 特定功能扩展 |
| `TypeName.Operators.cs` | `OptionT.Operators.cs` | 基础运算符 |
| `TypeName.Operators.TraitName.cs` | `OptionT.Operators.Functor.cs` | 特定类型类运算符 |
| `TypeName.Prelude.xxx.cs` | `OptionT.Prelude.mapapply.cs` | 全局函数 |
| `TypeName.TraitImpl.cs` | `OptionT.TraitImpl.cs` | Trait 实现 |

---

## 为什么这样组织？

### 1. 可维护性
- 每个文件有明确的单一职责
- 容易找到特定功能的实现位置
- 修改某一功能不会影响其他文件

### 2. 编译效率
- 小文件编译更快
- IDE 索引和导航更高效

### 3. 符合函数式编程惯例
- 按类型类组织代码是 Haskell/Scala 的标准做法
- 方便有函数式背景的开发者理解

### 4. C# partial class 特性
- C# 允许将一个类分散到多个文件
- 结合 `partial` 关键字实现代码分离

### 5. 渐进式学习
- 新手可以先只关注 `OptionT.cs` 和 `OptionT.Module.cs`
- 深入学习时再看 Operators 和 Trait 目录

---

## 如何添加新的 Monad Transformer

参照 OptionT 的结构，创建新类型（如 `MyT`）时需要：

1. 创建目录 `MyT/`
2. 创建 `MyT.cs` - 核心类型
3. 创建 `MyT.Module.cs` - 工厂方法
4. 创建 `Extensions/MyT.Extensions.cs` - 扩展方法
5. 创建 `Trait/MyT.TraitImpl.cs` - Trait 实现
6. 根据需要创建 `Operators/` 目录下的运算符文件
7. 可选：创建 `Prelude/` 目录下的全局函数
