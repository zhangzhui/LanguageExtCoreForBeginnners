# Runtime 组合而非继承

## 问题：Runtime 能否被继承扩展？

**答案：不能通过继承扩展，需要创建新的 Runtime。**

## 技术原因

### 1. Has 接口绑定具体类型

```csharp
public record Runtime(RuntimeEnv Env) : 
    Has<Eff<Runtime>, ConsoleIO>,  // 注意：Eff<Runtime> 是具体类型
    ...
```

如果尝试继承：
```csharp
public record MyRuntime : Runtime  // 继承
{
    // 想添加新能力
}
```

问题：`Has<Eff<Runtime>, ConsoleIO>` 仍然绑定到 `Eff<Runtime>`，而不是 `Eff<MyRuntime>`。
业务代码中 `where RT : Has<Eff<RT>, ConsoleIO>` 约束无法用 `MyRuntime` 满足。

### 2. 静态接口成员不能被继承重写

```csharp
// 这是静态成员，按类型绑定
static K<Eff<Runtime>, ConsoleIO> Has<Eff<Runtime>, ConsoleIO>.Ask { get; } = ...
```

静态接口成员是 C# 11 的特性，它们：
- 属于类型本身，不属于实例
- 不参与继承链
- 每个类型必须独立实现

### 3. 类型参数不会自动替换

即使 record 支持继承，`Has<Eff<Runtime>, X>` 中的 `Runtime` 不会自动变成子类型。

## 正确做法：组合

### 创建新 Runtime，复用现有实现

```csharp
public record MyRuntime(MyRuntimeEnv Env) : 
    // 复用系统能力（重新声明接口）
    Has<Eff<MyRuntime>, ConsoleIO>,
    Has<Eff<MyRuntime>, FileIO>,
    Has<Eff<MyRuntime>, TimeIO>,
    // 添加自己的能力
    Has<Eff<MyRuntime>, MyDatabaseIO>,
    Has<Eff<MyRuntime>, MyEmailIO>
{
    // ========== 复用现有实现 ==========
    static K<Eff<MyRuntime>, ConsoleIO> Has<Eff<MyRuntime>, ConsoleIO>.Ask { get; } =
        pure(LanguageExt.Sys.Live.Implementations.ConsoleIO.Default);
    
    static K<Eff<MyRuntime>, FileIO> Has<Eff<MyRuntime>, FileIO>.Ask { get; } =
        pure(LanguageExt.Sys.Live.Implementations.FileIO.Default);
    
    static K<Eff<MyRuntime>, TimeIO> Has<Eff<MyRuntime>, TimeIO>.Ask { get; } =
        pure(LanguageExt.Sys.Live.Implementations.TimeIO.Default);
    
    // ========== 自己的新能力 ==========
    static K<Eff<MyRuntime>, MyDatabaseIO> Has<Eff<MyRuntime>, MyDatabaseIO>.Ask { get; } =
        liftEff<MyRuntime, MyDatabaseIO>(rt => rt.Env.Database);
    
    static K<Eff<MyRuntime>, MyEmailIO> Has<Eff<MyRuntime>, MyEmailIO>.Ask { get; } =
        liftEff<MyRuntime, MyEmailIO>(rt => rt.Env.Email);
}

public record MyRuntimeEnv(
    MyDatabaseIO Database,
    MyEmailIO Email
);
```

### 实际例子（Newsletter Sample）

