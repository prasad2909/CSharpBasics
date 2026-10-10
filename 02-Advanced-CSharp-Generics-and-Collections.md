# Advanced C# — Generics and Collections

> Interview preparation for Senior .NET Developer / Technical Lead roles  
> Focus: type safety, reusable code, collection selection, equality, and performance.

## Learning objectives

By the end of this chapter, you should be able to:

- Explain why generics are useful in C#.
- Write and understand generic classes and methods.
- Apply generic constraints.
- Choose between `List<T>`, `Dictionary<TKey, TValue>`, `HashSet<T>`, `Queue<T>`, and `Stack<T>`.
- Explain equality and hashing requirements for hash-based collections.
- Discuss typical collection operation complexities and their trade-offs.

---

## 1. Why generics?

Generics let a class, method, interface, or struct work with a type supplied by the caller while retaining compile-time type safety.

### Without generics

```csharp
using System.Collections;

var values = new ArrayList();
values.Add(10);
values.Add("twenty");

int first = (int)values[0]; // Requires a cast.
int second = (int)values[1]; // Runtime InvalidCastException.
```

`ArrayList` stores values as `object`, so unrelated types can be added and mistakes may surface at runtime.

### With generics

```csharp
var values = new List<int> { 10, 20, 30 };
values.Add(40);

int first = values[0]; // No cast required.
// values.Add("fifty"); // Compile-time error.
```

**Benefits of generics**

- Compile-time type safety.
- Fewer casts and fewer runtime type errors.
- Reusable algorithms and data structures.
- Avoids boxing/unboxing for value types in common generic collection scenarios.

### Interview answer

Generics allow reusable, type-safe code without giving up compile-time checking. For example, `List<int>` can contain integers only, whereas a non-generic collection such as `ArrayList` can hold different object types and may require casts.

---

## 2. Generic classes

A generic class accepts a type parameter.

```csharp
public class Repository<T>
{
    private readonly List<T> _items = new();

    public void Add(T item) => _items.Add(item);

    public IReadOnlyList<T> GetAll() => _items;
}
```

Usage:

```csharp
var tradeRepository = new Repository<Trade>();
var customerRepository = new Repository<Customer>();
```

The same implementation can work with different entity types. `T` is a placeholder; the actual type is supplied when the generic type is constructed.

---

## 3. Generic methods and type inference

```csharp
static T Echo<T>(T value)
{
    return value;
}
```

Usage:

```csharp
string text = Echo("hello"); // T is inferred as string.
int number = Echo(25);       // T is inferred as int.
```

The compiler can often infer the type parameter from the arguments.

A type mismatch still fails:

```csharp
// int value = Echo("25"); // Compile-time error: result is string.
```

Explicit type arguments are possible:

```csharp
int value = Echo<int>(25);
```

---

## 4. Generic constraints

Constraints limit the types that can be supplied for a type parameter.

### Reference-type constraint

```csharp
static void Process<T>(T item) where T : class
{
    // T must be a reference type.
}
```

### Value-type constraint

```csharp
static void ProcessValue<T>(T item) where T : struct
{
    // T must be a non-nullable value type.
}
```

### Parameterless-constructor constraint

```csharp
static T Create<T>() where T : new()
{
    return new T();
}
```

The type must have a public parameterless constructor. The `new()` constraint is commonly used after other applicable constraints.

### Interface constraint

```csharp
public interface IEntity
{
    int Id { get; }
}

static int GetId<T>(T entity) where T : IEntity
{
    return entity.Id;
}
```

The compiler allows the method to use members guaranteed by `IEntity`.

**Important:** `where T : struct` does not allow a class type. `where T : class` does not allow a value type such as `int`.

---

## 5. `List<T>` fundamentals

`List<T>` is a dynamically sized, ordered collection. It preserves insertion order and allows duplicates.

```csharp
var numbers = new List<int> { 10, 20, 30 };

numbers.Insert(1, 15); // [10, 15, 20, 30]
numbers.Remove(20);    // [10, 15, 30]

Console.WriteLine(numbers[1]); // 15
Console.WriteLine(numbers.Count); // 3
```

### Key members

