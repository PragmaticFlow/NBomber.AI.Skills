---
name: nb-http
description: Helps write and validate NBomber HTTP flows, meaning C# load test scenarios that send HTTP requests with NBomber and NBomber.Http, in a .csproj project or a C# file-based app (dotnet run app.cs). Use it whenever the user writes a new NBomber HTTP test, or asks to review, validate, check or fix an existing one, even if they don't say "NBomber.Http". It covers the HTTP checks; use it together with nb-core, which covers the checks for every NBomber test (garbage collection, warm-up, failed steps).
license: MIT
metadata:
  author: NBomber
  version: 0.2.0
  category: testing
  tags: [load-testing, performance-testing, nbomber, http, csharp, dotnet]
---

# NBomber HTTP

Use this skill to write new NBomber HTTP flows, or to validate existing ones.

- **Writing a flow:** make the new code pass every check below.
- **Validating a flow:** go through every check below. Report each one as passed or failed. For each failed check, say where the problem is (file and line) and propose the exact change. Don't change the code until the user agrees.

## NBomber.Http version

Use NBomber.Http 7.0.0 or later. This skill and its checks are written for the 7.0.0 API:

```xml
<PackageReference Include="NBomber.Http" Version="7.0.0" />
```

```csharp
#:package NBomber.Http@7.0.0
```

Earlier versions behave differently in ways the checks rely on:

- **Sending requests.** Version 7.0.0 adds the extension methods `httpClient.SendRequest(request)` and `httpClient.SendRequest<T>(request)`. The static `Http.Send(client, request)` and `Http.Send<T>(...)` methods still work, but they are obsolete and cause compiler warning CS0618 (see check 6). Earlier versions have only `Http.Send`.
- **`Http.CreateDefaultClient()` defaults.** In 7.0.0 the client uses HTTP/2 when the server supports it, spreads new connections over all IP addresses of the host, recycles pooled connections after 3 minutes, and closes idle connections after 1.5 minutes. Earlier versions use only HTTP/1.1, send all connections to the first IP address, recycle connections after 10 minutes, close idle connections after 5 minutes, and accept only the `maxConnectionsPerServer` parameter (see check 1).
- **Compression.** Version 7.0.0 adds request body compression (`WithCompression`, and the `compression` parameter of `WithJsonBody`) and response decompression (the `automaticDecompression` parameter of `Http.CreateDefaultClient`). Earlier versions have neither (see check 7).
- **Data transfer.** Version 7.0.0 counts the size of requests and responses as HTTP/1.1 sends them, and counts a compressed body at its compressed size. Earlier versions count differently, so the data transfer stats of the same test can change after the upgrade.
- **JSON request bodies.** In 7.0.0, `WithJsonBody(data)` serializes with `Http.GlobalJsonSerializerOptions`, so property names are camelCase (`{"userId":1}`, see check 2). In earlier versions, `WithJsonBody(data)` ignores `Http.GlobalJsonSerializerOptions` and keeps the C# property names (PascalCase, `{"UserId":1}`).
- **JSON responses.** In 7.0.0, `SendRequest<T>` reports a 2xx response whose body isn't valid JSON as failed, and doesn't deserialize the body of a non-2xx response (see check 3). In earlier versions, `Http.Send<T>` treats a 2xx response with invalid JSON as a success, with `Data = null`, and deserializes the body of a non-2xx response too.
- **Response type.** In 7.0.0, `SendRequest<T>` returns `Response<HttpResponse<T>>`. In earlier versions, the C# method `Http.Send<T>(client, clientArgs, request)` returns `HttpResponse<object>` instead of `HttpResponse<T>`.

When you write a flow, reference NBomber.Http 7.0.0 or a later version. When you validate one, check the version first: if the `.csproj` or the `#:package` directive references a version older than 7.0.0, report it and propose upgrading to 7.0.0 or later. Upgrading changes the client defaults above, so tell the user that the results can differ from earlier runs.

If the user must compare the new results with runs of an earlier version, propose these settings. They bring back the earlier behavior:

