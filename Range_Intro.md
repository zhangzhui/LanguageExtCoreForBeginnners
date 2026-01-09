# LanguageExt 中的 Range 类型

Range 是 LanguageExt 库中的一个数据类型，用于表示一个值的范围。它提供了一种声明式和函数式的方式来处理数值序列。

## 设计哲学

Range 类型的设计遵循了 LanguageExt 的核心理念：
1. **不可变性** - Range 是不可变的数据结构
2. **惰性求值** - 使用 IEnumerable 实现惰性序列生成
3. **类型安全** - 利用泛型和类型约束确保类型安全
4. **函数式编程** - 实现了多种函数式编程接口和操作

## 核心特性

### 1. 类型定义
```csharp
public record Range<A>(A From, A To, A Step, IEnumerable<A> runRange) :
    IEnumerable<A>,
    K<Range, A>
```

### 2. 构造方法

Range 提供了多种构造方式：

#### fromMinMax
构造一个从最小值到最大值的范围：
```csharp
// 递增范围
var range1 = Range.fromMinMax(1, 5); // 1, 2, 3, 4, 5

// 递减范围
var range2 = Range.fromMinMax(5, 1); // 5, 4, 3, 2, 1
```

#### fromCount
构造一个指定起始值和元素数量的范围：
```csharp
// 5个元素，步长为2
var range = Range.fromCount(2, 5, 2); // 2, 4, 6, 8, 10

// 5个元素，步长为-2
var range2 = Range.fromCount(2, 5, -2); // 2, 0, -2, -4, -6
```

### 3. Prelude 中的便捷方法

LanguageExt 在 Prelude 中提供了便捷的 Range 构造方法：

```csharp
// 整数范围
var intRange = Prelude.Range(1, 5); // 1, 2, 3, 4, 5

// 字符范围
var charRange = Prelude.Range('a', 'e'); // 'a', 'b', 'c', 'd', 'e'
```

## 使用场景

### 1. 生成序列
```csharp
// 生成 1 到 10 的整数序列
var numbers = Range.fromMinMax(1, 10);

// 转换为列表
var list = numbers.ToLst();
```

### 2. 范围检查
```csharp
var range = Range.fromMinMax(1, 100);
bool inRange = range.As().InRange(50); // true
```

### 3. 范围重叠检查
```csharp
var range1 = Range.fromMinMax(1, 10);
var range2 = Range.fromMinMax(5, 15);
bool overlaps = range1.As().Overlaps(range2); // true
```

### 4. 函数式操作
```csharp
// 使用 Fold 操作
var range = Range.fromMinMax(1, 5);
var sum = Range.Fold(0, (acc, x) => acc + x, range); // 15

// 转换操作
var doubled = range.Select(x => x * 2).ToLst(); // 2, 4, 6, 8, 10
```

## 实际用例

### 1. 日期范围生成
```csharp
// 生成一周的日期
var startDate = DateTime.Now.Date;
var weekRange = Range.fromCount(startDate, 7, TimeSpan.FromDays(1));
```

### 2. 字符序列处理
```csharp
// 生成字母表
var alphabet = Range.fromMinMax('a', 'z');
```

### 3. 数学序列
```csharp
// 生成等差数列
var arithmetic = Range.fromCount(2, 10, 3); // 2, 5, 8, 11, 14, ...
```

## 特性优势

1. **内存效率** - 惰性求值，只在需要时生成值
2. **类型安全** - 编译时检查确保类型正确
3. **可组合性** - 可以与其他 LanguageExt 类型无缝组合
4. **函数式接口** - 支持 Map, Fold 等函数式操作
5. **可枚举** - 实现 IEnumerable 接口，可以使用 LINQ 操作

Range 类型在 LanguageExt 中是一个非常实用的工具，特别适合需要处理数值序列、范围检查和生成规律数据的场景。它的设计体现了函数式编程的核心思想，提供了类型安全、内存高效的解决方案。