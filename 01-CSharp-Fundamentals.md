# C# Fundamentals --- Senior Developer Interview Notes

> Status: In progress --- concepts explained; independent revision still
> required\
> Confidence: 2/4 (provisional; most diagnostic answers were correct,
> but reasoning was often not provided)\
> Last reviewed: 2026-10-10\
> Target: Senior .NET Developer / Hands-on Technical Lead\
> Repository location:
> `06-CSharp-Fundamentals/01-CSharp-Fundamentals.md`

------------------------------------------------------------------------

## 1. Learning objectives

By the end of this chapter, I should be able to:

-   [ ] Explain value types, reference types, assignment, and parameter
    passing.
-   [ ] Explain string immutability and distinguish `var`, `object`, and
    `dynamic`.
-   [ ] Explain boxing/unboxing, nullable values, `const`, and
    `readonly`.
-   [ ] Use enums, arrays, generic collections, and generic constraints.
-   [ ] Explain `class` versus `struct` without relying on the
    inaccurate "stack versus heap" rule.
-   [ ] Explain equality, `GetHashCode()`, and hash-based collection
    requirements.
-   [ ] Handle exceptions and disposable resources appropriately.
-   [ ] Predict program output and justify it step by step.
-   [ ] Answer senior-level follow-up questions about trade-offs,
    performance, and production use.

## 2. Prerequisites

-   Basic programming constructs: variables, conditions, loops, methods.
-   Basic understanding of classes and objects.
-   Familiarity with compiling and running a C# console application.

## 3. Mental model

C# is statically typed. A variable has a compile-time type, and that
type determines which operations the compiler permits. Some
operations---especially those involving `dynamic`---are resolved at
runtime.

Keep these distinctions separate:

1.  **Type category:** value type or reference type.
2.  **Variable assignment:** what is copied when one variable is
    assigned to another.
3.  **Object mutability:** whether the object's state can change.
4.  **Storage location:** stack, heap, registers, or other runtime
    implementation details.
5.  **Equality:** whether two values or objects count as equal for a
    particular type.

These are related concepts, but none should be treated as synonyms for
another.

## 4. Detailed explanation

### 4.1 Value types and reference types

Common value types include `int`, `bool`, `decimal`, `double`, `char`,
enums, and user-defined `struct` types. Common reference types include
classes, arrays, strings, delegates, and interface references.

#### Value-type assignment copies the value

``` csharp
int a = 10;
int b = a;

b = 20;

Console.WriteLine(a); // 10
Console.WriteLine(b); // 20
```

`b` receives a copy of `a`'s value. Updating `b` does not update `a`.

#### Reference-type assignment copies the reference

``` csharp
sealed class Counter
{
    public int Value { get; set; }
}

var c1 = new Counter { Value = 10 };
var c2 = c1;

c2.Value = 99;
Console.WriteLine(c1.Value); // 99

c2 = new Counter { Value = 50 };
Console.WriteLine(c1.Value); // 99
Console.WriteLine(c2.Value); // 50
```

Initially, `c1` and `c2` refer to the same object. Mutating that object
through `c2` is visible through `c1`. Reassigning `c2` makes it refer to
a different object; it does not change `c1`.

**Important:** Do not memorise "value types live on the stack and
reference types live on the heap." That is not a reliable rule. A value
type may be embedded in a heap object or array, and actual storage
depends on context and runtime implementation.

### 4.2 Strings are immutable

A string cannot be changed after it is created. Operations that appear
to modify a string create a new string value.

``` csharp
string s1 = "Hello";
string s2 = s1;

s2 += " World";

Console.WriteLine(s1); // Hello
Console.WriteLine(s2); // Hello World
```

The concatenation creates a new string, and `s2` is assigned to it. `s1`
remains unchanged.

For repeated construction in a loop, `StringBuilder` may reduce repeated
allocations and copying:

``` csharp
using System.Text;

var builder = new StringBuilder();

for (int i = 0; i < 10_000; i++)
{
    builder.Append(i);
}

string result = builder.ToString();
```

Do not over-optimise: ordinary `+` is perfectly reasonable for a small,
fixed number of concatenations. Measure performance-sensitive code.

### 4.3 `var` vs `object` vs `dynamic`

  -----------------------------------------------------------------------
  Feature           `var`             `object`          `dynamic`
  ----------------- ----------------- ----------------- -----------------
  Meaning           Compiler infers a The declared type Relevant
                    static type       is `object`       operations are
                                                        dynamically bound

  Type checking     Primarily compile Compile time;     Runtime for
                    time              casts may be      dynamic
                                      needed            operations

  Can later hold a  Not by            Yes               Yes
  different type?   reassignment to                     
                    an incompatible                     
                    type                                

  Typical use       Clear local       APIs or storage   Interop or APIs
                    declarations      requiring a       that genuinely
                                      general object    require dynamic
                                      reference         binding
  -----------------------------------------------------------------------

``` csharp
var age = 28;
// age = "Prasad"; // Compile-time error: age is int

object value = 10;
value = "Hello"; // Valid

dynamic data = 10;
Console.WriteLine(data + 5); // 15
data = "Hello";
Console.WriteLine(data.ToUpper()); // HELLO
```

`var` is not dynamic typing. The compiler infers `age` as `int`.

`object` allows different runtime objects, but the compiler only exposes
members of `object` without a cast or type check.

`dynamic` defers relevant member and operation binding to runtime. For
example:

``` csharp
dynamic value = 10;
value.ToUpper(); // Compiles, then fails at runtime
```

Prefer static typing for normal application code because it catches more
mistakes during compilation.

### 4.4 Boxing and unboxing

**Boxing** converts a value type to `object` or a compatible interface
reference. Conceptually, the value is copied into a boxed object
representation.

``` csharp
int number = 100;
object boxed = number; // Boxing

number = 200;

Console.WriteLine(number); // 200
Console.WriteLine(boxed);  // 100
```

The boxed copy is independent of the original variable.

**Unboxing** extracts the value from a boxed object and requires the
correct underlying value type:

``` csharp
object boxed = 100;

int number = (int)boxed; // Unboxing
long bigger = (long)(int)boxed; // Unbox int, then convert to long
```

This is not valid:

``` csharp
// long bigger = (long)boxed; // InvalidCastException at runtime
```

Although an `int` can be converted to `long`, a boxed `int` cannot be
directly unboxed as a `long`.

Boxing can introduce allocations and copying. Generic collections such
as `List<int>` avoid the boxing normally associated with storing and
retrieving integers through non-generic collections such as `ArrayList`.

### 4.5 Parameter passing: default, `ref`, `out`, and `in`

By default, C# passes arguments by value.

``` csharp
static void Change(int x) => x = 100;

int number = 10;
Change(number);
Console.WriteLine(number); // 10
```

For a reference type, the *reference itself* is copied by value. A
method can mutate the shared object, but reassigning its ordinary
parameter does not reassign the caller's variable.

``` csharp
sealed class Person
{
    public string Name { get; set; } = "";
}

static void Rename(Person person) => person.Name = "Rahul";

static void Replace(Person person) =>
    person = new Person { Name = "Rahul" };

var p = new Person { Name = "Prasad" };
Rename(p);
Console.WriteLine(p.Name); // Rahul

Replace(p);
Console.WriteLine(p.Name); // Rahul — Replace's local parameter was reassigned only
```

  -------------------------------------------------------------------------
  Keyword         Main purpose    Caller variable must   Method must assign
                                    be assigned before        before normal
                                                 call?              return?
  --------------- --------------- -------------------- --------------------
  Default         Ordinary             Yes, for normal                   No
                  parameter        definite assignment 
                  passing                              

  `ref`           Read/write the                   Yes                   No
                  caller's                             
                  variable                             

  `out`           Return a value                    No                  Yes
                  through a                            
                  parameter                            

  `in`            Pass by                          Yes                   No
                  reference with                       
                  read-only                            
                  access through                       
                  that parameter                       
  -------------------------------------------------------------------------

