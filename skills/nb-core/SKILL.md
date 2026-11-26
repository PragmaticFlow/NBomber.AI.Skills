---
name: nb-core
description: Helps write and validate the basic setup of NBomber load tests in C#, in a .csproj project or a C# file-based app (dotnet run app.cs), for any protocol. It checks the garbage collection settings, the warm-up, and how the scenario handles failed steps. Use it whenever the user writes a new NBomber test, or asks to review, validate, check or fix an existing one, even if they don't name these settings. For HTTP tests, use it together with nb-http.
license: MIT
metadata:
  author: NBomber
  version: 0.1.0
  category: testing
  tags: [load-testing, performance-testing, nbomber, csharp, dotnet]
---

# NBomber Core

Use this skill to write new NBomber tests, or to validate existing ones. Its checks apply to every NBomber test, whatever protocol it uses. Protocol skills (for example nb-http) add their own checks on top of these.

- **Writing a test:** make the new code pass every check below.
- **Validating a test:** go through every check below. Report each one as passed or failed. For each failed check, say where the problem is (file and line) and propose the exact change. Don't change the code until the user agrees.

## Checks

### 1. Server garbage collection

NBomber generates load from many concurrent tasks in one process. With the default workstation GC, garbage collection in the load generator can slow it down, so the test measures the client instead of the system under test. Server GC runs one heap and one GC thread per CPU core, and keeps the load generator fast.

First find out how the test is built:

- **Project:** a `.csproj` file builds the test code.
- **C# file-based app:** a single `.cs` file that runs with `dotnet run <file>.cs`, with `#:package` directives and no `.csproj`.

**Project.** The `.csproj` must contain both properties:

```xml
<PropertyGroup>
  <ServerGarbageCollection>true</ServerGarbageCollection>
  <ConcurrentGarbageCollection>true</ConcurrentGarbageCollection>
</PropertyGroup>
```

The properties can also come from a `Directory.Build.props` file in the project folder or a parent folder. Look there before you report them as missing. A property that is missing or set to `false` fails the check.

**C# file-based app.** The `.cs` file must contain all three directives, at the top with the other `#:` directives:

```csharp
#:property PublishAot=false
#:property ServerGarbageCollection=true
#:property ConcurrentGarbageCollection=true
```

A directive that is missing or set to `false` fails the check.

File-based apps use workstation GC by default, the same as projects. They also turn on native AOT by default (`PublishAot=true`), so `dotnet publish app.cs` tries to build a native AOT app instead of a normal .NET app. NBomber runs as a normal .NET app, so `PublishAot=false` keeps it that way.

Concurrent GC is on by default in both kinds of app, but the check still requires it explicitly. Concurrent (background) GC does most gen2 work while the test keeps running. Without it, every full collection pauses all of NBomber's tasks, and those pauses show up as extra latency in the measured responses. The explicit setting makes this choice visible, and gives both kinds of app the same rule.

### 2. The scenario has a useful warm-up

Before the measured load, NBomber runs a warm-up phase. It isn't included in the reports. During the warm-up, .NET compiles the test code (JIT), the clients open their connections, and the system under test fills its caches and connection pools. Without a warm-up, the first seconds of the test measure this setup work instead of the steady state.

NBomber has these settings for the warm-up (namespace `NBomber.CSharp`):

| Setting | What it does |
|---|---|
| no setting | Warm-up of 30 seconds. |
| `.WithWarmUpDuration(t)` | Warm-up of `t`. |
| `.WithWarmUpSimulations(...)` | A separate load profile for the warm-up. It runs to its full planned duration, and the warm-up duration is ignored. Needs NBomber 6.7.0 or later. |
| `.WithoutWarmUp()` | No warm-up. |

Without `WithWarmUpSimulations`, the warm-up doesn't have its own load. It runs the scenario's `LoadSimulations` from the start, and stops after the warm-up duration. So the warm-up only runs the *first part* of the main load profile. Then the main load starts from the beginning again.

The JSON config can set the same values for a scenario, in `ScenariosSettings`: `WarmUpDuration` (for example `"00:00:30"`) and `WarmUpSimulationsSettings`. The config values replace the values from the code. If the test loads a config (`NBomberRunner.LoadConfig(...)`, or `--config` on the command line), check the config too.

#### Warm-up duration

