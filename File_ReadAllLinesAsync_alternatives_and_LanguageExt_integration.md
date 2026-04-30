# File.ReadAllLinesAsync 的替代方案 与 LanguageExt 集成（AsIterable、IO、Pipes）及内存安全性

---

## 一、`File.ReadAllLinesAsync` 的替代方案

### 问题背景

`File.ReadAllLinesAsync` 虽然名字带 `Async`，但它的实际行为是：**一次性将文件所有行读入内存**，返回 `Task<string[]>`。对于大文件，这会造成显著的内存压力。

---

### 替代方案对比

| 方案 | 返回类型 | 是否流式 | 是否异步 | 适用场景 |
|------|----------|----------|----------|----------|
| `File.ReadAllLinesAsync` | `Task<string[]>` | ❌ 全量加载 | ✅ | 小文件 |
| `File.ReadLines` | `IEnumerable<string>` | ✅ 逐行 | ❌ 同步 | 中等文件，同步上下文 |
| `File.ReadLinesAsync` (.NET 9+) | `IAsyncEnumerable<string>` | ✅ 逐行 | ✅ | 大文件，异步上下文 |
| `StreamReader` 手动循环 | 自定义 | ✅ 逐行 | ✅ | 精细控制 |
| `PipeReader` (System.IO.Pipelines) | 自定义 | ✅ 块级 | ✅ | 超高性能场景 |

---

### 1. `File.ReadLines`（同步流式）

```csharp
foreach (var line in File.ReadLines("data.txt"))
{
    Console.WriteLine(line);
}
```

- **优点**：逐行读取，内存占用低（只保留当前行）
- **缺点**：同步阻塞，不适合 async/await 上下文

---

### 2. `File.ReadLinesAsync`（.NET 9+ 异步流式）✅ 推荐

```csharp
await foreach (var line in File.ReadLinesAsync("data.txt"))
{
    Console.WriteLine(line);
}
```

- **优点**：真正的异步流式读取，内存友好，支持 `await foreach`
- **缺点**：需要 .NET 9+

---

### 3. `StreamReader` 手动异步循环

```csharp
using var reader = new StreamReader("data.txt");
string? line;
while ((line = await reader.ReadLineAsync()) != null)
{
    Console.WriteLine(line);
}
```

- **优点**：兼容 .NET 旧版本，可自定义编码、缓冲区
- **缺点**：代码较繁琐

---

### 4. 封装成 `IAsyncEnumerable<string>`（兼容旧版 .NET）

```csharp
public static async IAsyncEnumerable<string> ReadLinesAsync(
    string path,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    using var reader = new StreamReader(path);
    string? line;
    while ((line = await reader.ReadLineAsync(ct)) != null)
    {
        ct.ThrowIfCancellationRequested();
        yield return line;
    }
}
```

使用：

```csharp
await foreach (var line in ReadLinesAsync("data.txt", cancellationToken))
{
    // 处理每一行
}
```

---

### 5. `PipeReader`（System.IO.Pipelines，超高性能）

```csharp
using var fs = File.OpenRead("data.txt");
var reader = PipeReader.Create(fs);

while (true)
{
    var result = await reader.ReadAsync();
    var buffer = result.Buffer;
    // 解析 buffer 中的行...
    reader.AdvanceTo(buffer.End);
    if (result.IsCompleted) break;
}
```

- **适用场景**：需要处理海量数据、网络流、零拷贝解析等极致性能场景

---

### 总结建议

- **.NET 9+**：直接用 `File.ReadLinesAsync`，简洁且高效
- **.NET 6/7/8**：封装 `StreamReader` 为 `IAsyncEnumerable<string>`
- **超大文件 / 高吞吐**：考虑 `PipeReader`
- **避免**：对大文件使用 `File.ReadAllLinesAsync`，它会一次性加载全部内容

---

## 二、将 `ReadLinesAsync` 与 LanguageExt 集成（AsIterable、IO、Pipes）及内存安全性

### 目标

将异步流式文件读取（`IAsyncEnumerable<string>`）融入 LanguageExt 的函数式体系，同时保持：

- ✅ 内存安全（流式，不全量加载）
- ✅ 纯函数式副作用管理（`IO` Monad）
- ✅ 可组合的管道（Pipes）
- ✅ 错误处理（`Either` / `fin`）

---