``` csharp
static void Update(int a, ref int b, out int c)
{
    a = 100;
    b = 200;
    c = 300;
}

int x = 10;
int y = 20;
int z;

Update(x, ref y, out z);
Console.WriteLine($"{x}, {y}, {z}"); // 10, 200, 300
```

`a` is a copy, `b` aliases the caller's variable, and `c` must be
assigned by the method.

`in` can avoid copying a large struct, but is not automatically faster
for every type. It prevents assignment to the parameter itself; it does
not make an object graph deeply immutable.

### 4.6 Nullable types and null-coalescing

A non-nullable value type such as `int` cannot represent absence. `int?`
is shorthand for `Nullable<int>` and can contain an integer or `null`.

``` csharp
int? amount = null;
int finalAmount = amount ?? 100;

amount = 50;

Console.WriteLine(finalAmount); // 100
Console.WriteLine(amount ?? 100); // 50
```

`??` returns the left value if it is non-null; otherwise, it returns the
right-hand fallback. It is evaluated at the point where the expression
executes. It does not create a live link between variables.

Other useful operators:

``` csharp
string? name = null;
Console.WriteLine(name?.ToUpper()); // null result, rendered as a blank line

int? quantity = null;
Console.WriteLine(quantity.HasValue); // False
Console.WriteLine(quantity.GetValueOrDefault()); // 0
```

Nullable reference types are different: `string?` is a compiler
nullability annotation, not a separate runtime string type. Enable
nullable reference types and check nullable values before dereferencing
them.

### 4.7 `const` vs `readonly`

  -----------------------------------------------------------------------
  Feature                 `const`                 `readonly` field
  ----------------------- ----------------------- -----------------------
  Value                   Compile-time constant   May be determined at
                                                  runtime

  Assignment              At declaration          At declaration or in
                                                  the appropriate
                                                  constructor

  Implicitly static?      Yes                     No

  Can use                 No                      Yes
  `DateTime.UtcNow`?                              
  -----------------------------------------------------------------------

``` csharp
class DealService
{
    public const int MaxRetries = 3;
    public readonly DateTime CreatedAt;

    public DealService()
    {
        CreatedAt = DateTime.UtcNow;
    }
}
```

A `readonly` reference field prevents reassignment of the reference
outside permitted initialization. It does not make the referenced object
immutable:

``` csharp
public readonly List<string> Names = new();

public void Example()
{
    Names.Add("Prasad"); // Allowed
    // Names = new List<string>(); // Not allowed here
}
```

### 4.8 Enums

Enums define named constants and default to an `int` underlying type
unless another integral type is specified.

``` csharp
enum DealStatus
{
    Draft,            // 0
    PendingApproval,  // 1
    Approved,         // 2
    Rejected          // 3
}

DealStatus status = DealStatus.Approved;
Console.WriteLine((int)status); // 2
Console.WriteLine(status);      // Approved
```

Explicit values are useful when mapping stable codes, but changing
persisted enum numbers can break compatibility. Casting an arbitrary
integer to an enum does not validate that it is a declared member:

``` csharp
DealStatus status = (DealStatus)999;
Console.WriteLine(Enum.IsDefined(status)); // False
```

Validate enum values received from external input when correctness
matters.

### 4.9 Arrays, `List<T>`, `Dictionary<TKey,TValue>`, and `HashSet<T>`

An array has a fixed length and supports O(1) indexed access:

``` csharp
int[] amounts = { 100, 200, 300 };
Console.WriteLine(amounts[1]);     // 200
Console.WriteLine(amounts.Length); // 3
```

