# 函数式程序设计的核心原则

## 1. **不可变性优先 (Immutability First)**

```csharp
// ❌ 错误：可变状态
class User {
    public string Name { get; set; }  // 可变
}

// ✅ 正确：不可变数据
record User(string Name, string Email);  // 或使用 language-ext 的 Record<T>
```

**为什么重要**：不可变数据消除了并发问题，让代码更容易推理。

---

## 2. **用类型表达意图 (Make Illegal States Unrepresentable)**

```csharp
// ❌ 错误：用 null 或 string 表示可能缺失的值
string? GetUserEmail(int id);  // null 是什么意思？没找到？出错了？

// ✅ 正确：用 Option 明确表达"可能没有"
Option<string> GetUserEmail(int id);

// ✅ 更好：用 Either/Fin 表达"可能失败"
Fin<User> GetUser(int id);  // 明确告诉调用者：这可能失败
```

---

## 3. **将副作用推到边缘 (Push Effects to the Edge)**

```csharp
// ❌ 错误：业务逻辑中混杂 IO
decimal CalculateDiscount(Order order) {
    var config = File.ReadAllText("config.json");  // IO 在中间！
    var rate = JsonParse(config).DiscountRate;
    return order.Total * rate;
}

// ✅ 正确：纯函数 + 依赖注入
decimal CalculateDiscount(Order order, decimal discountRate)
    => order.Total * discountRate;

// 或使用 Effect 系统
Eff<decimal> CalculateDiscount(Order order) =>
    from config <- readConfig()
    select order.Total * config.DiscountRate;
```

---

## 4. **组合优于继承 (Composition over Inheritance)**

```csharp
// ❌ 错误：复杂的继承层次
abstract class Animal { }
class Dog : Animal { }
class ServiceDog : Dog { }  // 继承爆炸

// ✅ 正确：用小函数组合
var processOrder =
    ValidateOrder
    .Compose(ApplyDiscount)
    .Compose(CalculateTax)
    .Compose(SaveOrder);
```

---

## 5. **使用 Railway Oriented Programming**

把操作链接成管道，错误自动传播：

```csharp
// 每一步都可能失败，但代码清晰流畅
Fin<OrderConfirmation> ProcessOrder(OrderRequest request) =>
    from validated  in ValidateOrder(request)
    from inventory  in CheckInventory(validated)
    from payment    in ProcessPayment(inventory)
    from confirmed  in ConfirmOrder(payment)
    select confirmed;

// 错误在任何一步发生都会短路，不需要 try-catch 或 if-null 检查
```

---

## 6. **模式匹配代替条件判断**

```csharp
// ❌ 错误：嵌套 if-else
if (result != null) {
    if (result.IsSuccess) {
        // ...
    } else {
        // ...
    }
}

// ✅ 正确：模式匹配
result.Match(
    Succ: value => HandleSuccess(value),
    Fail: error => HandleError(error)
);
```

---

## 实用技巧清单

| 技巧 | 说明 |
|------|------|
| **从数据开始设计** | 先定义不可变数据类型，再设计操作它们的函数 |
| **小函数** | 每个函数只做一件事，便于组合和测试 |
| **无副作用** | 纯函数：相同输入永远返回相同输出 |
| **显式依赖** | 所有依赖通过参数传入，不使用全局状态 |
| **用类型替代注释** | `NonEmptyString` 比 `string // must not be empty` 更好 |
| **延迟执行** | 使用 `Eff<T>` 或 `IO<T>` 来`描述`操作，而不是立即执行 |

---

## 一个简单的设计流程

```
1. 定义领域类型 (Domain Types)
   └─ record, union types, newtypes

2. 定义纯业务逻辑 (Pure Functions)
   └─ 输入 → 输出，无副作用

3. 定义效果 (Effects)
   └─ IO 操作用 Eff/IO 包装

4. 在边缘组合 (Composition at Edge)
   └─ Main/Controller 中组合并执行
```

---

## 最重要的一条

> **如果你只记住一件事**：把"做什么"和"怎么做"分开。
>
> 纯函数描述"做什么"，Effect 系统处理"怎么做"。
