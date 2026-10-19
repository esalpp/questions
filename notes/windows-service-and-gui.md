# Windows Service + GUI App: Refresher

Notes for talking about a Windows service that runs on devices and communicates with a desktop GUI app.

Practice quiz: [`windows-service-quiz.html`](../windows-service-quiz.html)

## 1. Windows services

**What it is:** a background process with no UI, managed by the **Service Control Manager (SCM)**. It can start at boot before anyone logs in, runs under a service account, and Windows can restart it if it crashes.

> One-line interview definition: "A Windows service is a background process with no UI, managed by the Service Control Manager. It can start at boot before anyone logs in, runs under a service account, and can be restarted automatically if it fails."

### Key facts

- **Session 0 isolation (since Vista):** all services run in session 0, separate from the users' desktops (session 1, 2, ...). A service **can't show UI**. This is also a security measure against "shatter attacks". That's why the system has a separate GUI app plus IPC.
- **Startup must be quick:** a classic `ServiceBase.OnStart` should start a background task and return. If it takes longer than about 30 seconds, you get **error 1053** ("the service did not respond in a timely fashion").
- **Current directory is `C:\Windows\System32`:** relative paths break. Use `Path.Combine(AppContext.BaseDirectory, "file")`.
- **Own profile and registry:** the service runs as its own account, so it has its own AppData and `HKEY_CURRENT_USER`, and no access to the user's mapped drives. Put data shared with the GUI in `C:\ProgramData\Company\...` (`SpecialFolder.CommonApplicationData`).

### Service accounts

| Account | Rights |
|---|---|
| LocalSystem | full control of the machine; acts as the computer account on the network |
| LocalService | minimal local rights; anonymous on the network |
| NetworkService | minimal local rights; acts as the computer account on the network |
| Virtual account (`NT SERVICE\Name`) | per-service identity, minimal rights (a good default) |
| gMSA / user account | a domain account, when it needs specific network resources |

Principle: **least privilege**. Be ready to explain which account yours used and why.

### Start types

| Type | Meaning |
|---|---|
| Automatic | starts during boot |
| Automatic (Delayed Start) | starts shortly after the other automatic services |
| Manual | starts only when something asks it to |
| Disabled | can't be started |

### Old vs new style

**.NET Framework (`ServiceBase`):**

```csharp
public partial class DeviceService : ServiceBase
{
    private CancellationTokenSource? _cts;
    private Task? _worker;

    protected override void OnStart(string[] args)
    {
        _cts = new CancellationTokenSource();
        _worker = Task.Run(() => RunAsync(_cts.Token));   // start the work and return right away
    }

    protected override void OnStop()
    {
        _cts?.Cancel();                                   // ask the work to stop
        _worker?.Wait(TimeSpan.FromSeconds(20));          // give it time to finish cleanly
    }
}
```

**Modern .NET (Worker Service):** uses the Generic Host, so you get dependency injection, configuration, logging and graceful shutdown.

```csharp
// Program.cs
var builder = Host.CreateApplicationBuilder(args);
builder.Services.AddWindowsService(o => o.ServiceName = "My Device Service");
builder.Services.AddHostedService<DeviceWorker>();
builder.Build().Run();

// DeviceWorker.cs
public sealed class DeviceWorker(ILogger<DeviceWorker> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromSeconds(30));
        try
        {
            do
            {
                await CheckDevicesAsync(stoppingToken);
            }
            while (await timer.WaitForNextTickAsync(stoppingToken));
        }
        catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
        {
            logger.LogInformation("Service stopping");     // normal stop
        }
        catch (Exception ex)
        {
            logger.LogCritical(ex, "Service crashed");
            Environment.Exit(1);   // non-zero exit so the SCM recovery actions restart us
        }
    }
}
```

`AddWindowsService()` (package `Microsoft.Extensions.Hosting.WindowsServices`) reports start and stop to the SCM, turns "service stopping" into a cancelled `stoppingToken`, sets the content root to the exe folder and adds Event Log logging.

**Gotcha:** since .NET 6, an exception in `ExecuteAsync` makes the host log it and stop *cleanly*. Windows sees a normal stop, so the recovery actions don't run. That's why the code calls `Environment.Exit(1)`.

### Install, update, recover