Arrays are reference types, even when their elements are value types.
Assigning an array variable copies the reference.

``` csharp
int[] a = { 10, 20, 30 };
int[] b = a;
b[0] = 99;
Console.WriteLine(a[0]); // 99
```

#### Typical collection complexity

  ------------------------------------------------------------------------------
  Collection / operation                Typical complexity Notes
  --------------------------- ---------------------------- ---------------------
  Array index access                                  O(1) Fixed length

  `List<T>` index access                              O(1) Resizable backing
                                                           array

  `List<T>.Add`                             Amortized O(1) Occasional resize
                                                           costs O(n)

  `List<T>.Contains`                                  O(n) Linear search

  Insert/remove near start of                         O(n) Elements may shift
  `List<T>`                                                

  `Dictionary<TKey,TValue>`                  Expected O(1) Hash-based; not an
  lookup                                                   absolute guarantee

  `HashSet<T>.Contains` /                    Expected O(1) Hash-based; not an
  `Add`                                                    absolute guarantee
  ------------------------------------------------------------------------------

Use a `List<T>` for an ordered, resizable sequence; a
`Dictionary<TKey,TValue>` for key-to-value lookup; and a `HashSet<T>`
for uniqueness and membership checks.

``` csharp
var deals = new Dictionary<int, string>
{
    [101] = "USD/INR",
    [102] = "EUR/USD"
};

if (deals.TryGetValue(102, out string? pair))
{
    Console.WriteLine(pair); // EUR/USD
}

var processedIds = new HashSet<int>();
Console.WriteLine(processedIds.Add(101)); // True
Console.WriteLine(processedIds.Add(101)); // False
```

For a deduplication requirement, `HashSet<int>` is usually more direct
than `Dictionary<int, string>`. `HashSet<T>.Add` returns `false` when an
equal element already exists.

Prefer `List<T>` over legacy non-generic `ArrayList` in new code.
Generics improve type safety and avoid boxing for normal value-type
collection operations.

### 4.10 `class` vs `struct`

A class is a reference type; a struct is a value type. Assignment copies
a class reference or copies the struct value.

``` csharp
struct Point
{
    public int X;
}

class Box
{
    public int Value;
}

Point p1 = new Point { X = 10 };
Point p2 = p1;
p2.X = 50;
Console.WriteLine(p1.X); // 10

Box b1 = new Box { Value = 10 };
Box b2 = b1;
b2.Value = 50;
Console.WriteLine(b1.Value); // 50
```

Structs cannot inherit from another class or struct, but can implement
interfaces. They can be mutable, but small immutable structs are often
easier to reason about. Do not choose a struct merely because of a
"stack allocation" assumption. Consider size, copying, identity,
mutability, and measured performance.

### 4.11 Equality and `GetHashCode()`

Equality depends on the type and its implementation. For an ordinary
class that has not customised equality, two separate instances are
normally unequal by reference, even if their properties match. Strings
use content-based equality.

``` csharp
class Deal
{
    public int Id { get; set; }
}

var d1 = new Deal { Id = 101 };
var d2 = new Deal { Id = 101 };

Console.WriteLine(d1 == d2);      // False for this class
Console.WriteLine(d1.Equals(d2)); // False for this class
```

For strings:

``` csharp
string s1 = new string("USD/INR".ToCharArray());
string s2 = new string("USD/INR".ToCharArray());

Console.WriteLine(s1 == s2);      // True
Console.WriteLine(s1.Equals(s2)); // True
```

**Hash-code contract:** If two objects are equal according to the
equality comparer, they must produce the same hash code. Unequal objects
may share a hash code; hash codes are not required to be unique.

`Dictionary<TKey,TValue>` and `HashSet<T>` use hash codes to find
candidate entries and equality to determine matches. If you implement
value-based equality for a type, implement `GetHashCode()` consistently
as well, or supply a suitable comparer. Records often provide convenient
value-based equality.

