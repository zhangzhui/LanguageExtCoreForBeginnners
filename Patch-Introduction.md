# Patch：文档差异与变更

**Patch** 是一个用于描述**文档差异（diff）**和**应用变更**的数据结构，类似于 Git 的 diff/patch 概念。

---

## 核心概念

### Patch 由一系列 Edit 组成

```csharp
Patch<EqA, A>  // 针对元素类型 A 的补丁，EqA 用于判断元素相等
```

三种编辑操作：
- **Insert** - 在位置 i 插入元素
- **Delete** - 在位置 i 删除元素
- **Replace** - 在位置 i 用新元素替换旧元素

---

## 基本用法

### 1. 计算两个文档的差异 (diff)

```csharp
using static LanguageExt.Prelude;
using static LanguageExt.Patch;
using LanguageExt.ClassInstances;

var docA = List("Hello", "World");
var docB = List("Hello", "Beautiful", "World", "!");

// 计算差异
Patch<EqString, string> patch = diff<EqString, string>(docA, docB);

// patch.Edits 包含:
// - Insert(1, "Beautiful")
// - Insert(3, "!")
```

### 2. 应用补丁 (apply)

```csharp
var docA = List("Hello", "World");
var docB = List("Hello", "Beautiful", "World");

var patch = diff<EqString, string>(docA, docB);

// 应用补丁，得到 docB
var result = patch.Apply(docA);  // ["Hello", "Beautiful", "World"]
```

### 3. 补丁求逆 (inverse)

```csharp
var patch = diff<EqString, string>(docA, docB);  // A -> B
var inversePatch = patch.Inverse();               // B -> A

// 应用逆补丁，从 B 还原回 A
var restored = inversePatch.Apply(docB);  // 等于 docA
```

### 4. 补丁合并 (append / +)

```csharp
var docA = List("a", "b");
var docB = List("a", "b", "c");
var docC = List("a", "c");

var patchAB = diff<EqString, string>(docA, docB);  // A -> B
var patchBC = diff<EqString, string>(docB, docC);  // B -> C

// 合并补丁：A -> B -> C
var patchAC = patchAB + patchBC;  // 或 append(patchAB, patchBC)

// 直接应用合并后的补丁
var result = patchAC.Apply(docA);  // 得到 docC
```

---

## 数学特性：Groupoid

Patch 是一个 **Groupoid**（带逆元的 Monoid）：

| 特性 | 说明 |
|------|------|
| **Monoid** | 有 `Empty`（空补丁）和 `+`（合并） |
| **逆元** | 每个 patch 都有 `Inverse()` |
| **结合律** | `(a + b) + c == a + (b + c)` |

```csharp
// 空补丁是单位元
patch + Patch.Empty == patch
Patch.Empty + patch == patch

// 逆元性质
patch + patch.Inverse() == Patch.Empty  // 应用后再撤销 = 无变化
```

---

## 辅助方法

```csharp
// 检查补丁是否可以应用到文档
bool canApply = patch.Applicable(document);

// 获取文档大小变化（Insert 数 - Delete 数）
int delta = patch.SizeChange();

// 检查两个补丁是否可以合并
bool canCompose = Patch.composable(patchA, patchB);
```

---

## 实际应用场景

1. **协同编辑** - 像 Google Docs 那样合并多人编辑
2. **版本控制** - 存储和应用增量变更
3. **撤销/重做** - 用 `Inverse()` 实现撤销
4. **冲突解决** - `transform` / `ours` / `theirs` 方法处理并发编辑

---

## 冲突解决

```csharp
// 当两个人同时编辑同一文档时
var original = List("a", "b", "c");
var aliceEdit = List("a", "x", "c");   // Alice 把 b 改成 x
var bobEdit = List("a", "b", "c", "d"); // Bob 在末尾加了 d

var patchAlice = diff<EqString, string>(original, aliceEdit);
var patchBob = diff<EqString, string>(original, bobEdit);

// 变换补丁以解决冲突
var (patchAlice2, patchBob2) = Patch.transform(patchAlice, patchBob);

// 两种方式得到相同结果
var result1 = patchBob2.Apply(aliceEdit);   // Alice 的基础上应用变换后的 Bob
var result2 = patchAlice2.Apply(bobEdit);   // Bob 的基础上应用变换后的 Alice
// 都是 ["a", "x", "c", "d"]
```

---

## 源码位置

- `LanguageExt.Core/DataTypes/Patch/Patch.cs` - 主类型定义
- `LanguageExt.Core/DataTypes/Patch/Patch.Module.cs` - 静态方法（diff, apply 等）
- `LanguageExt.Core/DataTypes/Patch/Patch.Internal.cs` - 内部算法实现