```powershell
# Create. In PowerShell, "sc" is an alias for Set-Content, so type sc.exe; a space after "=" is required
sc.exe create MyService binPath= "C:\App\MyService.exe" start= auto
New-Service -Name MyService -BinaryPathName "C:\App\MyService.exe" -StartupType Automatic

# Delayed start
sc.exe config MyService start= delayed-auto

# Restart after 5 s, 5 s, then 60 s; reset the failure count after one day
sc.exe failure MyService reset= 86400 actions= restart/5000/restart/5000/restart/60000

# Update: stop -> replace files -> start
sc.exe stop MyService
sc.exe start MyService

sc.exe delete MyService      # uninstall
```

Real products usually use an installer (an MSI built with WiX, Inno Setup and so on). Old .NET Framework projects may use `installutil.exe` with a `ProjectInstaller` class.

### Debugging

- A Worker Service run from Visual Studio (not started by the SCM) simply runs as a console app.
- Attach to the installed service with **Debug → Attach to Process**, running Visual Studio as administrator.
- Put `System.Diagnostics.Debugger.Launch();` at the start for bugs during startup.
- Old trick for `ServiceBase`: if `Environment.UserInteractive` is true, call `OnStart` yourself instead of `ServiceBase.Run`.
- On real devices you rely on **logging**: files (Serilog, NLog), the Event Log, or a central log system.

## 2. Service ↔ GUI communication (IPC)

### Options

| Option | Notes |
|---|---|
| **Named pipes** | local, fast, secure; you design the messages (e.g. JSON lines). The most common choice. |
| WCF (`net.pipe`) | typed contracts and callbacks over pipes; .NET Framework (CoreWCF on modern .NET) |
| gRPC | typed contracts and streaming; over localhost TCP or named pipes (.NET 8+) |
| Local HTTP / SignalR | Kestrel inside the service; easy to test |
| Memory-mapped files | shared memory; fastest for big data, needs your own synchronization |
| Named events / mutexes | signals only ("data ready"), no data |
| Files / registry | simple, but needs polling; slow and race-prone |
| `ServiceController` | start/stop/status, plus custom command numbers 128–255 (`OnCustomCommand`) |

Window messages (`WM_COPYDATA`) don't work, because a service in session 0 has no window on the user's desktop.

### Details that show real experience

**Pipe permissions:** a pipe created by LocalSystem gives other users only *read* access by default, so a normal-user GUI connecting with read/write gets `UnauthorizedAccessException`. That's why it can work for admins and fail for normal users. Fix it with an explicit ACL:

```csharp
var security = new PipeSecurity();
security.AddAccessRule(new PipeAccessRule(
    new SecurityIdentifier(WellKnownSidType.AuthenticatedUserSid, null),
    PipeAccessRights.ReadWrite, AccessControlType.Allow));
security.AddAccessRule(new PipeAccessRule(
    new SecurityIdentifier(WellKnownSidType.LocalSystemSid, null),
    PipeAccessRights.FullControl, AccessControlType.Allow));

var server = NamedPipeServerStreamAcl.Create(     // System.IO.Pipes.AccessControl
    "MyCompany.DeviceService", PipeDirection.InOut,
    NamedPipeServerStream.MaxAllowedServerInstances,
    PipeTransmissionMode.Byte, PipeOptions.Asynchronous,
    0, 0, security);
```

