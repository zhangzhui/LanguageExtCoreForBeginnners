# Prism 详细指南 - language-ext

> 本文档详细介绍 language-ext 库中的 Prism 概念，帮助理解其与 Lens 的区别及使用场景。

## 1. 概念背景：从 Lens 说起

### Lens 是什么

Lens 是一种函数式编程中的 **光学 (Optics)** 工具，用于聚焦和操作不可变数据结构中的嵌套字段。

```csharp
// Lens 的核心：Get 和 Set
public interface ILens<S, A>
{
    A Get(S source);           // 从 S 中获取 A
    S Set(A value, S source);  // 在 S 中设置 A，返回新的 S
}
```

### Lens 的局限性

Lens 假设目标字段 **总是存在**，这在以下场景中会遇到问题：

1. **可选字段**: `Option<T>` 类型，值可能不存在
2. **联合类型 (Sum Types)**: 如 `Either<L, R>`，只有一个分支有值
3. **集合元素**: 列表中的某个元素可能不存在
4. **多态结构**: 不同子类型有不同的字段

```csharp
// Lens 无法优雅处理这种情况
record Person(string Name, Option<Address> Address);

// 如果用 Lens，Get 返回什么？Option<Address> 还是 Address？
// 如果 Address 是 None，Set 操作有意义吗？
```

---

## 2. Prism 是什么

### 定义

Prism 是一种 **部分光学 (Partial Optics)**，用于聚焦可能不存在的值。它处理的是 "可能有，可能没有" 的场景。

```csharp
// Prism 的核心操作
public interface IPrism<S, A>
{
    Option<A> Get(S source);        // 尝试获取，返回 Option
    S Set(A value, S source);       // 如果目标存在则设置，否则返回原值
}
```

### 与 Lens 的核心区别

| 特性 | Lens | Prism |
|------|------|-------|
| 目标存在性 | 总是存在 | 可能不存在 |
| Get 返回值 | `A` | `Option<A>` |
| Set 行为 | 总是修改 | 仅当目标存在时修改 |
| 适用场景 | Product Types (记录) | Sum Types (联合类型) |

### 类比

- **Lens** 像一把精确的钥匙，总能打开特定的抽屉
- **Prism** 像一把可能匹配的钥匙，尝试打开，可能成功也可能失败

---

## 3. 使用场景

### 场景 1: 处理 Option 字段

```csharp
record User(string Name, Option<string> Email);

// 创建聚焦 Email 内部值的 Prism
var emailPrism = Prism<User, string>.New(
    get: user => user.Email,  // 返回 Option<string>
    set: (email, user) => user with { Email = Some(email) }
);

var user1 = new User("Alice", Some("alice@example.com"));
var user2 = new User("Bob", None);

// Get 操作
emailPrism.Get(user1);  // Some("alice@example.com")
emailPrism.Get(user2);  // None

// Set 操作 - 只有当原值存在时才更新
emailPrism.Set("new@example.com", user1);  // Email 被更新
emailPrism.Set("new@example.com", user2);  // 保持 None（不创建新值）
```

### 场景 2: 处理联合类型 (Either/Sum Types)

```csharp
// 定义一个联合类型
[Union]
public interface Shape
{
    Shape Circle(double Radius);
    Shape Rectangle(double Width, double Height);
}

// 创建聚焦 Circle 的 Prism
var circlePrism = Prism<Shape, double>.New(
    get: shape => shape switch
    {
        Circle c => Some(c.Radius),
        _ => None
    },
    set: (radius, shape) => shape switch
    {
        Circle _ => Circle(radius),
        _ => shape  // 非 Circle 保持不变
    }
);

var circle = Circle(5.0);
var rect = Rectangle(3.0, 4.0);

circlePrism.Get(circle);  // Some(5.0)
circlePrism.Get(rect);    // None

circlePrism.Set(10.0, circle);  // Circle(10.0)
circlePrism.Set(10.0, rect);    // Rectangle(3.0, 4.0) - 不变
```

### 场景 3: 处理集合元素

```csharp
// 聚焦列表第一个元素的 Prism
var firstPrism = Prism<Seq<int>, int>.New(
    get: seq => seq.HeadOrNone(),
    set: (value, seq) => seq.Match(
        Empty: () => seq,  // 空列表保持不变
        Cons: (_, tail) => value.Cons(tail)  // 替换头部
    )
);

var list1 = Seq(1, 2, 3);
var list2 = Seq<int>();

firstPrism.Get(list1);  // Some(1)
firstPrism.Get(list2);  // None

firstPrism.Set(100, list1);  // Seq(100, 2, 3)
firstPrism.Set(100, list2);  // Seq() - 空列表不变
```

