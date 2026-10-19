# C# Q&A

## What is `yield`?

`yield return` turns a method into an **iterator**. It hands back a sequence one item at a time, pausing after each item.

```csharp
IEnumerable<int> Evens(int max)
{
    for (int i = 0; i <= max; i += 2)
        yield return i;      // hand back one value, then pause here
}
foreach (var n in Evens(10)) Console.Write(n + " ");  // 0 2 4 6 8 10
```

- The compiler builds a hidden state machine. Calling `Evens(10)` runs **none** of your code yet.
- Each `MoveNext()` runs the method to the next `yield return`. `yield break` ends the sequence.
- Benefits: memory (one item at a time), laziness (`Take(3)` stops early, so infinite sequences work), and it's how LINQ works.

## What does `FirstOrDefault` return when nothing is found?

It returns `default(T)`.

| Type | Result |
|---|---|
| a class | `null` |
| `int`, `double`, `decimal` | `0` |
| `bool` | `false` |
| `char` | `'\0'` |
| nullable (`int?`) | `null` |
| a struct | all fields zero |

**Trap:** with value types you can't tell "nothing found" from "found 0". Cast to a nullable type (`.Select(x => (char?)x).FirstOrDefault()`), or use the .NET 6+ overload with your own default: `FirstOrDefault(predicate, -1)`.

`First()` throws when nothing matches. `SingleOrDefault()` throws when more than one item matches.

## Space complexity of "first non-repeating character"

The dictionary holds one entry per distinct character: **O(k)**. That's O(n) in the worst case, or **O(1)** for a fixed alphabet like ASCII (`int[128]`). The LINQ `GroupBy` version is always O(n), because it stores every character.

> "O(n) time; O(k) space for k distinct characters, which is O(1) for a fixed character set."

## Does boxing make the `object` an `int`?

Two types are involved:
- **The variable** `o` is declared as `object`. That's all the compiler knows.
- **The heap object** really is an `Int32` inside a box.

```csharp
int x = 5;
object o = x;
o.GetType();          // System.Int32
o is int;             // true
int y = o + 1;        // compile error
int z = (int)o + 1;   // unbox: 6
if (o is int n) { }   // safe unboxing
long w = (long)o;     // InvalidCastException: must unbox to exactly int
```

The box holds a **copy**, so changing `x` later doesn't affect it.

## Why can an `IEnumerable` result be deferred?

**`IEnumerable` means "can be enumerated", not "has been enumerated".**

```csharp
public interface IEnumerable<out T> { IEnumerator<T> GetEnumerator(); }
public interface IEnumerator<out T> : IDisposable { T Current { get; } bool MoveNext(); }
```

It promises "you can walk me", not when the items are produced. `Where` returns a **recipe** (the source plus the filter) that runs when someone calls `MoveNext()`.

```csharp
var query = numbers.Where(n => { Console.WriteLine($"checking {n}"); return n > 1; });
Console.WriteLine("query created");   // printed first
foreach (var n in query) { }          // "checking ..." prints now
```

Precise rule: returning `IEnumerable<T>` means it **may** be lazy. A `List<T>`, array or single value means the work is definitely done.
