# 转变思维：用函数式编程处理数据分类汇总

在函数式编程（FP）中，遇到这种需要"分类汇总"的场景，你的第一反应是"创建一个可变的 Map，然后循环遍历并向里面塞数据"——这非常典型，是指令式编程（Imperative Programming）的思维方式。指令式编程关注的是**"怎么做（How）"**，即一步步指挥计算机去修改状态。

而函数式编程关注的是**"做什么（What）"**（数据转换）以及**"状态的不可变性（Immutability）"**。

要转变思维，你可以从以下几个维度来理解和设计：

### 1. 核心思维转变
* **从"修改容器"到"数据管道（Pipeline）"：** 不要把数据看作是被你放进某个篮子里的东西，而是把数据看作`水流`。原始数据（文件列表）流经一个"分类器"，直接变换成了新的形态（Map）。
* **从"变量突变（Mutation）"到"状态传递（Fold/Reduce）"：** 如果必须手动累加数据，FP 的做法不是在原地修改旧的 Map，而是每一次累加都**返回一个包含新元素的全新的 Map**。由于 FP 语言/库（如你所在的 `language-ext`）底层的不可变数据结构（Persistent Data Structures）使用了结构共享（如 Trie 树），这种"创建新 Map"的操作是非常高效的，不会真的复制整个 Map。

### 2. 具体设计与实现方法

在 FP 中，处理这种需求通常有两种标准姿势：

#### 姿势一：声明式分组（GroupBy）—— 最推荐的日常写法
大多数 FP 语言或库都已经为你抽象了这种操作，通常叫 `groupBy`。你只需要告诉它"按照什么 Key 来分组"以及"Value 是什么"。

在 C# 结合 `language-ext` 中，你可以直接结合 LINQ 和不可变结构来做：

```csharp
using LanguageExt;
using static LanguageExt.Prelude;
using System.Linq;

// 假设你有这样一个序列：Seq<(string File, string Md5)>
Seq<(string File, string Md5)> fileHashes = Seq(
    ("a.txt", "hash1"),
    ("b.txt", "hash1"),
    ("c.txt", "hash2")
);

// 声明式转换：直接映射为一个不可变的 Map<string, Seq<string>>
Map<string, Seq<string>> groupedMap = toMap(
    fileHashes.GroupBy(x => x.Md5)
              .Select(g => (g.Key, toSeq(g.Select(x => x.File))))
);
```
这里没有任何变量被改变，数据从 `Seq` 流入 `GroupBy`，再转换成了不可变的 `Map`。

#### 姿势二：折叠 / 归约（Fold / Reduce）—— 理解 FP 底层机制
如果你想知道如果不使用内置的 `GroupBy`，在纯 FP 中是如何"不用可变状态"建立起这个字典的，答案是 **Fold（折叠）**。

`Fold` 需要两个东西：
1. **初始状态**：一个空的、不可变的 Map。
2. **累加函数**：接收"当前状态（旧 Map）"和"当前元素"，返回一个**新状态（新 Map）**。

```csharp
// 使用 Fold 从零构建不可变字典
Map<string, Seq<string>> groupedMap = fileHashes.Fold(
    Map<string, Seq<string>>.Empty, // 初始状态：空字典
    (stateMap, item) =>
        // 查找字典里是否已经有这个 MD5
        stateMap.Find(item.Md5).Match(
            // 如果有，获取现有的 seq，往里面添加新文件（返回新的 seq），并覆盖旧的 key（返回新的 Map）
            Some: existingSeq => stateMap.SetItem(item.Md5, existingSeq.Add(item.File)),
            // 如果没有，直接添加一个新的 key 和包含当前文件的 seq（返回新的 Map）
            None: () => stateMap.Add(item.Md5, Seq1(item.File))
        )
);
```

### 3. 如何在日常中刻意练习这种 FP 思想？

下次当你脑海中浮现出 **"我要创建一个 `var list = new List()` 或 `var dict = new Dictionary()` 然后写一个 `foreach` 循环"** 的冲动时，立刻停下来，问自己以下几个问题：

1. **我是在做一对一的转换吗？** -> 使用 `Map` / `Select`
2. **我是在把一个元素展开成零个或多个元素吗？** -> 使用 `Bind` / `SelectMany`
3. **我是在过滤数据吗？** -> 使用 `Filter` / `Where`
4. **我是在把数据按条件拆成两部分吗？** -> 使用 `partition`
5. **我是在把多个元素重新组织、分类吗？** -> 使用 `GroupBy`
6. **我是在排序数据吗？** -> 使用 `OrderBy` / `ThenBy`
7. **我是在把一个集合聚合成一个单一的值（或单一的复杂结构）吗？** -> 使用 `Fold` / `Aggregate`
8. **我是在处理可能不存在、可能失败的结果吗？** -> 使用 `Option` / `Either` / `Try`

通过强迫自己优先从这些“数据变换操作”里找答案，而不是直接写 `foreach` 去维护临时状态，你的思维就会自然而然地从"指令式"过渡到"函数式"了。

### 4. 补充例子：找出目录下的重复文件

为了更好地说明，我们可以结合实际应用场景：找出某个文件夹下所有重复的文件并把结果处理成我们需要的 `Map` 结构。用完全函数式的方式（结合 `language-ext`），这将会是一条非常干净的数据流：

```csharp
using System.IO;
using System.Security.Cryptography;
using LanguageExt;
using static LanguageExt.Prelude;
using System.Linq;

// 1. 定义一个计算文件 MD5 的纯函数（忽略异常处理的简化版）
string ComputeMd5(string filePath)
{
    using var md5 = MD5.Create();
    using var stream = File.OpenRead(filePath);
    return BitConverter.ToString(md5.ComputeHash(stream)).Replace("-", "").ToLowerInvariant();
}

// 2. 数据流水线
void FindDuplicateFiles(string directoryPath)
{
    // 从文件路径的序列开始
    Seq<string> files = toSeq(Directory.GetFiles(directoryPath));

    // 流水线转换
    Map<string, Seq<string>> duplicates = toMap(
        files
            // 转换：路径 -> (路径, MD5)
            .Map(path => (Path: path, Hash: ComputeMd5(path)))
            // 分组：按 MD5 聚合
            .GroupBy(x => x.Hash)
            // 转换：将 IGrouping 转为 Tuple，并提取出文件路径列表
            .Map(g => (Hash: g.Key, Files: toSeq(g.Select(x => x.Path))))
            // 过滤：只保留文件数量大于 1 的组（即重复文件）
            .Filter(x => x.Files.Count > 1)
    );

    // 3. 产生副作用（输出结果，在 FP 中副作用往往推迟到最后执行）
    duplicates.Iter((hash, duplicateFiles) =>
    {
        Console.WriteLine($"发现重复文件 (MD5: {hash}):");
        duplicateFiles.Iter(file => Console.WriteLine($" - {file}"));
    });
}
```

在这个例子中，没有任何临时字典被创建并手动 `Add`，也没有使用任何 `foreach` 循环进行状态累加。数据从 `GetFiles` 进来，经过一系列的 `Map`、`GroupBy`、`Filter` 转换，最后通过 `toMap` 直接变成了我们需要的 `Map<string, Seq<string>>`。
