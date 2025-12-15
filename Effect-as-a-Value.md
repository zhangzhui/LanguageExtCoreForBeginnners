# Eff：副作用作为值 (Effect as a Value)

## Eff 是什么

**Eff** 是 language-ext 中用于管理副作用的核心类型，代表`可能产生副作用的计算`。

```csharp
Eff<RT, A>
```

- `RT`：运行时环境类型（Runtime），提供执行副作用所需的能力
- `A`：计算成功后返回的值类型

**关键点**：Eff 是 **惰性的**，构建时不执行，只有调用 `Run()` 时才真正执行。

---

## 核心理解：Eff 并没有"消除"副作用

```csharp
// 方式1：直接执行
var content = File.ReadAllText("config.json");

// 方式2：包装在 Eff 里
Eff<string> readConfig = Eff(() => File.ReadAllText("config.json"));
readConfig.Run();  // 最终还是要执行，副作用依然发生
```

**最终执行时，副作用一样会发生。**

更准确的说法：

> **Eff 把副作用从"立即发生"变成"描述 + 稍后执行"**

这种模式叫 **"副作用作为值"(Effect as a Value)**：
- 副作用被表示为一个**数据结构**
- 可以传递、组合、变换这个数据结构
- 最后在程序边界统一执行

---

## Eff 的价值：控制权与可见性

### 1. 延迟到你选择的时机执行
```csharp
Eff<string> eff = Eff(() => File.ReadAllText("x.txt"));
// 这里可以组合、变换、传递，但什么都没发生

// 你决定何时、何地、如何执行
eff.Run();
```

### 2. 类型签名暴露意图
```csharp
// 看签名就知道有副作用
Eff<RT, string> LoadConfig();

// 纯函数
string ParseConfig(string json);
```

### 3. 可以在"执行前"做很多事
```csharp
Eff<string> final = eff
    .Retry(3)             // 失败重试
    .Timeout(5.Seconds()) // 超时控制
    .Map(s => s.Trim())   // 变换结果
    .Catch(e => "default"); // 错误恢复
```

如果是 `File.ReadAllText()` 直接调用，这些都要手写 try-catch 和循环。

### 4. 测试时可以不执行真实副作用
```csharp
// 生产环境
program.Run(RealRuntime);

// 测试环境
program.Run(MockRuntime);  // MockRuntime 不真的读文件
```

---

## 主要能力

### 错误处理内置
```csharp
Eff<int> computation = ...;

// 自动捕获异常，返回 Fin<A>（类似 Either<Error, A>）
Fin<int> result = computation.Run();

result.Match(
    Succ: value => Console.WriteLine($"成功: {value}"),
    Fail: error => Console.WriteLine($"失败: {error}")
);
```

### 组合性强
```csharp
Eff<string> program =
    from name   in readName()
    from age    in readAge()
    from _      in validate(name, age)
    from result in save(name, age)
    select result;
```

### 重试、超时等控制
```csharp
eff.Retry(Schedule.spaced(1.Seconds()) | Schedule.recurs(3))  // 重试3次，间隔1秒
   .Timeout(10.Seconds())  // 超时控制
```

### 依赖注入通过 RT
```csharp
// 定义运行时需要的能力
interface RT : Has<ConsoleIO>, Has<FileIO> { }

Eff<RT, Unit> program = 
    from line in Console<RT>.readLine
    from _    in File<RT>.writeAllText("log.txt", line)
    select unit;
```

测试时可以传入 mock 的 runtime。

---

## Eff vs IO vs Aff

| 类型 | 用途 |
|------|------|
| `Eff<A>` | 同步副作用，无需 runtime |
| `Eff<RT, A>` | 同步副作用，需要 runtime 提供能力 |
| `Aff<A>` | **异步**副作用（返回 `ValueTask`） |
| `Aff<RT, A>` | 异步副作用 + runtime |
| `IO<A>` | 最简单的副作用封装 |

---

## 类比：IQueryable

就像 LINQ to SQL：
```csharp
var query = db.Users.Where(u => u.Age > 18);  // 只是构建查询，没执行
var result = query.ToList();  // 现在才执行
```

`IQueryable` 也没有"消除"数据库查询，只是让你能在执行前组合和优化它。

**Eff 对副作用做了同样的事。**

---

## 总结

Eff 让你：
1. **显式标记**哪些代码有副作用
2. **延迟执行**，可以组合、变换、重试
3. **统一错误处理**
4. **依赖注入**通过类型系统保证
5. **可测试**：传入不同 runtime 即可
