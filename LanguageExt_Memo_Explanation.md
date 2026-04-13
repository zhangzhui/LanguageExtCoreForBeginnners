# LanguageExt 中的 `Memo`（记忆化）详解

> 适用版本：LanguageExt v4 / v5
> 语言：C#
> 作者：Paul Louth（[louthy/language-ext](https://github.com/louthy/language-ext)）

---

## 一、概念：什么是 Memo（记忆化）？

**Memo**（Memoization，记忆化）是函数式编程中的一种**优化技术**，其核心思想是：

> 将一个**纯函数**的计算结果缓存起来，当相同的输入再次出现时，直接返回缓存结果，而不重新执行计算。

在 LanguageExt 中，`Memo` 以若干扩展方法和工具函数的形式提供，主要通过 `memo` 函数将普通委托（`Func<A, B>`）包装为具有缓存能力的版本。

### 核心特征

| 特征 | 说明 |
|------|------|
| **透明性** | 对调用者完全透明，行为与原函数一致 |
| **幂等性** | 相同输入始终返回相同输出（要求原函数为纯函数） |
| **惰性求值** | 首次调用时才计算，后续命中缓存直接返回 |
| **线程安全** | LanguageExt 的实现保证并发场景下的安全性 |

---

## 二、作者意图：Paul Louth 为何引入 Memo？

Paul Louth 设计 LanguageExt 的目标是将 **Haskell / 函数式编程的精髓** 带入 C# 生态。`Memo` 的引入源于以下几点核心意图：

### 1. 拥抱纯函数（Pure Functions）
函数式编程要求函数是**无副作用的纯函数**。纯函数天然满足记忆化的前提条件，Memo 正是对这一特性的最佳利用。

### 2. 消除重复计算的副作用
在递归、管道组合或多次遍历数据流的场景中，同一计算可能被多次触发。Memo 以最小侵入性的方式解决性能瓶颈，**无需修改原有业务逻辑**。

### 3. 与惰性求值（Lazy Evaluation）协同
LanguageExt 大量使用 `Lazy<T>`、`Option<T>`、`Either<L, R>` 等惰性/延迟结构，Memo 与之天然契合，构成完整的**延迟计算 + 缓存**链路。

### 4. 提升 DSL 表达力
通过 `memo` 包装后的函数仍可自由组合、柯里化、部分应用，不破坏函数式 DSL 的流畅性。

---

## 三、使用场景

### 场景 1：递归计算优化（经典斐波那契）
递归函数存在大量重叠子问题，Memo 可将指数复杂度降至线性。

### 场景 2：昂贵的 I/O 或计算结果缓存
对数据库查询、文件解析、网络请求等耗时操作的结果进行缓存（需保证结果不变性）。

### 场景 3：配置/元数据的按需加载
系统配置在运行期间不变，使用 Memo 实现"首次读取后永久缓存"。

### 场景 4：函数组合管道中的中间结果缓存
在复杂的函数组合链中，对中间转换结果进行记忆化，避免重复处理相同输入。

### 场景 5：部分应用函数的结果缓存
对柯里化后固定部分参数的函数进行记忆化，提升复用效率。

---

## 四、代码案例

### 4.1 基本用法：包装一个纯函数

```csharp
using LanguageExt;
using static LanguageExt.Prelude;

// 原始纯函数：计算平方
Func<int, int> square = x => {
    Console.WriteLine($"  [计算中] square({x})");
    return x * x;
};

// 使用 memo 包装，生成具有缓存能力的版本
var memoSquare = memo(square);

Console.WriteLine(memoSquare(5));  // 输出：[计算中] square(5)  → 25
Console.WriteLine(memoSquare(5));  // 直接命中缓存            → 25
Console.WriteLine(memoSquare(3));  // 输出：[计算中] square(3)  → 9
Console.WriteLine(memoSquare(3));  // 直接命中缓存            → 9
```

**输出：**
```
[计算中] square(5)
25
25
[计算中] square(3)
9
9
```

---

### 4.2 递归记忆化：斐波那契数列

```csharp
using LanguageExt;
using static LanguageExt.Prelude;

// 声明可被递归引用的 memo 函数
Func<long, long> fib = null!;

fib = memo<long, long>(n =>
    n <= 1
        ? n
        : fib(n - 1) + fib(n - 2)   // 递归调用已被 memo 包装的版本
);

Console.WriteLine(fib(10));   // 55
Console.WriteLine(fib(40));   // 102334155（瞬时完成，无重复计算）
Console.WriteLine(fib(70));   // 190392490709135（毫秒级）
```

> **注意**：若不使用 `memo`，`fib(40)` 会触发约 10 亿次函数调用；使用 `memo` 后仅需 40 次。

---

### 4.3 与 Option 组合：安全的缓存查询

```csharp
using LanguageExt;
using static LanguageExt.Prelude;

// 模拟一个可能失败的数据库查询
Func<int, Option<string>> queryDb = id => {
    Console.WriteLine($"  [DB查询] id={id}");
    return id > 0
        ? Some($"用户_{id}")
        : None;
};

// memo 同样适用于返回 Option 的函数
var cachedQuery = memo(queryDb);

var result1 = cachedQuery(1);   // 触发 DB 查询
var result2 = cachedQuery(1);   // 命中缓存，不触发 DB 查询
var result3 = cachedQuery(-1);  // 触发 DB 查询，返回 None
var result4 = cachedQuery(-1);  // 命中缓存，返回缓存的 None

result1.IfSome(name => Console.WriteLine($"查询结果：{name}"));
result3.IfNone(() => Console.WriteLine("用户不存在（来自缓存）"));
```

---

### 4.4 多参数函数的记忆化

```csharp
using LanguageExt;
using static LanguageExt.Prelude;

// 二元函数的 memo（先柯里化再 memo）
Func<int, int, int> add = (a, b) => {
    Console.WriteLine($"  [计算中] {a} + {b}");
    return a + b;
};

// 方式一：柯里化后逐层 memo
var curriedAdd = curry(add);
var memoAdd5 = memo(curriedAdd(5));   // 固定第一个参数为 5

Console.WriteLine(memoAdd5(3));   // [计算中] 5 + 3 → 8
Console.WriteLine(memoAdd5(3));   // 命中缓存        → 8
Console.WriteLine(memoAdd5(7));   // [计算中] 5 + 7 → 12
```

---

### 4.5 配置加载场景

```csharp
using LanguageExt;
using static LanguageExt.Prelude;

// 模拟从磁盘加载配置（昂贵操作）
Func<string, string> loadConfig = key => {
    Console.WriteLine($"  [磁盘读取] key={key}");
    // 实际项目中替换为真实的文件/数据库读取
    return key switch {
        "db_conn"  => "Server=localhost;Database=MyDb;",
        "api_key"  => "sk-xxxxxxxxxxxxxxxx",
        "max_conn" => "100",
        _          => string.Empty
    };
};

var config = memo(loadConfig);

// 应用启动时多处读取同一配置项，只触发一次磁盘 I/O
var conn1 = config("db_conn");   // 触发磁盘读取
var conn2 = config("db_conn");   // 命中缓存
var conn3 = config("db_conn");   // 命中缓存

Console.WriteLine($"连接字符串：{conn1}");
```

---

## 五、对比表格

### 5.1 Memo vs. 其他缓存方案

| 对比维度 | `LanguageExt.memo` | `Dictionary` 手动缓存 | `IMemoryCache`（ASP.NET） | `Lazy<T>` |
|----------|--------------------|-----------------------|---------------------------|-----------|
| **适用对象** | 任意纯函数 `Func<A, B>` | 手动管理键值 | 任意对象（DI 注入） | 单一无参值 |
| **线程安全** | ✅ 内置保证 | ❌ 需手动加锁 | ✅ 内置保证 | ✅ 内置保证 |
| **代码侵入性** | ✅ 极低，一行包装 | ❌ 高，需改写逻辑 | ❌ 中，需注入依赖 | ✅ 低 |
| **函数组合友好** | ✅ 完全兼容 FP 风格 | ❌ 命令式风格 | ❌ 命令式风格 | ⚠️ 有限支持 |
| **缓存粒度** | 按函数输入参数 | 自定义 | 自定义 Key | 整体单值 |
| **TTL / 过期策略** | ❌ 无（永久缓存） | ⚠️ 需手动实现 | ✅ 丰富的过期策略 | ❌ 无 |
| **内存管理** | ⚠️ 需注意缓存膨胀 | ⚠️ 需手动清理 | ✅ 自动淘汰策略 | ✅ 单值无问题 |
| **适用场景** | 纯函数 + FP 管道 | 简单键值缓存 | Web 应用级缓存 | 单例延迟初始化 |

---

### 5.2 Memo vs. 相关 FP 概念

| 概念 | 定义 | 与 Memo 的关系 |
|------|------|----------------|
| **Pure Function（纯函数）** | 无副作用、相同输入相同输出 | Memo 的**前提条件**，只有纯函数才适合记忆化 |
| **Lazy Evaluation（惰性求值）** | 值在需要时才计算 | Memo 兼具惰性求值特性（首次调用才计算） |
| **Referential Transparency（引用透明）** | 表达式可被其值替换 | Memo 保留引用透明性，缓存不改变语义 |
| **Currying（柯里化）** | 多参函数转为单参函数链 | 与 Memo 组合可对部分应用结果缓存 |
| **Higher-Order Function（高阶函数）** | 接受或返回函数的函数 | `memo` 本身是高阶函数，接受函数返回函数 |

---

### 5.3 适用 vs. 不适用场景

| 场景 | 是否适合 Memo | 原因 |
|------|:---:|------|
| 纯数学计算（斐波那契、阶乘） | ✅ | 纯函数，结果确定性强 |
| 静态配置/元数据读取 | ✅ | 运行期结果不变 |
| 数据转换/序列化（输入固定） | ✅ | 相同输入产生相同输出 |
| 随机数生成 | ❌ | 非纯函数，每次结果不同 |
| 依赖当前时间的计算 | ❌ | 相同输入，不同时间结果不同 |
| 有副作用的 I/O 操作 | ❌ | 副作用不应被缓存跳过 |
| 结果会随状态变化的查询 | ❌ | 数据库记录可能更新，缓存会过时 |
| 参数空间极大的函数 | ⚠️ | 缓存命中率低，可能造成内存浪费 |

---

## 六、注意事项与最佳实践

1. **仅对纯函数使用 Memo**：若函数有副作用（写日志、修改状态、I/O），Memo 会跳过副作用，导致行为不符合预期。

2. **注意内存泄漏**：`memo` 默认使用无界缓存，对于参数值域极大的函数（如以任意字符串为 Key），可能导致内存持续增长。此时应考虑 `IMemoryCache` 等带 LRU 策略的方案。

3. **引用类型参数需正确实现 `Equals` / `GetHashCode`**：Memo 内部使用字典存储，Key 的相等性判断依赖这两个方法。

4. **递归函数的 memo 需先声明变量**：如案例 4.2 所示，需先声明 `Func` 变量，再赋值 `memo(...)` 包装体，否则递归调用无法引用到已记忆化的版本。

5. **与 `Lazy<T>` 的区分**：`Lazy<T>` 适合无参的单次初始化；`memo` 适合有参数、多次调用、需按参数缓存的场景。

---

## 七、参考资源

- [LanguageExt GitHub 仓库](https://github.com/louthy/language-ext)
- [LanguageExt Wiki](https://github.com/louthy/language-ext/wiki)
- [Memoization - Wikipedia](https://en.wikipedia.org/wiki/Memoization)
- [Paul Louth 的博客](https://paullouth.com/)
