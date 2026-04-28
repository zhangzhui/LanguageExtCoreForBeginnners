在 LanguageExt 中，`Bind`（以及 LINQ 中的 `SelectMany` / `from...select`）**默认严格要求是相同的 Monad 类型**。

总结来说，它的设计规则如下：

### 1. 核心规则：必须保持相同的 Monad "形状"
在 LanguageExt 的类型系统和 Monad Trait 定义中，`Bind` 的签名严格绑定在同一个高级类型（Higher-Kinded Type）上：
* `Option<A>.Bind` 必须返回 `Option<B>`
* `Either<L, A>.Bind` 必须返回 `Either<L, B>`（且左侧的错误类型 `L` 必须相同）
* `Eff<A>.Bind` 必须返回 `Eff<B>`
* `Seq<A>.Bind` 必须返回 `Seq<B>`

### 2. 为什么不能直接跨 Monad Bind？
因为不同的 Monad 有不同的短路/求值语义：
如果允许 `Option<A>.Bind(a => Either<L, B>)`，当 `Option` 是 `None` 时，系统不知道该如何凭空构造出一个左值的 `L` 错误塞进 `Either` 里。

### 3. 如果需要跨 Monad 组合（例如 Option 混 Either，或者 Eff 混 Aff），怎么处理？

虽然不能直接 Bind，但 LanguageExt 提供了标准的函数式方法来处理混合嵌套：

#### A. 显式转换（ToX 扩展方法）
这是最常见的做法，先将一种 Monad 转换成另一种，然后再 Bind：
```csharp
Option<int> opt = Some(5);
Either<string, int> eith = Right<string, int>(10);

// 将 Option 转换为 Either 再组合
var result = from a in opt.ToEither("缺省错误信息")
             from b in eith
             select a + b;
```

#### B. 提升（Lifting / Monad Transformers）
如果业务中经常需要处理嵌套类型（比如 `IO<Option<A>>` 或 `Eff<Either<L, A>>`），LanguageExt 提供了 Monad Transformers（例如 `OptionT`, `EitherT`, `FinT` 等）：
它们专门用来把一个 Monad 嵌套在另一个里面，并提供一体化的 `Bind` 体验。

#### C. 在 Effects (Eff/IO) 中的特殊处理
在现代的 `Eff` / `IO` 系统中，它们提供了专用的扩展和 lift 方法，让你能将底层的 `Option`、`Either` 或者无状态的 `Eff` 提升到有状态或异步的 `Aff` / `IO` 栈中去执行。

**结论**：`Bind` 必须是同一个 Monad。当你遇到不同类型的 Monad 需要串联计算时，你需要通过 `.ToEither()`、`.ToOption()`、`.ToEff()` 等转换方法，先将它们对齐到同一个更宽泛的上下文（通常是错误包含力最强的那一个，比如 Either 或 Eff），然后再进行 Bind。