| Earlier behavior | Setting in 7.0.0 |
|---|---|
| HTTP/1.1 only | `Http.CreateDefaultClient(httpVersion: "1.1")` |
| Connections recycled after 10 minutes | `pooledConnectionLifetime: TimeSpan.FromMinutes(10)` |
| All connections to the first IP address | `dnsSelection: DnsSelection.Default` |
| PascalCase names in JSON request bodies | `.WithJsonBody(data, new JsonSerializerOptions())` |

The idle timeout and the data transfer counting have no setting. The idle timeout matters only when connections stay idle for more than 1.5 minutes, for example in a test with a very low request rate.

The JSON names can also break requests, not only change results. If the API matches property names case-sensitively and expects PascalCase, the camelCase body from 7.0.0 loses its values. In that case PascalCase is a requirement. Set it once for the whole test, and keep the other web defaults, so that `SendRequest<T>` still reads camelCase responses (see checks 2 and 3):

```csharp
Http.GlobalJsonSerializerOptions = new JsonSerializerOptions(JsonSerializerOptions.Web)
{
    PropertyNamingPolicy = null   // keep the C# property names
};
```

## The nb-core checks

The checks below are only about HTTP. An NBomber HTTP test must also pass the checks of the nb-core skill, which apply to every NBomber test:

1. Server garbage collection.
2. The scenario has a useful warm-up.
3. No unnecessary IsError checks after steps.

When you write or validate an HTTP flow, also follow the nb-core skill, and report its checks in the same list as the checks below. If the nb-core skill isn't installed, tell the user that these checks were skipped.

## Checks

### 1. HttpClient comes from Http.CreateDefaultClient()

Every `HttpClient` the test creates must come from `Http.CreateDefaultClient()` (namespace `NBomber.Http.CSharp`):

```csharp
var httpClient = Http.CreateDefaultClient();
```

This method configures the client for the highest throughput on the load-generating machine. Every setting has an optional parameter with this default:

| Parameter | Default | What it does |
|---|---|---|
| `maxConnectionsPerServer` | `5000` | The maximum number of connections to one endpoint. That's a reasonable limit for load testing, especially from a single node. |
| `pooledConnectionLifetime` | 3 minutes | How long a pooled connection is used before it's closed. Idle connections close after 1.5 minutes. |
| `dnsSelection` | `DnsSelection.Random` | Which IP address a new connection uses when the host name resolves to several addresses. |
| `httpVersion` | `"2.0"` | The HTTP version of every request. |
| `versionPolicy` | `HttpVersionPolicy.RequestVersionOrLower` | Lets the client fall back to a lower HTTP version. |
| `automaticDecompression` | `DecompressionMethods.None` | Which compressed response bodies the client decompresses. |
| `requestTimeout` | 1 minute | `HttpClient.Timeout`. `Timeout.InfiniteTimeSpan` disables it. |

The `DnsSelection` enum is in the `NBomber.Http` namespace. `DecompressionMethods` and `HttpVersionPolicy` are in `System.Net` and `System.Net.Http`.

**DNS changes.** A pooled connection stays open to the IP address that DNS returned when the connection opened. After `pooledConnectionLifetime` the handler closes it, and the next connection resolves the host name again. So one long-lived client (see check 4) moves to the new address within 3 minutes when DNS changes, for example after a blue-green deployment, a failover, or when a load balancer changes its IPs. A plain `new HttpClient()` has no connection lifetime (`PooledConnectionLifetime` is infinite by default), so it keeps sending requests to the old address for the whole test.

**Several IP addresses.** When the host name resolves to several addresses (DNS round-robin), .NET usually opens all connections to the first address, so one node gets the whole load. With `DnsSelection.Random` (the default) or `DnsSelection.RoundRobin`, the client spreads new connections over all addresses, as many independent users do. If a connection to the selected address fails, the client tries the next addresses.

**HTTP version.** By default the client uses HTTP/2 for `https` URLs when the server supports it, and HTTP/1.1 for `http` URLs and for servers without HTTP/2. HTTP/2 sends many requests in parallel over one connection, and the client opens more connections when the server's stream limit is reached. If the real clients of the API use HTTP/1.1, pass `httpVersion: "1.1"`. A request with its own version from `.WithVersion("1.1")` keeps that version.