### 1. 使用 `AsIterable` 将 `IAsyncEnumerable` 接入 LanguageExt

LanguageExt 提供了 `AsIterable()` 扩展，可以将 `IAsyncEnumerable<T>` 转为 `IAsyncEnumerable<T>` 的 LanguageExt 包装（`AsyncEnumerable<T>`），从而使用 LINQ 风格的函数式操作：

```csharp
using LanguageExt;
using static LanguageExt.Prelude;

var lines = File.ReadLinesAsync("data.txt")
               .AsIterable()            // 转为 LanguageExt 的 AsyncIterable
               .Filter(l => !string.IsNullOrWhiteSpace(l))
               .Map(l => l.Trim());

await foreach (var line in lines)
{
    Console.WriteLine(line);
}
```

**内存安全性**：`AsIterable()` 不会缓冲全部数据，仍然是流式处理，每次只处理一行。

---

### 2. 用 `IO` Monad 封装文件读取副作用

```csharp
using LanguageExt.Effects.Traits;
using LanguageExt.Sys.Traits;

// 将文件读取包装进 IO<RT, Error, IAsyncEnumerable<string>>
IO<IAsyncEnumerable<string>> ReadFileLines(string path) =>
    IO.lift(() => File.ReadLinesAsync(path));

// 组合使用
var program =
    from lines in ReadFileLines("data.txt")
    from _     in IO.lift(async () =>
    {
        await foreach (var line in lines)
        {
            Console.WriteLine(line);
        }
    })
    select unit;

await program.RunAsync();
```

**优点**：
- 副作用被延迟、显式地封装在 `IO` 中
- 便于测试（可替换为内存中的假实现）
- 错误通过 `IO` 的错误通道传播，无需 try/catch

---

### 3. 使用 LanguageExt Pipes 构建流式处理管道

LanguageExt 的 `Pipes` 模块（类似 Haskell 的 `pipes`/`conduit`）非常适合流式数据处理：

```csharp
using LanguageExt.Pipes;

// Producer：从文件中逐行产出
Producer<string, IO.RT, Unit> FileProducer(string path) =>
    Produce.from(File.ReadLinesAsync(path));

// Pipe：过滤空行 + Trim
Pipe<string, string, IO.RT, Unit> CleanPipe() =>
    Pipe.filter<string>(l => !string.IsNullOrWhiteSpace(l))
    | Pipe.map<string, string>(l => l.Trim());

// Consumer：打印到控制台
Consumer<string, IO.RT, Unit> PrintConsumer() =>
    Consumer.forEach<string>(line => Console.WriteLine(line));

// 组合管道
var pipeline = FileProducer("data.txt") | CleanPipe() | PrintConsumer();

await pipeline.RunAsync();
```

**内存安全性分析**：
- `Producer` 基于 `IAsyncEnumerable`，**逐行推送**，不缓存
- `Pipe` 中的每个操作符均为流式变换，**不积累全部数据**
- `Consumer` 逐条消费，**背压（backpressure）自然成立**

---

### 4. 错误处理：结合 `fin` / `Either`

```csharp
IO<Seq<string>> SafeReadLines(string path) =>
    IO.lift(async () =>
    {
        var builder = Seq<string>.Empty;
        await foreach (var line in File.ReadLinesAsync(path))
        {
            builder = builder.Add(line);
        }
        return builder;
    });

// 如果希望保持流式（不收集），使用 IO<IAsyncEnumerable<string>>
IO<IAsyncEnumerable<string>> SafeReadLinesStreaming(string path) =>
    IO.lift(() =>
        File.Exists(path)
            ? File.ReadLinesAsync(path)
            : throw new FileNotFoundException(path));
```

> ⚠️ 注意：如果使用 `Seq<string>` 收集所有行，则会失去流式优势。推荐保持 `IAsyncEnumerable<string>` 传递。

---

### 5. 完整示例：IO + Pipes + 错误处理

```csharp
var program =
    from lines in IO.lift(() => File.ReadLinesAsync("data.txt"))
    from result in IO.lift(async () =>
    {
        var count = 0;
        await foreach (var line in lines
                           .AsIterable()
                           .Filter(l => l.StartsWith("ERROR")))
        {
            Console.WriteLine($"[错误日志] {line}");
            count++;
        }
        return count;
    })
    select result;

var errorCount = await program.RunAsync();
Console.WriteLine($"共发现 {errorCount} 条错误日志");
```

