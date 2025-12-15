# Eff 中的 Runtime (RT) 到底是什么

## Runtime 在这里的含义

它**不是**指 .NET Runtime 或 JVM 那种"运行时环境"。

在 Eff 的语境中，**Runtime 就是"依赖的容器"**，更准确的名字可能是：
- `Environment`
- `Context`
- `Dependencies`
- `Capabilities`

---

## 为什么叫 Runtime？

这个命名来自 **Haskell 的 IO Monad** 和 **ZIO (Scala)**。

在这些语言中，思路是：
> 纯函数式代码本身不能执行副作用，需要"某个东西"来实际执行它们。这个"东西"就叫 Runtime。

就像：
- 你写了一个**脚本**（Eff 描述）
- 需要一个**解释器**（Runtime）来执行它

---

## 实际上 RT 就是依赖注入

```csharp
// RT 定义了"这个程序需要什么能力"
interface MyRT : 
    Has<ConsoleIO>,    // 需要控制台能力
    Has<FileIO>,       // 需要文件系统能力
    Has<HttpClient>    // 需要网络能力
{ }
```

这和传统 DI 是一回事：

```csharp
// 传统 DI
class MyService(IConsole console, IFileSystem fs, IHttpClient http) { }

// Eff 风格
Eff<MyRT, A>  // MyRT 声明了需要这些依赖
```

---

## 为什么叫"提供执行副作用的能力"？

因为 RT 里的东西**就是执行副作用的实现**：

```csharp
// 生产环境的 Runtime
class ProdRT : MyRT {
    ConsoleIO Console => RealConsoleIO.Default;  // 真的读写控制台
    FileIO File => RealFileIO.Default;           // 真的读写文件
}

// 测试环境的 Runtime  
class TestRT : MyRT {
    ConsoleIO Console => FakeConsoleIO(...);     // 假的，记录调用
    FileIO File => InMemoryFileIO(...);          // 内存模拟
}
```

**Runtime 决定了副作用"如何"被执行。**

---

## 更直白的理解

把 `Eff<RT, A>` 理解成：

```
Eff<需要什么依赖, 返回什么结果>
```

或者：

```
RT => A  （给我依赖，我给你结果）
```

---

## 总结

| 术语 | 实际含义 |
|------|---------|
| Runtime (RT) | 依赖容器 / 能力提供者 |
| `Has<X>` | 声明"我需要 X 这个依赖" |
| `Run(runtime)` | 注入依赖并执行 |

**"Runtime" 只是个名字**，换成 `Env`、`Deps`、`Ctx` 理解起来可能更容易。language-ext 沿用了 ZIO 的术语而已。
