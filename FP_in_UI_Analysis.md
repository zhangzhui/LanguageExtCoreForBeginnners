这是一个非常深刻且普遍的观察。特别是在像 C# (WPF/WinForms) 或 Java (Swing/JavaFX) 这样的生态中，**你会有这种感觉，通常是因为"UI 框架本身是面向对象且基于副作用的"。**

当框架强制你通过 `button.Text = "Hello"`（副作用）来改变界面时，FP 确实显得有点"使不上劲"。

但如果你换个角度，**FP 在 UI 程序中的作用并不是"写算法"，而是"管理状态的变化"**。

以下是为什么你觉得"由于 UI 框架限制，FP 能写的部分很少"，以及 FP 实际上藏在哪里的深度解析：

### 1. 传统的 UI 写法掩盖了"业务逻辑"
在传统的 UI 开发中（MVC 或 MVVM），我们习惯把逻辑直接写在事件处理器里：
```csharp
// 典型的命令式 UI 代码
void OnButtonClick() {
    if (this.Input.Text == "") return; // 逻辑混合在 UI 控制中
    this.LoadingSpinner.Show();        // 副作用
    try {
        var data = await _service.GetData(); // 副作用
        this.ResultLabel.Text = data;        // 修改状态 + 更新 UI 混合
    } catch {
        this.ErrorLabel.Text = "Error";
    } finally {
        this.LoadingSpinner.Hide();
    }
}
```
在这个函数里，**状态变更、UI 更新、业务规则、副作用**全混在一起。如果你把 UI 刨出去，确实感觉"没剩啥了"。

### 2. FP 的视角：UI = f(State)
FP 试图将**状态的计算**与**状态的渲染**剥离。
如果用 FP (或者 React/Redux, Elm 架构) 的思路重写，你会发现"剩下的部分"其实很大，那就是**纯粹的状态机**。

在 FP 架构（如 Model-View-Update）中，你的程序被拆分为：

*   **Model (State)**: 这是不可变的数据结构。
*   **Update (Reducer)**: `(OldState, Event) -> NewState`。**这是一个纯函数**。
*   **View**: `State -> UI`。

**刨去 UI 后，剩下的就是那个 Update 函数。** 它负责：
1.  **输入验证**: 比如 `Input` 是否合法？（使用 `Validation<T>`）
2.  **状态转换**: 从 `Loading` 状态变为 `Success(Data)` 状态。
3.  **决策**: 下一步该发起什么副作用（但不执行它）。

### 3. "隐形"的复杂性：中间状态的管理
很多 UI Bug 都是因为状态管理不当造成的（比如：加载时按钮没禁用、报错后 Loading 圈没消失）。
FP 通过**代数数据类型 (ADT)** 来消除这些不可能的状态。

**命令式写法 (分散的 bool):**
```csharp
class ViewModel {
    bool IsLoading;
    string ErrorMessage;
    Data Result;
    // Bug 隐患：可能同时 IsLoading = true 且 ErrorMessage != null
}
```

**FP 写法 (联合类型):**
```csharp
// 这种定义本身就是核心逻辑的一部分
type UIState =
    | Idle
    | Loading
    | Success of Data
    | Failure of string
```
当你在写如何处理这些状态转换的代码时，你就是在写纯 FP 代码，而这正是 UI 程序的核心健壮性所在。

### 4. 响应式编程 (FRP)
UI 本质上是异步的（等待点击、等待网络）。
FP 处理这种场景有强大的工具：**Observable (Rx)**。
如果你把点击看作"事件流 (Stream)"，把输入框看作"数据流"，通过 `Map`, `Filter`, `CombineLatest` 来组合它们，那么**整个 UI 的交互逻辑就变成了一段声明式的 FP 代码**，而不是散落在各处的 `void OnClick`。

### 总结
你感觉"能写的部分很少"，通常是因为：
1.  **业务逻辑太简单**：如果是纯 CRUD 程序，确实没多少逻辑。
2.  **框架限制**：WPF/WinForms 逼着你用 Mutation，导致你很难把"逻辑"从"修改属性"中剥离出来。
3.  **习惯问题**：我们习惯把"做什么 (逻辑)"和"怎么显示 (UI)"写在一起。

**一旦你开始尝试"MVU (Model-View-Update)"架构，或者使用 Rx，你会发现 90% 的代码（状态管理、事件流转、数据验证）都可以变成 FP，只有最后 10% 负责把数据"画"到屏幕上的代码是命令式的。**