**`Global\` prefix:** named mutexes, events, semaphores and memory-mapped files are **per session** by default (`Local\`). The service (session 0) and the GUI (session 1) need `Global\Name`. Let the service create the object and the GUI open it. Named **pipes** are machine-wide and don't need the prefix.

**Message framing:** a byte-mode pipe is a stream, so one write can arrive in several reads. Use one JSON message per line (`WriteLineAsync` / `ReadLineAsync`) or a length prefix. Windows also offers `PipeTransmissionMode.Message`.

**Multiple clients:** each `NamedPipeServerStream` is one connection. Run an accept loop: create an instance, wait for a client, hand it to its own task, then create the next instance.

**Push vs poll:** keep the connection open, so the GUI sends requests and the service also pushes events on it. Give every message a type:

```
{"type":"getStatus"}                                      GUI -> service (request)
{"type":"status","devices":[{"id":"A1","online":true}]}   service -> GUI (reply)
{"type":"deviceDisconnected","id":"A1"}                   service -> GUI (pushed event)
```

**Resilience:** the service never depends on the GUI. It keeps the current state, and when a GUI connects it sends a **snapshot**, then updates. The GUI detects a broken pipe (an `IOException`, or `ReadLineAsync` returning null), shows "disconnected", and retries with a timeout.

**Security:** the service is a SYSTEM-level endpoint, which makes it a privilege-escalation risk. Restrict the ACL, accept only known commands, validate all input, never run paths or commands taken from messages, and identify callers (`pipe.GetImpersonationUserName()`).

**WCF:** a `[ServiceContract]` interface with `[OperationContract]` methods, plus a `CallbackContract` for duplex. Modern .NET ships only WCF clients. For hosting, use CoreWCF, or switch to gRPC, plain pipes or HTTP.

### Pipe server sketch (service)

```csharp
while (!stoppingToken.IsCancellationRequested)
{
    var pipe = CreatePipe();                          // with the ACL above
    await pipe.WaitForConnectionAsync(stoppingToken);
    _ = HandleClientAsync(pipe, stoppingToken);       // serve it; loop to accept the next
}

async Task HandleClientAsync(NamedPipeServerStream pipe, CancellationToken ct)
{
    await using (pipe)
    {
        try
        {
            using var reader = new StreamReader(pipe, leaveOpen: true);
            using var writer = new StreamWriter(pipe, leaveOpen: true) { AutoFlush = true };
            while (await reader.ReadLineAsync(ct) is { } line)          // null = client disconnected
            {
                var request = JsonSerializer.Deserialize<Request>(line);
                await writer.WriteLineAsync(JsonSerializer.Serialize(Handle(request)));
            }
        }
        catch (IOException) { /* client vanished */ }
        catch (Exception ex) { logger.LogError(ex, "Pipe client failed"); }
    }
}
```

### Pipe client sketch (WPF GUI)

```csharp
private async Task RunPipeClientAsync(CancellationToken ct)    // started from Window_Loaded
{
    while (!ct.IsCancellationRequested)
    {
        try
        {
            await using var pipe = new NamedPipeClientStream(
                ".", "MyCompany.DeviceService", PipeDirection.InOut, PipeOptions.Asynchronous);
            await pipe.ConnectAsync(5000, ct);         // timeout: never hang forever
            StatusText.Text = "Connected";

            using var reader = new StreamReader(pipe);
            while (await reader.ReadLineAsync(ct) is { } line)
                StatusText.Text = JsonSerializer.Deserialize<StatusUpdate>(line)?.Message;  // UI thread
        }
        catch (TimeoutException) { StatusText.Text = "Service not running. Retrying..."; }
        catch (IOException)      { StatusText.Text = "Connection lost. Retrying..."; }

        await Task.Delay(TimeSpan.FromSeconds(3), ct);
    }
}
```

## 3. GUI side

- Controls may only be touched on the **UI thread**. From another thread, WPF throws "The calling thread cannot access this object because a different thread owns it", and WinForms throws "Cross-thread operation not valid".
- Ways back to the UI thread: `Dispatcher.Invoke` (WPF), `Control.Invoke` / `BeginInvoke` (WinForms), `await` started from the UI thread (SynchronizationContext), or `IProgress<T>` / `Progress<T>`.
- Never block the UI thread on I/O. Use `ConnectAsync(timeout, token)`, not `Connect()`.
- Start at login: the `HKCU\...\CurrentVersion\Run` registry key, the Startup folder, or a scheduled task. (A service *can* launch a process in the user's session with `WTSQueryUserToken` + `CreateProcessAsUser`, but that's advanced.)
- One instance per user: a named `Mutex` **without** `Global\`. With `Global\`, only one instance is allowed on the whole machine.

## 4. Likely interview questions about this project

1. Walk me through the architecture. (Draw it: device ↔ service (session 0) ⇄ pipe ⇄ GUI (user session).)
2. Why a separate service and GUI? (Availability at boot, session 0, privileges, reliability.)
3. How did they communicate, and why that method?
4. What happens if the service restarts, or the GUI isn't running?
5. How was it secured?
6. How was it installed and updated?
7. How did you log and debug it?
8. What was the hardest bug? (Answer with STAR.)
9. Which account did the service run as?
10. What would you change if you built it again?
