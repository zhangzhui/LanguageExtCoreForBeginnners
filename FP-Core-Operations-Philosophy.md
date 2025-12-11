# 为什么 map/bind/apply/traverse/sequence 这几个函数就够了？

## 核心设计哲学：组合性（Composability）

函数式编程的核心信条是：**复杂系统应该由简单、可组合的部件构建**。

这几个函数代表了处理"容器中的值"的**所有基本操作模式**：

```
┌─────────────────────────────────────────────────────────────┐
│                    容器 F<A> 中的值                          │
├─────────────────────────────────────────────────────────────┤
│  map       : 变换容器里的值         (A → B) → F<A> → F<B>    │
│  bind      : 链式组合有副作用的操作  (A → F<B>) → F<A> → F<B> │
│  apply     : 在容器中应用函数       F<A→B> → F<A> → F<B>     │
│  traverse  : 遍历并收集效果         (A → F<B>) → T<A> → F<T<B>│
│  sequence  : 翻转嵌套结构           T<F<A>> → F<T<A>>        │
└─────────────────────────────────────────────────────────────┘
```

## 每个函数解决什么问题？

### 1. **Map（Functor）**
最基本的操作：在不改变容器结构的情况下变换内部值。

```csharp
// 问题：如何在 Option 内部做计算？
Option<int> x = Some(5);
Option<int> y = x.Map(n => n * 2);  // Some(10)

// 无需拆箱，保持上下文（None 仍然是 None）
```

### 2. **Bind/FlatMap（Monad）**
解决"函数返回容器"时的嵌套问题。

```csharp
// 问题：连续调用返回 Option 的函数
Option<User> GetUser(int id);
Option<Address> GetAddress(User u);

// 没有 Bind：Option<Option<Address>> 嵌套了！
// 有了 Bind：
Option<Address> result = GetUser(1).Bind(u => GetAddress(u));
```

### 3. **Apply（Applicative）**
解决"多个独立容器值"的组合问题。

```csharp
// 问题：两个 Option 都有值时才计算
Option<int> a = Some(3);
Option<int> b = Some(4);

// Apply 允许把普通函数提升到容器世界
Option<int> sum = Some<Func<int,int,int>>((x,y) => x + y)
                    .Apply(a)
                    .Apply(b);  // Some(7)
```

### 4. **Traverse**
解决"对集合中每个元素执行有副作用操作，并收集结果"。

```csharp
// 问题：验证一组数据，任何一个失败就全部失败
List<string> inputs = ["1", "2", "3"];
Func<string, Option<int>> parse = s => ...;

// Traverse：List<Option<int>> 变成 Option<List<int>>
Option<List<int>> result = inputs.Traverse(parse);
// 全部成功：Some([1, 2, 3])
// 任一失败：None
```

### 5. **Sequence**
Traverse 的特例，当你已经有了 `T<F<A>>` 时翻转层次。

```csharp
List<Option<int>> list = [Some(1), Some(2), Some(3)];
Option<List<int>> result = list.Sequence();  // Some([1, 2, 3])
```

## 设计哲学：抽象的层次结构

这些函数形成一个**能力递增的层次**：

```
                    ┌─────────────┐
                    │  Traversable │  ← 可遍历并收集效果
                    └──────┬──────┘
                           │
                    ┌──────┴──────┐
                    │    Monad    │  ← 可顺序组合（有依赖）
                    └──────┬──────┘
                           │
                    ┌──────┴──────┐
                    │ Applicative │  ← 可并行组合（无依赖）
                    └──────┬──────┘
                           │
                    ┌──────┴──────┐
                    │   Functor   │  ← 可变换内部值
                    └─────────────┘
```

**核心原则：使用最弱的抽象**
- 只需要变换？用 Functor（map）
- 需要独立组合？用 Applicative（apply）
- 需要依赖前一步结果？用 Monad（bind）
- 需要遍历集合？用 Traversable

## 为什么这就"够了"？

### 1. **数学完备性**
这些抽象来自范畴论，经过数学证明是处理"上下文中的计算"的**完备基础操作**。

### 2. **正交性（Orthogonality）**
每个操作解决一个独立问题，互不重叠：

| 操作 | 解决的问题 |
|------|-----------|
| map | 值的变换 |
| bind | 顺序依赖 |
| apply | 独立组合 |
| traverse | 效果收集 |

### 3. **组合爆炸 → 组合简化**

传统 OOP 方式：每种容器都要写专门方法
```csharp
// 要为每种组合写代码
List<Option<T>> → Option<List<T>>  // 专门方法
Array<Either<E,T>> → Either<E,Array<T>>  // 又一个专门方法
// ... 组合爆炸
```

FP 方式：一个通用模式
```csharp
// Traverse 对所有 Traversable + Applicative 组合都有效
T<F<A>> → F<T<A>>  // 一个抽象，无限组合
```

### 4. **最小化原语，最大化表达力**

就像数学中：
- **加法 + 乘法** 足以构建所有算术
- **与/或/非** 足以构建所有布尔逻辑

函数式编程中：
- **map/bind/apply/traverse/sequence** 足以表达所有"容器内计算"的模式

## 类比：乐高积木

```
传统编程 = 买特定形状的模型（功能固定）
函数式编程 = 买基础积木块（无限组合）

map, bind, apply, traverse = 基础积木块
复杂程序 = 积木块的组合
```

## 实际例子：组合的力量

```csharp
// 复杂场景：批量获取用户，每个可能失败，要收集所有错误
Seq<int> userIds = [1, 2, 3];

// 只用这几个基本操作
var result = userIds
    .Traverse(id => GetUser(id))           // Seq<IO<User>> → IO<Seq<User>>
    .Bind(users => users.Traverse(Validate)) // 验证每个用户
    .Map(users => users.Filter(IsActive));   // 过滤活跃用户

// 没有循环，没有临时变量，没有空检查
// 所有错误处理、副作用管理都由这些操作自动处理
```

## 总结

| 设计原则 | 体现 |
|---------|------|
| **最小化接口** | 5个函数覆盖所有场景 |
| **正交性** | 每个函数解决一个独立问题 |
| **组合性** | 简单操作组合成复杂行为 |
| **数学基础** | 范畴论保证完备性 |
| **类型驱动** | 类型签名即文档 |

这就是为什么函数式编程看起来"很少的东西"却能解决复杂问题——**不是功能少，而是抽象层次高，每个抽象都恰好解决一类问题**。