Avoid mutating fields that determine equality or hashing while an object
is used as a hash-based key.

### 4.12 Exceptions and resource disposal

Use exceptions for exceptional failures, not as the default control flow
for expected invalid input.

``` csharp
if (int.TryParse("123", out int number))
{
    Console.WriteLine(number); // 123
}
else
{
    Console.WriteLine("Invalid number");
}
```

For operations that can fail exceptionally:

``` csharp
try
{
    int number = int.Parse("abc");
}
catch (FormatException ex)
{
    Console.WriteLine($"Invalid number: {ex.Message}");
}
```

When rethrowing the current exception, use `throw;` to preserve the
original stack trace:

``` csharp
try
{
    DoWork();
}
catch
{
    LogFailure();
    throw;
}
```

Avoid catching and swallowing exceptions without a recovery strategy.
Catch specific exceptions when possible, add useful context, and
preserve failure semantics.

Dispose resources deterministically:

``` csharp
using var stream = File.OpenRead("deals.txt");
// Read from the stream.
```

The compiler arranges disposal when execution leaves the scope. Use
`await using` for types implementing `IAsyncDisposable` when
asynchronous disposal is appropriate. For database access, ensure
connections and data readers are disposed; connection-pool exhaustion
can have several causes, so diagnose using evidence.

## 5. Consolidated code example

``` csharp
using System;
using System.Collections.Generic;

enum DealStatus
{
    Draft,
    PendingApproval,
    Approved,
    Rejected
}

static class Program
{
    static void Main()
    {
        int original = 10;
        object boxed = original;
        original = 20;

        Console.WriteLine(original); // 20
        Console.WriteLine(boxed);    // 10

        var deals = new Dictionary<int, string>
        {
            [101] = "USD/INR",
            [102] = "EUR/USD"
        };

        if (deals.TryGetValue(102, out string? pair))
            Console.WriteLine(pair);

        var processed = new HashSet<int>();
        Console.WriteLine(processed.Add(101)); // True
        Console.WriteLine(processed.Add(101)); // False

        DealStatus status = DealStatus.Approved;
        Console.WriteLine(status); // Approved
    }
}
```

### Expected output

``` text
20
10
EUR/USD
True
False
Approved
```

## 6. Comparisons

  --------------------------------------------------------------------------------
  Comparison                  Key difference            Choose / remember
  --------------------------- ------------------------- --------------------------
  Value type vs reference     Assignment copies value   Reason about mutations and
  type                        vs reference              reassignments

  `var` vs `object` vs        Inference vs general base Prefer static typing by
  `dynamic`                   type vs runtime binding   default

  `ref` vs `out` vs `in`      Read/write alias vs       Use only when the method
                              output alias vs read-only contract benefits
                              alias                     

  `const` vs `readonly`       Compile-time constant vs  Use `readonly` for runtime
                              constructor-initialised   values
                              field                     

  Array vs `List<T>`          Fixed length vs resizable Choose based on collection
                                                        requirements

  `List<T>` vs `HashSet<T>`   Ordered sequence vs       Use set for
                              unique membership set     deduplication/membership

  `Dictionary<TKey,TValue>`   Key-value mapping vs      Store associated data only
  vs `HashSet<T>`             unique values             when needed

  `class` vs `struct`         Reference semantics vs    Decide based on identity,
                              value semantics           size, mutability and use

  `Equals()` vs               Equality vs hash bucket   Equal values must have
  `GetHashCode()`             assistance                equal hash codes

  `Parse` vs `TryParse`       Throws on invalid input   Prefer `TryParse` for
                              vs reports failure        expected invalid user
                                                        input
  --------------------------------------------------------------------------------

## 7. Real-world scenario

### Requirement: process a batch of FX deal IDs without duplicates