**HTTP/2 without TLS (h2c).** An `http` URL has no TLS, so the client can't negotiate the version with the server. With the default `versionPolicy`, the client then uses HTTP/1.1 and reports no error. Some services accept HTTP/2 over plain `http`, for example a gRPC service or an internal service behind a proxy. To test such a service with HTTP/2, pass the exact version:

```csharp
var httpClient = Http.CreateDefaultClient(
    httpVersion: "2.0",
    versionPolicy: HttpVersionPolicy.RequestVersionExact);
```

With `RequestVersionExact`, a request fails if the server doesn't support that version. A wrong setting then shows up as failed requests, not as a test that silently uses HTTP/1.1. If a test of an `http` URL is meant to use HTTP/2 (for example the code, its comments or the scenario name say so), and the client doesn't use `RequestVersionExact`, report it: the test runs over HTTP/1.1. For HTTP/3, pass `httpVersion: "3.0"`. HTTP/3 needs QUIC support in the operating system and on the server.

**Compressed responses.** The client doesn't decompress response bodies by default, and doesn't send `Accept-Encoding`. If the API sends compressed responses and the test reads the body (for example with `SendRequest<T>`, see check 3), pass `automaticDecompression: DecompressionMethods.GZip | DecompressionMethods.Brotli`. The client then decompresses the body itself, and the reported size still counts the compressed bytes that came over the network. `Deflate` isn't supported and is ignored. Check 7 has the compression rules.

Any other way of creating the client fails the check, for example `new HttpClient()` or `new HttpClient(new SocketsHttpHandler { ... })`. Propose replacing it with `Http.CreateDefaultClient()`, and map the custom settings to its parameters:

| Custom setting | Parameter of `Http.CreateDefaultClient` |
|---|---|
| `MaxConnectionsPerServer = N` | `maxConnectionsPerServer: N` |
| `PooledConnectionLifetime = t` | `pooledConnectionLifetime: t` |
| `httpClient.Timeout = t` | `requestTimeout: t` |
| `DefaultRequestVersion = v` | `httpVersion: "1.1"` or `"2.0"`. Read the note below the table. |
| `DefaultVersionPolicy = p` | `versionPolicy: p`. Read the note below the table. |
| `AutomaticDecompression = m` | `automaticDecompression: m`. With the .NET setting, the reported size is the decompressed size, not the bytes received. |

**`DefaultRequestVersion` on a custom client has no effect.** NBomber sends each request with `SendAsync(HttpRequestMessage)`. That method uses the version of the request, not the `DefaultRequestVersion` and `DefaultVersionPolicy` of the client. A request from `Http.CreateRequest` has version 1.1. So a custom client with `DefaultRequestVersion = HttpVersion.Version20` actually sends HTTP/1.1, unless the code sets the version on each request (`.WithVersion(...)` or `request.Version`). `Http.CreateDefaultClient()` sets the version on every request. After the change the test really uses HTTP/2 (for `https` URLs, when the server supports it), so the results can differ from earlier runs. Tell the user. To keep the HTTP/1.1 behavior, propose `httpVersion: "1.1"`.

If the custom handler has a setting that has no parameter (for example a client certificate or a proxy), keep the custom handler and report it as a suggestion only. Name the defaults from the first table that it doesn't have.

A client from `IHttpClientFactory` doesn't fail the check. The factory manages the handlers and their lifetime itself, so it isn't a bad practice. But report it as a suggestion and propose `Http.CreateDefaultClient()`: the factory's default handler doesn't have the load-testing settings above, and the test doesn't need the DI setup that the factory requires.

`Http.CreateDefaultClient(...)` with any of the parameters above also passes. They exist to change the defaults on purpose, for example `maxConnectionsPerServer` to limit the number of connections, or `httpVersion: "1.1"` to test like HTTP/1.1 clients. Setting `httpClient.Timeout` on a client from `Http.CreateDefaultClient()` also works. Report it as a suggestion and propose the `requestTimeout` parameter.