---

### 内存安全性总结

| 操作 | 是否流式 | 内存安全 | 说明 |
|------|----------|----------|------|
| `File.ReadLinesAsync` | ✅ | ✅ | 基础异步流 |
| `.AsIterable()` | ✅ | ✅ | 不缓冲，流式变换 |
| `IO.lift(...)` 包装 | ✅ | ✅ | 延迟执行，不提前拉取 |
| `Pipes` 管道 | ✅ | ✅ | 背压支持，逐项处理 |
| 收集为 `Seq<T>` / `List<T>` | ❌ | ⚠️ | 全量加载，慎用于大文件 |

---

### 最终建议

> 在 LanguageExt 体系中处理大文件时：
> 1. **数据源**：用 `File.ReadLinesAsync`（.NET 9+）或封装的 `IAsyncEnumerable<string>`
> 2. **副作用管理**：用 `IO` Monad 包裹，保持纯函数式
> 3. **流式变换**：用 `.AsIterable()` + `Filter/Map` 或 `Pipes` 管道
> 4. **避免**：在管道中途使用 `.ToList()` / `ToSeq()` 等收集操作，除非数据量可控

---

## 三、源码证明：`AsIterable()` 为何是惰性且内存安全的

上文多次声称 `AsIterable()` 是流式的、不会将文件内容加载进内存。本节直接引用 LanguageExt 库的两处关键源码，作为严格的实现层证明。

---

### 证据一：`Iterable.Extensions.cs` — `AsIterable()` 只是一个轻量包装

> 📄 文件路径：`LanguageExt.Core/Immutable Collections/Iterable/Extensions/Iterable.Extensions.cs`

```csharp
public static Iterable<A> AsIterable<A>(this IAsyncEnumerable<A> xs) =>
    new IterableAsyncEnumerable<A>(IO.pure(xs));
```

**逐行解读：**

| 代码片段 | 含义 |
|----------|------|
| `this IAsyncEnumerable<A> xs` | 接收原始的异步流（如 `File.ReadLinesAsync` 的返回值） |
| `IO.pure(xs)` | 将流包裹进一个**纯 IO 值**，此刻**不执行任何 IO 操作**，不触碰文件 |
| `new IterableAsyncEnumerable<A>(...)` | 仅构造一个持有 `IO<IAsyncEnumerable<A>>` 引用的包装对象 |

> ✅ **结论**：`AsIterable()` 调用的瞬间，**零字节被读取**。它只是把流的引用装进了一个 `IO` 容器，文件系统完全未被触碰。没有缓冲、没有预取、没有分配行数组。

---

### 证据二：`Iterable.AsyncEnumerable.cs` — 消费时才用 `await foreach` 逐项拉取

> 📄 文件路径：`LanguageExt.Core/Immutable Collections/Iterable/DSL/Iterable.AsyncEnumerable.cs`

#### 2a. 类声明与构造函数

```csharp
sealed class IterableAsyncEnumerable<A>(IO<IAsyncEnumerable<A>> runEnumerable) : Iterable<A>
{
    internal override bool IsAsync => true;
    // ...
}
```

`runEnumerable` 是一个 `IO<IAsyncEnumerable<A>>`——即一个**描述"如何获取流"的惰性动作**，而非流本身的内容。整个类在构造时不拉取任何数据。

---

#### 2b. `Map` 与 `Filter`：变换本身也是惰性的

```csharp
public override Iterable<B> Map<B>(Func<A, B> f) =>
    new IterableAsyncEnumerable<B>(AsAsyncEnumerableIO().Map(xs => xs.Select(f)));

public override Iterable<A> Filter(Func<A, bool> f) =>
    new IterableAsyncEnumerable<A>(AsAsyncEnumerableIO().Map(xs => xs.Where(f)));
```

`Map` 和 `Filter` 都只是在 `IO` 层面**叠加描述**（`IO.Map`），返回的仍是一个新的 `IterableAsyncEnumerable` 包装对象。整条变换链在被"运行"之前，**一个元素都不会被处理**。

---

#### 2c. `FoldWhileIO`：终结操作，真正消费流的地方