---

## 4. Prism API

### 基本操作

```csharp
// 创建 Prism
var prism = Prism<S, A>.New(
    get: s => ...,   // S -> Option<A>
    set: (a, s) => ... // (A, S) -> S
);

// Get - 尝试获取值
Option<A> value = prism.Get(source);

// Set - 尝试设置值（仅当目标存在）
S newSource = prism.Set(newValue, source);
```

### 组合 Prism

```csharp
// Prism 可以与其他光学工具组合
var combined = outerPrism.Then(innerPrism);
var combined = lens.Then(prism);  // Lens + Prism = Prism
```

### Update 操作

```csharp
// 使用函数更新（如果值存在）
var updated = prism.Update(x => x * 2, source);

// 等价于
var updated = prism.Get(source)
    .Map(x => x * 2)
    .Match(
        Some: newVal => prism.Set(newVal, source),
        None: () => source
    );
```

---

## 5. Lens vs Prism 对比

### 代码对比

```csharp
// Lens - 总是能获取值
record Person(string Name, int Age);
var ageLens = Lens<Person, int>.New(
    get: p => p.Age,
    set: (age, p) => p with { Age = age }
);

int age = ageLens.Get(person);  // 总是返回 int

// Prism - 可能获取不到值
record Person2(string Name, Option<int> Age);
var agePrism = Prism<Person2, int>.New(
    get: p => p.Age,
    set: (age, p) => p with { Age = Some(age) }
);

Option<int> age = agePrism.Get(person2);  // 返回 Option<int>
```

### 选择指南

```
使用 Lens 当：
  - 字段总是存在
  - 处理 record/class 的直接字段
  - 需要保证 Get 总是成功

使用 Prism 当：
  - 字段可能不存在 (Option<T>)
  - 处理联合类型的特定分支
  - 聚焦集合中的元素
  - 需要优雅处理 "不存在" 的情况
```

---

## 6. 实际应用建议

### 建议 1: 与 Lens 配合使用

```csharp
// 复杂结构常常需要 Lens 和 Prism 配合
record Company(string Name, Option<Address> HeadOffice);
record Address(string City, string Street);

// Lens: Company -> Option<Address>
var headOfficeLens = Lens<Company, Option<Address>>.New(...);

// Prism: Option<Address> -> Address
var someAddressPrism = Prism<Option<Address>, Address>.New(
    get: opt => opt,
    set: (addr, _) => Some(addr)
);

// Lens: Address -> string
var cityLens = Lens<Address, string>.New(...);

// 组合: Company -> Option<string>
var headOfficeCityPrism = headOfficeLens
    .Then(someAddressPrism)
    .Then(cityLens);
```

### 建议 2: 为联合类型定义 Prism

```csharp
// 为每个分支定义一个 Prism
public static class ShapePrisms
{
    public static readonly Prism<Shape, double> CircleRadius = ...;
    public static readonly Prism<Shape, (double, double)> RectangleDimensions = ...;
}
```

### 建议 3: 使用 language-ext 内置的 Prism

language-ext 已经为常见类型提供了 Prism：

```csharp
// Option<T> 的 Prism
Option<int>.some  // Prism 聚焦 Some 值

// Either<L, R> 的 Prism
Either<string, int>.right  // Prism 聚焦 Right 值
Either<string, int>.left   // Prism 聚焦 Left 值
```

---

## 7. 总结

### Prism 的核心价值

1. **类型安全地处理部分数据**: 通过 `Option<T>` 明确表达 "可能不存在"
2. **与不可变数据完美配合**: 所有操作返回新值，不修改原数据
3. **可组合性**: 可以与 Lens、其他 Prism 组合，构建复杂的数据访问路径
4. **模式匹配的泛化**: Prism 是对模式匹配的抽象，使其可重用和可组合

### 光学工具家族

```
Optics (光学工具)
├── Lens      - 聚焦总是存在的部分 (Product Types)
├── Prism     - 聚焦可能存在的部分 (Sum Types)
├── Affine    - Lens + Prism 的组合
├── Traversal - 聚焦多个目标 (集合)
└── Iso       - 双向无损转换
```

### 一句话总结

> **Lens 是 "有" 的光学，Prism 是 "可能有" 的光学。**
> 当你的数据结构中包含 `Option`、`Either` 或其他表示可选性的类型时，Prism 是正确的工具。

---

*文档创建时间: 2024-12*
*适用于: language-ext v5.x*