**HttpMetricsPlugin.** If the test registers `new HttpMetricsPlugin([HttpVersion.Version1])` (or `new HttpMetricsPlugin()`, which also monitors only HTTP/1.1) and the client can use HTTP/2 (the default `httpVersion`, and `https` URLs), report it as a suggestion. The HTTP/2 connections don't show in the metrics. Propose `new HttpMetricsPlugin([HttpVersion.Version1, HttpVersion.Version2])`, or `httpVersion: "1.1"` if the test must use HTTP/1.1. This `HttpVersion` is the enum from `NBomber.Http`. If the file also has `using System.Net;`, the name is ambiguous (error CS0104), so write `NBomber.Http.HttpVersion.Version2`.

`Http.CreateDefaultClient()` exists only when the project targets .NET 8 or later. If the project targets an older framework, report that and propose moving to .NET 8 or later.

### 2. JSON request bodies use WithJsonBody

NBomber.Http has a helper for JSON requests. `WithJsonBody` serializes the object with `System.Text.Json`, and it puts `Content-Type: application/json; charset=utf-8` on the body:

```csharp
var user = new UserData { UserId = 1, Title = "anton" };

var request = Http.CreateRequest("POST", "https://nbomber.com")
    .WithJsonBody(user);

var response = await httpClient.SendRequest(request);
```

`WithJsonBody(data)` serializes with `Http.GlobalJsonSerializerOptions`, which is `JsonSerializerOptions.Web` by default, so property names are camelCase: `{"userId":1,"title":"anton"}`. If the API needs other settings (for example snake_case names), set `Http.GlobalJsonSerializerOptions` once for the whole test, or pass options for one request: `.WithJsonBody(data, options)`.

Code that builds a JSON body by hand fails the check. Propose `.WithJsonBody(data)` instead:

- **Serializing by hand:** `JsonSerializer.Serialize(data)` plus `.WithBody(new StringContent(json, Encoding.UTF8, "application/json"))`. This works, but it's boilerplate that `WithJsonBody` already does.
- **Content-Type set with `WithHeader`:** `.WithHeader("Content-Type", "application/json")` plus `.WithBody(new StringContent(json))`. This is a bug, not only boilerplate. `WithHeader` adds request headers only, and `Content-Type` is a content header, so it's dropped silently. The `StringContent` then goes out as `text/plain; charset=utf-8`, and the API typically answers 415 Unsupported Media Type.

Passing a JSON string or an `HttpContent` to `WithJsonBody` also fails the check. `WithJsonBody` serializes whatever it gets. A JSON string becomes a quoted JSON string (`"{\"userId\":1}"`), and a `StringContent` becomes `{"Headers":[...]}`. Propose passing the object itself.

Don't report a JSON body that exists only as text, for example a payload loaded from a file or a recorded request. With no object to serialize, `.WithBody(new StringContent(json, Encoding.UTF8, "application/json"))` is correct.

### 3. JSON responses use SendRequest<T>

To read a JSON response, use `httpClient.SendRequest<T>`. It sends the request and deserializes the body into `T` with `Http.GlobalJsonSerializerOptions`, or with the options you pass for one request: `httpClient.SendRequest<T>(request, jsonOptions: options)`:

```csharp
var response = await httpClient.SendRequest<UserData>(request);

UserData user = response.Payload.Value.Data;              // the deserialized body
HttpResponseMessage raw = response.Payload.Value.Response; // status, headers
```

`httpClient.SendRequest<T>` reports the result like this:

| Response | Result | `Data` |
|---|---|---|
| 2xx with a JSON body | OK | the deserialized value |
| 204 No Content, or an empty body | OK | `default(T)` |
| 2xx with a body that isn't valid JSON for `T` | failed. The status code is kept, and `Message` contains the deserialization error. | `default(T)` |
| non-2xx | failed. The body isn't deserialized. Read `Response.Content` to see it. | `default(T)` |

`SendRequest<T>` deserializes the body as it is received. If the API sends a compressed body (`Content-Encoding: gzip` or `br`) and the client doesn't decompress it, every request fails with a deserialization error. Create the client with `automaticDecompression` (see check 1).

