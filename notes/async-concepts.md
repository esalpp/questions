# Async Concepts

- [Catching exceptions in async void event handlers](#catching-exceptions-in-async-void-event-handlers)
- [SynchronizationContext](#synchronizationcontext)
- [ValueTask](#valuetask)
- [CancellationTokenSource vs CancellationToken](#cancellationtokensource-vs-cancellationtoken)

---

## Catching exceptions in async void event handlers

Put a `try/catch` **inside** the handler, around everything, including the `await`s. The caller can't catch these exceptions, but the handler can.

```csharp
private async void SaveButton_Click(object sender, RoutedEventArgs e)
{
    SaveButton.IsEnabled = false;            // stop double-clicks
    try
    {
        await SaveAsync();
        StatusText.Text = "Saved";
    }
    catch (OperationCanceledException)
    {
        StatusText.Text = "Cancelled";        // not really an error
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Save failed");
        MessageBox.Show($"Could not save: {ex.Message}");
    }
    finally
    {
        SaveButton.IsEnabled = true;
    }
}
```

This works because the try/catch is part of the same async method (the same state machine).

**Catching from outside doesn't work:** an `async void` method has no `Task` to carry the exception back to the caller.

**Better:** keep the handler thin, and put the logic in an `async Task` method that you can await and test.

**Where an uncaught exception goes:** to the SynchronizationContext that was active when the handler **started**. That applies even after `ConfigureAwait(false)`, and even for exceptions thrown before the first `await`.

| Started on | Goes to | Result |
|---|---|---|
| UI thread (WPF / WinForms) | the UI thread | a global UI handler can catch it and keep running |
| No context (console, thread pool, ASP.NET Core) | the thread pool | **the process crashes** |

**Global safety nets (WPF):**

```csharp
DispatcherUnhandledException += (s, args) => { Log(args.Exception); args.Handled = true; };
AppDomain.CurrentDomain.UnhandledException += (s, args) => Log((Exception)args.ExceptionObject); // log only
TaskScheduler.UnobservedTaskException += (s, args) => { Log(args.Exception); args.SetObserved(); };
```

**WinForms:** call `Application.SetUnhandledExceptionMode(UnhandledExceptionMode.CatchException)` and subscribe to `Application.ThreadException` before creating any forms.

- `AppDomain.UnhandledException` can't save the app; it's only for logging.
- `TaskScheduler.UnobservedTaskException` is for un-awaited faulted Tasks, not async void.

**Fire-and-forget helper:**

```csharp
public static async void FireAndForget(this Task task, Action<Exception> onError)
{
    try { await task; }
    catch (Exception ex) { onError(ex); }
}
```

---

## SynchronizationContext

It answers **"when this code continues later, which thread or place should it run on?"**

UI frameworks only allow the UI thread to touch controls. `SynchronizationContext` is the general way to "come back" to that thread.

```csharp
public class SynchronizationContext
{
    public virtual void Post(SendOrPostCallback d, object? state);  // queue it; don't wait
    public virtual void Send(SendOrPostCallback d, object? state);  // run it and wait
    public static SynchronizationContext? Current { get; }
}
```

| Environment | `SynchronizationContext.Current` |
|---|---|
| WPF | `DispatcherSynchronizationContext` (UI thread) |
| WinForms | `WindowsFormsSynchronizationContext` (UI thread) |
| MAUI / WinUI | a UI-thread context |
| Blazor | a renderer context (one piece at a time) |
| Classic ASP.NET | `AspNetSynchronizationContext` (one thread at a time per request) |
| ASP.NET Core, console, thread pool | **null**, so code continues on the thread pool |

**What `await` does:** if the task isn't finished, it captures `Current` and later `Post`s the rest of the method to it. If the task is already complete, it simply continues synchronously.

```csharp
private async void LoadButton_Click(object sender, EventArgs e)
{
    string data = await LoadDataAsync();   // captures the UI context; the thread is free while waiting
    resultLabel.Text = data;               // back on the UI thread, so this is safe
}
```

**When it matters:**
1. Updating the UI after `await`.
2. `ConfigureAwait(false)`: don't capture the context. Use it in library code.
3. **The deadlock:**

```csharp
private void Button_Click(object sender, RoutedEventArgs e)
{
    string text = LoadAsync().Result;   // the UI thread blocks
}
async Task<string> LoadAsync()
{
    await Task.Delay(1000);             // captures the UI context
    return "done";                      // needs the UI thread, which is blocked: deadlock
}
```

Fix it with `await` all the way up, or `ConfigureAwait(false)` inside `LoadAsync`.

4. `async void` exceptions are posted to it.

| | Answers |
|---|---|
| SynchronizationContext | *where* code continues |
| TaskScheduler | where a Task runs (`TaskScheduler.FromCurrentSynchronizationContext()`) |
| ExecutionContext | *what data* travels with the code (`AsyncLocal`, culture) |

---

## ValueTask

**Problem:** `Task<T>` is a class, so a call usually allocates, even when the result is already available (a cache hit).

**`ValueTask<T>`** is a struct that holds **either** the result directly (no allocation) **or** a `Task<T>` / pooled object when it really had to wait.

```csharp
public async ValueTask<User> GetUserAsync(int id)
{
    if (_cache.TryGetValue(id, out var user))
        return user;                       // no allocation
    user = await _db.Users.FindAsync(id);  // truly async
    _cache[id] = user;
    return user;
}

User user = await GetUserAsync(5);         // the caller code is the same as with Task
```

**The catch:** it may be backed by a pooled object that's reused after you await it. Think of a restaurant buzzer vs a receipt.

```csharp
var u = await GetUserAsync(5);             // ✅ await once, immediately

var vt = GetUserAsync(5);
await vt; await vt;                        // ❌ never await twice
// ❌ don't await it concurrently, or read .Result before it completes

Task<User> t = GetUserAsync(5).AsTask();   // ✅ convert if you need more
await Task.WhenAll(GetUserAsync(1).AsTask(), GetUserAsync(2).AsTask());
```

| | `Task<T>` | `ValueTask<T>` |
|---|---|---|
| Kind | class | struct |
| Allocates on a synchronous result | usually | no |
| Await more than once | ✅ | ❌ |
| `WhenAll` | ✅ | after `.AsTask()` |
| Default choice | **yes** | hot paths only |

You'll see it in `Stream.ReadAsync(Memory<byte>)`, `IAsyncEnumerator.MoveNextAsync`, `IAsyncDisposable.DisposeAsync` and channels.

**Use it when:** the method is called very often, usually completes synchronously, and allocations matter.

> "`ValueTask<T>` is a struct that avoids allocating a `Task` when an async method can return synchronously. It must be awaited only once, so I default to `Task<T>`."

---

## CancellationTokenSource vs CancellationToken

**The source is the controller, not a token.** It creates tokens and is the only thing that can cancel. The token is a read-only signal.

| | `CancellationTokenSource` | `CancellationToken` |
|---|---|---|
| Role | decides *when* to cancel | lets code *notice* it |
| Can cancel | ✅ `Cancel()`, `CancelAfter()` | ❌ |
| Can check | | ✅ `IsCancellationRequested`, `ThrowIfCancellationRequested()`, `Register()` |
| Kind | class; dispose it | small struct pointing to its source |

Analogy: the fire alarm switch (the source) vs the bells in every room (the tokens).

```csharp
using var cts = new CancellationTokenSource();
Task work = DownloadFilesAsync(urls, cts.Token);   // pass the TOKEN
cancelButton.Click += (_, _) => cts.Cancel();

async Task DownloadFilesAsync(IEnumerable<string> urls, CancellationToken token)
{
    foreach (var url in urls)
    {
        token.ThrowIfCancellationRequested();
        await _http.GetStringAsync(url, token);
    }
}
```

**Source features:**
- `new CancellationTokenSource(TimeSpan)` / `CancelAfter`: timeouts.
- `CreateLinkedTokenSource(a, b)`: cancelled when any of them is.
- `CancelAsync()` (.NET 8+).

```csharp
using var cts = CancellationTokenSource.CreateLinkedTokenSource(ctx.RequestAborted);
cts.CancelAfter(TimeSpan.FromSeconds(10));
```

**Usually you only receive tokens:** `HttpContext.RequestAborted`, the `stoppingToken` in `BackgroundService`, or your method's parameter.

**Mistakes to avoid:**
- Expecting a forced stop (it's cooperative).
- Not passing the token down.
- Not disposing the source.
- Reusing a cancelled source.
- Logging cancellation as an error.
