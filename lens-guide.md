## Lens 详解 - 函数式编程中的"透镜"

### 一、背景：为什么需要 Lens？

在函数式编程中，我们追求**不可变数据结构**(immutable data)。但这带来一个问题：当你有嵌套很深的数据结构时，如何优雅地更新其中的某个值？

**传统方式的痛点：**

```csharp
// 假设有这样的嵌套结构
var book = new Book(
    "Editors of the World",
    "Sarah Bloggs",
    new Editor(
        "Joe Bloggs",
        50000,
        new Car("Maserati", "Ghibli", 20000)
    )
);

// 要更新 book.Editor.Car.Mileage，传统方式需要：
var newBook = book with {
    Editor = book.Editor with {
        Car = book.Editor.Car with {
            Mileage = 25000
        }
    }
};
// 嵌套层次越深，代码越臃肿！
```

**Lens 的解决方案：**
Lens 是一种**可组合的双向访问器**，可以：
1. **Get**: 从整体中获取部分
2. **Set**: 在整体中设置部分（返回新的整体）

---

### 二、Lens 的核心定义

language-ext 中 Lens 的定义位于 `LanguageExt.Core/Lens/LensAB.cs:8`：

```csharp
public readonly struct Lens<A, B>
{
    public readonly Func<A, B> Get;           // A -> B (获取器)
    public readonly Func<B, Func<A, A>> SetF; // B -> A -> A (设置器，柯里化形式)

    public A Set(B value, A cont) =>
        SetF(value)(cont);

    public static Lens<A, B> New(Func<A, B> Get, Func<B, Func<A, A>> Set) =>
        new (Get, Set);

    // 使用函数更新值
    public Func<A, A> Update(Func<B, B> f)
    {
        var self = this;
        return a => self.Set(f(self.Get(a)), a);
    }

    public A Update(Func<B, B> f, A value) =>
        Set(f(Get(value)), value);
}
```

**关键点：**
- `Lens<A, B>` 表示从类型 `A` 中"聚焦"到类型 `B` 的镜头
- `Get`: 从 A 中提取 B
- `Set`: 给定新的 B 值和原来的 A，返回新的 A

---

### 三、基本用法

#### 1. 创建一个 Lens

```csharp
// 为 Person 的 Name 属性创建 Lens
public record Person(string Name, int Age);

var nameLens = Lens<Person, string>.New(
    Get: p => p.Name,
    Set: newName => p => p with { Name = newName }
);
```

#### 2. 使用 Lens

```csharp
var person = new Person("Alice", 30);

// Get: 获取值
string name = nameLens.Get(person);  // "Alice"

// Set: 设置值（返回新对象）
var person2 = nameLens.Set("Bob", person);  // Person("Bob", 30)

// Update: 使用函数更新
var person3 = nameLens.Update(n => n.ToUpper(), person);  // Person("ALICE", 30)
```

---

### 四、Lens 的组合 - 核心威力

**Lens 最强大的特性是可组合性！** 可以将多个 Lens 组合起来访问深层嵌套的属性。

#### 方式一：使用 `lens()` 函数组合

```csharp
// 假设有自动生成的 Lens
// Book.editor: Lens<Book, Editor>
// Editor.car: Lens<Editor, Car>
// Car.mileage: Lens<Car, int>

// 组合成一个 Lens，直接访问 book -> mileage
var bookEditorCarMileage = lens(Book.editor, Editor.car, Car.mileage);

// 现在可以直接操作！
var mileage = bookEditorCarMileage.Get(book);           // 20000
var newBook = bookEditorCarMileage.Set(25000, book);    // 更新里程
```

#### 方式二：使用 `|` 操作符组合

在 `Lens.Operators.cs:7` 中定义了组合操作符：

```csharp
// 使用管道符组合
var combinedLens = Book.editor | Editor.car | Car.mileage;
```

---

### 五、内置的 Lens 工具函数

language-ext 提供了一些常用的 Lens，位于 `Lens.cs`:

| 函数 | 用途 |
|------|------|
| `Lens.fst<A,B>()` | 获取/设置元组的第一个元素 |
| `Lens.snd<A,B>()` | 获取/设置元组的第二个元素 |
| `Lens.thrd<A,B,C>()` | 获取/设置元组的第三个元素 |
| `Lens.identity<A>()` | 恒等 Lens (Get=id, Set=const) |
| `Lens.tuple(...)` | 组合多个 Lens 作用于元组 |
| `Lens.cond(pred, then, else)` | 条件 Lens |
| `Lens.enumMap(lens)` | 将 Lens 映射到集合上 |

**示例：元组操作**

```csharp
var pair = (10, "hello");

var fstLens = Lens.fst<int, string>();
var sndLens = Lens.snd<int, string>();

int first = fstLens.Get(pair);           // 10
var pair2 = sndLens.Set("world", pair);  // (10, "world")
```

---

### 六、与集合配合使用

测试文件 `LensTests.cs:30` 展示了与 `Map` 配合的用法：

```csharp
// Map<K, V>.item(key) 返回 Lens<Map<K,V>, V>
Lens<Person, ApptState> arrive(int id) =>
    lens(Person.appts, Map<int, Appt>.item(id), Appt.state);

// 使用
var person2 = arrive(2).Set(ApptState.Arrived, person);
// 这等价于：person.Appts[2].State = Arrived，但是不可变的！
```

---

### 七、Lens 与 Prism

**Prism** 是 Lens 的扩展，处理**可能不存在**的情况（返回 `Option<B>` 而不是 `B`）。

```csharp
// Lens: Get 总是成功
Lens<A, B>  →  Get: A → B

// Prism: Get 可能失败
Prism<A, B> →  Get: A → Option<B>
```

**Lens 可以转换为 Prism：**

```csharp
Prism<A, B> prism = lens.ToPrism();
```

---

### 八、实际应用场景

#### 场景1：Redux-like 状态管理

```csharp
record AppState(UserState User, CartState Cart);
record UserState(string Name, bool LoggedIn);

var userNameLens = lens(AppState.user, UserState.name);

// 纯函数式更新
AppState Reducer(AppState state, Action action) => action switch
{
    SetUserName a => userNameLens.Set(a.Name, state),
    _ => state
};
```

#### 场景2：配置更新

```csharp
record Config(DatabaseConfig Db, LoggingConfig Log);
record DatabaseConfig(string Host, int Port, int Timeout);

var dbTimeoutLens = lens(Config.db, DatabaseConfig.timeout);

// 轻松更新深层配置
var newConfig = dbTimeoutLens.Update(t => t * 2, config);
```

#### 场景3：测试数据构造

```csharp
// 从基础测试数据派生变体
var baseUser = CreateDefaultUser();
var userWithHighBalance = balanceLens.Set(10000, baseUser);
var userWithLowBalance = balanceLens.Set(10, baseUser);
```

---

### 九、总结

| 特性 | 说明 |
|------|------|
| **不可变性** | Lens 不修改原数据，返回新副本 |
| **可组合性** | 多个 Lens 可以组合访问深层属性 |
| **类型安全** | 编译时检查，避免运行时错误 |
| **双向性** | 既能 Get 也能 Set |
| **函数式** | 与其他 FP 概念（如 Map/Bind）配合良好 |

**Lens 本质上是将"属性访问路径"抽象为一等公民（first-class value）**，使得路径本身可以被传递、组合、复用，极大提升了操作不可变数据结构的人体工程学。
