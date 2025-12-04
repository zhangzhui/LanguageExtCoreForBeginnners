# Runtime 设计原则、背景及优缺点

## 1. 背景与来源

### 1.1 函数式编程中的 Effect 系统演进

```
传统 IO Monad (Haskell)
    ↓
Reader + IO 组合
    ↓
ZIO (Scala) - ZIO[R, E, A]  
    ↓
language-ext Eff<RT, A>
```

Runtime 设计的核心灵感来自 **ZIO (Scala)** 的 `ZIO[R, E, A]` 类型：
- `R` = Environment/Runtime（依赖/环境）
- `E` = Error（错误类型）
- `A` = Success value（成功值）

language-ext 的 `Eff<RT, A>` 采用了类似的设计，其中 `RT` 就是 Runtime 类型。

### 1.2 为什么需要 Runtime？

传统依赖注入（DI）的问题：
1. **隐式依赖**：构造函数注入隐藏了真正的依赖关系
2. **不纯粹**：注入的服务通常直接执行副作用
3. **难以测试**：需要 mock 框架
4. **运行时错误**：DI 容器配置错误只在运行时发现

## 2. 核心设计原则

### 2.1 Has Trait - 结构化依赖访问

```csharp
// Has.Trait.cs
public interface Has<in M, VALUE>
{
    public static abstract K<M, VALUE> Ask { get; }
}
```

`Has<M, VALUE>` 表示 "monad M 可以访问类型为 VALUE 的能力"。

### 2.2 Runtime 作为能力的聚合

```csharp
// Live/Runtime.cs
public record Runtime(RuntimeEnv Env) : 
    Has<Eff<Runtime>, ConsoleIO>,    // 有控制台能力
    Has<Eff<Runtime>, FileIO>,       // 有文件IO能力
    Has<Eff<Runtime>, TimeIO>,       // 有时间能力
    Has<Eff<Runtime>, DirectoryIO>,  // 有目录能力
    ...
{
    // 每个 Has 接口提供获取该能力的方式
    static K<Eff<Runtime>, ConsoleIO> Has<Eff<Runtime>, ConsoleIO>.Ask { get; } =
        pure(Implementations.ConsoleIO.Default);
}
```

### 2.3 IO Trait 接口 - 抽象副作用

```csharp
// Traits/ConsoleIO.cs
public interface ConsoleIO
{
    IO<Unit> Clear();
    IO<Option<ConsoleKeyInfo>> ReadKey();
    IO<Option<string>> ReadLine();
    IO<Unit> WriteLine(string value);
    // ...
}
```

关键点：**所有方法返回 `IO<T>` 而不是直接返回 `T`**，这保证了：
- 副作用被显式标记
- 操作是惰性的（lazy）
- 可以组合和转换

### 2.4 Live vs Test 实现

**Live 实现**（真实副作用）：
```csharp
// Live/Implementations/ConsoleIO.cs
public record ConsoleIO : Sys.Traits.ConsoleIO
{
    public IO<Option<string>> ReadLine() =>
        lift(() => Optional(Console.ReadLine()));  // 真实调用 System.Console
}
```

**Test 实现**（内存模拟）：
```csharp
// Test/Implementations/ConsoleIO.cs
public record ConsoleIO(MemoryConsole mem) : Sys.Traits.ConsoleIO
{
    public IO<Option<string>> ReadLine() =>
        lift(() => mem.ReadLine());  // 从内存缓冲区读取
}
```

## 3. 使用模式

### 3.1 业务逻辑使用泛型约束

```csharp
// Console.cs - 业务逻辑与 Runtime 无关
public static class Console<M, RT>
    where M : MonadIO<M>, Fallible<Error, M>
    where RT : Has<M, ConsoleIO>  // 约束：RT 必须提供 ConsoleIO
{
    static K<M, ConsoleIO> consoleIO =>
        Has<M, RT, ConsoleIO>.ask;  // 从环境获取 ConsoleIO

    public static K<M, string> readLine =>
        from t in consoleIO
        from k in t.ReadLine()
        from r in k.Match(Some: M.Pure, None: M.Fail<string>(Errors.EndOfStream))
        select r;
}
```

### 3.2 特化版本简化使用

```csharp
// Console.Eff.cs - Eff 特化版本
public static class Console<RT>
    where RT : Has<Eff<RT>, ConsoleIO>
{
    public static Eff<RT, string> readLine =>
        Console<Eff<RT>, RT>.readLine.As();
}
```

### 3.3 应用程序入口

```csharp
// 使用 Live Runtime
var runtime = Runtime.New();
var result = myProgram.Run(runtime, EnvIO.New());

// 使用 Test Runtime
var testRuntime = Test.Runtime.New();
var testResult = myProgram.Run(testRuntime, EnvIO.New());
```

## 4. 优点

### 4.1 编译时依赖检查
```csharp
// 如果 RT 没有实现 Has<Eff<RT>, ConsoleIO>，编译器会报错
public static Eff<RT, Unit> Program<RT>()
    where RT : Has<Eff<RT>, ConsoleIO>, Has<Eff<RT>, FileIO>
{
    // 类型系统保证 RT 必须提供 ConsoleIO 和 FileIO
}
```

### 4.2 纯函数式测试
```csharp
// 测试不需要 mock 框架
[Fact]
public void TestConsoleProgram()
{
    var runtime = Test.Runtime.New();
    runtime.Env.Console.WriteInput("test input");
    
    var result = MyProgram.Run(runtime, EnvIO.New());
    
    Assert.Equal("expected output", runtime.Env.Console.Output);
}
```

