# Face-to-Face Interview Prep

Expect about half the time on **your own experience** and half on **quick C# and .NET concepts** explained out loud.

## Part 1: Your experience (prepare these the most)

**"Tell me about yourself."** About 90 seconds: who you are now → main experience → one thing you're proud of → why this job.

> "I'm a .NET developer with X years of experience, mostly building [...] with C#. At [company] I worked on [product], where I [main responsibility]. Something I'm proud of is [one concrete result]. I'm looking for [what this role offers]."

**"Walk me through a project."** Cover what the product did, the stack, the architecture, **your** part ("I", not only "we"), and one problem you solved with its result. Practise drawing it.

**Likely follow-ups:**
- "Why X instead of Y?"
- "How did you deploy it?"
- "How did you test it?"
- "How did you handle errors and logging?"
- "What would you change if you built it again?"

**Behavioral questions.** Answer with **STAR**: Situation → Task → Action (most of the answer) → Result.

| Question | What they're checking |
|---|---|
| The hardest bug you fixed | how you debug: logs, reproduce, narrow down, fix, prevent |
| A performance problem you solved | profiling, N+1 queries, indexes, caching |
| A disagreement with a teammate | respectful, focused on data, open to being wrong |
| A mistake you made | ownership plus what you learned |
| A tight deadline | prioritizing, communicating early |
| Learning something new quickly | docs, small prototypes, asking |
| How your team worked | Scrum or Kanban, code reviews, git flow |

Prepare 3–4 real stories; one story can answer several questions.

**"Why are you leaving?" / "Why this company?"** Stay positive: growth, new challenges, their product or tech.

## Part 2: C# basics

| Question | Short answer |
|---|---|
| Value vs reference type? | Value types hold the data and are copied (`int`, `struct`, `DateTime`). Reference types hold a reference to a heap object, so copies share it (`class`, `string`, arrays). |
| `class` vs `struct`? | Class: reference type with inheritance. Struct: value type, no inheritance, for small immutable data. |
| `record`? | A class with value-based equality, `with` copies and a nice `ToString`. |
| Interface vs abstract class? | Interface: a contract with no state; implement many. Abstract class: state and shared code; inherit only one. |
| `const` vs `readonly`? | `const` is compile-time and copied into callers. `readonly` is set once at run time. |
| Why is `string` immutable? | Safe to share and to use as a key; changes create new strings. Use `StringBuilder` in loops. |
| Boxing? | A value type copied to the heap as `object`. It allocates; generics avoid it. |
| `==` vs `Equals`? | `==` is picked at compile time (reference comparison for classes, unless overridden). `Equals` is virtual. |
| `ref` / `out` / `in`? | `ref`: read and write. `out`: must be assigned by the method. `in`: read-only by reference. |
| Delegates and events? | A delegate is a type-safe function pointer. An event can only be `+=` / `-=` from outside. |
| Extension methods? | Static methods with `this` on the first parameter. That's how LINQ works. |
| Generics and constraints? | Type-safe reuse; `where T : class, new()`. |
| Nullable reference types? | Compile-time null warnings only. |
| `IDisposable` / `using`? | Deterministic cleanup; `Dispose()` runs even on exceptions. |
| Garbage collection? | Generational (0/1/2); large objects (85 KB+) go on the LOH. |
| `throw` vs `throw ex`? | `throw;` keeps the stack trace. |

**OOP pillars:** encapsulation, inheritance, polymorphism (`virtual`/`override`), abstraction.

## Part 3: Async and LINQ

- `async/await` frees the thread while waiting for I/O; it doesn't create threads by itself.
- `Task` vs `Thread`: a unit of work vs an OS thread.
- Avoid `async void` except for event handlers.
- `.Result` / `.Wait()` block a thread and can deadlock where there's a SynchronizationContext.
- `Task.WhenAll` runs independent operations concurrently.
- CancellationToken: cooperative cancellation.
- `lock` vs `SemaphoreSlim` vs `Mutex`: in-process vs async-friendly vs cross-process.
- `IEnumerable` vs `IQueryable`: in-memory delegates vs an expression tree translated to SQL.
- Deferred execution: the query runs only when enumerated.

## Part 4: ASP.NET Core and web APIs

| Question | Short answer |
|---|---|
| Middleware? | A pipeline; order matters (authentication before authorization). |
| DI lifetimes? | Transient, Scoped (per request, e.g. DbContext), Singleton. |
| Captive dependency? | A singleton holding a scoped service. Fix it with `IServiceScopeFactory`. |
| Controllers vs Minimal APIs? | Structure and filters vs lightweight. |
| Global error handling? | Exception middleware / `UseExceptionHandler`, returning `ProblemDetails`. |
| Authentication vs authorization? | Who you are vs what you may do (JWT + `[Authorize]`). |
| Configuration? | `appsettings.json`, environment variables and secrets, with the options pattern. |
| PUT vs PATCH vs POST? | Replace vs partial update vs create. |
| Status codes | 200, 201, 204, 400, 401, 403, 404, 409, 500 |
| `HttpClient`? | Use `IHttpClientFactory`. |
| API versioning? | URL, header or query string. |
| Caching? | `IMemoryCache`, Redis / `IDistributedCache`, output caching. |

## Part 5: Databases and EF Core

- EF Core vs Dapper: productivity and tracking vs speed and raw SQL.
- N+1 problem: fix it with `Include` or a projection.
- `AsNoTracking`: faster read-only queries.
- Migrations, indexes (clustered vs non-clustered), transactions / ACID.
- SQL injection: always use parameters.
- Finding slow queries: execution plan, indexes, logged SQL.

## Part 6: Design

**SOLID:** Single responsibility, Open/closed, Liskov substitution, Interface segregation, Dependency inversion.

**Patterns:** Repository, Unit of Work, Factory, Strategy, Singleton (vs DI), Decorator, Observer, Mediator, CQRS.

Also be ready for: clean / onion architecture, unit vs integration tests, what makes code "good", and monolith vs microservices.

## Part 7: Tools and process

- Git: branching, merge vs rebase, conflicts, pull requests.
- CI/CD: what your pipeline did, and with which tool.
- Docker: image vs container.
- Code reviews.
- Debugging in production: logs, correlation IDs.

## Part 8: Questions to ask them

- "What does a typical day or sprint look like?"
- "What's the tech stack, and are there plans to change it?"
- "What would success look like in the first 3 months?"
- "What's the team's biggest challenge right now?"
- "How do code reviews and testing work here?"

## Tips

- **Think out loud.** If you don't know, say how you'd find out.
- **Short answer first, then offer more.**
- **Connect answers to your own experience.**
- It's fine to pause, or to ask them to clarify.