**Requirements** - Process each deal ID only once per batch. - Identify
repeated IDs efficiently. - Preserve associated deal information
separately when required. - Handle unknown deal IDs without throwing an
avoidable lookup exception.

``` csharp
var processed = new HashSet<int>();
var deals = new Dictionary<int, string>
{
    [101] = "USD/INR",
    [102] = "EUR/USD"
};

int[] incoming = { 101, 102, 101, 999 };

foreach (int id in incoming)
{
    if (!processed.Add(id))
    {
        Console.WriteLine($"Duplicate deal: {id}");
        continue;
    }

    if (deals.TryGetValue(id, out string? pair))
        Console.WriteLine($"Processing {id}: {pair}");
    else
        Console.WriteLine($"Unknown deal: {id}");
}
```

**Trade-offs** - `HashSet<int>` tracks uniqueness; the dictionary maps
IDs to pairs. - Both provide expected O(1) hash-based membership/lookup
in typical conditions. - The example deduplicates only within this
in-memory batch. It does not guarantee exactly-once processing across
multiple service instances, retries, or restarts. Production systems may
also need database constraints, idempotency keys, and transaction
design. - Validate external data and log useful context without leaking
sensitive information.

## 8. Common mistakes and anti-patterns

-   Assuming all value types are stack allocated and all reference types
    are heap allocated.
-   Assuming a reference-type variable stores the object itself rather
    than a reference.
-   Assuming `var` means dynamically typed.
-   Assuming strings can be modified in place.
-   Casting a boxed `int` directly to `long`.
-   Using `ref` when a normal return value would be clearer.
-   Assuming `readonly` makes the referenced object immutable.
-   Assuming an integer-to-enum cast validates the enum value.
-   Using `List<T>.Contains` for large, repeated membership checks when
    a set is more appropriate.
-   Using `Dictionary<TKey,TValue>` when only uniqueness is required.
-   Overriding `Equals()` without implementing a consistent
    `GetHashCode()`.
-   Mutating hash-relevant fields after inserting an object into a
    dictionary or set.
-   Catching and swallowing every exception.
-   Forgetting to dispose streams, readers, and other disposable
    resources.
-   Optimising allocations without measuring a real bottleneck.

## 9. Interview questions

### Basic

1.  What is the difference between a value type and a reference type?
2.  Why is `string` a reference type but immutable?
3.  What is the difference between `var`, `object`, and `dynamic`?
4.  What are boxing and unboxing?
5.  What does `int?` mean?
6.  What is the difference between `const` and `readonly`?
7.  What is an enum, and what is its default underlying type?
8.  What is the difference between an array and a `List<T>`?

### Intermediate

1.  What changes when a reference-type argument is passed by value?
2.  Compare `ref`, `out`, and `in`.
3.  Why can a boxed `int` not be directly unboxed as a `long`?
4.  Compare `List<T>`, `Dictionary<TKey,TValue>`, and `HashSet<T>`.
5.  Explain the typical time complexity of lookup and insertion in a
    dictionary.
6.  Compare a class and a struct without using the stack-versus-heap
    shortcut.
7.  What is the difference between `==` and `Equals()`?
8.  Why must equal objects have equal hash codes?
9.  Why might `TryParse` be preferable to `Parse`?
10. What does `using` guarantee for disposable resources?

### Advanced / scenario-based

1.  A method receives a class instance. How can it change the object's
    state without being able to replace the caller's variable?
2.  A `HashSet<T>` appears to contain an object, but `Contains()` later
    returns `false`. What equality or mutability problems would you
    investigate?
3.  A hot path boxes integers repeatedly. How would you identify and
    reduce the allocations?
4.  A large `List<T>` is searched repeatedly for membership. How would
    you decide whether to use a `HashSet<T>` or dictionary?
5.  An enum value arrives from an API as an integer. How would you
    validate and handle unsupported values?
6.  A SQL connection pool is exhausted. What would you investigate
    beyond merely increasing the pool size?
