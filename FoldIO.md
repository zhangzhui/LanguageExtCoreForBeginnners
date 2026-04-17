`FoldIO` 是 LanguageExt v5（及其 IO/Eff 体系）中一个非常强大的机制。它将**副作用的重复执行**和**状态折叠（聚合）**优雅地组合在了一起。

以下是关于 `FoldIO` 的原理、使用场景以及具体代码示例的详细介绍。

### 1. 原理
在函数式编程中，传统的 `Fold`（或 LINQ 里的 `Aggregate`）是用来遍历集合并将元素逐步"折叠/累加"成一个最终状态的。
而 `FoldIO` 则是针对**副作用（IO 效应）**的折叠：
- 它接受一个单体 IO 操作（比如拉取一次 API、读取一次控制台、生成一个随机数等）。
- 它接受一个调度策略 `Schedule`（控制执行的次数、间隔、回退等）。
- 每次 IO 操作成功返回值 `A` 后，它会调用指定的 `folder` 函数：`S -> A -> S`，将新产生的值合并到当前积累的状态 `S` 中。
- 随后它会根据 `Schedule` 的时间间隔等待，然后再次执行 IO，进行下一轮状态累加。
- 当 `Schedule` 耗尽（或通过 `FoldUntilIO` / `FoldWhileIO` 触发了终止条件）时，它返回整个生命周期中计算出的最终状态 `S`。

**简而言之：`FoldIO = 重复执行同一个 IO + 收集每次执行的结果`**。

### 2. 使用场景
`FoldIO` 以及它的衍生函数 `FoldWhileIO` / `FoldUntilIO` 非常适合以下场景：

* **轮询并累加数据**：你需要定时或基于特定条件向外部接口请求数据。每次请求你都会拿到一小批数据，你希望不断重试直到抓取完全部批次，最后合并成一个大集合。
* **带状态监控的周期性任务**：例如每 5 秒从传感器或消息队列读取一条数据，并计算这些数据的移动平均值。当发生某种条件时停止读取并返回平均值。
* **交互式读取循环**：在控制台/后台服务中不断地接收用户/客户端输入，将其记录在状态字典或列表里，直到接收到 "退出" 指令为止。

### 3. 具体例子

以下给出三个逐步深入的例子，展示如何在实际中使用。

#### 例子 1：按调度策略重复执行并求和
这是一个最基础的示例。我们定义了一个模拟的掷骰子 IO，然后通过 `FoldIO` 让它执行 5 次，将每次的结果相加。

```csharp
using System;
using LanguageExt;
using static LanguageExt.Prelude;

public class FoldIOExamples
{
    public static void Example1_Sum()
    {
        // 1. 定义一个纯粹的 IO 操作描述（只有在 Run 时才会执行）
        var rollDice = IO.lift(() => new Random().Next(1, 7));

        // 2. 将其在指定调度下执行并进行折叠累加
        // Schedule.Recurs(4) 意味着除了最初的 1 次执行外，还会重复执行 4 次（共执行 5 次）
        var sumIO = rollDice.FoldIO(
            Schedule.Recurs(4),                 // 调度策略
            initialState: 0,                    // 初始状态为 0
            folder: (sum, dice) => sum + dice   // 将每次的骰子点数累加
        );

        // 3. 真正执行整个流程
        int finalSum = sumIO.Run();
        Console.WriteLine($"掷骰子 5 次的总和是: {finalSum}");
    }
}
```

#### 例子 2：定时采集数据到集合中
如果我们需要每隔固定的时间采集一次数据，可以使用时间相关的 `Schedule`，并在 `folder` 中将单次数据合并进一个不可变集合（如 `Seq<T>`）中。

```csharp
    public static void Example2_Collect()
    {
        var readSensor = IO.lift(() =>
        {
            Console.WriteLine($"[{DateTime.Now:HH:mm:ss}] 正在读取传感器数据...");
            return Math.Round(new Random().NextDouble() * 100, 2);
        });

        // 调度策略：每隔 1 秒执行一次，直到重复 2 次（总共执行 3 次）
        var schedule = Schedule.Spaced(TimeSpan.FromSeconds(1)) | Schedule.Recurs(2);

        var collectDataIO = readSensor.FoldIO(
            schedule,
            initialState: Seq<double>(),          // 初始状态为空的不可变序列
            folder: (seq, value) => seq.Add(value)// 每次获取的值追加到集合后
        );

        Seq<double> results = collectDataIO.Run();
        Console.WriteLine($"传感器数据收集完毕: {string.Join(", ", results)}");
    }
```

#### 例子 3：轮询交互直到特定条件触发 (`FoldUntilIO`)
有时我们不知道具体要循环多少次，我们需要基于**拉取到的值**或者**累加的状态**来决定何时停止。这时候可以用 `FoldUntilIO` 或 `FoldWhileIO`。

```csharp
    public static void Example3_Interactive()
    {
        var readInput = IO.lift(() =>
        {
            Console.Write("请输入单词 (输入 exit 退出): ");
            return Console.ReadLine() ?? "";
        });

        // 不断读取用户的输入并拼接成句子
        // 当用户输入的内容转换为小写等于 "exit" 时，循环提前终止
        var sentenceIO = readInput.FoldUntilIO(
            Schedule.Forever,                             // 理论上支持无限次重复
            initialState: "",                             // 初始句子为空
            folder: (sentence, word) => sentence + word + " ",
            valueIs: word => word.ToLower() == "exit"     // 当单次 IO 产生的值满足此条件时停止
        );

        string sentence = sentenceIO.Run();
        // 如果最后一次输入是 "exit"，这个词也会被传递给 valueIs 校验并导致结束，
        // 具体 "exit" 是否会被折叠到 state 里取决于底层实现，
        // 在 LanguageExt 的设计中，触发终止条件的那次 value 也会参与到当前的 folding 计算中。
        Console.WriteLine($"你拼接的句子是: {sentence}");
    }
```