No warm-up setting passes: the default is 30 seconds. `.WithWarmUpDuration(t)` passes when `t` is 5 seconds or more. A shorter warm-up fails the check, because it is too short for the JIT and the connections. Propose 5 seconds or more, or removing the call to keep the 30-second default.

The warm-up must not be longer than the main load. NBomber reports an error when the warm-up duration is longer than the scenario duration.

#### WithoutWarmUp

`.WithoutWarmUp()` is correct only for testing and development runs, for example:

- a check-only or debug switch that runs one iteration to see that the steps work (like `NBOMBER_CHECK_ONLY` from nb-load-test-creator),
- a smoke test that checks that the test code works, not the performance.

Don't report it in these cases.

`.WithoutWarmUp()` in a test that runs the real load fails the check. Propose removing it (the 30-second default) or `.WithWarmUpDuration(...)` of 5 seconds or more. Never propose `.WithoutWarmUp()` as a fix, for example to make a test start faster.

#### A very low initial load

Because the warm-up runs the start of the main load profile, a low start gives a warm-up that does nothing. For example, the main load starts with 1 request every 10 seconds. The default warm-up of 30 seconds then runs only 3 iterations: the JIT, the connections and the caches aren't warm when the measured load starts.

To find out, look at the load simulations during the warm-up window (the first 30 seconds, or the `WithWarmUpDuration` value). The check fails when that part of the load runs fewer than about 1 iteration per second on average, that is, fewer iterations than the warm-up has seconds. Examples:

- `Simulation.Inject(rate: 1, interval: TimeSpan.FromSeconds(10), ...)`: 0.1 iterations per second.
- `Simulation.Pause(...)` as the first simulation: no iterations.
- A `RampingInject` from 0 that ramps slowly, so the rate is still near zero at the end of the warm-up window. For example, a ramp to 20 per second over 10 minutes reaches only 1 per second after 30 seconds.
- A `RampingConstant` from 0 that still has 0 copies at the end of the warm-up window.

A `KeepConstant` or `RampingConstant` with 1 copy or more doesn't fail: each copy runs its iterations one after another.

Propose `.WithWarmUpSimulations(...)` with a short load that runs the scenario many times:

```csharp
var scenario = Scenario.Create("orders", async context => { ... })
    .WithWarmUpSimulations(
        Simulation.Inject(rate: 5, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromSeconds(30)))
    .WithLoadSimulations(
        Simulation.Inject(rate: 1, interval: TimeSpan.FromSeconds(10), during: TimeSpan.FromMinutes(30)));
```

For the warm-up rate, use the peak rate of the main load if it is 1 iteration per second or more. If the peak is lower, use a low rate such as 1 to 5 iterations per second. A warm-up much stronger than the main load can change the state of the system under test, for example fill the database with more data than the test creates. A duration of 5 to 30 seconds is usually enough.

When the scenario has `WithWarmUpSimulations`, NBomber ignores the warm-up duration. If the scenario also calls `.WithWarmUpDuration(...)`, NBomber logs a warning. Report it as a suggestion and propose removing the call.

`WithWarmUpSimulations` exists in NBomber 6.7.0 and later. If the `.csproj` or the `#:package` directive references an older version, report it and propose upgrading NBomber to 6.7.0 or later.

### 3. No unnecessary IsError checks after steps

By default, when a step returns a failed response or throws, NBomber stops the current iteration and starts the next one. The code after the failed `Step.Run` never runs. So a check like this one is unnecessary:

```csharp
var login = await Step.Run("login", context, async () => { ... });
if (login.IsError) return login;     // never true here: a failed step already ended the iteration

var profile = await Step.Run("get_profile", context, async () => { ... });
```

If the scenario doesn't call `.WithRestartIterationOnFail(shouldRestart: false)`, any code that runs only when a `Step.Run` result has `IsError == true` fails the check. Propose removing it.

Don't report these cases:

- **The scenario calls `.WithRestartIterationOnFail(shouldRestart: false)`.** The code after a failed step still runs, so the checks are needed. This setting exists only in code, not in the JSON config.
- **A check inside a step's lambda that protects later lines in the same lambda.** For example, `if (response.IsError) return response;` after a client call, before the code reads the response data. The step hasn't ended yet at that point. Only report it when nothing in the lambda follows it except `return response;`.
- **A check on a client call result in a scenario without steps** (for example `httpClient.SendRequest` called directly in the scenario lambda). Without `Step.Run` nothing restarts the iteration, so the code after the call runs either way.