Code that reads a JSON response by hand fails the check. Propose `httpClient.SendRequest<T>` instead:

- `httpClient.SendRequest(...)` followed by `ReadAsStringAsync()` and `JsonSerializer.Deserialize<T>(...)`, or by `ReadFromJsonAsync<T>()`.
- `JsonDocument.Parse(...)` to pick out a few fields. Propose a small record with only the fields the test needs, for example `record CreatedItem(int Id);`.

Hand-written parsing doesn't turn an invalid body into a failed request. It either throws (reported as `-101` with the exception message) or keeps going with missing data. `httpClient.SendRequest<T>` reports the request as failed, keeps the real status code, and says why.

Don't report code that needs the body as text, for example to check an HTML page or to compare the exact response text.

**Check the fields that later steps use.** Valid JSON with different field names than `T` doesn't cause an error. The response is OK, and the missing properties keep their default values (`0`, `null`). If a later step uses a field from `Data`, for example an id in the next URL, the step that reads it must check it and fail otherwise:

```csharp
await Step.Run("POST /api/items", context, async () =>
{
    var response = await httpClient.SendRequest<CreatedItem>(request);

    if (!response.IsError && response.Payload.Value.Data?.Id is null or 0)
        return Response.Fail<HttpResponse<CreatedItem>>(
            statusCode: response.StatusCode, message: "response has no id", sizeBytes: response.SizeBytes);

    itemId = response.Payload.Value.Data!.Id;
    return response;
});
```

`Response.Fail<HttpResponse<CreatedItem>>` has the same type as the response from `httpClient.SendRequest<CreatedItem>`. All return paths of a step must have the same type, otherwise `Step.Run` can't infer it (error CS0411). Passing `sizeBytes: response.SizeBytes` keeps the data transfer stats correct.

If a later step uses a field from `Data` and nothing checks it, report it and propose a check like the one above. Don't report a null check on `Data`: it's still needed, because 204 responses and empty bodies return `Data = default(T)`.

### 4. The HttpClient is reused, not created per request

Create the `HttpClient` once, outside the scenario lambda, and use it for all requests of the scenario:

```csharp
var httpClient = Http.CreateDefaultClient();

var scenario = Scenario.Create("http_scenario", async context =>
{
    var request = Http.CreateRequest("GET", "https://nbomber.com");
    return await httpClient.SendRequest(request);
});
```

`HttpClient` is safe for concurrent requests, so all copies of the scenario can use the same instance. Its handler keeps a pool of open connections, and the requests reuse them.

**Good practice: one shared client for each scenario.** All copies of the scenario use the same client. The only exception is a session that lives in cookies. One client keeps one set of cookies, so each copy then needs its own client (see check 5). A token in a header doesn't need its own client: each copy sends its token with each request on the shared client (see check 5).

Several scenarios can share one client, but a separate client for each scenario is better. Each client has its own connection pool and its own cookies. So the connections, the `maxConnectionsPerServer` limit and the cookies of one scenario don't affect the other scenarios.

A client that is created inside the scenario lambda fails the check, because the lambda runs on every iteration:

```csharp
var scenario = Scenario.Create("http_scenario", async context =>
{
    using var httpClient = Http.CreateDefaultClient();   // a new client on every iteration
    var request = Http.CreateRequest("GET", "https://nbomber.com");
    return await httpClient.SendRequest(request);
});
```

This causes two problems:

- **Wrong measurements.** Each new client opens new connections. Every iteration then includes a TCP handshake and a TLS handshake, so the test measures connection setup in addition to the API.
- **Socket exhaustion.** When a client is disposed, the operating system keeps its sockets in the `TIME_WAIT` state for some time. Under high load the load generator runs out of free ports, and requests start to fail with socket errors.

The same applies to a client created inside a `Step.Run` lambda, or in a helper method that the scenario calls on each iteration. Propose creating the client once, outside the scenario lambda, and using that instance in the scenario.

**Don't dispose the client during the test.** Disposing it is costly, and later requests on a disposed client throw `ObjectDisposedException`. Code that disposes the shared client inside the scenario lambda fails the check. Disposing it after `NBomberRunner.Run()` returns, or in `WithClean`, is correct.

