# 在 FP 中处理 UI Side Effect 的建议

在 `language-ext` 项目的 FP 实践中，处理 UI 相关的 Side Effect（副作用）确实是一个核心且常见的问题。基于该项目中的示例代码（特别是 `Samples/TestBed.WPF` 和 `Samples/BlazorApp`），我们可以总结出一套标准的处理模式。

其核心思想遵循 **"Imperative Shell, Functional Core" (指令式外壳，函数式核心)** 原则，通过 IO/Eff Monad 将副作用隔离在边界。

以下是处理 UI Side Effect 的几个标准建议和做法：

### 1. 将 UI 操作封装为 IO/Eff
不要直接在业务逻辑中操作 UI 控件。相反，应该将“读取”和“修改”UI 的操作封装在 `IO<A>` 或 `Eff<RT, A>` 中。

**标准做法：** 使用 `lift` 将副作用包装起来。

```csharp
// ❌ 错误做法：直接修改
void UpdateLabel(string text) {
    MyLabel.Content = text;
}

// ✅ 正确做法：封装为 Eff/IO
Eff<RT, Unit> SetContent(ContentControl control, string text) =>
    lift(() => control.Content = text);
```
*(参考自 `Samples/TestBed.WPF/WindowIO.cs`)*

### 2. 使用 "Bridge" 方法连接 UI 事件
UI 框架（如 WPF, WinForms, Blazor, MAUI）本质上是面向对象和事件驱动的。你需要一个“桥梁”将框架的事件（Imperative）转换为你的 FP 世界（Functional）。

**标准做法：** 在事件处理函数中调用 `.Run()` 执行副作用。

```csharp
// UI 事件处理器 (Imperative Shell)
void Button_Click(object sender, RoutedEventArgs e) =>
    Handle(OnButtonClick);

// 纯函数式描述 (Functional Core)
Eff<RT, Unit> OnButtonClick =>
    from _1 in ResetCount
    from _2 in SetContent(CounterLabel, "0")
    select unit;

// 辅助方法 (Bridge)
void Handle<A>(Eff<RT, A> effect) =>
    effect.Run(Runtime, Env).ThrowIfFail();
```
*(参考自 `Samples/TestBed.WPF/MainWindow.xaml.cs`)*

### 3. 使用 `Atom<T>` 管理可变状态
在 FP 中我们尽量避免变量，但 UI 组件通常需要持有状态。`language-ext` 提供了 `Atom<T>` 作为线程安全的、可观察的可变状态容器。

**标准做法：** 状态定义为 `Atom<T>`，修改操作封装在 IO 中。

```csharp
// 定义状态
readonly Atom<int> count = Atom(0);

// 修改状态的 IO 操作
IO<int> IncrementCount =>
    count.SwapIO(x => x + 1); // SwapIO 返回一个执行修改的 IO
```
*(参考自 `Samples/BlazorApp/Components/Pages/Counter.razor`)*

### 4. 高级交互：使用 Pipes (Streams)
对于连续的事件流（如鼠标移动、文本输入、WebSocket 消息），单纯的 IO 可能不够用。可以使用 `Pipes` 将事件转换为数据流。

**标准做法：** 创建一个 `Producer` 监听 UI 事件，然后用 `Consumer` 处理逻辑。

```csharp
// 定义一个鼠标移动的生产者
Producer<MouseEventArgs, Eff<RT>, Unit> OnMouseMove => ...

// 消费这个流并更新 UI
Consumer<MouseEventArgs, Eff<RT>, Unit> ShowMousePos =>
    from e in Proxy.awaiting<MouseEventArgs>()
    from _ in SetContent(CoordLabel, $"{e.GetPosition(this)}")
    select unit;
```
*(参考自 `Samples/TestBed.WPF/MainWindow.xaml.cs`)*

### 总结
处理 UI Side Effect 的标准流程是：
1.  **Wrap**: 用 `lift` 把所有 `control.Property = value` 包装成 `IO`/`Eff`。
2.  **Compose**: 使用 LINQ (`from ... select`) 组合这些小的 IO 操作和业务逻辑。
3.  **Run**: 仅在 UI 框架的 Event Handler 最外层调用 `.Run()`。

这样你的核心逻辑保持纯净（即它是对“要做什么”的描述，而不是“正在做”），使得代码更易于测试和推导。
