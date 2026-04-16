# 什么是"副作用"，为什么 `Iter` 是专门用来执行副作用的

在函数式编程（FP）中，“副作用”（Side Effect）是一个极其重要的核心概念。要理解为什么 Iter 叫“执行副作用”，我们需要先明白什么是“纯函数”，什么是“副作用”。

---

## 一、什么是"副作用"（Side Effect）

在 FP 中，一个**纯函数（Pure Function）**是指：
- 给定相同的输入，`永远`返回相同的输出。
- `不`与外部世界发生任何交互（`不`修改外部变量、`不`读写文件、`不`访问数据库、`不`打印控制台日志、`不`发送网络请求）。

相反，如果一个函数在计算结果之外，还对系统状态产生了影响，或者与外部世界进行了交互，这种行为就被称为副作用（Side Effect）。

在函数式编程中，**副作用**指的是：

> 一个操作**除了返回值之外**，还对"外部世界"产生了影响。

常见的副作用包括：

| 副作用类型 | 举例 |
|---|---|
| 打印到控制台 | `Console.WriteLine(...)` |
| 写入文件 / 数据库 | `File.WriteAllText(...)` |
| 发送网络请求 | `HttpClient.PostAsync(...)` |
| 修改外部变量 | `externalList.Add(...)` |
| 触发 UI 更新 | `button.Text = "..."` |

**纯函数**（Pure Function）是没有副作用的：给定相同输入，永远返回相同输出，不改变任何外部状态。

---

## 二. 为什么 Iter 是专门用来执行副作用的？

在 language-ext（或类似的 FP 库）中，像 Map、Bind、Filter 这样的函数，设计初衷都是纯粹的数据转换。它们接收输入，计算后包装在一个新的容器中返回。

如果你在 Map 里面写了数据库保存操作，虽然代码能跑，但`破坏了`函数式编程“数据流转”与“状态改变”分离的`原则`。

- 错误示范（在map中写副作用）

```csharp
    Option<User> userOpt = GetUser();

    // 不推荐：Map 应该只做转换，但这里却写了数据库，且返回了毫无意义的 Unit
    Option<Unit> result = userOpt.Map(user =>
    {
        db.Save(user); // 产生了副作用
        return unit;
    });

```
    
- 正确示范（使用Iter隔离副作用）

```csharp
    Option<User> userOpt = GetUser();

    // 推荐：明确告诉阅读代码的人，“我要在这里执行副作用了”
    userOpt.Iter(user =>
    {
        db.Save(user); // 将 User 存入数据库
        Console.WriteLine($"Saved {user.Name}"); // 打印日志
    });
```


Iter（在有些语言中叫 ForEach 或 Do）是专门为你提供的一个“合法出口”，让你在不改变容器内数据流向的情况下，去影响外部世界。

它的特征非常明显：
- 它没有返回值（在 C# 中通常返回 Unit，等同于 void）。
- 既然它没有返回值，调用它就肯定不是为了“计算某个结果”，而是为了“做某件事情”（即副作用）。

---


## 三、`Map` vs `Iter`：最关键的区别

### `Map` —— 有返回值，不执行副作用

```csharp
// Map：把 Option<int> 变成 Option<string>，有返回值
Option<int> opt = Some(42);
Option<string> result = opt.Map(x => x.ToString()); // result = Some("42")
```

`Map` 的 lambda 必须**返回一个新值**，整个表达式也有返回值。

---

### `Iter` —— 无返回值，专门执行副作用

```csharp
// Iter：只是"对里面的值做点什么"，不关心返回值
Option<int> opt = Some(42);
opt.Iter(x => Console.WriteLine($"值是: {x}")); // 打印：值是: 42
```

`Iter` 的 lambda 返回 `void`（或 `Unit`），整个表达式也返回 `Unit`。

---

## 三、代码对比：`Map` vs `Iter`

```csharp
Option<string> name = Some("Alice");

// ✅ 用 Map：转换值，有返回值
Option<string> upper = name.Map(n => n.ToUpper());
// upper = Some("ALICE")

// ✅ 用 Iter：执行副作用，无返回值
name.Iter(n => Console.WriteLine($"Hello, {n}!"));
// 打印：Hello, Alice!

// ❌ 错误用法：用 Map 来打印（虽然能运行，但语义错误）
// name.Map(n => { Console.WriteLine(n); return n; }); // 不推荐！
```

### 当值不存在时（`None`），两者都安全跳过：

```csharp
Option<string> noName = None;

noName.Map(n => n.ToUpper());  // 什么都不做，返回 None
noName.Iter(n => Console.WriteLine(n)); // 什么都不做，不会报错
```

---

## 四、为什么叫 `Iter`？

`Iter` 是 **Iterate（迭代）** 的缩写。
  - 对于一个集合（如 Seq<T> 或 List<T>），Iter 意味着“遍历”里面的每一个元素，并对它们依次执行某个动作（类似于 foreach 循环）。
  - 对于 Option<T>，在函数式思维中，Option 可以被看作是一个“最多只能装 1 个元素的集合”。如果是 Some，它就迭代这 1 个元素；如果是 None，它就迭代 0 个元素（什么都不做）。

其思想来源于函数式语言（如 F#、Haskell）：

- 把 `Option` / `List` 等容器"迭代一遍"
- 对每个元素执行一个**带副作用的操作**
- 不收集、不转换，只是"走一遍，做点事"

### F# 中的对应函数：

```fsharp
// F# 中 Option.iter
let name = Some "Alice"
name |> Option.iter (fun n -> printfn "Hello, %s!" n)
// 打印：Hello, Alice!
```

`LanguageExt` 的 `Iter` 正是从 F# 的这一惯例继承而来。

---

## 五、一句话总结

| 方法 | 用途 | Lambda 返回值 | 整体返回值 |
|---|---|---|---|
| `Map` | **转换**容器内的值 | 新值（`T → R`） | `Option<R>` |
| `Iter` | **执行副作用**（打印、写日志等） | 无（`T → void`） | `Unit` |

> **`Iter` 就是：我不想改变这个值，我只想对它"做点什么"。**

>叫它“执行副作用”，是因为 Iter 接收的是一个 Action<T>（只做动作，不求回报）。它宣告着纯粹的数学计算（Map/Bind）在此暂停，程序开始正式干预外部世界（打印、存储、发送）。