Don't report these cases:

- **A dedicated client for each copy of the scenario,** created once and stored in `context.ScenarioInstanceData` (see check 5). Only fail the check when the code creates a new client on every iteration instead of reading the stored one. If the scenario doesn't keep a session in cookies, report the dedicated clients as a suggestion and propose one shared client. Each copy then opens its own connections, so the test opens many more connections and does many more TLS handshakes than it needs.
- **A client created in `WithInit`** and kept in a variable or field outside the scenario lambda.
- **One client shared by several scenarios.** It works, so it doesn't fail the check. Mention the separate client for each scenario only if the scenarios need different settings or cookies.

### 5. Each virtual user has its own session

If the scenario simulates users that log in, each copy of the scenario must send its own credentials. Where the session lives decides how:

- **A token in a header** (JWT, OAuth bearer token): all copies share one client. Each copy keeps its own token and sends it with each request.
- **A cookie** (server-side session, cookie authentication): each copy needs its own client, because one client keeps one set of cookies.

To keep per-copy state, use `context.ScenarioInstanceData`. This dictionary belongs to one copy of the scenario and keeps its values between iterations.

**Token in a header.** A header added to a request goes only with that request, and `HttpClient` is safe for concurrent requests. So the copies share one client (check 4), and each copy sends its own token:

```csharp
record LoginResult(string Token);

var httpClient = Http.CreateDefaultClient();

var scenario = Scenario.Create("user_session", async context =>
{
    if (!context.ScenarioInstanceData.ContainsKey("token"))
    {
        await Step.Run("login", context, async () =>
        {
            var request = Http.CreateRequest("POST", "https://nbomber.com/api/login")
                .WithJsonBody(new { login = "user", password = "pass" });

            var response = await httpClient.SendRequest<LoginResult>(request);
            if (response.IsError) return response;   // protects the lines below (nb-core check 3)

            var token = response.Payload.Value.Data?.Token;
            if (string.IsNullOrEmpty(token))
                return Response.Fail<HttpResponse<LoginResult>>(
                    statusCode: response.StatusCode, message: "response has no token", sizeBytes: response.SizeBytes);

            context.ScenarioInstanceData["token"] = token;   // the token of this copy only
            return response;
        });
    }

    var token = (string)context.ScenarioInstanceData["token"];

    await Step.Run("get_data", context, async () =>
    {
        var request = Http.CreateRequest("GET", "https://nbomber.com/api/data")
            .WithHeader("Authorization", $"Bearer {token}");

        return await httpClient.SendRequest(request);
    });

    return Response.Ok();
});
```

The shared client still stores the cookies from every response, even when the API uses tokens. If the API also sets cookies, the client sends the last cookie it got with the requests of all copies. Two examples:

- A load balancer affinity cookie (for example `AWSALB` or `ARRAffinity`) sends the whole load to one server node.
- A refresh token in a cookie refreshes the session of another user.

The test code usually doesn't show whether the API sets cookies, so don't fail the check for it. When the flow uses a shared client with token auth, tell the user in one line to look for `Set-Cookie` headers in the responses. If the API sets cookies, use a dedicated client for each copy, as shown below.

A token expires, for example a JWT after 5 to 60 minutes. If the test runs longer than the token lifetime, requests start to fail with 401 Unauthorized. If the flow logs in only once for each copy, report it as a suggestion: remove the stored token when a request returns 401, so that the next iteration logs in again.

**Session in a cookie.** Give each copy its own client, and store it in `context.ScenarioInstanceData`, so each copy creates its client once and reuses it:

```csharp
var scenario = Scenario.Create("user_session", async context =>
{
    if (!context.ScenarioInstanceData.TryGetValue("http_client", out var client))
    {
        client = Http.CreateDefaultClient();
        context.ScenarioInstanceData["http_client"] = client;   // store it before the login step
    }
    var httpClient = (HttpClient)client;

    if (!context.ScenarioInstanceData.ContainsKey("logged_in"))
    {
        await Step.Run("login", context, async () =>
        {
            var request = Http.CreateRequest("POST", "https://nbomber.com/api/login")
                .WithJsonBody(new { login = "user", password = "pass" });

            return await httpClient.SendRequest(request);   // the session cookies stay in this client
        });

        context.ScenarioInstanceData["logged_in"] = true;
    }

    await Step.Run("get_data", context, async () =>
    {
        var request = Http.CreateRequest("GET", "https://nbomber.com/api/data");
        return await httpClient.SendRequest(request);       // sends the cookies of this copy only
    });

    return Response.Ok();
});
```