```csharp
// Samples/Newsletter/Newsletter/Effects/Runtime.cs
public record Runtime(RuntimeEnv Env) :
    Has<Eff<Runtime>, WebIO>,       // 自定义能力
    Has<Eff<Runtime>, FileIO>,      // 复用系统能力
    Has<Eff<Runtime>, JsonIO>,      // 自定义能力
    Has<Eff<Runtime>, ConsoleIO>,   // 复用系统能力
    Has<Eff<Runtime>, EncodingIO>,
    Has<Eff<Runtime>, DirectoryIO>,
    Has<Eff<Runtime>, Config>,
    Has<Eff<Runtime>, HttpClient>,
    IDisposable
{
    // 复用 LanguageExt.Sys 的实现
    static K<Eff<Runtime>, FileIO> Has<Eff<Runtime>, FileIO>.Ask =
        pure<Eff<Runtime>, FileIO>(LanguageExt.Sys.Live.Implementations.FileIO.Default);
    
    static K<Eff<Runtime>, ConsoleIO> Has<Eff<Runtime>, ConsoleIO>.Ask =
        pure<Eff<Runtime>, ConsoleIO>(LanguageExt.Sys.Live.Implementations.ConsoleIO.Default);
    
    static K<Eff<Runtime>, EncodingIO> Has<Eff<Runtime>, EncodingIO>.Ask => 
        pure<Eff<Runtime>, EncodingIO>(LanguageExt.Sys.Live.Implementations.EncodingIO.Default);

    static K<Eff<Runtime>, DirectoryIO> Has<Eff<Runtime>, DirectoryIO>.Ask =
        pure<Eff<Runtime>, DirectoryIO>(LanguageExt.Sys.Live.Implementations.DirectoryIO.Default);
    
    // 自己的实现
    static K<Eff<Runtime>, JsonIO> Has<Eff<Runtime>, JsonIO>.Ask => 
        pure<Eff<Runtime>, JsonIO>(Impl.Json.Default); 
    
    static K<Eff<Runtime>, WebIO> Has<Eff<Runtime>, WebIO>.Ask =
        pure<Eff<Runtime>, WebIO>(Impl.Web.Default);

    // 从环境读取
    static K<Eff<Runtime>, Config> Has<Eff<Runtime>, Config>.Ask { get; } = 
        liftEff<Runtime, Config>(rt => rt.Env.Config);

    static K<Eff<Runtime>, HttpClient> Has<Eff<Runtime>, HttpClient>.Ask { get; } = 
        liftEff<Runtime, HttpClient>(rt => rt.Env.HttpClient);
}
```

## OOP 继承 vs FP 组合对比

| 方面 | OOP 继承 | FP 组合 |
|------|----------|---------|
| 语法 | `class B : A` | `record B` 实现相同接口 |
| 多态 | 运行时多态 | 编译时类型约束 |
| 复用 | 隐式获得父类行为 | 显式选择复用哪些实现 |
| 耦合 | 紧耦合，脆弱基类问题 | 松耦合，无继承层次 |
| 可见性 | 子类可能不需要父类所有成员 | 只声明需要的能力 |
| 灵活性 | 单继承限制 | 可任意组合能力 |

## 为什么这是有意为之的设计？

1. **避免继承耦合** - 不会因为父类修改而影响子类
2. **显式声明** - 每个 Runtime 清晰声明自己的所有能力
3. **类型精确** - 类型系统可以精确检查依赖关系
4. **灵活组合** - 可以任意组合需要的能力，不受继承层次限制

## 模式总结

```
┌─────────────────────────────────────────────────────────────┐
│  组合模式步骤：                                               │
│                                                             │
│  1. 创建新的 Runtime record                                  │
│  2. 声明所有需要的 Has<Eff<MyRuntime>, X> 接口               │
│  3. 对于系统能力：复用 LanguageExt.Sys 的实现                 │
│     pure(LanguageExt.Sys.Live.Implementations.XXX.Default)  │
│  4. 对于自定义能力：提供自己的实现                             │
│     liftEff<MyRuntime, X>(rt => rt.Env.XXX)                 │
│  5. 定义 RuntimeEnv 存放运行时状态和配置                      │
└─────────────────────────────────────────────────────────────┘
```

## 样板代码问题

这种设计的代价是需要写更多样板代码。可能的缓解方式：
- 使用 Source Generator 自动生成
- 创建项目模板
- 定义辅助基类型（但不用于继承，只用于代码共享）