### 4.3 依赖关系显式化
```csharp
// 函数签名明确告诉你它需要什么
public static Eff<RT, Result> ProcessFile<RT>(string path)
    where RT : Has<Eff<RT>, FileIO>,      // 需要文件系统
               Has<Eff<RT>, ConsoleIO>,   // 需要控制台
               Has<Eff<RT>, TimeIO>       // 需要时间服务
```

### 4.4 组合性
```csharp
// 不同能力可以自由组合
from file in File<RT>.readAllText(path)
from _    in Console<RT>.writeLine(file)
from now  in Time<RT>.now
select now
```

### 4.5 局部环境修改（Local）
```csharp
public interface Local<M, InnerEnv> : Has<M, InnerEnv>
{
    public static abstract K<M, A> With<A>(Func<InnerEnv, InnerEnv> f, K<M, A> ma);
}
```

允许在子计算中临时修改环境，而不影响外部。

## 5. 缺点与权衡

### 5.1 类型签名复杂
```csharp
// 约束列表可能变得很长
where RT : Has<Eff<RT>, ConsoleIO>,
           Has<Eff<RT>, FileIO>,
           Has<Eff<RT>, TimeIO>,
           Has<Eff<RT>, EnvironmentIO>,
           Has<Eff<RT>, DirectoryIO>,
           Has<Eff<RT>, EncodingIO>
```

### 5.2 学习曲线陡峭
- 需要理解高阶类型 (Higher-Kinded Types)
- 需要理解 Monad、Functor、Applicative
- 需要理解 Effect 系统

### 5.3 样板代码
每个新项目需要：
1. 定义 Runtime 类型
2. 实现所有需要的 Has 接口
3. 可能需要自定义 IO traits

```csharp
// 每个项目都要写类似的代码
public record Runtime(RuntimeEnv Env) : 
    Has<Eff<Runtime>, ConsoleIO>,
    Has<Eff<Runtime>, FileIO>,
    // ... 重复的模式
{
    static K<Eff<Runtime>, ConsoleIO> Has<Eff<Runtime>, ConsoleIO>.Ask { get; } =
        pure(Implementations.ConsoleIO.Default);
    // ... 重复的实现
}
```

### 5.4 运行时开销
- 间接层增加
- 闭包分配
- 可能的性能影响（虽然通常不显著）

### 5.5 IDE 支持有限
- 复杂泛型约束降低代码补全质量
- 错误信息可能难以理解

### 5.6 与现有代码集成困难
- 需要包装所有外部依赖为 IO trait
- 调用第三方库需要 `lift` 包装

## 6. 与其他方案的对比

### 6.1 vs 传统 DI (如 Microsoft.Extensions.DI)

| 方面 | Runtime | 传统 DI |
|------|---------|---------|
| 依赖声明 | 编译时（类型约束） | 运行时（容器） |
| 副作用处理 | 显式（IO monad） | 隐式 |
| 测试 | 纯函数式 | 需要 mock |
| 学习曲线 | 陡峭 | 平缓 |
| IDE 支持 | 一般 | 良好 |

### 6.2 vs ZIO (Scala)

| 方面 | language-ext | ZIO |
|------|--------------|-----|
| 错误类型 | 固定为 Error | 泛型 E |
| 语法 | C# LINQ | Scala for-comprehension |
| 生态系统 | 较小 | 丰富 |
| 类型推导 | 有限 | 更强 |

## 7. 最佳实践

### 7.1 定义领域特定的 IO Traits
```csharp
public interface OrderRepositoryIO
{
    IO<Option<Order>> GetById(OrderId id);
    IO<Unit> Save(Order order);
}
```

### 7.2 使用组合 Runtime
```csharp
// 基础系统 Runtime
public record BaseRuntime : Has<Eff<BaseRuntime>, ConsoleIO>, ...

// 应用特定 Runtime 组合基础 + 领域
public record AppRuntime : 
    Has<Eff<AppRuntime>, ConsoleIO>,
    Has<Eff<AppRuntime>, OrderRepositoryIO>,
    ...
```

### 7.3 保持 IO Traits 细粒度
```csharp
// 好：细粒度，易于组合和测试
public interface TimeIO { IO<DateTime> Now { get; } }
public interface ClockIO { IO<Unit> Delay(TimeSpan duration); }

// 不好：粗粒度，难以部分 mock
public interface SystemIO { /* 包含所有系统操作 */ }
```

### 7.4 利用 Local 进行作用域隔离
```csharp
// 在子操作中使用不同的配置
from result in Local<Eff<RT>, Config>.With(
    cfg => cfg with { Timeout = TimeSpan.FromSeconds(30) },
    longRunningOperation)
select result
```

## 8. 总结

Runtime 设计是 language-ext 实现 **依赖注入 + 副作用管理** 的核心机制，它：

1. **借鉴 ZIO** 的 Environment 模式
2. **使用 Higher-Kinded Types** 实现类型安全的依赖访问
3. **通过 Has trait** 声明能力依赖
4. **分离接口与实现** 支持 Live/Test 切换
5. **编译时检查** 保证依赖完整性

适用于：
- 需要高度可测试性的项目
- 追求纯函数式风格的团队
- 复杂业务逻辑需要清晰依赖管理

不适用于：
- 团队 FP 经验有限
- 简单 CRUD 应用
- 对性能极度敏感的场景