Store the client right after you create it, before the login step. If the login fails, NBomber ends the iteration (see nb-core check 3). A client that isn't stored yet is lost, and the next iteration creates another one, which is the problem from check 4. Keep the login state in a separate key, so that a failed login repeats only the login.

These cases fail the check:

- **Sessions in a shared client.** The scenario logs in, the later requests don't send their own auth header (the session lives in cookies), and all copies use one shared client. Propose a dedicated client for each copy, as in the cookie example above.
- **`DefaultRequestHeaders` changed during the test.** The code changes `httpClient.DefaultRequestHeaders` inside the scenario lambda, for example to set `Authorization` after the login, on a client that the copies share. Changing `DefaultRequestHeaders` while requests are in progress isn't thread-safe, and all copies send the last value that was set. Propose sending the header with each request, `.WithHeader("Authorization", $"Bearer {token}")`, and storing the token in `context.ScenarioInstanceData`.
- **A token in a shared variable.** The login stores the token in a variable or field outside the scenario lambda, and all copies read it. All copies then send the token of the last login. Propose storing the token in `context.ScenarioInstanceData`, as in the token example above.
- **The client is stored only after the login step.** A failed login then creates a new client on every iteration. Propose storing the client right after it's created.

A dedicated client that isn't stored, and is created again on every iteration, fails check 4.

Don't report these cases:

- **A shared client with token auth,** where each copy stores its own token and sends it with each request.
- **A shared client without per-user sessions:** no login, or one token for the whole test, for example an API key, or a service token that `WithInit` gets once.

### 6. Requests are sent with SendRequest, not the obsolete Http.Send

In NBomber.Http 7.0.0, requests are sent with the extension methods on `HttpClient` (namespace `NBomber.Http.CSharp`):

```csharp
var response = await httpClient.SendRequest(request);
var typed    = await httpClient.SendRequest<UserData>(request);
```

The static `Http.Send` methods are obsolete. They still work, but each call causes compiler warning CS0618, and a project with `TreatWarningsAsErrors` doesn't build. Every `Http.Send(...)` or `Http.Send<T>(...)` call fails the check. Propose the replacement:

| Obsolete call | Replacement |
|---|---|
| `Http.Send(httpClient, request)` | `httpClient.SendRequest(request)` |
| `Http.Send<T>(httpClient, request)` | `httpClient.SendRequest<T>(request)` |
| `Http.Send(httpClient, clientArgs, request)` | `httpClient.SendRequest(request, logger: ..., traceId: ..., httpCompletion: ..., cancellationToken: ...)` |
| `Http.Send<T>(httpClient, clientArgs, request)` | `httpClient.SendRequest<T>(request, jsonOptions: ..., logger: ..., traceId: ..., httpCompletion: ..., cancellationToken: ...)` |

All named parameters are optional. When the code builds `HttpClientArgs.Create(...)`, move each value it sets to the parameter with the same meaning, and remove the `HttpClientArgs` variable if nothing else uses it. For example, tracing with `HttpClientArgs.Create(logger: context.Logger)` becomes `httpClient.SendRequest(request, logger: context.Logger)`. The logger writes the request and the response at the `Debug` level.

The method name is `SendRequest`, not `Send`, because `HttpClient` has its own `Send` and `SendAsync` methods, and C# calls those instead of an extension method with the same name. Don't propose `httpClient.Send(request)`: it calls the .NET method, which returns an `HttpResponseMessage` instead of an NBomber response.

The response is the same as before: `SendRequest` returns `Response<HttpResponseMessage>`, and `SendRequest<T>` returns `Response<HttpResponse<T>>`, so the code that reads `response.Payload.Value` doesn't change.

