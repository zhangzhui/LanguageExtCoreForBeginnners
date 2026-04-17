在 `LanguageExt` 的设计中，`Schedule` 被定义为一个"**时间间隔（Duration）的流**"（a potentially infinite stream of durations）。它提供了一整套非常强大和灵活的函数式 DSL，用于控制重试（Retry）、重复（Repeat）或时间折叠（Fold）的调度策略。

通过查看源码，`Schedule` 支持各种基础策略以及多种极其强大的组合和修饰器。我们可以将它们大致分为四大类：

### 1. 基础间隔与次数策略
这些是最基础的调度流，用于设定次数或明确的时间间隔。
*   **`Schedule.Never`**：从不执行。
*   **`Schedule.Once`**：仅执行一次（等同于执行次数为 1）。
*   **`Schedule.Forever`**：无限次执行，不带任何时间延迟（立即触发）。
*   **`Schedule.recurs(int times)`**：限制执行或重试的**最大次数**（如重试 3 次就停止）。
*   **`Schedule.spaced(Duration space)`**：每次执行之间保持**固定的间隔时间**（如每隔 5 秒执行一次）。
*   **`Schedule.fixedInterval(Duration interval)`**：固定周期执行。与 `spaced` 不同，如果 IO 动作执行本身耗时，`fixedInterval` 会尽力"扣除"消耗的时间，让每次触发都落在固定的时钟频率上。
*   **`Schedule.TimeSeries(params Duration[] durations)`**：通过**自定义的一系列间隔**组成策略（例如给定延迟数组 `1s, 5s, 10s`，执行三次分别对应各自的延迟）。

### 2. 退避策略（Backoff）
主要用于出错重试时的容错机制，避免给下游服务或网络带来持续的高压。
*   **`Schedule.linear(Duration seed, double factor = 1)`**：**线性退避**。时间间隔呈线性增长，比如初始 `1s`，随次数逐渐变为 `2s`, `3s`, `4s`...
*   **`Schedule.exponential(Duration seed, double factor = 2)`**：**指数退避**。最经典的重试机制，每次间隔成倍（factor）增长，例如初始 `1s`，之后是 `2s`, `4s`, `8s`...
*   **`Schedule.fibonacci(Duration seed)`**：**斐波那契退避**。按斐波那契数列增长（例如 `1s, 1s, 2s, 3s, 5s, 8s`）。通常比指数退避稍微温和一些。

### 3. 基于真实时钟/日历策略 (Cron-like)
这些调度允许程序直接与自然时间（或 Cron 表达式概念）对齐，主要用于定时任务。
*   **`Schedule.windowed(Duration interval)`**：划分时间窗口，每次执行后会**睡眠到下一个时间窗口的边界**（例如每 10 秒一个窗口，动作在第 3 秒完成，就会睡眠 7 秒等待下一个整 10 秒）。
*   **`Schedule.secondOfMinute(int second)`**：在每分钟指定的**第 N 秒**触发。
*   **`Schedule.minuteOfHour(int minute)`**：在每小时的**第 N 分钟**触发。
*   **`Schedule.hourOfDay(int hour)`**：在每天的**第 N 小时**触发。
*   **`Schedule.dayOfWeek(DayOfWeek day)`**：在每周的**星期几**触发。

### 4. 强大的组合子（Modifiers & Combinators）
由于 `Schedule` 在概念上实现了 `Semigroup`，并大量运用了函数式编程设计（类似于 RxJS），所以能够通过操作符或装饰器把上面的基础调度组合成非常复杂的策略。

#### 限制条件 (Limits)
*   **`Schedule.upto(Duration max)`**：设置调度的**最长存活时间**（例如，无论重试几次，整体花费时间不能超过 1 分钟）。
*   **`Schedule.maxDelay(Duration max)`**：设置单次退避延迟的**上限**（例如结合指数退避使用，每次间隔加倍，但最大不超过 30 秒）。
*   **`Schedule.maxCumulativeDelay(Duration max)`**：当所有延迟累加超过一个阈值后终止。

#### 抖动与随机分布 (Jitter)
在大规模分布式系统中，如果多个节点同时发生故障并同时触发重试（比如使用相同的指数退避），会导致"惊群效应（Thundering Herd）"。抖动用于打散并发请求：
*   **`Schedule.jitter(...)`**：为计算出的延迟时间增加随机性偏移（Jitter）。例如让本应是 5 秒的延迟变成 `4.5s ~ 5.5s` 之间的随机值。
*   **`Schedule.decorrelate(...)`**：在时间线上进行去相关性处理，让并行的调度在时间轴上产生错位。

#### 组合操作符 (`|` 和 `&`)
*   **交集 (`&`)**：`A & B` 会取两者的 **Max** 延迟，并在较短的 schedule 结束时停止。例如 `Schedule.exponential(10*ms) & Schedule.spaced(300*ms)` 代表：以指数退避，但如果指数退避算出来的间隔**小于** 300ms，就强制等待 300ms（变相的底线限制）。
*   **并集 (`|`)**：`A | B` 会取两者的 **Min** 延迟，并会在较长的 schedule 结束时停止。例如 `Schedule.recurs(5) | Schedule.exponential(10*ms)` 代表最多重试 5 次，并在每次重试之间按指数间隔等待。

### 总结范例

有了以上类型，在 `LanguageExt` 中要写出 "**尝试执行一个操作，最多重试 5 次；采用从 100ms 起步的指数退避，最大不超过 5秒 间隔；为了防雪崩加入随机抖动；如果总时长超过 30 秒直接放弃**" 的严谨策略，只需要简短地组合：

```csharp
var mySchedule = Schedule.recurs(5)
                 | Schedule.exponential(100 * ms)
                 | Schedule.maxDelay(5 * seconds)
                 | Schedule.jitter(0.5)
                 | Schedule.upto(30 * seconds);

// 使用它折叠或者重试IO
myEffect.RetryIO(mySchedule);
```
