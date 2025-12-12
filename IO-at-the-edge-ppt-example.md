# 关于"将IO放到程序边缘"

这是一个很好的问题！你说的场景（操作PPT、删除元素、导出图片）确实充满了side effects，但这**恰恰是可以实现的**，而且是函数式编程的核心模式之一。

## 核心思路：**描述 vs 执行**

关键在于区分：
1. **描述要做什么**（纯函数，构建数据结构）
2. **实际执行**（在程序边缘运行）

## 方法一：使用 `IO<A>` monad

```csharp
// 1. 定义纯函数的"描述"层
IO<Presentation> LoadPresentation(string path) =>
    IO.lift(() => new Presentation(path));

IO<Seq<Shape>> FindShapesWithKeyword(Presentation ppt, string keyword) =>
    IO.pure(ppt.Shapes.Where(s => s.Text.Contains(keyword)).ToSeq());

IO<Unit> DeleteShape(Shape shape) =>
    IO.lift(() => { shape.Delete(); return unit; });

IO<Unit> ExportAsImage(Presentation ppt, string outputPath) =>
    IO.lift(() => { ppt.SaveAsImage(outputPath); return unit; });

// 2. 组合成一个"程序描述" - 这里都是纯的！
IO<Unit> ProcessPpt(string inputPath, string keyword, string outputPath) =>
    from ppt in LoadPresentation(inputPath)
    from shapes in FindShapesWithKeyword(ppt, keyword)
    from _ in shapes.Traverse(DeleteShape)  // 删除所有匹配的元素
    from __ in ExportAsImage(ppt, outputPath)
    select unit;

// 3. 只在程序的最边缘执行
public static void Main()
{
    ProcessPpt("input.pptx", "删除我", "output.png")
        .Run();  // <-- IO在这里才真正执行
}
```

## 方法二：使用 `Eff<RT, A>`（带运行时依赖）

如果需要更好的可测试性：

```csharp
// 定义 Trait
public interface PptIO
{
    IO<Presentation> Load(string path);
    IO<Unit> DeleteShape(Shape shape);
    IO<Unit> Export(Presentation ppt, string path);
}

// 业务逻辑完全纯净
Eff<RT, Unit> ProcessPpt<RT>(string inputPath, string keyword, string outputPath)
    where RT : struct, Has<Eff<RT>, PptIO> =>
    from ppt in load(inputPath)
    from shapes in findShapes(ppt, keyword)
    from _ in shapes.Traverse(deleteShape)
    from __ in export(ppt, outputPath)
    select unit;
```

## 为什么这样是"IO在边缘"？

```
┌─────────────────────────────────────────┐
│  Main() - IO 执行点 (不纯)               │  ← 程序边缘
├─────────────────────────────────────────┤
│  ProcessPpt(...) - 返回 IO<Unit>        │
│  ↓                                       │
│  组合多个 IO 操作 (纯！只是组合描述)      │  ← 核心业务逻辑
│  ↓                                       │
│  LoadPresentation, DeleteShape 等        │
│  (每个都返回 IO<A>，不执行)              │
└─────────────────────────────────────────┘
```

**关键点**：
- `IO<A>` 是一个**描述**，不是执行
- `from ... in ...` 只是在**组合描述**
- `.Run()` 才是真正执行的地方

## 这样做的好处

1. **可测试**：可以mock整个IO层
2. **可推理**：核心逻辑是纯函数
3. **可组合**：可以自由组合、retry、并行化
4. **延迟执行**：可以在执行前分析/优化
