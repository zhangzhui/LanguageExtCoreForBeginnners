# Monad 与 Transformer 分析

## 1. 项目中的 Monad 分类

| 基础 Monad | 对应 Transformer | 说明 |
|-----------|-----------------|------|
| **Option** | OptionT | 可选值 |
| **Either** | EitherT | 二选一（成功/失败） |
| **Fin** | FinT | 带 Error 的结果 |
| **Try** | TryT | 异常捕获 |
| **Validation** | ValidationT | 验证（可累积错误） |
| **Reader** | ReaderT | 环境读取 |
| **Writer** | WriterT | 日志输出 |
| **State** | StateT | 状态管理 |
| **Identity** | IdentityT | 恒等（基础包装） |
| **Chronicle** | ChronicleT | 带日志的结果 |
| - | RWST | Reader+Writer+State 组合 |
| **Free** | - | 自由 Monad（无 Transformer） |
| **Cont** | ContT | 延续 |
| **These** | - | 无 Transformer |
| **Trampoline** | - | 无 Transformer（尾递归优化） |

---

## 2. 规律总结

### 需要 Transformer 的 Monad 特征

```
核心规律：需要与其他效果组合的 Monad 才需要 Transformer
```

**规律 1：携带"额外信息"的 Monad 需要 Transformer**

| 类型 | 携带的信息 | 为何需要 Transformer |
|-----|----------|-------------------|
| Option | 有/无 | 需要在 IO 中表达"可能没有" |
| Either/Fin | 错误值 | 需要在 IO 中表达"可能失败" |
| Reader | 环境依赖 | 需要在其他效果中读取配置 |
| Writer | 日志 | 需要在其他效果中写日志 |
| State | 状态 | 需要在其他效果中管理状态 |

**规律 2：不需要 Transformer 的 Monad 特征**

| 类型 | 原因 |
|-----|-----|
| **Free** | 本身就是"构建器"，通过解释器运行，不需要嵌套 |
| **Trampoline** | 仅用于尾递归优化，不涉及效果组合 |
| **These** | 较少用于效果组合场景 |

---

## 3. 记忆技巧

### 口诀：效果要叠加，Transformer 来帮忙

```
┌─────────────────────────────────────────────────────┐
│  问自己：这个 Monad 的"效果"需要和其他效果组合吗？  │
│                                                     │
│  Option  → "可能没有" + IO = OptionT<IO, A>        │
│  Either  → "可能失败" + IO = EitherT<IO, L, A>     │
│  Reader  → "需要环境" + IO = ReaderT<IO, Env, A>   │
│  State   → "有状态"   + IO = StateT<IO, S, A>      │
└─────────────────────────────────────────────────────┘
```

### 思维模型：洋葱模型

```
最外层：IO (副作用)
  ↓
中间层：ReaderT (环境)
  ↓  
内层：EitherT (错误处理)
  ↓
核心：业务值 A

完整类型：ReaderT<EitherT<IO, Error>, Config, A>
```

### 简化记忆表

| 如果你需要... | 用 Transformer... |
|-------------|------------------|
| IO 中可能失败 | `EitherT<IO, E, A>` 或 `FinT<IO, A>` |
| IO 中可能为空 | `OptionT<IO, A>` |
| IO 中需要配置 | `ReaderT<IO, Config, A>` |
| IO 中需要状态 | `StateT<IO, S, A>` |
| IO 中需要日志 | `WriterT<IO, W, A>` |
| 全都要 | `RWST<M, R, W, S, A>` |

---

## 4. 为什么需要 Transformer？

**问题场景**：
```csharp
// 你有两个效果：
IO<A>           // 副作用
Option<A>       // 可能为空

// 直接嵌套的问题：
IO<Option<A>>   // 操作不便，需要两层 Map/Bind
Option<IO<A>>   // 语义混乱

// Transformer 解决方案：
OptionT<IO, A>  // 统一的 Monad 接口，一层操作
```

**Transformer 的本质**：
```
Transformer = 把内层 Monad 的操作"提升"到组合类型上
```

---

## 5. 快速判断法

```
问题：Monad X 需要 Transformer 吗？

判断流程：
1. X 是纯计算工具吗？(如 Trampoline) → 不需要
2. X 是解释器模式吗？(如 Free) → 不需要  
3. X 需要和 IO/Task 等组合吗？→ 需要 Transformer
4. X 携带需要传递的上下文吗？(环境/状态/日志) → 需要 Transformer
```
