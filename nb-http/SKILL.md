---
name: nb-http
description: Helps write and validate NBomber HTTP flows, meaning C# load test scenarios that send HTTP requests with NBomber and NBomber.Http, in a .csproj project or a C# file-based app (dotnet run app.cs). Use it whenever the user writes a new NBomber HTTP test, or asks to review, validate, check or fix an existing one, even if they don't say "NBomber.Http".
license: MIT
metadata:
  author: NBomber
  version: 0.1.0
  category: testing
  tags: [load-testing, performance-testing, nbomber, http, csharp, dotnet]
---

# NBomber HTTP

Use this skill to write new NBomber HTTP flows, or to validate existing ones.

- **Writing a flow:** make the new code pass every check below.
- **Validating a flow:** go through every check below. Report each one as passed or failed. For each failed check, say where the problem is (file and line) and propose the exact change. Don't change the code until the user agrees.

## NBomber.Http version

This skill assumes NBomber.Http 6.3.0 or later. Earlier versions behave differently in ways the checks rely on:

- `WithJsonBody(data)` ignores `Http.GlobalJsonSerializerOptions` and keeps the C# property names (PascalCase, `{"UserId":1}`).
- `Http.Send<T>` treats a 2xx response whose body isn't valid JSON as a success, with `Data = null`.
- In C#, `Http.Send<T>(client, clientArgs, request)` returns `HttpResponse<object>` instead of `HttpResponse<T>`.

When you write a flow, use the latest stable NBomber.Http. When you validate one, check the version first: if the `.csproj` or the `#:package` directive references a version older than 6.3.0, report it and propose upgrading to the latest stable version.

## Checks

### 1. Server garbage collection

NBomber generates load from many concurrent tasks in one process. With the default workstation GC, garbage collection in the load generator can slow it down, so the test measures the client instead of the API. Server GC runs one heap and one GC thread per CPU core, and keeps the load generator fast.

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

### 2. HttpClient comes from Http.CreateDefaultClient()

Every `HttpClient` the test creates must come from `Http.CreateDefaultClient()` (namespace `NBomber.Http.CSharp`):

```csharp
var httpClient = Http.CreateDefaultClient();
```

This method configures the client for the highest throughput on the load-generating machine. Its `SocketsHttpHandler` uses these settings:

- `MaxConnectionsPerServer = 5000`. That's a reasonable limit for load testing, especially from a single node.
- `PooledConnectionLifetime = 10 minutes` and `PooledConnectionIdleTimeout = 5 minutes`, so pooled connections are reused and then recycled.
- `HttpClient.Timeout = 1 minute`.

Any other way of creating the client fails the check, for example `new HttpClient()`, `new HttpClient(new SocketsHttpHandler { ... })` or `IHttpClientFactory`. Propose replacing it with `Http.CreateDefaultClient()`.

`Http.CreateDefaultClient(maxConnectionsPerServer: N)` with another value also passes. This parameter exists to limit the number of connections on purpose.

`Http.CreateDefaultClient()` exists only when the project targets .NET 8 or later. If the project targets an older framework, report that and propose moving to .NET 8 or later.

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
- **A check inside a step's lambda that protects later lines in the same lambda.** For example, `if (response.IsError) return response;` after `Http.Send`, before the code reads `response.Payload.Value.Data`. The step hasn't ended yet at that point. Only report it when nothing in the lambda follows it except `return response;`.
- **A check on an `Http.Send` result in a scenario without steps.** Without `Step.Run` nothing restarts the iteration, so the code after the request runs either way.

### 4. JSON request bodies use WithJsonBody

NBomber.Http has a helper for JSON requests. `WithJsonBody` serializes the object with `System.Text.Json`, and it puts `Content-Type: application/json; charset=utf-8` on the body:

```csharp
var user = new UserData { UserId = 1, Title = "anton" };

var request = Http.CreateRequest("POST", "https://nbomber.com")
    .WithJsonBody(user);

var response = await Http.Send(httpClient, request);
```

`WithJsonBody(data)` serializes with `Http.GlobalJsonSerializerOptions`, which is `JsonSerializerOptions.Web` by default, so property names are camelCase: `{"userId":1,"title":"anton"}`. If the API needs other settings (for example snake_case names), set `Http.GlobalJsonSerializerOptions` once for the whole test, or pass options for one request: `.WithJsonBody(data, options)`.

