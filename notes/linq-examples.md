# LINQ Query Examples

Results were worked out by hand from this sample data (not run).

```csharp
record Employee(int Id, string Name, string Dept, decimal Salary, int? ManagerId);
record Department(string Code, string Name);
record Order(int Id, int EmployeeId, decimal Amount, DateTime Date);

var employees = new List<Employee>
{
    new(1, "Ann", "IT",    9000, null),
    new(2, "Bob", "IT",    7000, 1),
    new(3, "Cem", "HR",    5000, 1),
    new(4, "Dee", "IT",    7000, 1),
    new(5, "Eve", "Sales", 6000, 3),
    new(6, "Fay", "HR",    5500, 3),
};
var departments = new List<Department>
{
    new("IT", "Engineering"), new("HR", "Human Resources"),
    new("Sales", "Sales"),    new("Ops", "Operations"),   // Ops has no employees
};
var orders = new List<Order>
{
    new(101, 5, 1200, new(2024, 1, 15)),
    new(102, 5,  800, new(2024, 2, 3)),
    new(103, 2,  300, new(2024, 2, 20)),
    new(104, 5,  500, new(2024, 3, 9)),
};
```

## Filter, project, sort

```csharp
employees.Where(e => e.Dept == "IT" && e.Salary >= 7000);   // Ann, Bob, Dee

employees.Select(e => e.Name);                               // Ann, Bob, Cem, Dee, Eve, Fay
employees.Select(e => new { e.Name, Annual = e.Salary * 12 });
employees.Select((e, i) => $"{i + 1}. {e.Name}");            // "1. Ann", ...

var high = from e in employees
           let annual = e.Salary * 12
           where annual > 80000
           select new { e.Name, annual };                    // Ann, Bob, Dee

employees.OrderByDescending(e => e.Salary).ThenBy(e => e.Name);
// Ann 9000, Bob 7000, Dee 7000, Eve 6000, Fay 5500, Cem 5000
```

Use `ThenBy`, not a second `OrderBy`.

## Picking one item and quantifiers

```csharp
employees.First(e => e.Dept == "HR");             // Cem
employees.FirstOrDefault(e => e.Dept == "Ops");   // null
employees.Single(e => e.Id == 3);                 // Cem
employees.Single(e => e.Dept == "IT");            // throws (more than one match)
employees.Any(e => e.Salary > 8000);              // true
employees.All(e => e.Salary >= 5000);             // true
```

| | No match | More than one match |
|---|---|---|
| `First` | throws | first |
| `FirstOrDefault` | default | first |
| `Single` | throws | throws |
| `SingleOrDefault` | default | throws |

Use `Any()`, not `Count() > 0`.

## Aggregation

```csharp
employees.Count();                       // 6
employees.Sum(e => e.Salary);            // 39500
employees.Average(e => e.Salary);        // 6583.33...
employees.Max(e => e.Salary);            // 9000 (the value)
employees.MaxBy(e => e.Salary);          // Ann (the item, .NET 6+)
new[] { 1, 2, 3, 4 }.Aggregate((a, x) => a * x);   // 24
```

## Grouping

```csharp
employees.GroupBy(e => e.Dept)
         .Select(g => new { Dept = g.Key, Count = g.Count(), Total = g.Sum(e => e.Salary) });
// IT 3 23000, HR 2 10500, Sales 1 6000

employees.GroupBy(e => e.Dept).Where(g => g.Count() > 1).Select(g => g.Key);   // IT, HR (like HAVING)

employees.GroupBy(e => new { e.Dept, e.Salary }).Where(g => g.Count() > 1);    // IT 7000: Bob, Dee

employees.CountBy(e => e.Dept);                                                  // .NET 9+
employees.AggregateBy(e => e.Dept, 0m, (sum, e) => sum + e.Salary);             // .NET 9+
```

## Joins

```csharp
// Inner join
from e in employees
join d in departments on e.Dept equals d.Code
select new { e.Name, Department = d.Name };

// Group join: Engineering 3, Human Resources 2, Sales 1, Operations 0
departments.GroupJoin(employees, d => d.Code, e => e.Dept,
    (d, emps) => new { d.Name, Count = emps.Count() });

// Left join: Operations -> (none)
from d in departments
join e in employees on d.Code equals e.Dept into g
from e in g.DefaultIfEmpty()
select new { Dept = d.Name, Employee = e?.Name ?? "(none)" };
// .NET 10+: departments.LeftJoin(employees, d => d.Code, e => e.Dept, (d, e) => ...)

// Self join (key types must match exactly)
from e in employees
join m in employees on e.ManagerId equals (int?)m.Id
select new { e.Name, Manager = m.Name };   // Bob-Ann, Cem-Ann, Dee-Ann, Eve-Cem, Fay-Cem
```

## Flatten, sets, paging, conversion

```csharp
new[] { "hello world", "linq is fun" }.SelectMany(s => s.Split(' '));   // hello, world, linq, is, fun

int[] a = { 1, 2, 3, 4 }, b = { 3, 4, 5 };
a.Union(b);  a.Intersect(b);  a.Except(b);  a.Concat(b);   // 1-5 | 3,4 | 1,2 | keeps duplicates
employees.DistinctBy(e => e.Dept);                       // Ann, Cem, Eve

employees.OrderBy(e => e.Id).Skip(2).Take(2);            // page 2: Cem, Dee
nums.TakeWhile(n => n % 2 == 1);  Enumerable.Range(1, 7).Chunk(3);

employees.ToDictionary(e => e.Id);                       // throws on duplicate keys
employees.ToLookup(e => e.Dept)["Ops"];                  // empty; never throws
mixed.OfType<string>();                                  // skips other types; Cast<string>() throws
people.Zip(scores, (p, s) => $"{p}:{s}");                // stops at the shorter sequence
```

## Interview-style problems

```csharp
// Second-highest distinct salary: 7000
employees.Select(e => e.Salary).Distinct().OrderByDescending(s => s)
         .Skip(1).Select(s => (decimal?)s).FirstOrDefault();

// Highest paid per department: Ann, Fay, Eve
employees.GroupBy(e => e.Dept).Select(g => g.MaxBy(e => e.Salary));

// Top 2 per department
employees.GroupBy(e => e.Dept).SelectMany(g => g.OrderByDescending(e => e.Salary).Take(2));

// Duplicate values: 7000
employees.GroupBy(e => e.Salary).Where(g => g.Count() > 1).Select(g => g.Key);

// Word frequency
text.Split(' ').GroupBy(w => w)
    .Select(g => new { Word = g.Key, Count = g.Count() })
    .OrderByDescending(x => x.Count).ThenBy(x => x.Word);

// Employees with no orders: Ann, Cem, Dee, Fay
var withOrders = orders.Select(o => o.EmployeeId).ToHashSet();
employees.Where(e => !withOrders.Contains(e.Id));
employees.ExceptBy(orders.Select(o => o.EmployeeId), e => e.Id);   // .NET 6+

// Best seller: Eve, 2500
orders.GroupBy(o => o.EmployeeId)
      .Select(g => new { EmployeeId = g.Key, Total = g.Sum(o => o.Amount) })
      .OrderByDescending(x => x.Total)
      .Join(employees, x => x.EmployeeId, e => e.Id, (x, e) => new { e.Name, x.Total })
      .First();
```

## Method vs query syntax

Both compile to the same thing. Query syntax only covers some operators, but reads better for joins and `let`.

## On EF Core

The same queries are translated into SQL. Custom C# methods usually can't be translated. Project with `Select`, use `AsNoTracking()` for reads, and remember that nothing runs until `ToList` / `First` / `Count`.
