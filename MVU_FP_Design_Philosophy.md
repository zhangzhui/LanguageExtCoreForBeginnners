# MVU (Model-View-Update) 与 FP (函数式编程) 设计理念

MVU (Model-View-Update) 架构结合 FP (函数式编程) 是现代前端和 GUI 开发中非常流行且强大的组合（比如 React 的 Redux 模式, Elm 语言, 以及 C# 的 Elmish）。

为了让你“通俗易懂”且“方便记忆”，我们可以用一个核心隐喻贯穿始终：**老式胶片电影（或定格动画）**。

## 1. 核心公式 (记这一个就够了)

在 MVU 中，整个世界只有两个公式：

1.  **渲染公式：** `UI = f(State)`
    *   (界面 只是 状态 的投影)
2.  **更新公式：** `NewState = f(CurrentState, Message)`
    *   (新状态 是 旧状态加上一个消息 算出来的)

## 2. 设计理念：从“操控木偶”到“播放电影”

### 传统思维 (MVC / MVVM / 面向对象)：操控木偶
以前我们写代码像是在**操控木偶**。
*   你需要记住木偶现在的姿势（State分散在各个控件里）。
*   当用户点击按钮，你手动去**修改**木偶的手臂位置（`button.Text = "Clicked"`）。
*   **痛点：** 如果操作多了，你很容易忘记木偶现在的姿势，导致胳膊拧大腿，状态不同步（Bug 的温床）。

### MVU + FP 思维：播放胶片电影
现在我们是在**制作定格动画**。
*   **不可变 (Immutability)：** 每一帧画面（Model/State）拍好后就**不能改**了。
*   **纯函数 (Pure Function)：** 如果要动一下手，我们不是去改上一帧，而是**根据上一帧产生一张全新的下一帧**。
*   **单向流 (Unidirectional Flow)：** 永远是“生成新帧 -> 投影到大银幕(View) -> 观众反应(Msg) -> 决定下一帧怎么拍(Update)”。

## 3. 需要做的三个思维转变

| 维度 | 传统思维 (OOP/Imperative) | MVU + FP 思维 | 通俗解释 |
| :--- | :--- | :--- | :--- |
| **状态** | **变量 (Variable)** <br> `x = x + 1` | **快照 (Snapshot)** <br> `newX = oldX + 1` | 以前是把杯子里的水换掉；现在是拿出一个新杯子装新水，旧杯子不动。 |
| **逻辑** | **命令 (Command)** <br> "去把那个按钮禁用掉！" | **声明 (Declarative)** <br> "如果状态是加载中，按钮就是禁用的。" | 以前是“指手画脚”；现在是“定义规则”。 |
| **副作用** | **随处发生** <br> 在点击事件里直接发网络请求。 | **受控边缘** <br> 发请求只产生一个“命令描述”，由运行时去执行。 | 以前是想干嘛就干嘛；现在是“填写申请单”，由系统统一处理。 |

## 4. 一个通俗易懂的例子：计数器

假设我们要写一个点击 “+” 号数字加 1 的程序。

### 第一步：Model (定格画面)
首先，定义世界长什么样。它只是数据，没有行为。
```csharp
// 这里的 record 意味着它是不可变的，就像一张洗出来的照片
record Model(int Count, bool IsLoading);
```

### 第二步：Msg (观众的反馈)
定义世界上可能发生什么事情。这只是一个信号。
```csharp
// 这是一个枚举，列出所有可能的动作
enum Msg
{
    Increment,      // 用户点了 +
    Decrement,      // 用户点了 -
    Reset           // 用户点了重置
}
```

### 第三步：Update (剪辑师)
这是最核心的 FP 部分。**输入旧状态和消息，输出新状态。**
注意：这里**没有**修改任何东西，只是算出了新东西。
```csharp
Model Update(Model current, Msg message)
{
    return message switch
    {
        // 收到加号消息 -> 拿旧的 Count + 1 -> 返回一张新照片(New Model)
        Msg.Increment => current with { Count = current.Count + 1 },

        // 收到减号消息 -> 拿旧的 Count - 1 -> 返回一张新照片
        Msg.Decrement => current with { Count = current.Count - 1 },

        // 收到重置 -> 返回全新的初始照片
        Msg.Reset => new Model(0, false)
    };
}
```

### 第四步：View (投影仪)
根据当前的照片（Model），把 UI 画出来。
```csharp
void View(Model model)
{
    // 完全根据传入的 model 决定显示什么
    // 不存在 "if (button.Text == ...)" 这种判断
    ShowLabel(model.Count.ToString());

    if (model.IsLoading)
        ShowSpinner();
}
```

## 5. 总结

**MVU 就是把写程序变成了做定格动画：不修改旧画面，永远只根据上一帧和剧本(Msg)产生下一帧。**
