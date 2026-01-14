# 什么样的 Monad 需要 Run 接口?

通常来说,**代表计算、动作或函数**的 Monad 需要 `Run` 接口(或者叫 `Eval`、`Execute`)。

与之相对的是代表**数据容器**的 Monad(如 `Option`, `List`, `Either`),它们通常不需要 `Run`,因为数据已经在里面了。

需要 `Run` 接口的 Monad 通常具有**惰性(Lazy)**和**依赖上下文**的特性。它们本质上是对**函数**的封装。

以下是几类典型的需要 `Run` 的 Monad:

### 1. 依赖环境的 Monad (Reader)
*   **本质**:`Func<Environment, T>` (函数)
*   **为什么需要 Run**:因为它是一个等待参数的函数。如果你不给它"环境"(Environment),它就无法计算出结果。
*   **例子**:
    ```csharp
    // 定义逻辑
    var computation = from c in Reader.ask<Config>() select c.Port;
    // 必须 Run 传入 Config 才能得到 int
    int port = computation.Run(new Config(8080));
    ```

### 2. 依赖状态的 Monad (State)
*   **本质**:`Func<State, (Result, State)>` (状态机函数)
*   **为什么需要 Run**:它描述的是状态的**变更过程**。你需要给它一个"初始状态",它才能开始运转,并吐出"结果"和"新状态"。
*   **例子**:
    ```csharp
    // 定义状态变更逻辑
    var counter = from x in State.get<int>()
                  from _ in State.put(x + 1)
                  select x;
    // 必须 Run 传入初始状态 0
    var (value, newState) = counter.Run(0);
    ```

### 3. 处理副作用的 Monad (IO, Eff, Task)
*   **本质**:`Func<void, T>` (或者更复杂的异步/副作用描述)
*   **为什么需要 Run**:为了保持"纯函数"特性,这些 Monad 只是**描述**了要做什么(比如"写文件"、"读数据库"),但不会立即做。调用 `Run` (或者 `RunUnsafe` / `Await`) 才是真正扣动扳机,让副作用发生的时刻。
*   **例子**:
    ```csharp
    // 只是描述要打印,还没打印
    var program = IO.print("Hello");
    // 真正执行打印
    program.Run();
    ```

### 4. 累积日志的 Monad (Writer)
*   **本质**:`(Result, Log)` (虽然它是数据,但在某些实现中为了惰性求值,也需要触发)
*   **为什么需要 Run**:通常用于解包,将计算结果和积累的日志(Log)分离开来。
*   **例子**:
    ```csharp
    var writer = from _ in Writer.tell("Started")
                 select 42;
    // 获取结果和日志
    var (result, log) = writer.Run();
    ```

---

### 总结对比

| Monad 类型 | 本质 | 就像是... | 是否需要 Run? |
| :--- | :--- | :--- | :--- |
| **Option / Either** | 数据 | 一个**盒子**。打开看里面有值就是有,没有就没有。 | ❌ 不需要 |
| **List / Seq** | 数据 | 一排**盒子**。 | ❌ 不需要 |
| **Reader** | 计算 | 一个**机器**。你需要通电(给环境)它才转。 | ✅ 需要 `Run(env)` |
| **State** | 计算 | 一个**流水线**。你需要放原料(初始状态)它才产出。 | ✅ 需要 `Run(initial)` |
| **IO / Eff** | 动作 | 一个**剧本**。你需要喊 Action(执行)演员才演。 | ✅ 需要 `Run()` |

**一句话总结:**
如果你手中的 Monad **本身不包含值**,而是**包含了"如何产生值"的配方**,那么它就需要通过 `Run` 来把配方变成实际的值。
