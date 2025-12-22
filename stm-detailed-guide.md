# STM（软件事务内存）详解

## 什么是 STM？

STM 是一种**无锁并发控制机制**，使用 **MVCC（多版本并发控制）** 来管理共享可变状态。它借鉴了数据库事务的思想，提供 **ACI** 特性：

| 特性 | 说明 |
|------|------|
| **Atomic（原子性）** | 事务内所有修改要么全部成功，要么全部回滚 |
| **Consistent（一致性）** | 可通过验证器确保数据始终有效 |
| **Isolated（隔离性）** | 事务之间互不干扰 |

---

## 核心类型

### 1. `Ref<A>` - 事务性引用

`Ref` 是 STM 的核心，它包装一个可变值，**只能在事务内修改**：

```csharp
using LanguageExt;
using static LanguageExt.Prelude;

// 创建 Ref（可选验证器）
var counter = Ref(0);                              // 无验证
var balance = Ref(1000, b => b >= 0);             // 带验证：余额不能为负

// 读取（可在事务外读取）
int value = counter.Value;

// 写入（必须在事务内！）
atomic(() => counter.Value = 10);

// Swap - 原子更新
atomic(() => counter.Swap(x => x + 1));
```

### 2. `Atom<A>` - 原子引用（简化版）

当你只需要**单个独立状态**，不需要跨多个值的事务时，使用 `Atom`：

```csharp
// 创建 Atom
var atom = Atom(0);

// Swap - 无锁原子更新（可能重试，所以函数必须无副作用！）
atom.Swap(x => x + 1);

// 带验证的 Atom
var validated = Atom(100, x => x >= 0);  // 返回 Option<Atom<int>>

// 读取
int val = atom.Value;
```

**Atom vs Ref 区别**：
- `Atom` - 单值、独立、无需事务、CAS 自旋
- `Ref` - 可组合、需要事务、支持多值协调

---

## 事务 API

### `atomic` / `snapshot` / `serial`

```csharp
// 基本事务（默认 Snapshot 隔离）
atomic(() => {
    ref1.Value = 10;
    ref2.Value = 20;
});

// 返回值
var result = atomic(() => ref1.Value + ref2.Value);

// 显式指定隔离级别
atomic(() => { ... }, Isolation.Serialisable);

// 便捷方法
snapshot(() => { ... });  // 等同于 atomic(..., Isolation.Snapshot)
serial(() => { ... });    // 等同于 atomic(..., Isolation.Serialisable)
```

### 隔离级别

```csharp
public enum Isolation
{
    Snapshot,      // 只检查写入冲突
    Serialisable   // 检查读取+写入冲突（更严格）
}
```

**Snapshot 示例**：
```csharp
var x = Ref(1);
var y = Ref(2);

// 如果其他线程在事务执行期间修改了 y，事务不会回滚
// 因为 y 只被读取，没有被写入
snapshot(() => x.Value = y.Value + 1);
```

**Serialisable 示例**：
```csharp
var x = Ref(1);
var y = Ref(2);

// 如果其他线程在事务执行期间修改了 y，事务会回滚重试
// 因为 Serialisable 检查所有读取的值
serial(() => x.Value = y.Value + 1);
```

---

## 实战示例：银行转账

```csharp
// 定义账户（不可变类型！）
public class Account
{
    public readonly int Balance;
    
    private Account(int balance) => Balance = balance;
    
    public Account AddBalance(int amount) => new Account(Balance + amount);
    
    // 验证器：余额不能为负
    public static bool Validate(Account a) => a.Balance >= 0;
    
    // 工厂方法
    public static Ref<Account> New(int balance) =>
        Ref(new Account(balance), Account.Validate);
}

// 转账操作
public static class Transfer
{
    public static Unit Do(Ref<Account> from, Ref<Account> to, int amount) =>
        atomic(() =>
        {
            // 这两个操作要么同时成功，要么同时失败
            from.Value = from.Value.AddBalance(-amount);
            to.Value = to.Value.AddBalance(amount);
        });
}

// 使用
var accountA = Account.New(200);  // 初始余额 200
var accountB = Account.New(0);    // 初始余额 0

Transfer.Do(accountA, accountB, 100);

// 查询余额（也在事务内，保证一致性）
var (balA, balB) = atomic(() => (accountA.Value.Balance, accountB.Value.Balance));
Console.WriteLine($"A: {balA}, B: {balB}");  // A: 100, B: 100
```

**如果转账金额超过余额呢？**

由于验证器 `Balance >= 0`，事务会因验证失败而**不提交**，两个账户都保持原状！

---

## Commute - 可交换操作

