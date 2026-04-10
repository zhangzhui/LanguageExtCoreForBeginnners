# `fun` 函数详解：C# `language-ext` 库中的 Lambda 利器

> 适用版本：`language-ext` v4 / v5
> 命名空间：`LanguageExt`（通过 `using static LanguageExt.Prelude;` 引入）

---

## 目录

1. [什么是 `fun`？](#什么是-fun)
2. [为何需要 `fun`？——C# 类型推断的局限](#为何需要-fun——c-类型推断的局限)
3. [用途一：简化 `Func` 类型推断](#用途一简化-func-类型推断)
   - [基本原理](#基本原理)
   - [代码示例集](#代码示例集——func-类型推断)
4. [用途二：将 `Action` 转换为 `Func<…, Unit>`](#用途二将-action-转换为-func-unit)
   - [为什么要把 `Action` 变成 `Func`？](#为什么要把-action-变成-func)
   - [代码示例集](#代码示例集——action-转-func-unit)
5. [`fun` 的所有重载签名一览](#fun-的所有重载签名一览)
6. [综合实战示例](#综合实战示例)
7. [常见误区与注意事项](#常见误区与注意事项)
8. [总结](#总结)

---

## 什么是 `fun`？

`fun` 是 `language-ext` 库的 `Prelude` 模块中提供的一个静态辅助函数。
它本质上是一个**恒等包装器（identity wrapper）**——它接收一个 lambda 表达式，原封不动地返回同一个 delegate，
但在这个过程中，C# 编译器会**利用目标类型推断**将 lambda 解析为具体的 `Func<...>` 类型。

```csharp
// 引入 language-ext Prelude，使 fun 可以直接调用
using static LanguageExt.Prelude;
```

`fun` 有两大类重载：

| 类别 | 作用 |
|------|------|
| `fun(Func<T1, …, R> f)` | 将 lambda 推断并固化为 `Func` 委托 |
| `fun(Action<T1, …> f)` | 将 `Action`（void 返回）包装为返回 `Unit` 的 `Func` |

---

## 为何需要 `fun`？——C# 类型推断的局限

C# 的 lambda 表达式本身**没有类型**，它只能根据赋值目标推断类型。
这在很多场景下会导致冗长的显式类型声明，甚至引发编译错误。

### 问题场景 1：`var` 无法推断 lambda 类型

```csharp
// ❌ 编译错误：无法将 lambda 表达式分配给隐式类型的局部变量
var add = (int x, int y) => x + y;
```

在 C# 10 之前，上面的代码直接报错。即使在 C# 10+ 中可以推断，
`language-ext` 的函数式管道（如 `curry`、`memo`、`pipe`）仍然**需要明确的 `Func<>` 类型**，
否则后续的链式调用无法进行。

### 问题场景 2：方法重载歧义

```csharp
// ❌ 编译错误：重载解析歧义，不知道应该用 Action<int> 还是 Func<int, int>
SomeMethod(x => x * 2);
```

### 解决方案：`fun`

```csharp
// ✅ fun 明确告知编译器：这是一个 Func<int, int, int>
var add = fun((int x, int y) => x + y);
```

---

## 用途一：简化 `Func` 类型推断

### 基本原理

`fun` 利用 C# 的**目标类型推断**机制：当 lambda 作为参数传入 `fun(Func<T1, T2, R> f)` 时，
编译器会把 lambda 的参数类型与返回类型对应到泛型参数，从而完成类型绑定。
最终你得到的是一个类型明确的 `Func<T1, T2, R>` 对象，可以安全地传递、组合、存储。

### 代码示例集——Func 类型推断

---

#### 示例 1：无参函数——获取当前时间戳

```csharp
using static LanguageExt.Prelude;

// ❌ 不使用 fun：必须手动写出完整类型
Func<DateTime> getNow_verbose = () => DateTime.UtcNow;

// ✅ 使用 fun：let 编译器自动推断 Func<DateTime>
var getNow = fun(() => DateTime.UtcNow);

// 使用
Console.WriteLine(getNow()); // 输出：2024-01-15 08:30:00
```

---

#### 示例 2：单参数函数——字符串转大写

```csharp
using static LanguageExt.Prelude;

// ❌ 不使用 fun
Func<string, string> toUpper_verbose = s => s.ToUpper();

// ✅ 使用 fun：参数类型在 lambda 内标注，返回类型自动推断
var toUpper = fun((string s) => s.ToUpper());

// 与 language-ext 管道结合使用
var result = toUpper("hello"); // "HELLO"
Console.WriteLine(result);
```

---

#### 示例 3：双参数函数——整数加法与柯里化

```csharp
using static LanguageExt.Prelude;
using LanguageExt;

// ✅ fun 推断为 Func<int, int, int>
var add = fun((int x, int y) => x + y);

// 直接调用
Console.WriteLine(add(3, 5)); // 8

// 与 curry 结合——将双参函数变为两个单参函数的链式调用
var curriedAdd = curry(add);
var add10      = curriedAdd(10);  // 固定第一个参数为 10
Console.WriteLine(add10(7));     // 17
Console.WriteLine(add10(20));    // 30
```

---

#### 示例 4：双参数函数——字符串格式化

```csharp
using static LanguageExt.Prelude;

// ✅ fun 推断为 Func<string, int, string>
var formatScore = fun((string name, int score) => $"{name} 的得分是：{score} 分");

Console.WriteLine(formatScore("Alice", 95)); // Alice 的得分是：95 分
Console.WriteLine(formatScore("Bob",   82)); // Bob 的得分是：82 分

// 柯里化后批量应用
var curriedFormat = curry(formatScore);
var aliceScore    = curriedFormat("Alice");
Console.WriteLine(aliceScore(100)); // Alice 的得分是：100 分
Console.WriteLine(aliceScore(88));  // Alice 的得分是：88 分
```

---

#### 示例 5：三参数函数——计算长方体体积

```csharp
using static LanguageExt.Prelude;

// ✅ fun 推断为 Func<double, double, double, double>
var volume = fun((double l, double w, double h) => l * w * h);

Console.WriteLine(volume(3.0, 4.0, 5.0)); // 60

// 与 memo（记忆化）结合，避免重复计算
var memoVolume = memo(volume);
Console.WriteLine(memoVolume(3.0, 4.0, 5.0)); // 60（首次计算并缓存）
Console.WriteLine(memoVolume(3.0, 4.0, 5.0)); // 60（直接读缓存）
```

---

#### 示例 6：在高阶函数中作为参数传递

```csharp
using static LanguageExt.Prelude;
using System.Collections.Generic;

// ✅ 将 fun 定义的函数传入高阶函数
var isEven  = fun((int n) => n % 2 == 0);
var doubled = fun((int n) => n * 2);

var numbers = new List<int> { 1, 2, 3, 4, 5, 6 };

// 配合 LINQ 使用（fun 返回标准 Func，完全兼容 LINQ）
var evenDoubled = numbers
    .Where(isEven)
    .Select(doubled);

foreach (var n in evenDoubled)
    Console.Write($"{n} "); // 4 8 12
```

---

#### 示例 7：与 `pipe` 函数组合——构建数据处理管道

```csharp
using static LanguageExt.Prelude;

var trim      = fun((string s) => s.Trim());
var toLower   = fun((string s) => s.ToLower());
var addPrefix = fun((string s) => $"[处理结果] {s}");

// pipe 将多个函数串联：trim → toLower → addPrefix
var process = fun((string s) => pipe(s, trim, toLower, addPrefix));

Console.WriteLine(process("  Hello World  "));
// 输出：[处理结果] hello world
```

---

#### 示例 8：与 `Option` 类型结合——安全的数值解析

```csharp
using static LanguageExt.Prelude;
using LanguageExt;

// ✅ 定义安全解析函数，返回 Option<int>
var tryParseInt = fun((string s) =>
    int.TryParse(s, out var n) ? Some(n) : None
);

// 配合 Option 的 Map 链式操作
var doubled = fun((int n) => n * 2);

var result1 = tryParseInt("42").Map(doubled);  // Some(84)
var result2 = tryParseInt("abc").Map(doubled); // None

Console.WriteLine(result1); // Some(84)
Console.WriteLine(result2); // None
```

---

#### 示例 9：递归与 `fun`——利用变量引用自身

```csharp
using static LanguageExt.Prelude;

// fun 推断类型后，变量可以在闭包中引用自身实现递归
Func<int, int> factorial = null!;
factorial = fun((int n) => n <= 1 ? 1 : n * factorial(n - 1));

Console.WriteLine(factorial(5));  // 120
Console.WriteLine(factorial(10)); // 3628800
```

---

#### 示例 10：存储在字典中的函数映射表

```csharp
using static LanguageExt.Prelude;
using System.Collections.Generic;

// ✅ 因为 fun 返回的是明确类型的 Func<double, double>，可以安全存入集合
var operations = new Dictionary<string, Func<double, double>>
{
    ["平方"]   = fun((double x) => x * x),
    ["平方根"] = fun((double x) => Math.Sqrt(x)),
    ["倒数"]   = fun((double x) => 1.0 / x),
    ["取反"]   = fun((double x) => -x),
};

foreach (var (name, op) in operations)
    Console.WriteLine($"{name}(4) = {op(4)}");

// 输出：
// 平方(4)   = 16
// 平方根(4) = 2
// 倒数(4)   = 0.25
// 取反(4)   = -4
```

---

## 用途二：将 `Action` 转换为 `Func<…, Unit>`

### 为什么要把 `Action` 变成 `Func`？

在函数式编程中，**一切皆为值**，**一切皆有返回**。
`Action`（`void` 返回）打破了这个规则，它无法参与：

- `Func` 组合链（`compose`、`pipe`）
- `Option<T>.Map()`、`Either<L, R>.Map()` 等泛型映射
- `Task` 管道（`Task<Unit>` vs `Task<void>`）
- 柯里化（`curry`）
- 任何期望 `Func<…>` 的高阶函数

`Unit` 是 `language-ext` 中代替 `void` 的类型，它是一个只有单一值（`unit`）的结构体。
`fun(Action)` 系列重载会**自动在 Action 执行完毕后返回 `unit`**，
使副作用操作能够无缝融入函数式管道。

```csharp
// Unit 的本质
var u = unit; // LanguageExt.Unit 的唯一实例
```

### 代码示例集——Action 转 `Func<…, Unit>`

---

#### 示例 1：最简单的转换——无参 Action

```csharp
using static LanguageExt.Prelude;
using LanguageExt;

// ❌ 原始 Action，无法参与函数式组合
Action printHello = () => Console.WriteLine("Hello!");

// ✅ 转换为 Func<Unit>，可以参与函数式管道
var printHelloF = fun(() => Console.WriteLine("Hello!"));
// printHelloF 的类型是 Func<Unit>

Unit result = printHelloF(); // 打印 "Hello!"，并返回 unit
Console.WriteLine(result);   // ()   （Unit 的字符串表示）
```

---

#### 示例 2：单参数——控制台日志记录器

```csharp
using static LanguageExt.Prelude;
using LanguageExt;

// ✅ 将日志写入 Action 包装为 Func<string, Unit>
var log = fun((string message) => Console.WriteLine($"[LOG] {message}"));
// 类型：Func<string, Unit>

log("系统启动");     // [LOG] 系统启动
log("数据加载完成"); // [LOG] 数据加载完成

// 可以参与 Option.Map 链（副作用日志）
Option<string> userName = Some("Alice");
userName.Map(name => { log($"欢迎用户：{name}"); return name; });
```

---

#### 示例 3：单参数——文件写入操作

```csharp
using static LanguageExt.Prelude;
using LanguageExt;

// ✅ 文件操作（副作用）包装为 Func<string, Unit>
var writeToFile = fun((string content) =>
    File.AppendAllText("output.txt", content + Environment.NewLine)
);
// 类型：Func<string, Unit>

writeToFile("第一行数据");
writeToFile("第二行数据");

// 可以将 writeToFile 传入任何接受 Func<string, Unit> 的高阶函数
var lines = new[] { "行A", "行B", "行C" };
Array.ForEach(lines, line => writeToFile(line));
```

---

#### 示例 4：双参数——带分类的日志系统与柯里化

```csharp
using static LanguageExt.Prelude;
using LanguageExt;

// ✅ 双参数 Action 转换为 Func<string, string, Unit>
var log = fun((string level, string message) =>
{
    var timestamp = DateTime.Now.ToString("HH:mm:ss");
    Console.WriteLine($"[{timestamp}][{level.ToUpper()}] {message}");
});
// 类型：Func<string, string, Unit>

log("info",    "服务已启动");       // [08:30:01][INFO] 服务已启动
log("warning", "内存使用率 > 80%"); // [08:30:02][WARNING] 内存使用率 > 80%
log("error",   "数据库连接超时");   // [08:30:03][ERROR] 数据库连接超时

// 柯里化：固定日志级别，生成专用日志函数
var curriedLog = curry(log);
var logInfo    = curriedLog("info");
var logError   = curriedLog("error");

logInfo("初始化完成");  // [08:30:04][INFO] 初始化完成
logError("未知错误");   // [08:30:05][ERROR] 未知错误
```

---

#### 示例 5：双参数——键值存储（副作用写入）

```csharp
using static LanguageExt.Prelude;
using LanguageExt;
using System.Collections.Generic;

var store = new Dictionary<string, string>();

// ✅ 将字典写入包装为 Func<string, string, Unit>
var save = fun((string key, string value) => store[key] = value);
// 类型：Func<string, string, Unit>

save("name",    "Alice");
save("email",   "alice@example.com");
save("country", "China");

foreach (var (k, v) in store)
    Console.WriteLine($"{k}: {v}");
// name:    Alice
// email:   alice@example.com
// country: China
```

---

#### 示例 6：在函数式管道中处理副作用——`IfSome` + 日志

```csharp
using static LanguageExt.Prelude;
using LanguageExt;

var log    = fun((string msg) => Console.WriteLine($"[日志] {msg}"));
var notify = fun((string msg) => Console.WriteLine($"[通知] {msg}"));

// 模拟：查找用户，若存在则记录日志并发送通知
Option<string> FindUser(int id) =>
    id == 1 ? Some("Alice") : None;

var user = FindUser(1);

// IfSome 接受 Action<T>，配合 fun 转换后的 Func 都可使用
user.IfSome(name =>
{
    log($"用户 {name} 登录");
    notify($"欢迎回来，{name}！");
});
// [日志] 用户 Alice 登录
// [通知] 欢迎回来，Alice！

// 找不到用户时，不执行任何操作
FindUser(999).IfSome(name => log($"不会执行：{name}"));
```

---

#### 示例 7：三参数——数据库审计日志与柯里化

```csharp
using static LanguageExt.Prelude;
using LanguageExt;

// ✅ 三参数 Action 转换为 Func<string, string, string, Unit>
var auditLog = fun((string user, string action, string resource) =>
{
    var ts = DateTimeOffset.UtcNow.ToUnixTimeSeconds();
    Console.WriteLine($"[AUDIT] ts={ts} user={user} action={action} resource={resource}");
});
// 类型：Func<string, string, string, Unit>

auditLog("alice", "READ",   "/api/users");    // [AUDIT] ts=... user=alice action=READ ...
auditLog("bob",   "WRITE",  "/api/orders");   // [AUDIT] ts=... user=bob   action=WRITE ...
auditLog("admin", "DELETE", "/api/sessions"); // [AUDIT] ts=... user=admin action=DELETE ...

// 柯里化：固定当前用户，生成该用户专属的审计函数
var curriedAudit = curry(auditLog);
var aliceAudit   = curriedAudit("alice");       // 固定 user="alice"
var aliceCreate  = aliceAudit("CREATE");        // 继续固定 action="CREATE"

aliceCreate("/api/products");  // [AUDIT] ... user=alice action=CREATE resource=/api/products
aliceCreate("/api/inventory"); // [AUDIT] ... user=alice action=CREATE resource=/api/inventory
```

---

#### 示例 8：将 `Action` 序列转换为 `Func` 列表并批量执行

```csharp
using static LanguageExt.Prelude;
using LanguageExt;
using System.Collections.Generic;

// 一组副作用操作（例如应用初始化步骤）
// 因为都是 Func<Unit>，可以统一放入同一个 List，进行统一管理和执行
var initSteps = new List<Func<Unit>>
{
    fun(() => Console.WriteLine("步骤1：加载配置文件")),
    fun(() => Console.WriteLine("步骤2：建立数据库连接")),
    fun(() => Console.WriteLine("步骤3：预热缓存")),
    fun(() => Console.WriteLine("步骤4：启动后台任务")),
    fun(() => Console.WriteLine("步骤5：注册健康检查")),
};

// 批量执行
initSteps.ForEach(step => step());
// 步骤1：加载配置文件
// 步骤2：建立数据库连接
// 步骤3：预热缓存
// 步骤4：启动后台任务
// 步骤5：注册健康检查
```

---

#### 示例 9：与 `Try` 结合——安全的副作用执行

```csharp
using static LanguageExt.Prelude;
using LanguageExt;

// ✅ 将可能抛出异常的副作用包装为 Func<Unit>，再用 Try 包裹
var riskyWrite = fun(() =>
{
    // 模拟可能失败的操作
    if (new Random().Next(2) == 0)
        throw new IOException("磁盘写入失败");
    Console.WriteLine("数据写入成功");
});
// 类型：Func<Unit>

// Try 包裹 Func<Unit>，将异常转为 Result<Unit>
var safeWrite = Try(riskyWrite);
var result    = safeWrite.Match(
    Succ: _ => "操作成功",
    Fail: ex => $"操作失败：{ex.Message}"
);
Console.WriteLine(result); // "操作成功" 或 "操作失败：磁盘写入失败"
```

---

#### 示例 10：`fun` + 请求中间件链

```csharp
using static LanguageExt.Prelude;
using LanguageExt;
using System.Collections.Generic;

// 每个中间件都是 Func<string, Unit>，接受请求字符串并执行副作用
// 因为都是相同的 Func<string, Unit> 类型，可以放入统一的 List 管理
var middlewares = new List<Func<string, Unit>>
{
    fun((string req) => Console.WriteLine($"[认证]   验证请求：{req}")),
    fun((string req) => Console.WriteLine($"[限流]   检查频率：{req}")),
    fun((string req) => Console.WriteLine($"[日志]   记录请求：{req}")),
    fun((string req) => Console.WriteLine($"[处理]   处理请求：{req}")),
};

// 按顺序执行所有中间件
var request = "GET /api/data";
middlewares.ForEach(middleware => middleware(request));

// [认证]   验证请求：GET /api/data
// [限流]   检查频率：GET /api/data
// [日志]   记录请求：GET /api/data
// [处理]   处理请求：GET /api/data
```

---

## `fun` 的所有重载签名一览

### Func 类型推断重载（有返回值版）

| 重载 | 推断为 |
|------|--------|
| `fun<R>(Func<R> f)` | `Func<R>` |
| `fun<T1, R>(Func<T1, R> f)` | `Func<T1, R>` |
| `fun<T1, T2, R>(Func<T1, T2, R> f)` | `Func<T1, T2, R>` |
| `fun<T1, T2, T3, R>(...)` | `Func<T1, T2, T3, R>` |
| `fun<T1, ..., T4, R>(...)` | `Func<T1, ..., T4, R>` |
| ... | ... |
| `fun<T1, ..., T16, R>(...)` | `Func<T1, ..., T16, R>`（最多 16 个参数）|

### Action 转 Func<…, Unit> 重载（副作用版）

| 重载 | 转换为 |
|------|--------|
| `fun(Action f)` | `Func<Unit>` |
| `fun<T1>(Action<T1> f)` | `Func<T1, Unit>` |
| `fun<T1, T2>(Action<T1, T2> f)` | `Func<T1, T2, Unit>` |
| `fun<T1, T2, T3>(Action<T1, T2, T3> f)` | `Func<T1, T2, T3, Unit>` |
| `fun<T1, T2, T3, T4>(...)` | `Func<T1, T2, T3, T4, Unit>` |
| `fun<T1, ..., T5>(...)` | `Func<T1, ..., T5, Unit>` |
| `fun<T1, ..., T6>(...)` | `Func<T1, ..., T6, Unit>` |
| `fun<T1, ..., T7>(...)` | `Func<T1, ..., T7, Unit>` |

---

## 综合实战示例

### 实战：构建一个小型函数式用户注册流程

```csharp
using static LanguageExt.Prelude;
using LanguageExt;

// ============================================================
// 数据模型
// ============================================================
record UserInput(string Name, string Email, int Age);
record ValidatedUser(string Name, string Email, int Age);

// ============================================================
// 用 fun 定义所有纯函数和副作用函数
// ============================================================

// 纯函数：验证（返回 Either<错误信息, 合法值>）
var validateName  = fun((string name) =>
    string.IsNullOrWhiteSpace(name)
        ? Left<string, string>("姓名不能为空")
        : Right<string, string>(name));

var validateEmail = fun((string email) =>
    email.Contains('@')
        ? Right<string, string>(email)
        : Left<string, string>("邮箱格式不正确"));

var validateAge   = fun((int age) =>
    age is >= 18 and <= 120
        ? Right<string, int>(age)
        : Left<string, int>("年龄必须在 18 到 120 之间"));

// 纯函数：构建已验证用户
var buildUser = fun((string name, string email, int age) =>
    new ValidatedUser(name, email, age));

// 副作用函数（Action → Func<…, Unit>）——可参与函数式组合
var logSuccess       = fun((ValidatedUser u) =>
    Console.WriteLine($"[成功] 用户注册：{u.Name} <{u.Email}>，年龄：{u.Age}"));

var logError         = fun((string error) =>
    Console.WriteLine($"[失败] 注册失败：{error}"));

var saveToDb         = fun((ValidatedUser u) =>
    Console.WriteLine($"[数据库] 保存用户：{u.Name}"));

var sendWelcomeEmail = fun((ValidatedUser u) =>
    Console.WriteLine($"[邮件] 发送欢迎邮件至：{u.Email}"));

// ============================================================
// 注册流程：使用 LINQ 查询语法组合 Either 验证链
// ============================================================
void Register(UserInput input)
{
    var result =
        from name  in validateName(input.Name)
        from email in validateEmail(input.Email)
        from age   in validateAge(input.Age)
        select buildUser(name, email, age);

    result.Match(
        Right: user =>
        {
            logSuccess(user);
            saveToDb(user);
            sendWelcomeEmail(user);
        },
        Left: error => logError(error)
    );
}

// ============================================================
// 测试
// ============================================================
Register(new UserInput("Alice", "alice@example.com", 28));
// [成功] 用户注册：Alice <alice@example.com>，年龄：28
// [数据库] 保存用户：Alice
// [邮件] 发送欢迎邮件至：alice@example.com

Register(new UserInput("", "bob@example.com", 25));
// [失败] 注册失败：姓名不能为空

Register(new UserInput("Carol", "carol-no-at", 30));
// [失败] 注册失败：邮箱格式不正确

Register(new UserInput("Dave", "dave@example.com", 15));
// [失败] 注册失败：年龄必须在 18 到 120 之间
```

---

## 常见误区与注意事项

### ❌ 误区 1：认为 `fun` 会改变函数行为

```csharp
// fun 是纯粹的类型辅助工具，不改变任何执行逻辑
var f = fun((int x) => x * 2);
// 完全等价于：
Func<int, int> g = x => x * 2;
// f 和 g 的执行结果完全相同，fun 只是让编译器完成类型推断
```

### ❌ 误区 2：忽略 Action 版 `fun` 的返回值类型

```csharp
var log = fun((string s) => Console.WriteLine(s));
// log 的返回类型是 Unit，不是 void

// ❌ 错误写法（void 不是 Unit）：
// void result = log("test");

// ✅ 正确写法：
Unit u = log("test");  // 显式接收
var  v = log("test");  // 隐式推断为 Unit
_      = log("test");  // discard（丢弃返回值）
```

### ❌ 误区 3：忘记引入 `using static LanguageExt.Prelude;`

```csharp
// ❌ 没有引入时编译器找不到 fun
var f = fun((int x) => x + 1); // 错误：名称"fun"不存在

// ✅ 在文件顶部添加：
using static LanguageExt.Prelude;
// 或在项目的全局 usings 文件中添加一次：
// global using static LanguageExt.Prelude;
```

### ✅ 最佳实践：只在需要函数组合时使用 `fun`

```csharp
// 当 lambda 只用一次且不需要组合时，直接写 lambda 即可
var nums       = new[] { 1, 2, 3 };
var doubled    = nums.Select(x => x * 2); // ✅ 直接写 lambda，无需 fun

// 当需要传递给 language-ext 的高阶函数（curry, memo, compose 等）时，使用 fun
var double_     = fun((int x) => x * 2);
var memoDoubled = memo(double_);           // ✅ fun 后才能 memo
var curriedAdd  = curry(fun((int x, int y) => x + y)); // ✅ fun 后才能 curry
```

---

## 总结

| 特性 | 说明 |
|------|------|
| **本质** | 恒等包装器，利用目标类型推断固化 lambda 的 `Func<>` 类型 |
| **用途 1** | 将无类型 lambda 固化为具体 `Func<T1, …, R>`，便于存储、传递、组合 |
| **用途 2** | 将 `Action`（void）转换为 `Func<…, Unit>`，使副作用参与函数式管道 |
| **Func 版参数上限** | 最多支持 **16 个**参数 |
| **Action 版参数上限** | 最多支持 **7 个**参数 |
| **性能开销** | 几乎为零——编译期类型推断，运行时仅传递委托引用 |
| **核心价值** | 减少冗余类型声明，提升代码可读性，无缝融入 language-ext 函数式生态 |

```csharp
// 一句话总结：
// fun 让你写更少的类型声明，做更多的函数式组合。

var f = fun((int x, int y) => x + y);         // ✅ 简洁的 Func 推断
var g = fun((string s) => Console.Write(s));  // ✅ 优雅的 Action → Func<Unit>
```

---

> **参考资料**
>
> - [language-ext GitHub 仓库](https://github.com/louthy/language-ext)
> - [language-ext API 文档 — Lambda function inference](https://louthy.github.io/language-ext/LanguageExt.Core/Prelude/Lambda%20function%20inference/index.html)
> - [language-ext Prelude 总览](https://louthy.github.io/language-ext/LanguageExt.Core/Prelude/index.html)