### 7. Compression uses the NBomber.Http helpers

NBomber.Http can compress request bodies and decompress response bodies. Use its helpers. Don't compress or decompress in the test code.

Use compression only when the real clients of the API use it. Compression changes the work on the server (CPU time to compress and decompress) and the number of bytes on the network. A test that compresses differently from the real clients measures a different load.

**Request compression.** To send a compressed body, pass `compression` to `WithJsonBody`, or call `WithCompression` after any other body:

```csharp
var request = Http.CreateRequest("POST", "https://nbomber.com/api/users")
    .WithJsonBody(user, compression: RequestCompression.GZip);

var upload = Http.CreateRequest("POST", "https://nbomber.com/api/text")
    .WithBody(new StringContent(text, Encoding.UTF8, "text/plain"))
    .WithCompression(RequestCompression.GZip);
```

The `RequestCompression` enum is in the `NBomber.Http` namespace. Its values are `None`, `GZip` (`Content-Encoding: gzip`) and `Brotli` (`Content-Encoding: br`). The helper compresses the body when the request is built, adds the `Content-Encoding` header, and keeps the other content headers, for example `Content-Type`. The body goes out with an exact `Content-Length`, not chunked, and the reported size counts the compressed bytes. A request without a body, or with `RequestCompression.None`, doesn't change.

The API must accept compressed bodies. For example, ASP.NET Core needs the request decompression middleware (`app.UseRequestDecompression()`). Without it, the API reads the compressed bytes as they are, and typically answers 400 Bad Request or 415 Unsupported Media Type.

These cases fail the check:

- **Compressing by hand.** The test writes the body to a `GZipStream` or a `BrotliStream`, and sends the bytes with `Content-Encoding` added. Propose `.WithJsonBody(data, compression: ...)` or `.WithCompression(...)`.
- **`Content-Encoding` set with `WithHeader`.** `.WithHeader("Content-Encoding", "gzip")` is dropped silently, because `Content-Encoding` is a content header (see check 2). The API gets the body without the header: compressed bytes that it can't read, or a body that was never compressed. Propose `.WithCompression(...)`.
- **`WithCompression` before the body.** `.WithCompression(...)` followed by `.WithBody(...)` or `.WithJsonBody(data)`. The new body replaces the compressed one, and the request goes out uncompressed. Propose calling `WithCompression` last, or passing `compression` to `WithJsonBody`.

Don't report a large body that doesn't change, compressed once outside the scenario lambda. The helper compresses the body again on every iteration, and at a high request rate a large body takes much CPU time on the load generator. Keeping the compressed bytes and creating a `new ByteArrayContent(bytes)` with `content.Headers.ContentEncoding.Add("gzip")` for each request is correct.

**Response decompression.** Create the client with `automaticDecompression` (see check 1):

```csharp
var httpClient = Http.CreateDefaultClient(
    automaticDecompression: DecompressionMethods.GZip | DecompressionMethods.Brotli);
```

The client then sends an `Accept-Encoding` header with these methods, and decompresses `gzip` and `br` response bodies. `SendRequest<T>` deserializes the decompressed body (see check 3). The reported size counts the compressed bytes that came over the network. `DecompressionMethods.Deflate` isn't supported: the flag is ignored, and a `deflate` body stays compressed.

These cases fail the check:

- **Decompressing by hand.** The test reads the body through a `GZipStream` or a `BrotliStream`, for example over `response.Content.ReadAsStreamAsync()`. Propose `automaticDecompression`, and `SendRequest<T>` for a JSON body.
- **`Accept-Encoding` without `automaticDecompression`.** The test sends `.WithHeader("Accept-Encoding", "gzip")`, and the client doesn't have `automaticDecompression`. The API sends compressed bodies that the client doesn't decompress, so `SendRequest<T>` fails every request with a deserialization error. Propose `automaticDecompression`, and remove the header: the client sends it itself.

If the test simulates browser traffic, and the API compresses its responses, a client without `automaticDecompression` is a suggestion only. Browsers always send `Accept-Encoding`. Without it, the API skips the compression work, and the test receives more bytes than real users do.