当更新函数满足**交换律**（顺序无关）时，使用 `commute` 可获得更好的并发性能：

```csharp
var counter = Ref(0);

// 普通更新 - 冲突时整个事务重试
atomic(() => counter.Value = counter.Value + 1);

// Commute - 冲突时只重试该操作
atomic(() => commute(counter, x => x + 1));
```

**工作原理**：
1. 事务内先用当前值计算
2. 提交时，用**最新提交值**重新计算
3. 因为是可交换的，结果相同

**典型场景**：计数器、存款

```csharp
var bank = Account.New(0);

// 存款是可交换的：存100再存50 = 存50再存100
void Deposit(Ref<Account> account, int amount) =>
    atomic(() => commute(account, a => a.AddBalance(amount)));

// 并发存款
Parallel.Invoke(
    () => Deposit(bank, 100),
    () => Deposit(bank, 50),
    () => Deposit(bank, 30)
);

// 结果一定是 180
```

---

## Change 事件

`Ref` 支持监听值变化：

```csharp
var account = Account.New(100);

// 订阅变化事件（事务提交后触发）
account.Change += newValue => 
    Console.WriteLine($"Balance changed to: {newValue.Balance}");

atomic(() =>
{
    // 事务内多次修改
    account.Value = account.Value.AddBalance(-10);
    account.Value = account.Value.AddBalance(-5);
    // Change 事件还未触发！
});
// 事务提交后，Change 事件触发一次，值为最终状态
```

---

## 与 IO/Eff 集成

STM 可以与 Effect 系统无缝集成：

```csharp
var counter = Ref(0);

// 使用 ValueIO 属性
IO<int> readCounter = counter.ValueIO;
IO<int> incrementIO = counter.SwapIO(x => x + 1);

// 在 Eff 事务中使用
Eff<Unit> program = 
    from _ in STM.DoTransaction(
        from val in counter.ValueIO
        from __ in IO.lift(() => { counter.Value = val + 1; return unit; })
        select unit,
        Isolation.Snapshot)
    select unit;
```

---

## 最佳实践

### ✅ 推荐

```csharp
// 1. 使用不可变类型作为 Ref 的值
public record Account(decimal Balance);

// 2. 提供验证器保护数据完整性
var account = Ref(new Account(100), a => a.Balance >= 0);

// 3. 保持事务短小
atomic(() => {
    // 只做必要的读写
});

// 4. 对可交换操作使用 commute
atomic(() => commute(counter, x => x + 1));
```

### ❌ 避免

```csharp
// 1. 不要在事务内执行 I/O（事务可能重试！）
atomic(() => {
    Console.WriteLine("This may print multiple times!");  // ❌
    ref1.Value = 10;
});

// 2. 不要使用可变类型
var bad = Ref(new List<int>());  // ❌ List 是可变的！

// 3. 不要让 Swap 函数有副作用
atom.Swap(x => {
    Console.WriteLine(x);  // ❌ 可能执行多次
    return x + 1;
});
```

---

## 何时使用 STM vs 其他方案？

| 场景 | 推荐方案 |
|------|----------|
| 单个独立状态 | `Atom<A>` |
| 多个状态需要协调 | `Ref<A>` + `atomic` |
| 高频读、低频写 | `Atom<A>` |
| 需要回滚保护 | `Ref<A>` + 验证器 |
| 简单的线程安全集合 | `AtomHashMap<K,V>` / `AtomSeq<A>` |

---

## 总结

```
┌─────────────────────────────────────────────────────────────┐
│                        STM 架构                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Ref<A>  ←──────────────→  Ref<B>  ←──────────────→ ...    │
│     │                         │                             │
│     └─────────┬───────────────┘                             │
│               │                                             │
│               ▼                                             │
│      atomic(() => { ... })                                  │
│               │                                             │
│     ┌─────────┴─────────┐                                   │
│     │                   │                                   │
│  成功提交            冲突重试                                 │
│     │                   │                                   │
│     ▼                   └──→ 自动使用最新快照重新执行         │
│  触发 Change 事件                                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 相关源码位置

- `LanguageExt.Core/Concurrency/STM/STM.cs` - STM 核心实现
- `LanguageExt.Core/Concurrency/STM/Ref.cs` - Ref 类型
- `LanguageExt.Core/Concurrency/STM/Isolation.cs` - 隔离级别枚举
- `LanguageExt.Core/Concurrency/Atom/Atom.cs` - Atom 类型
- `LanguageExt.Core/Concurrency/Prelude.Concurrency.cs` - 便捷方法（atomic, snapshot, serial 等）
- `LanguageExt.Tests/RefTest.cs` - 测试示例