7.  When would you choose a struct over a class in a
    performance-sensitive API, and what would you measure?
8.  How would you design idempotent deal processing beyond using an
    in-memory `HashSet<T>`?

### Follow-up questions

1.  Does `readonly List<int>` prevent adding elements to the list? Why
    not?
2.  Can two unequal objects have the same hash code?
3.  Does `dynamic` eliminate runtime errors?
4.  Does `HashSet<T>` guarantee O(1) operations in every case?
5.  Does assigning one array variable to another clone the array?
6.  Can a struct implement an interface?
7.  Does `finally` run if the process is forcibly terminated?
8.  Does `TryParse` mean the application no longer needs input
    validation?

## 10. Exercises

### Exercise 1 --- Predict assignment behaviour

**Problem:** Predict the output and explain each line.

``` csharp
int a = 5;
int b = a;
b = 8;

Console.WriteLine($"{a}, {b}");
```

**Expected output:**

``` text
5, 8
```

**Complexity:** O(1) time, O(1) extra space.

### Exercise 2 --- Reference reassignment

**Problem:** Explain why the first and second outputs differ.

``` csharp
sealed class Item
{
    public int Value { get; set; }
}

var x = new Item { Value = 1 };
var y = x;
y.Value = 2;
y = new Item { Value = 3 };

Console.WriteLine(x.Value);
Console.WriteLine(y.Value);
```

**Expected output:**

``` text
2
3
```

### Exercise 3 --- Deduplicate IDs

**Problem:** Given `int[] ids = { 1, 2, 1, 3, 2 }`, print each unique ID
once.

**Hint:** Use `HashSet<int>` or preserve order with a set plus a list.

**One solution:**

``` csharp
int[] ids = { 1, 2, 1, 3, 2 };
var seen = new HashSet<int>();

foreach (int id in ids)
{
    if (seen.Add(id))
        Console.WriteLine(id);
}
```

This solution prints `1`, `2`, `3` in first-seen order because the input
is traversed in order and output occurs only on successful insertion.
The set itself should not be treated as an ordered collection.

**Complexity:** Expected O(n) time and O(u) extra space, where `u` is
the number of unique IDs.

### Exercise 4 --- Safe dictionary lookup

**Problem:** Write a method that returns a deal pair for an ID, without
throwing when the ID is absent.

**Hint:** Use `TryGetValue`.

### Exercise 5 --- Equality contract

**Problem:** Create a `DealId` value object where two instances with the
same ID are equal. Implement equality and hash codes consistently, or
use an appropriate record/value-object approach. Add tests for equal
values, unequal values, and use in a `HashSet<T>`.

### Exercise 6 --- Nullable input

**Problem:** Accept a nullable `decimal` deal amount and calculate a
total using zero when the amount is absent. Explain why storing the
fallback in a variable does not make it update when the original
nullable variable changes.

### Exercise 7 --- Explain a production trade-off

**Problem:** A service checks whether 50,000 IDs have been processed.
Compare repeated `List<int>.Contains` with a `HashSet<int>`. State
assumptions, time complexity, memory implications, and when preserving
order matters.

## 11. Troubleshooting / edge cases

-   **Unexpected shared changes:** Determine whether two variables
    reference the same mutable object.
-   **Invalid cast during unboxing:** Check the actual boxed runtime
    type.
-   **Null reference exceptions:** Enable nullable reference types and
    validate data at boundaries.
-   **Unexpected enum values:** Validate external numeric values rather
    than assuming casts are safe.
-   **Dictionary/set lookup failures:** Review equality, hash-code
    consistency, and mutation of key fields.
-   **Slow membership checks:** Measure collection size and access
    patterns; consider hash-based lookup.
-   **Excessive allocations:** Profile before changing code; check
    boxing, string concatenation, and collection resizing.