```csharp
public override IO<S> FoldWhileIO<S>(
    Func<S, A, S> f,
    Func<(S State, A Value), bool> predicate,
    S initialState) =>
    IO.liftVAsync(async env =>
    {
        var s  = initialState;
        var xs = await runEnumerable.RunAsync(env);   // ① 此时才真正"打开"流
        await foreach (var x in xs.WithCancellation(env.Token))  // ② 逐项拉取，每次一个元素
        {
            if (!predicate((s, x)))
            {
                return s;   // ③ 满足终止条件时立即停止，后续数据根本不被读取
            }
            s = f(s, x);
        }
        return s;
    });
```

**三个关键时刻的注解：**

| 标注 | 发生了什么 | 内存含义 |
|------|-----------|----------|
| ① `await runEnumerable.RunAsync(env)` | `IO` 动作被执行，获得 `IAsyncEnumerable<A>` 的引用 | 仍然没有读取文件内容，只是拿到了迭代器句柄 |
| ② `await foreach (var x in xs...)` | 每次循环只从流中拉取**一个元素** | 内存中同时只存在**一行**数据 |
| ③ `return s`（提前退出） | 谓词为假时立刻返回，迭代器被丢弃 | 文件剩余部分**永远不会被读取** |

同样的模式出现在全部四个 `FoldWhileIO` / `FoldUntilIO` 重载中：

```csharp
// FoldUntilIO 重载同样使用 await foreach 逐项消费
public override IO<S> FoldUntilIO<S>(Func<S, A, S> f, Func<(S State, A Value), bool> predicate, S initialState) =>
    IO.liftVAsync(async env =>
    {
        var s  = initialState;
        var xs = await runEnumerable.RunAsync(env);
        await foreach (var x in xs.WithCancellation(env.Token))
        {
            s = f(s, x);
            if (predicate((s, x)))
            {
                return s;   // 同样支持提前终止
            }
        }
        return s;
    });
```

---

#### 2d. `AsAsyncEnumerableIO`：透传流，不缓冲

```csharp
public override IO<IAsyncEnumerable<A>> AsAsyncEnumerableIO()
{
    return IO.lift(env => go(env, runEnumerable));

    static async IAsyncEnumerable<A> go(EnvIO env, IO<IAsyncEnumerable<A>> run)
    {
        var xs = await run.RunAsync(env);
        await foreach (var x in xs.WithCancellation(env.Token))
        {
            yield return x;   // 逐项 yield，绝不积累
        }
    }
}
```

这里使用了 C# 的 `async IAsyncEnumerable` + `yield return`，这是**编译器级别的流式保证**：每个元素在被下游请求之前不会被产出，也不会被缓存。

---

### 完整调用链路图示

```
File.ReadLinesAsync("data.txt")          ← IAsyncEnumerable<string>（流，未读取）
    │
    ▼
.AsIterable()                            ← 仅构造 IterableAsyncEnumerable<string>
    │                                       包装 IO.pure(xs)，零 IO 操作
    ▼
.Filter(l => !string.IsNullOrEmpty(l))   ← 返回新的 IterableAsyncEnumerable<string>
    │                                       内部叠加 xs.Where(f)，仍未执行
    ▼
.Map(l => l.Trim())                      ← 再次返回新的包装，叠加 xs.Select(f)
    │                                       仍未执行任何 IO
    ▼
.FoldWhileIO(...)  或  await foreach     ← ✅ 此处才是"运行边界"
    │                                       IO 动作被触发，await foreach 开始逐行拉取
    ▼
每次循环：从磁盘读取一行 → 变换 → 消费    ← 内存中始终只有一行
```

---

### 源码证明总结

| 证明点 | 源码位置 | 结论 |
|--------|---------|------|
| `AsIterable()` 不读取数据 | `Iterable.Extensions.cs` 第 19–20 行 | 仅调用 `IO.pure(xs)`，纯包装，零 IO |
| `Map` / `Filter` 不触发执行 | `Iterable.AsyncEnumerable.cs` 第 51–55 行 | 在 `IO` 层叠加描述，返回新包装 |
| 消费时逐项 `await foreach` | `Iterable.AsyncEnumerable.cs` 第 57–92 行 | 每次只处理一个元素，内存恒定 |
| 支持提前终止 | `FoldWhileIO` / `FoldUntilIO` 中的 `return s` | 未读数据永远不被加载 |
| 透传不缓冲 | `AsAsyncEnumerableIO` 中的 `yield return` | 编译器保证逐项产出 |