- `Add(item)`: adds an item to the end.
- `Insert(index, item)`: inserts at an index and shifts subsequent items.
- `Remove(item)`: removes the first matching item, if present.
- `RemoveAt(index)`: removes the item at an index.
- `Contains(item)`: checks whether an equal item exists.
- `Count`: number of items currently stored.
- `Capacity`: allocated storage capacity; it can be greater than `Count`.

Indexes are zero-based. Inserting or removing in the middle can shift later elements.

### Typical time complexity

| Operation | Typical complexity |
|---|---:|
| Access by index | O(1) |
| Add at end | O(1) amortized |
| Insert at beginning or middle | O(n) |
| Remove by index | O(n) |
| `Contains` / search | O(n) |
| Iterate through all items | O(n) |

“Amortized O(1)” for `Add` means most appends are constant time, while an occasional capacity expansion costs O(n).

### When to use it

Use `List<T>` when you need an ordered sequence, index-based access, duplicates, or iteration in sequence order.

---

## 6. `Dictionary<TKey, TValue>`

A dictionary maps unique keys to values. It is useful when a value must be retrieved using a key such as a trade ID, customer ID, or configuration name.

```csharp
var deals = new Dictionary<int, string>
{
    [101] = "USD/INR",
    [102] = "EUR/USD"
};

deals[102] = "GBP/USD"; // Updates the existing key.
deals[103] = "USD/JPY"; // Adds a new key.

Console.WriteLine(deals.Count); // 3
Console.WriteLine(deals[102]);  // GBP/USD
```

### Adding versus updating

```csharp
deals.Add(104, "AUD/USD"); // Adds; duplicate key throws.
deals[104] = "AUD/NZD";    // Adds if absent, updates if present.
```

Use `Add` when a duplicate key should be treated as an error. Use the indexer when either adding or replacing is intended.

### Safe lookup with `TryGetValue`

```csharp
if (deals.TryGetValue(102, out string? pair))
{
    Console.WriteLine(pair);
}
else
{
    Console.WriteLine("Deal not found");
}
```

`TryGetValue` returns `true` if the key exists and `false` otherwise. It is generally preferable to using exceptions for normal missing-key cases.

- `deals[missingKey]` throws `KeyNotFoundException`.
- `deals.Add(existingKey, value)` throws `ArgumentException`.
- `ContainsKey(key)` checks for key presence.

C# is case-sensitive: write `TryGetValue`, not `tryGetValue`.

### Typical complexity

| Operation | Typical average complexity |
|---|---:|
| Lookup by key | O(1) |
| Insert / update | O(1) |
| Remove by key | O(1) |
| Search through values | O(n) |

These are average-case expectations, not absolute guarantees. Hash collisions and resizing can affect performance.

### When to use it

Use a dictionary when you need key-to-value lookup. It is not intended to be a list with numeric positions: dictionary keys identify entries, while list indexes identify positions.

---

## 7. `HashSet<T>` and equality

A `HashSet<T>` stores unique elements according to its equality comparer.

```csharp
var tradeIds = new HashSet<int>();

Console.WriteLine(tradeIds.Add(101)); // True
Console.WriteLine(tradeIds.Add(102)); // True
Console.WriteLine(tradeIds.Add(101)); // False
Console.WriteLine(tradeIds.Count);    // 2
```

`Add` returns `true` if the element was added and `false` if an equal element already exists.

### Equality and hashing contract

For hash-based collections:

- If `Equals` says two objects are equal, their hash codes must be equal.
- Equal hash codes do **not** prove that two objects are equal.
- Hash collisions are allowed; the collection uses equality to distinguish colliding elements.
- If equality-relevant fields change while an object is stored in a hash-based collection, lookup/removal behavior can become unreliable. Avoid mutating those fields while the object is stored.

For custom types, equality may be defined through `IEquatable<T>`, `Equals`, `GetHashCode`, or an explicit `IEqualityComparer<T>`.

Example using a value type:

```csharp
var ids = new HashSet<int> { 101, 102, 101 };
Console.WriteLine(ids.Count); // 2
```

### Typical complexity

| Operation | Typical average complexity |
|---|---:|
| `Add` | O(1) |
| `Contains` | O(1) |
| `Remove` | O(1) |
| Iterate through all elements | O(n) |

### When to use it

Use `HashSet<T>` when uniqueness or fast membership testing matters and you do not need index-based access. For example, it can track trade IDs that have already been processed.

---

## 8. `Queue<T>` — FIFO