-   **Resource exhaustion:** Verify deterministic disposal and
    investigate connection/reader lifetimes, long-running requests, and
    pool metrics.
-   **Exception noise:** Catch expected failures appropriately and
    preserve stack traces when rethrowing.

## 12. Quick revision sheet

-   Value-type assignment copies the value; reference-type assignment
    copies the reference.
-   Mutating a shared object differs from reassigning a reference
    variable.
-   Strings are immutable; repeated concatenation in loops may create
    unnecessary allocations.
-   `var` is compile-time inference; `object` is a type; `dynamic`
    defers binding to runtime.
-   Boxing wraps a value-type copy; unboxing requires the matching boxed
    value type.
-   `ref` aliases the caller's variable; `out` must be assigned; `in` is
    read-only through the parameter.
-   `int?` represents an integer or `null`; `??` supplies a fallback
    when evaluated.
-   `const` is compile-time; `readonly` can be initialised at runtime
    during permitted initialisation.
-   Arrays have fixed length; `List<T>` is resizable; dictionaries map
    keys to values; sets enforce uniqueness.
-   Generic constraints express requirements on type arguments.
-   A class has reference semantics; a struct has value semantics. Avoid
    simplistic stack/heap rules.
-   Equal objects must have equal hash codes; hash codes need not be
    unique.
-   Prefer `TryParse` for expected invalid input, and dispose resources
    deterministically.
-   Hash-based complexity is typically expected O(1), not an
    unconditional guarantee.

## 13. Self-assessment

-   [ ] I can explain the mental model without notes.
-   [ ] I can write and run the examples independently.
-   [ ] I can answer the basic and intermediate questions aloud.
-   [ ] I can solve all exercises without copying.
-   [ ] I can explain the trade-offs and production failure modes.
-   [ ] I have reattempted the diagnostic questions I got wrong.
-   [ ] I have reviewed this chapter after a delay.

### Diagnostic history

  -----------------------------------------------------------------------
  Diagnostic              Result                  Revision note
  ----------------------- ----------------------- -----------------------
  1 --- integer           Correct (B)             Add reasoning aloud
  assignment                                      

  2 --- class reference   Incorrect (selected B;  Distinguish object
  reassignment            correct A)              mutation from reference
                                                  reassignment

  3 --- `var`, `object`,  Correct (A)             Explain compile-time vs
  `dynamic`                                       runtime binding

  4 --- boxing/unboxing   Correct (B)             Trace original, boxed,
                                                  and unboxed values
                                                  separately

  5 --- default, `ref`,   Correct (C)             Explain each
  `out`                                           parameter's effect

  6 --- nullable and `??` Correct (C)             Remember fallback
                                                  expressions are
                                                  evaluated when executed

  7 --- array references  Correct (B)             Array assignment copies
  and enums                                       a reference

  8 --- class vs struct   Correct (C)             Value copy vs reference
                                                  copy

  9 --- collection        Incorrect (selected B;  Choose `HashSet<T>` for
  selection               best answer C)          unique membership when
                                                  no associated value is
                                                  required

  10 --- equality/hash    Correct (C)             Equal values must have
  code                                            equal hash codes
  -----------------------------------------------------------------------

## 14. Revision priorities

1.  Re-explain diagnostic 2 without notes: mutating a shared object
    versus reassigning one reference.
2.  Re-explain diagnostic 9: why a `HashSet<int>` is more appropriate
    than a `Dictionary<int, string>` when only uniqueness and membership
    are required.
3.  Practise the difference between `var`, `object`, and `dynamic`.
4.  Implement a custom equality type and test it in a `HashSet<T>`.
5.  Complete the exercises before marking the chapter complete.

------------------------------------------------------------------------

## Next topic

**Advanced C# --- Generics, Collections, LINQ, Delegates, Async/Await,
Threading, CLR, Garbage Collection, and Disposal.**

Do not mark this chapter complete until the exercises and
self-assessment have been attempted independently.
