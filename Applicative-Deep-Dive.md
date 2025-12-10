# Applicative 深入解析：为什么是"独立计算"？

## Applicative 的定义

```csharp
// 两个核心操作
Pure : A → F<A>                        // 把值放入容器
Apply: F<A → B> → F<A> → F<B>          // 把容器里的函数应用到容器里的值
```

## 为什么说是"独立计算"？

关键在于对比 **Applicative** 和 **Monad**：

### Monad 的 Bind（依赖）

```csharp
// Bind 签名
Bind: F<A> → (A → F<B>) → F<B>
//              ↑
//          需要拿到 A 的值，才能决定下一步产生什么 F<B>
```

```csharp
// 例子：第二步依赖第一步的结果
from userId in GetUserId()           // 先拿到 userId
from user   in GetUser(userId)       // 才能用 userId 去查 user
select user;
```

第二个计算 `GetUser(userId)` **必须等待**第一个计算完成，因为它需要 `userId` 这个值。

### Applicative 的 Apply（独立）

```csharp
// Apply 签名
Apply: F<A → B> → F<A> → F<B>
//     ↑           ↑
//     两个 F 是独立的，互不依赖
```

```csharp
// 例子：三个查询互不依赖
var result = (GetName(), GetAge(), GetEmail())
    .Apply((name, age, email) => new User(name, age, email));
```

`GetName()`、`GetAge()`、`GetEmail()` 三个计算**互不知道对方的存在**，它们可以：
- 并行执行
- 按任意顺序执行
- 独立失败

## 原理详解

### 用柯里化理解 Apply

假设我们有一个函数和两个容器值：

```csharp
Func<int, int, int> add = (a, b) => a + b;
Option<int> optA = Some(3);
Option<int> optB = Some(5);
```

**步骤分解：**

```csharp
// 1. 把函数放入容器
Option<Func<int, int, int>> optAdd = Pure(add);  // Some(add)

// 2. 柯里化：add 变成 int → (int → int)
Option<Func<int, Func<int, int>>> curriedAdd = Some(a => b => a + b);

// 3. 第一次 Apply：把 optA 喂给柯里化函数
Option<Func<int, int>> partial = curriedAdd.Apply(optA);  // Some(b => 3 + b)

// 4. 第二次 Apply：把 optB 喂给部分应用的函数
Option<int> result = partial.Apply(optB);  // Some(8)
```

**关键洞察**：每次 Apply 时，我们只是把**已经存在的容器**组合起来，不需要打开任何容器来决定下一步做什么。

### 图解对比

```
Monad (依赖链):
┌───────┐    拆开取值A    ┌─────────────┐    拆开取值B    ┌───────┐
│ F<A>  │ ────────────→ │ A → F<B>    │ ────────────→ │ F<B>  │
└───────┘                └─────────────┘                └───────┘
                              ↑
                         需要A才能构造F<B>

Applicative (独立组合):
┌───────────┐
│ F<A → B>  │──┐
└───────────┘  │    同时存在
               ├──→ Apply ──→ F<B>
┌───────────┐  │    组合
│   F<A>    │──┘
└───────────┘
    ↑
两个F独立存在，互不依赖
```

## 实际例子

### 例1：表单验证（独立收集所有错误）

```csharp
// 每个验证独立执行
Validation<Error, string> validName  = ValidateName(input.Name);
Validation<Error, int>    validAge   = ValidateAge(input.Age);
Validation<Error, string> validEmail = ValidateEmail(input.Email);

// Applicative 组合：即使某个失败，其他仍然执行
var result = (validName, validAge, validEmail)
    .Apply((name, age, email) => new User(name, age, email));

// 结果：Success(User) 或 Failure([所有的错误])
```

如果用 Monad，第一个失败就短路了，无法收集所有错误。

### 例2：并行 IO

```csharp
// 三个独立的 HTTP 请求
IO<User>     getUser     = FetchUser(id);
IO<List<Order>> getOrders = FetchOrders(id);
IO<Settings> getSettings = FetchSettings(id);

// Applicative：可以并行！
var result = (getUser, getOrders, getSettings)
    .Apply((user, orders, settings) => new Dashboard(user, orders, settings));
```

编译器/运行时**知道**这三个计算独立，可以安全并行。

### 例3：解析器组合

```csharp
// 独立的解析器
Parser<string> parseTag   = String("<tag>");
Parser<string> parseBody  = Many(Letter);
Parser<string> parseClose = String("</tag>");

// Applicative 组合
var parseElement = (parseTag, parseBody, parseClose)
    .Apply((open, body, close) => body);
```

## Applicative vs Monad 的能力边界

```
                    Applicative          Monad
                    ─────────────────────────────
能做什么？          静态结构组合         动态结构
计算图              编译时确定           运行时构建
并行可能性          是                   否（必须顺序）
错误收集            可以收集             短路
分析可能性          可静态分析           不可
```

### 关键区别代码

```csharp
// Applicative：结构在编译时确定
(fa, fb).Apply((a, b) => ...)   // 不管 a 是什么，fb 都会执行

// Monad：结构在运行时确定
fa.Bind(a => a > 0 ? fb : fc)   // 必须知道 a 的值，才知道执行 fb 还是 fc
```

## 记忆口诀

> **Applicative = 独立的事情一起做**
>
> **Monad = 后面的事依赖前面的结果**

或者：

> Applicative 像**并行流水线**：每个工位独立工作
>
> Monad 像**串行流水线**：下一步要看上一步的结果

## 总结对比表

| 特性 | Applicative | Monad |
|------|-------------|-------|
| 核心操作 | `Apply: F<A→B> → F<A> → F<B>` | `Bind: F<A> → (A→F<B>) → F<B>` |
| 计算依赖 | 独立 | 依赖 |
| 执行顺序 | 可并行 | 必须顺序 |
| 错误处理 | 收集所有 | 短路 |
| 结构 | 静态确定 | 动态构建 |

## 核心洞察

Applicative 的"独立"来自其类型签名：`F<A→B>` 和 `F<A>` 都是**已经存在**的值，组合它们不需要先解开任何一个。

而 Monad 的 `A → F<B>` 需要先拿到 `A`，才能决定产生什么 `F<B>`。

这就是为什么 Applicative 表示"独立计算"的根本原因。