Code that builds a JSON body by hand fails the check. Propose `.WithJsonBody(data)` instead:

- **Serializing by hand:** `JsonSerializer.Serialize(data)` plus `.WithBody(new StringContent(json, Encoding.UTF8, "application/json"))`. This works, but it's boilerplate that `WithJsonBody` already does.
- **Content-Type set with `WithHeader`:** `.WithHeader("Content-Type", "application/json")` plus `.WithBody(new StringContent(json))`. This is a bug, not only boilerplate. `WithHeader` adds request headers only, and `Content-Type` is a content header, so it's dropped silently. The `StringContent` then goes out as `text/plain; charset=utf-8`, and the API typically answers 415 Unsupported Media Type.

Passing a JSON string or an `HttpContent` to `WithJsonBody` also fails the check. `WithJsonBody` serializes whatever it gets. A JSON string becomes a quoted JSON string (`"{\"userId\":1}"`), and a `StringContent` becomes `{"Headers":[...]}`. Propose passing the object itself.

Don't report a JSON body that exists only as text, for example a payload loaded from a file or a recorded request. With no object to serialize, `.WithBody(new StringContent(json, Encoding.UTF8, "application/json"))` is correct.

### 5. JSON responses use Http.Send<T>

To read a JSON response, use `Http.Send<T>`. It sends the request and deserializes the body into `T` with `Http.GlobalJsonSerializerOptions`, or with `clientArgs.JsonSerializerOptions` when you pass `HttpClientArgs`:

```csharp
var response = await Http.Send<UserData>(httpClient, request);

UserData user = response.Payload.Value.Data;              // the deserialized body
HttpResponseMessage raw = response.Payload.Value.Response; // status, headers
```

`Http.Send<T>` reports the result like this:

| Response | Result | `Data` |
|---|---|---|
| 2xx with a JSON body | OK | the deserialized value |
| 204 No Content, or an empty body | OK | `default(T)` |
| 2xx with a body that isn't valid JSON for `T` | failed. The status code is kept, and `Message` contains the deserialization error. | `default(T)` |
| non-2xx | failed. The body isn't deserialized. Read `Response.Content` to see it. | `default(T)` |

Code that reads a JSON response by hand fails the check. Propose `Http.Send<T>` instead:

- `Http.Send(...)` followed by `ReadAsStringAsync()` and `JsonSerializer.Deserialize<T>(...)`, or by `ReadFromJsonAsync<T>()`.
- `JsonDocument.Parse(...)` to pick out a few fields. Propose a small record with only the fields the test needs, for example `record CreatedItem(int Id);`.

Hand-written parsing doesn't turn an invalid body into a failed request. It either throws (reported as `-101` with the exception message) or keeps going with missing data. `Http.Send<T>` reports the request as failed, keeps the real status code, and says why.

Don't report code that needs the body as text, for example to check an HTML page or to compare the exact response text.

**Check the fields that later steps use.** Valid JSON with different field names than `T` doesn't cause an error. The response is OK, and the missing properties keep their default values (`0`, `null`). If a later step uses a field from `Data`, for example an id in the next URL, the step that reads it must check it and fail otherwise:

```csharp
await Step.Run("POST /api/items", context, async () =>
{
    var response = await Http.Send<CreatedItem>(httpClient, request);

    if (!response.IsError && response.Payload.Value.Data?.Id is null or 0)
        return Response.Fail<HttpResponse<CreatedItem>>(
            statusCode: response.StatusCode, message: "response has no id", sizeBytes: response.SizeBytes);

    itemId = response.Payload.Value.Data!.Id;
    return response;
});
```

`Response.Fail<HttpResponse<CreatedItem>>` has the same type as the response from `Http.Send<CreatedItem>`. All return paths of a step must have the same type, otherwise `Step.Run` can't infer it (error CS0411). Passing `sizeBytes: response.SizeBytes` keeps the data transfer stats correct.

If a later step uses a field from `Data` and nothing checks it, report it and propose a check like the one above. Don't report a null check on `Data`: it's still needed, because 204 responses and empty bodies return `Data = default(T)`.
