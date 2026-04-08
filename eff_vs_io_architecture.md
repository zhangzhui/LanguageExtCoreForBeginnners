在 `language-ext` 的架构设计中，`Eff` 和 `IO` 处于不同的抽象层级，分别解决不同层面的问题。这决定了它们对"环境"（Environment/Runtime）的定义和使用方式截然不同。

简单来说：**`Eff` 的 Runtime 是为了业务依赖注入（DI），而 `IO` 的 Env 是为了底层运行机制（线程、取消、资源）。**

以下是具体的架构差异：

### 1. `IO<A>`：底层的副作用执行引擎
`IO` 关心的是**代码如何在系统底层安全地执行**。它不关心你的业务依赖（如数据库、缓存）。在执行 `IO<A>`（如调用 `RunAsync()`）时所需的 `EnvIO` 是一个具象的**操作执行上下文**，它包含：
* `CancellationToken`：用于控制和传递异步操作的取消信号。
* `Resources`：用于追踪和管理 IDisposable 对象的生命周期，确保在发生异常时资源被安全释放。
* `SynchronizationContext`：用于管理线程上下文（比如在 UI 线程上恢复执行）。

**为什么 `IO` 不需要泛型 Env？**
因为无论你做什么副作用（读文件、网络请求），底层执行所需的机制（资源管理、取消逻辑）都是通用的、非业务特定的。因此它不需要泛型的 `Env`，只需要内部统一固定的 `EnvIO` 即可。

### 2. `Eff<RT, A>`：应用层的业务依赖层 (Capability / DI)
`Eff` 关心的是**业务逻辑和外部依赖**。在 `language-ext` 的源码中，`Eff<RT, A>` 实际上就是 `ReaderT<RT, IO, A>` 的封装结构。这里的 `RT` (Runtime) 是你的**业务执行环境**（例如 `HasDatabase`、`HasLogging`）。

**为什么 `Eff` 必须要有泛型 `RT` (Runtime)？**
因为在纯函数式编程中，我们需要把函数所依赖的外部能力（依赖注入）明确标注在类型签名上。`Eff` 通过 Reader Monad 模式，允许你在编译期静态声明一段代码需要哪些外部依赖。由于每个应用/组件的依赖都不同，所以它必须是泛型的。

### 3. 架构上的解耦设计 (Separation of Concerns)
这种设计的精妙之处在于**将"业务依赖"和"运行时执行机制"彻底解耦**，分为两层：

1. **上层业务**：你使用 `Eff<RT, A>` 编写业务逻辑，传入泛型的业务环境 `RT`。
2. **底层执行**：当你要真正执行这个 `Eff` 时，你提供一个具体的 `RT` 实例给它，此时它就会被降级（计算）成一个纯粹的 `IO<A>`。
3. **最终运行**：这个剥离了业务依赖的 `IO<A>` 最后再结合系统分配的 `EnvIO` 开始实际运行（处理线程、清理资源）。

**补充：没有业务依赖的场景**
如果你有一段代码仅仅是纯副作用（比如打印日志），不需要任何复杂的数据库或配置依赖，`language-ext` 也提供了无 Runtime 泛型的 `Eff<A>`。在源码中，它仅仅是 `Eff<MinRT, A>` 的别名。`MinRT` 相当于一个空的占位符 Runtime，这让你在不需要依赖注入时，依然能保持统一的 `Eff` 编程模型。

### 4. `IO` 与外部 `CancellationToken` 的结合
在实际应用中，将外部的 `CancellationToken`（例如 ASP.NET Core 的 `HttpContext.RequestAborted`）与 `IO` 结合非常简单，框架原生提供了支持。

#### 方式一：直接传给 `RunAsync`（最简单推荐）
`IO<A>` 提供了一个直接接收 `CancellationToken` 的重载方法：
```csharp
CancellationToken externalToken = ...;
var result = await myIo.RunAsync(externalToken);
```
**底层机制：** 源码内部会自动用你的 token 创建一个绑定在 using 生命周期内的 `EnvIO`。

#### 方式二：手动构建 `EnvIO`（高级控制）
如果你还需要同时配置资源追踪（`Resources`）或者同步上下文（`SynchronizationContext`）：
```csharp
using var env = EnvIO.New(token: externalToken);
var result = await myIo.RunAsync(env);
```

#### 核心解密：外部 Token 的桥接机制
在 `EnvIO.New(token: externalToken)` 的源码实现中，框架并没有粗暴地直接使用传入的外部 Token，而是做了严谨的**桥接设计**：
1. 创建了一个 `EnvIO` **内部专属**的 `CancellationTokenSource`。
2. 使用 `externalToken.Register(() => 内部Source.Cancel())` 将外部 Token 注册了回调。
3. 把合并后的内部 Token 提供给 `IO` 引擎使用。

**设计初衷：** `IO` 引擎不仅需要响应"外部取消"（如用户关闭了页面），还可能需要处理"内部取消"（比如在 `IO` 操作中设置了 `Timeout`）。通过桥接机制，无论是外部取消还是内部超时，`EnvIO` 都能统一捕捉并安全地熔断正在运行的副作用，同时触发 `Resources` 绑定的资源清理逻辑。
