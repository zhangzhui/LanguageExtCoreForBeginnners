# How to instantiate and work with its core types without using the new keyword.

This document explains the core functional programming patterns in Language-Ext and how to instantiate and work with its core types without using the `new` keyword.

## Core Types and Instantiation Patterns

Language-Ext provides functional programming constructs for C# that avoid using the `new` keyword for type instantiation. Instead, it uses static factory methods and Prelude functions.

### Option < A >

The `Option<A>` type represents a value that may or may not exist (Some or None). It's an alternative to using `null`.

#### Instantiation:

```csharp
// Using Prelude functions (preferred)
Option<int> someValue = Some(123);
Option<int> noneValue = None;

// Or using the static methods directly
Option<int> someValue2 = Option<int>.Some(123);
Option<int> noneValue2 = Option<int>.None;
```

#### Usage Examples:

```csharp
// Basic Option usage
var opt1 = Some(42);
var opt2 = Option<int>.None;

// Pattern matching
string result1 = opt1.Match(
    Some: x => $"Value is {x}",
    None: () => "No value"
);

// LINQ syntax with automatic None propagation
var optResult = from x in Some(10)
                from y in Some(20)
                select x + y; // Results in Some(30)

var optResult2 = from x in Some(10)
                 from y in Option<int>.None
                 select x + y; // Results in None
```

### Either<L, R>

The `Either<L, R>` type represents a value that is either a Left (typically an error) or a Right (typically a success value).

#### Instantiation:

```csharp
// Using Prelude functions
Either<string, int> success = Right(123);
Either<string, int> error = Left("Something went wrong");

// Or using the static methods directly
Either<string, int> success2 = Either<string, int>.Right(123);
Either<string, int> error2 = Either<string, int>.Left("Something went wrong");

// Convenience types (implicit casting)
Either<string, int> success3 = Right(123);
Either<string, int> error3 = Left("Something went wrong");
```

#### Usage Examples:

```csharp
// Basic Either usage
var success = Right<string, int>(42);
var error = Left<string, int>("Error occurred");

// Pattern matching
string result = success.Match(
    Right: x => $"Success: {x}",
    Left: err => $"Error: {err}"
);

// LINQ syntax with automatic Left propagation (like exception handling)
var eitherResult = from x in Right<string, int>(10)
                   from y in Right<string, int>(20)
                   select x + y; // Results in Right(30)

var eitherResult2 = from x in Right<string, int>(10)
                    from y in Left<string, int>("Failed")
                    select x + y; // Results in Left("Failed")
```

### Seq < A >

The `Seq<A>` type is a high-performance immutable sequence (similar to a list but with better performance characteristics).

#### Instantiation:

```csharp
// Using Prelude functions (preferred)
Seq<int> sequence = Seq(1, 2, 3, 4, 5);
Seq<string> emptySeq = Seq<string>();

// Or using the static methods directly
Seq<int> sequence2 = SeqStrict.create(1, 2, 3, 4, 5);
```

#### Usage Examples:

```csharp
// Basic Seq usage
var numbers = Seq(1, 2, 3, 4, 5);
var empty = Seq<int>();

// Operations
var doubled = numbers.Map(x => x * 2); // Seq(2, 4, 6, 8, 10)
var evens = numbers.Filter(x => x % 2 == 0); // Seq(2, 4)
var sum = numbers.Fold(0, (acc, x) => acc + x); // 15

// LINQ syntax
var result = from x in Seq(1, 2, 3)
             from y in Seq(10, 20)
             select x + y; // Seq(11, 21, 12, 22, 13, 23)
```

## Type Instantiation Comparison Table

| Type | Instantiation Method | Example | Notes |
|------|---------------------|---------|-------|
| `Option<A>` | `Some(value)` or `Option<A>.Some(value)` | `Some(42)` | Represents optional values |
| `Option<A>` | `None` or `Option<A>.None` | `None` | Represents absence of value |
| `Either<L, R>` | `Right(value)` or `Either<L, R>.Right(value)` | `Right(42)` | Represents success value |
| `Either<L, R>` | `Left(value)` or `Either<L, R>.Left(value)` | `Left("Error")` | Represents error value |
| `Seq<A>` | `Seq(items...)` or `SeqStrict.create(items...)` | `Seq(1, 2, 3)` | Immutable sequence/list |
| `HashMap<K, V>` | `HashMap((key, value), ...)` | `HashMap(("a", 1), ("b", 2))` | Immutable dictionary |
| `HashSet<A>` | `HashSet(items...)` | `HashSet(1, 2, 3)` | Immutable set |

## Key Concepts

### 1. Monad Transformers

Language-Ext uses monad transformers to combine effects. For example, `OptionT<M, A>` adds optional behavior to any monad `M`.

```csharp
// OptionT<IO, int> combines Option with IO effect
OptionT<IO, int> computation = OptionT.lift<IO, int>(Some(42));

// To run the computation and get IO<Option<int>>
IO<Option<int>> resultIO = computation.Run();

// To get the final result
Option<int> result = resultIO.Run();
```

### 2. Automatic Propagation

When using LINQ syntax or methods like `Map`, `Bind`, etc., Language-Ext automatically propagates "failure" states:
- For `Option<A>`, `None` values short-circuit the computation
- For `Either<L, R>`, `Left` values short-circuit the computation

### 3. Pure Functions

All operations in Language-Ext are pure - they don't modify existing data structures but return new ones.

## Working with Effects (IO)

Language-Ext provides an `IO` monad for handling side effects in a pure functional way.

```csharp
// Creating IO operations
IO<int> readNumber = IO.lift(() => {
    Console.Write("Enter a number: ");
    return int.Parse(Console.ReadLine());
});

IO<Unit> writeMessage = IO.lift(() => {
    Console.WriteLine("Hello, World!");
    return unit; // unit is LanguageExt's equivalent of void
});

// Combining IO operations
IO<int> combined = from num in readNumber
                   from _ in writeMessage
                   select num * 2;
```

This approach ensures side effects are explicit and composable while maintaining referential transparency.

## Summary

If you don't know how to initialize a monad, please consider **lift** function.