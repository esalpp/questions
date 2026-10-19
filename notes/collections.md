# .NET Collections Compared

## Interfaces

```
IEnumerable<T>        can be walked with foreach (that's all)
  └─ ICollection<T>   + Count, Add, Remove, Contains, Clear
       ├─ IList<T>         + access by index, Insert, RemoveAt
       ├─ ISet<T>          + unique items, set operations
       └─ IDictionary<K,V> + key → value lookup

Read-only: IReadOnlyCollection<T>, IReadOnlyList<T>, IReadOnlyDictionary<K,V>
```

Accept the most general interface you need, and return something specific. Careful: enumerating a lazy `IEnumerable<T>` parameter twice runs it twice.

## Everyday collections

| Collection | Built on | Index | Find | Add | Remove | Use it when |
|---|---|---|---|---|---|---|
| `T[]` | contiguous memory | O(1) | O(n) | fixed | fixed | the size is fixed |
| `List<T>` | growable array | O(1) | O(n) | O(1)* | O(n) | **the default** |
| `LinkedList<T>` | linked nodes | none | O(n) | O(1) at a node | O(1) at a node | rarely wins in practice |
| `Dictionary<K,V>` | hash table | none | **O(1)** | O(1)* | O(1) | lookup by key |
| `HashSet<T>` | hash table | none | **O(1)** | O(1)* | O(1) | unique items, fast `Contains` |
| `SortedDictionary<K,V>` | red-black tree | none | O(log n) | O(log n) | O(log n) | keys kept sorted |
| `SortedList<K,V>` | sorted arrays | O(1) | O(log n) | O(n) | O(n) | sorted, mostly read |
| `SortedSet<T>` | red-black tree | none | O(log n) | O(log n) | O(log n) | sorted unique items, Min/Max |
| `Queue<T>` | circular array | none | O(n) | O(1)* | O(1) | first in, first out |
| `Stack<T>` | array | none | O(n) | O(1)* | O(1) | last in, first out |
| `PriorityQueue<E,P>` | binary heap | none | none | O(log n) | O(log n) | lowest priority first |

\* **Amortized:** when the array is full it doubles in size (an O(n) copy), but averaged over all adds it's O(1). Pre-size with `new List<T>(capacity)`.

- `Dictionary` enumeration order **isn't guaranteed**.
- `List.Contains` is O(n) and `HashSet.Contains` is O(1). Switching can turn O(n²) into O(n).
- `LinkedList` causes cache misses and allocates on every add.

## Thread-safe collections (`System.Collections.Concurrent`)

| Collection | What it is |
|---|---|
| `ConcurrentDictionary<K,V>` | atomic `GetOrAdd` / `AddOrUpdate` (the factory may run twice) |
| `ConcurrentQueue<T>` / `ConcurrentStack<T>` | lock-free; `TryDequeue` / `TryPop` |
| `ConcurrentBag<T>` | unordered; best when the same thread adds and takes |
| `BlockingCollection<T>` | blocking producer/consumer (older) |
| `Channel<T>` | the modern **async** producer/consumer |

Single calls are atomic, but sequences of calls (check, then act) are not.

## Read-only vs immutable vs frozen

| Kind | Example | Can it change? |
|---|---|---|
| Read-only view | `AsReadOnly()`, `IReadOnlyList<T>` | yes, by the owner of the underlying list |
| Immutable | `ImmutableList<T>`, `ImmutableArray<T>` | never; "adding" returns a new collection |
| Frozen (.NET 8+) | `FrozenDictionary<K,V>` | never; slow to build, very fast to read |

## Legacy (avoid)

`ArrayList`, `Hashtable`, and the non-generic `Queue` and `Stack` box value types and aren't type-safe.

## Which one should I use?

- Lookup by key → `Dictionary` (sorted: `SortedDictionary`; built once: `FrozenDictionary`; shared between threads: `ConcurrentDictionary`)
- Unique items / fast contains → `HashSet` (sorted: `SortedSet`)
- Processing order → `Queue` / `Stack` / `PriorityQueue`
- Anything else → `List<T>` (or an array if the size is fixed)
