# IO.cs 与 IO.Monad.cs 的区别与联系

这两个文件共同构成了 `LanguageExt` 库中 `IO` Monad 的完整实现。它们通过 `partial class` 机制将功能分成了两个部分:核心实现与特质(Type Class)适配。

## 1. `IO.cs` —— 核心定义与运行时 (The Core Implementation)
这个文件包含了 `IO<A>` 类型的主要逻辑和运行时解释器。它是 `IO` 的"肉身"。

*   **定义数据结构**:定义了 `public abstract record IO<A>`,这是表示副作用计算的核心类型。
*   **核心操作**:实现了具体的 Monad 操作,如 `Map`, `Bind`, `SelectMany` (LINQ 支持), `Bracket` (资源管理), `Retry`, `Repeat` 等。
*   **运行时 (Interpreter)**:包含最关键的 `Run` 和 `RunAsync` 方法。这是 `IO` 运行时的核心循环(代码 800-1000 行左右的 `while` 循环),负责解释和执行 IO 指令,处理异步操作、栈管理和错误传播。
*   **核心逻辑**:所有的实际业务逻辑(如重试逻辑 `RetryUntil`,折叠逻辑 `Fold`)都写在这里。

## 2. `IO.Monad.cs` —— 特质适配 (Trait Implementation)
这个文件包含了 `IO` 类型对 `LanguageExt` 特质系统(Traits/Type Classes)的实现。它是 `IO` 的"名片",让 `IO` 能被当作通用的 Monad 使用。

*   **实现接口**:实现了 `MonadUnliftIO<IO>`, `Final<IO>`, `Fallible<IO>`, `Alternative<IO>` 等高阶接口。
*   **静态桥接**:这里的方法大多是静态的(`static K<IO, A> ...`),它们通常不包含具体逻辑,而是直接调用 `IO.cs` 中定义的实例方法。
    *   例如:`Monad<IO>.Bind` 只是简单地调用了 `ma.As().Bind(f)`。
*   **通用性**:这使得你可以编写针对泛型 `M` 的代码(`where M : Monad<M>`),并将 `IO` 作为 `M` 传入使用。

## 总结与联系

| 特征 | `IO.cs` | `IO.Monad.cs` |
| :--- | :--- | :--- |
| **角色** | **核心实现者** (Concrete Type) | **适配器** (Type Class Instance) |
| **关注点** | 具体的执行逻辑、状态机、LINQ 支持、性能优化 | 满足接口契约、泛型编程支持 |
| **代码风格** | 包含复杂的 `switch` 解释器循环和具体算法 | 主要是委托调用 (Delegation) 的样板代码 |
| **关系** | 提供实际干活的方法(如 `Bind`, `Run`) | 调用 `IO.cs` 的方法来满足 Trait 接口要求 |

**一句话概括**:
`IO.cs` 负责 **"怎么做"** (具体的异步执行和副作用处理),而 `IO.Monad.cs` 负责 **"是什么"** (声明它是 Monad、可失败的、支持并发的),从而让 `IO` 能融入 LanguageExt 庞大的函数式抽象体系中。
