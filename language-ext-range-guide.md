# C# `language-ext` 库中的 `Range` 数据类型详解

---

## 目录

1. [什么是 `Range`？](#1-什么是-range)
2. [泛型特性：支持多种类型](#2-泛型特性支持多种类型)
3. [创建方式：`fromMinMax` 与 `fromCount`](#3-创建方式fromminmax-与-fromcount)
4. [实用扩展方法：`InRange` 与 `Overlaps`](#4-实用扩展方法inrange-与-overlaps)
5. [综合示例](#5-综合示例)
6. [总结](#6-总结)

---

## 1. 什么是 `Range`？

在 `language-ext` 库中，`Range<A>` 是一个用于表示**连续值区间**的不可变数据类型。它封装了一个范围的起点（`Min`）、终点（`Max`）和步长（`Step`），并通过泛型约束支持任何实现了 `Ord<A>`（可排序）和 `Arithmetic<A>`（可运算）的类型。

与 C# 原生的 `System.Range`（仅支持整数索引）不同，`language-ext` 的 `Range<A>` 更加通用，具备以下核心特点：

- **不可变性（Immutability）**：创建后不可修改，符合函数式编程理念。
- **泛型支持**：不局限于整数，可用于字符、长整型等任意支持算术运算的类型。
- **可枚举**：`Range<A>` 实现了 `IEnumerable<A>`，可直接用于 `foreach` 循环或 LINQ 查询。
- **内置实用方法**：提供 `InRange`、`Overlaps` 等方法，简化区间判断逻辑。

```csharp
// 引入必要的命名空间
using LanguageExt;
using static LanguageExt.Prelude;
```

---

## 2. 泛型特性：支持多种类型

`Range<A>` 的类型参数 `A` 受到 `struct`、`Ord<A>`、`Arithmetic<A>` 等约束，因此它可以支持所有满足条件的值类型。

### 2.1 支持 `int`（整数范围）

整数是最常见的使用场景，例如表示索引区间或数值范围：

```csharp
// 表示整数范围 [1, 10]，步长为 1
var intRange = Range(1, 10);

foreach (var n in intRange)
{
    Console.Write(n + " "); // 输出: 1 2 3 4 5 6 7 8 9 10
}
```

### 2.2 支持 `char`（字符范围）

`Range<char>` 可以表示连续的字符区间，非常适合处理字母表或字符集：

```csharp
// 表示字符范围 ['a', 'e']，步长为 1
var charRange = Range('a', 'e');

foreach (var c in charRange)
{
    Console.Write(c + " "); // 输出: a b c d e
}
```

### 2.3 支持 `long`（长整型范围）

当需要处理大数值区间时，可以使用 `long`：

```csharp
// 表示长整型范围 [1000000000L, 1000000005L]
var longRange = Range(1_000_000_000L, 1_000_000_005L);

foreach (var n in longRange)
{
    Console.Write(n + " "); // 输出: 1000000000 1000000001 1000000002 1000000003 1000000004 1000000005
}
```

### 2.4 自定义步长

`Range` 支持自定义步长，可以表示非连续的区间（如偶数、奇数序列）：

```csharp
// 表示步长为 2 的整数范围：0, 2, 4, 6, 8, 10
var evenRange = Range(0, 10, 2);

foreach (var n in evenRange)
{
    Console.Write(n + " "); // 输出: 0 2 4 6 8 10
}
```

---

## 3. 创建方式：`fromMinMax` 与 `fromCount`

`language-ext` 提供了两种语义清晰的工厂方法来创建 `Range<A>` 实例，分别适用于不同的场景。

### 3.1 `fromMinMax`：通过最小值和最大值创建

`fromMinMax` 明确指定区间的**起始值（Min）**和**终止值（Max）**，适用于你已知区间两端边界的场景。

**方法签名：**

```csharp
Range<A> fromMinMax<A>(A min, A max, A step)
```

**示例：**

```csharp
// 创建整数范围 [1, 100]，步长为 1
var range1 = fromMinMax(1, 100, 1);
Console.WriteLine($"Min: {range1.Min}, Max: {range1.Max}");
// 输出: Min: 1, Max: 100

// 创建字符范围 ['A', 'Z']，步长为 1
var alphabetRange = fromMinMax('A', 'Z', (char)1);
Console.WriteLine($"大写字母范围: {alphabetRange.Min} 到 {alphabetRange.Max}");
// 输出: 大写字母范围: A 到 Z

// 创建步长为 5 的整数范围 [0, 50]
var step5Range = fromMinMax(0, 50, 5);
foreach (var n in step5Range)
{
    Console.Write(n + " "); // 输出: 0 5 10 15 20 25 30 35 40 45 50
}
```

> **注意**：`fromMinMax` 创建的是**闭区间** `[min, max]`，即包含两端的端点值。

### 3.2 `fromCount`：通过起始值和数量创建

`fromCount` 指定区间的**起始值（start）**和**元素数量（count）**，适用于你已知起点和需要多少个元素的场景。

**方法签名：**

```csharp
Range<A> fromCount<A>(A start, int count, A step)
```

**示例：**

```csharp
// 从 1 开始，取 5 个整数，步长为 1 → [1, 2, 3, 4, 5]
var range2 = fromCount(1, 5, 1);
foreach (var n in range2)
{
    Console.Write(n + " "); // 输出: 1 2 3 4 5
}

// 从 'a' 开始，取 5 个字符，步长为 1 → ['a', 'b', 'c', 'd', 'e']
var charRange2 = fromCount('a', 5, (char)1);
foreach (var c in charRange2)
{
    Console.Write(c + " "); // 输出: a b c d e
}

// 从 10 开始，取 4 个偶数，步长为 2 → [10, 12, 14, 16]
var evenCount = fromCount(10, 4, 2);
foreach (var n in evenCount)
{
    Console.Write(n + " "); // 输出: 10 12 14 16
}
```

### 3.3 两种方式的对比

| 特性           | `fromMinMax(min, max, step)` | `fromCount(start, count, step)` |
|----------------|------------------------------|---------------------------------|
| **适用场景**   | 已知两端边界值               | 已知起点和元素个数              |
| **语义**       | 定义区间的范围               | 定义序列的起点和长度            |
| **端点包含性** | 包含 `min` 和 `max`          | 从 `start` 开始共 `count` 个元素|
| **示例**       | `fromMinMax(1, 10, 1)`       | `fromCount(1, 10, 1)`           |

---

## 4. 实用扩展方法：`InRange` 与 `Overlaps`

`language-ext` 为 `Range<A>` 提供了两个常用的扩展方法，用于判断值是否在区间内，以及两个区间是否重叠。

### 4.1 `InRange`：判断值是否在区间内

`InRange` 方法用于检查某个值是否落在给定的 `Range<A>` 区间之内。

**方法签名：**

```csharp
bool InRange<A>(this Range<A> range, A value)
```

**示例：**

```csharp
var scoreRange = fromMinMax(0, 100, 1);

// 检查分数是否合法
bool isValid1 = scoreRange.InRange(85);   // true  → 85 在 [0, 100] 内
bool isValid2 = scoreRange.InRange(105);  // false → 105 超出范围
bool isValid3 = scoreRange.InRange(0);    // true  → 0 是边界值，包含在内
bool isValid4 = scoreRange.InRange(100);  // true  → 100 是边界值，包含在内

Console.WriteLine($"85 分有效: {isValid1}");   // 输出: 85 分有效: True
Console.WriteLine($"105 分有效: {isValid2}");  // 输出: 105 分有效: False

// 对字符范围使用 InRange
var lowercaseRange = fromMinMax('a', 'z', (char)1);

bool isLower1 = lowercaseRange.InRange('m');  // true  → 'm' 是小写字母
bool isLower2 = lowercaseRange.InRange('A');  // false → 'A' 是大写字母
bool isLower3 = lowercaseRange.InRange('z');  // true  → 'z' 是边界值

Console.WriteLine($"'m' 是小写字母: {isLower1}");  // 输出: 'm' 是小写字母: True
Console.WriteLine($"'A' 是小写字母: {isLower2}");  // 输出: 'A' 是小写字母: False
```

#### 结合 LINQ 过滤集合中的有效值

```csharp
var validAgeRange = fromMinMax(0, 120, 1);
var ages = new[] { -5, 0, 18, 25, 99, 121, 200 };

var validAges = ages.Where(age => validAgeRange.InRange(age)).ToList();
// validAges = [0, 18, 25, 99]

Console.WriteLine("合法年龄: " + string.Join(", ", validAges));
// 输出: 合法年龄: 0, 18, 25, 99
```

### 4.2 `Overlaps`：判断两个区间是否重叠

`Overlaps` 方法用于检查两个 `Range<A>` 区间是否有交集（重叠部分）。

**方法签名：**

```csharp
bool Overlaps<A>(this Range<A> range1, Range<A> range2)
```

**示例：**

```csharp
var range1 = fromMinMax(1, 10, 1);   // [1, 10]
var range2 = fromMinMax(5, 15, 1);   // [5, 15]
var range3 = fromMinMax(11, 20, 1);  // [11, 20]
var range4 = fromMinMax(10, 20, 1);  // [10, 20]

bool overlap1 = range1.Overlaps(range2);  // true  → [1,10] 与 [5,15] 重叠于 [5,10]
bool overlap2 = range1.Overlaps(range3);  // false → [1,10] 与 [11,20] 不重叠
bool overlap3 = range1.Overlaps(range4);  // true  → [1,10] 与 [10,20] 在端点 10 处相交

Console.WriteLine($"[1,10] 与 [5,15] 重叠: {overlap1}");   // 输出: True
Console.WriteLine($"[1,10] 与 [11,20] 重叠: {overlap2}");  // 输出: False
Console.WriteLine($"[1,10] 与 [10,20] 重叠: {overlap3}");  // 输出: True
```

#### 实际应用：会议时间冲突检测

```csharp
// 用整数表示时间（分钟），检测会议是否冲突
var meeting1 = fromMinMax(540, 660, 1);   // 09:00 - 11:00（540分钟 - 660分钟）
var meeting2 = fromMinMax(600, 720, 1);   // 10:00 - 12:00（600分钟 - 720分钟）
var meeting3 = fromMinMax(660, 780, 1);   // 11:00 - 13:00（660分钟 - 780分钟）
var meeting4 = fromMinMax(800, 900, 1);   // 13:20 - 15:00（800分钟 - 900分钟）

bool conflict1 = meeting1.Overlaps(meeting2);  // true  → 09:00-11:00 与 10:00-12:00 冲突
bool conflict2 = meeting1.Overlaps(meeting3);  // true  → 在 11:00 处重叠
bool conflict3 = meeting1.Overlaps(meeting4);  // false → 完全不冲突

Console.WriteLine($"会议1 与 会议2 冲突: {conflict1}");  // 输出: True
Console.WriteLine($"会议1 与 会议3 冲突: {conflict2}");  // 输出: True
Console.WriteLine($"会议1 与 会议4 冲突: {conflict3}");  // 输出: False
```

---

## 5. 综合示例

下面通过一个完整的示例，展示 `Range<A>` 各个特性的协同使用。

```csharp
using System;
using System.Linq;
using LanguageExt;
using static LanguageExt.Prelude;

class RangeDemo
{
    static void Main()
    {
        Console.WriteLine("=== language-ext Range 综合演示 ===\n");

        // ── 1. 基本整数范围 ──────────────────────────────────────────
        Console.WriteLine("【1. 整数范围枚举】");
        var intRange = fromMinMax(1, 5, 1);
        Console.WriteLine("fromMinMax(1, 5, 1): " + string.Join(", ", intRange));
        // 输出: 1, 2, 3, 4, 5

        var countRange = fromCount(10, 5, 1);
        Console.WriteLine("fromCount(10, 5, 1): " + string.Join(", ", countRange));
        // 输出: 10, 11, 12, 13, 14

        // ── 2. 字符范围 ──────────────────────────────────────────────
        Console.WriteLine("\n【2. 字符范围枚举】");
        var alphabetRange = fromMinMax('A', 'E', (char)1);
        Console.WriteLine("fromMinMax('A', 'E', 1): " + string.Join(", ", alphabetRange));
        // 输出: A, B, C, D, E

        // ── 3. 自定义步长 ────────────────────────────────────────────
        Console.WriteLine("\n【3. 自定义步长】");
        var oddRange = fromCount(1, 5, 2);
        Console.WriteLine("奇数序列 fromCount(1, 5, 2): " + string.Join(", ", oddRange));
        // 输出: 1, 3, 5, 7, 9

        // ── 4. InRange 判断 ──────────────────────────────────────────
        Console.WriteLine("\n【4. InRange 判断】");
        var gradeRange = fromMinMax(60, 100, 1);
        var scores = new[] { 45, 60, 75, 99, 100, 101 };
        foreach (var score in scores)
        {
            Console.WriteLine($"  分数 {score} 及格: {gradeRange.InRange(score)}");
        }
        // 输出:
        //   分数 45 及格: False
        //   分数 60 及格: True
        //   分数 75 及格: True
        //   分数 99 及格: True
        //   分数 100 及格: True
        //   分数 101 及格: False

        // ── 5. Overlaps 重叠判断 ─────────────────────────────────────
        Console.WriteLine("\n【5. Overlaps 重叠判断】");
        var rangeA = fromMinMax(1, 10, 1);
        var rangeB = fromMinMax(8, 15, 1);
        var rangeC = fromMinMax(11, 20, 1);

        Console.WriteLine($"  [1,10] 与 [8,15]  重叠: {rangeA.Overlaps(rangeB)}");   // True
        Console.WriteLine($"  [1,10] 与 [11,20] 重叠: {rangeA.Overlaps(rangeC)}");   // False

        // ── 6. 结合 LINQ 使用 ────────────────────────────────────────
        Console.WriteLine("\n【6. 结合 LINQ 使用】");
        var hundredRange = fromMinMax(1, 100, 1);
        var multiplesOf7 = hundredRange
            .Where(n => n % 7 == 0)
            .ToList();
        Console.WriteLine("1~100 中 7 的倍数: " + string.Join(", ", multiplesOf7));
        // 输出: 7, 14, 21, 28, 35, 42, 49, 56, 63, 70, 77, 84, 91, 98

        Console.WriteLine("\n=== 演示结束 ===");
    }
}
```

---

## 6. 总结

| 功能                   | API                              | 说明                                         |
|------------------------|----------------------------------|----------------------------------------------|
| **创建（按边界）**     | `fromMinMax(min, max, step)`     | 指定最小值与最大值来创建闭区间               |
| **创建（按数量）**     | `fromCount(start, count, step)`  | 指定起始值与元素个数来创建序列               |
| **枚举元素**           | `foreach` / LINQ                 | `Range<A>` 实现 `IEnumerable<A>`，可直接迭代 |
| **判断值是否在区间内** | `.InRange(value)`                | 返回 `bool`，包含端点                        |
| **判断两区间是否重叠** | `.Overlaps(otherRange)`          | 返回 `bool`，包含端点相交的情况              |
| **泛型支持**           | `int`, `char`, `long`, ...       | 支持所有实现 `Ord<A>` + `Arithmetic<A>` 的类型 |

`Range<A>` 是 `language-ext` 中一个简洁而强大的工具，将函数式编程的**不可变性**与**泛型抽象**融入了区间表达，让区间相关的逻辑更加直观、安全、易于组合。

---

> **参考资源**
> - [language-ext GitHub 仓库](https://github.com/louthy/language-ext)
> - [language-ext 官方文档](https://louthy.github.io/language-ext/)
