# LanguageExt Collections Summary

A reference guide for the immutable collection types provided by the [LanguageExt](https://github.com/louthy/language-ext) library.

---

## Seq\<A\>

**Underlying structure**: Lazy linked-list / chunked array hybrid

**Evaluation**: Lazy — elements are only materialised when accessed

**Random access**: O(n)

**Modification (prepend/append)**: O(1) amortised

**IEnumerable interop**: Excellent — wraps any `IEnumerable<A>` without immediate enumeration

`Seq<A>` is the go-to general-purpose sequence in LanguageExt. Because it is lazy, wrapping an existing `IEnumerable<A>` is essentially free. It supports efficient head/tail decomposition, making it natural for recursive and pattern-matching style code.

```csharp
Seq<int> seq = Seq(1, 2, 3, 4, 5);
Seq<int> prepended = seq.Cons(0);       // O(1)
var (head, tail) = seq;                 // deconstruct
```

---

## Arr\<A\>

**Underlying structure**: Immutable array (contiguous memory)

**Evaluation**: Strict/Eager

**Random access**: O(1)

**Modification**: O(n) — requires copying the entire array

**IEnumerable interop**: Good

`Arr<A>` is a thin immutable wrapper around a plain array. It is the right choice when you need fast indexed access and the collection is built once and read many times. Mutations are expensive because the whole array must be copied.

```csharp
Arr<int> arr = Array(1, 2, 3, 4, 5);
int third = arr[2];                     // O(1)
Arr<int> updated = arr.SetItem(2, 99);  // O(n) copy
```

---

## Lst\<A\>

**Underlying structure**: AVL-tree-backed persistent list

**Evaluation**: Strict/Eager

**Random access**: O(log n)

**Modification (insert/remove at index)**: O(log n)

**IEnumerable interop**: Good

`Lst<A>` provides a balanced-tree persistent list. Unlike `Arr<A>`, arbitrary insertions and removals are O(log n) rather than O(n), making it suitable for collections that are frequently modified at arbitrary positions.

```csharp
Lst<int> lst = List(1, 2, 3, 4, 5);
Lst<int> inserted = lst.Insert(2, 99); // O(log n)
Lst<int> removed  = lst.RemoveAt(1);   // O(log n)
```

---

## Map\<K, V\>

**Underlying structure**: AVL tree (ordered by key)

**Evaluation**: Strict/Eager

**Lookup / insert / delete**: O(log n)

**Key ordering**: Maintained — iteration is in key order

**IEnumerable interop**: Good (`IEnumerable<(K Key, V Value)>`)

`Map<K, V>` is an ordered, immutable dictionary. Because it is tree-based, keys are always iterated in sorted order. Prefer it when you need deterministic ordering or range queries.

```csharp
Map<string, int> map = Map(("a", 1), ("b", 2), ("c", 3));
Map<string, int> updated = map.Add("d", 4);       // O(log n)
Option<int> value = map.Find("b");                // O(log n)
```

---

## HashMap\<K, V\>

**Underlying structure**: Hash array mapped trie (HAMT)

**Evaluation**: Strict/Eager

**Lookup / insert / delete**: O(1) average

**Key ordering**: Not maintained

**IEnumerable interop**: Good

`HashMap<K, V>` is an unordered, immutable dictionary backed by a HAMT. It offers better average-case performance than `Map<K, V>` for pure key-based lookups when ordering is not required.

```csharp
HashMap<string, int> hmap = HashMap(("a", 1), ("b", 2));
HashMap<string, int> updated = hmap.Add("c", 3);  // O(1) avg
Option<int> value = hmap.Find("a");               // O(1) avg
```

---

## Set\<A\>

**Underlying structure**: AVL tree (ordered)

**Evaluation**: Strict/Eager

**Contains / insert / delete**: O(log n)

**Key ordering**: Maintained

**IEnumerable interop**: Good

`Set<A>` is an ordered, immutable set. Elements are always iterated in sorted order. Use it when you need set operations (union, intersection, difference) with deterministic ordering.

```csharp
Set<int> setA = Set(1, 2, 3, 4);
Set<int> setB = Set(3, 4, 5, 6);
Set<int> union = setA.Union(setB);         // {1,2,3,4,5,6}
Set<int> intersect = setA.Intersect(setB); // {3,4}
```

---

## HashSet\<A\>

**Underlying structure**: Hash array mapped trie (HAMT)

**Evaluation**: Strict/Eager

**Contains / insert / delete**: O(1) average

**Key ordering**: Not maintained

**IEnumerable interop**: Good

`HashSet<A>` is an unordered, immutable set. It provides faster membership tests than `Set<A>` when ordering is irrelevant.

```csharp
HashSet<int> hs = HashSet(1, 2, 3, 4);
bool has3 = hs.Contains(3);              // O(1) avg
HashSet<int> added = hs.Add(5);          // O(1) avg
```

---

## Que\<A\>

**Underlying structure**: Pair of immutable lists (banker's queue)

**Evaluation**: Strict/Eager

**Enqueue (add to back)**: O(1) amortised

**Dequeue (remove from front)**: O(1) amortised

**Random access**: O(n)

**IEnumerable interop**: Good

`Que<A>` is a persistent FIFO queue. It is the right choice for producer/consumer patterns or breadth-first traversal where you need efficient enqueue and dequeue without mutation.

```csharp
Que<int> queue = Queue(1, 2, 3);
Que<int> enqueued = queue.Enqueue(4);          // O(1) amortised
(int head, Que<int> rest) = queue.Dequeue();   // O(1) amortised
```

---

## Stck\<A\>

**Underlying structure**: Immutable linked list

**Evaluation**: Strict/Eager

**Push (add to top)**: O(1)

**Pop (remove from top)**: O(1)

**Random access**: O(n)

**IEnumerable interop**: Good

`Stck<A>` is a persistent LIFO stack. Push and pop are both O(1). Use it for depth-first traversal, undo stacks, or any last-in-first-out scenario.

```csharp
Stck<int> stack = Stack(1, 2, 3);
Stck<int> pushed = stack.Push(4);              // O(1)
(int top, Stck<int> rest) = stack.Pop();       // O(1)
```

---

## Comprehensive Comparison Table

| Collection      | Underlying Structure        | Evaluation | Random Access | Modification Cost          | IEnumerable Interop | Core Use Case                                      |
|-----------------|-----------------------------|------------|---------------|----------------------------|---------------------|----------------------------------------------------|
| `Seq<A>`        | Lazy chunked list           | Lazy       | O(n)          | O(1) prepend/append        | Excellent           | General-purpose sequences, wrapping IEnumerable    |
| `Arr<A>`        | Contiguous immutable array  | Strict/Eager  | O(1)          | O(n) full copy             | Good                | Read-heavy, index-access collections               |
| `Lst<A>`        | AVL tree list               | Strict/Eager  | O(log n)      | O(log n) insert/remove     | Good                | Ordered lists with frequent arbitrary modification |
| `Map<K,V>`      | AVL tree (ordered)          | Strict/Eager  | O(log n)      | O(log n) add/remove        | Good                | Ordered key-value store, range queries             |
| `HashMap<K,V>`  | HAMT (unordered)            | Strict/Eager  | O(1) avg      | O(1) avg add/remove        | Good                | Fast unordered key-value lookups                   |
| `Set<A>`        | AVL tree (ordered)          | Strict/Eager  | O(log n)      | O(log n) add/remove        | Good                | Ordered unique elements, set algebra               |
| `HashSet<A>`    | HAMT (unordered)            | Strict/Eager  | O(1) avg      | O(1) avg add/remove        | Good                | Fast unordered membership tests                    |
| `Que<A>`        | Banker's queue (two lists)  | Strict/Eager | O(n)          | O(1) amortised enqueue/dequeue | Good            | FIFO queues, breadth-first traversal               |
| `Stck<A>`       | Immutable linked list       | Strict/Eager  | O(n)          | O(1) push/pop              | Good                | LIFO stacks, depth-first traversal, undo           |

---

## One-Sentence Selection Guide

> Use `Seq` by default; switch to `Arr` for index-heavy reads, `Lst` for frequent mid-list edits, `Map`/`Set` when you need sorted keys, `HashMap`/`HashSet` for fastest lookups without ordering, `Que` for FIFO, and `Stck` for LIFO.