FIFO means **First In, First Out**. The oldest item is removed first.

```csharp
var queue = new Queue<int>();

queue.Enqueue(101);
queue.Enqueue(102);
queue.Enqueue(103);

Console.WriteLine(queue.Dequeue()); // 101
Console.WriteLine(queue.Peek());    // 102 (does not remove it)
```

- `Enqueue`: add to the back.
- `Dequeue`: remove and return the front item.
- `Peek`: inspect the front item without removing it.

Typical enqueue/dequeue operations are O(1).

**Use cases:** work queues, ordered request processing, breadth-first search, and buffering.

---

## 9. `Stack<T>` — LIFO

LIFO means **Last In, First Out**. The newest item is removed first.

```csharp
var stack = new Stack<int>();

stack.Push(101);
stack.Push(102);
stack.Push(103);

Console.WriteLine(stack.Pop());  // 103
Console.WriteLine(stack.Peek()); // 102 (does not remove it)
```

- `Push`: add to the top.
- `Pop`: remove and return the top item.
- `Peek`: inspect the top item without removing it.

Typical push/pop operations are O(1).

**Use cases:** undo/redo logic, parsing, depth-first search, and tracking nested operations.

---

## 10. Choosing the right collection

| Requirement | Suitable collection | Reason |
|---|---|---|
| Ordered sequence, index access, duplicates allowed | `List<T>` | Preserves sequence and supports indexes |
| Lookup a value using a unique key | `Dictionary<TKey, TValue>` | Key-based lookup |
| Unique items and membership testing | `HashSet<T>` | Prevents equal duplicates |
| Process oldest item first | `Queue<T>` | FIFO behavior |
| Process newest item first | `Stack<T>` | LIFO behavior |

### Practical banking example

Suppose a service receives trade IDs:

- Use a `HashSet<int>` to check whether a trade ID has already been processed.
- Use a `Dictionary<int, Trade>` to retrieve a trade by its ID.
- Use a `Queue<Trade>` to process incoming trades in arrival order.
- Use a `List<Trade>` when you need an ordered collection with index access.
- Use a `Stack<Operation>` to model undo operations.

The collection should be selected based on the operations and semantics required—not simply because one type seems more familiar.

---

## 11. Common interview pitfalls

1. **Confusing `Count` and `Capacity`:** `Count` is the number of stored items; `Capacity` is allocated storage.
2. **Assuming `List<T>.Contains` is O(1):** it is O(n) because it may inspect each element.
3. **Assuming dictionary assignment always adds:** assigning to an existing key updates its value.
4. **Using the dictionary indexer for an uncertain key:** use `TryGetValue` to handle missing keys safely.
5. **Assuming a `HashSet<T>` detects duplicates only by reference:** behavior depends on equality and the comparer.
6. **Assuming equal hash codes mean equal objects:** collisions are possible.
7. **Confusing FIFO and LIFO:** queues remove the oldest item; stacks remove the newest item.
8. **Treating Big-O values as unconditional guarantees:** hash-based operations are typically O(1) on average; worst-case behavior can differ.

---

## 12. Interview-ready recap

Be prepared to answer these succinctly:

- **Why generics?** Reusable, type-safe code with compile-time checking.
- **What is a generic constraint?** A rule limiting valid type arguments and sometimes enabling access to specific members or constructors.
- **When do you choose `List<T>`?** For ordered data, index access, and duplicates.
- **When do you choose `Dictionary<TKey, TValue>`?** For key-based lookup and mapping.
- **When do you choose `HashSet<T>`?** For uniqueness and membership checks.
- **What must be true of `Equals` and `GetHashCode`?** If two objects are equal, they must have the same hash code.
- **Queue versus stack?** FIFO versus LIFO.
- **Why use `TryGetValue`?** To check for a key and retrieve its value without relying on an exception for a missing key.

## Completion checklist

- [ ] I can explain and write a generic class and method.
- [ ] I understand `class`, `struct`, `new()`, and interface constraints.
- [ ] I can explain `List<T>` operations and complexity.
- [ ] I can safely retrieve values from a dictionary.
- [ ] I understand hash-set uniqueness and the equality/hash-code contract.
- [ ] I can distinguish queue FIFO from stack LIFO.
- [ ] I can choose a collection for a real .NET scenario.
