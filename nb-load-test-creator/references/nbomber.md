# NBomber code patterns

Packages: `NBomber` and `NBomber.Http`, always at their latest stable version. Add them with `dotnet add package` and never write a version number from memory. See "Dependencies: always the latest version" in SKILL.md.

The examples below use the NBomber 6 API. If the latest version has changed the API, fix the generated code to match it, and don't pin an older package.

## Full example

A scenario for `POST /pets`, `GET /pets/{petId}` and `DELETE /pets/{petId}`, with bearer auth and the Load preset:

```csharp
using System.Text;
using System.Text.Json;
using NBomber.CSharp;
using NBomber.Http.CSharp;

// Settings: change these to adjust the test
const string BaseUrl = "https://api.example.com/v1";
const int TargetRate = 50;                                  // scenario runs per second
var rampUp = TimeSpan.FromSeconds(30);
var duration = TimeSpan.FromMinutes(5);

var token = Environment.GetEnvironmentVariable("API_TOKEN")
    ?? throw new InvalidOperationException("Set the API_TOKEN environment variable before running the test.");

using var httpClient = new HttpClient();

var scenario = Scenario.Create("pets_api", async context =>
{
    string? petId = null;

    var createPet = await Step.Run("POST /pets", context, async () =>
    {
        var body = JsonSerializer.Serialize(new
        {
            name = $"pet-{Guid.NewGuid():N}",
            tag = "dog"                                     // TODO: generated value, replace with real data if needed
        });

        var request = Http.CreateRequest("POST", $"{BaseUrl}/pets")
            .WithHeader("Authorization", $"Bearer {token}")
            .WithHeader("Accept", "application/json")
            .WithBody(new StringContent(body, Encoding.UTF8, "application/json"));

        var response = await Http.Send(httpClient, request);

        if (!response.IsError)
        {
            var json = await response.Payload.Value.Content.ReadAsStringAsync();
            using var doc = JsonDocument.Parse(json);
            petId = doc.RootElement.GetProperty("id").ToString();
        }

        return response;
    });
    if (createPet.IsError) return createPet;

    var getPet = await Step.Run("GET /pets/{petId}", context, async () =>
    {
        var request = Http.CreateRequest("GET", $"{BaseUrl}/pets/{Uri.EscapeDataString(petId!)}")
            .WithHeader("Authorization", $"Bearer {token}")
            .WithHeader("Accept", "application/json");

        return await Http.Send(httpClient, request);
    });
    if (getPet.IsError) return getPet;

    var deletePet = await Step.Run("DELETE /pets/{petId}", context, async () =>
    {
        var request = Http.CreateRequest("DELETE", $"{BaseUrl}/pets/{Uri.EscapeDataString(petId!)}")
            .WithHeader("Authorization", $"Bearer {token}");

        return await Http.Send(httpClient, request);
    });

    return deletePet;
})
.WithWarmUpDuration(TimeSpan.FromSeconds(5))
.WithLoadSimulations(
    Simulation.RampingInject(rate: TargetRate, interval: TimeSpan.FromSeconds(1), during: rampUp),
    Simulation.Inject(rate: TargetRate, interval: TimeSpan.FromSeconds(1), during: duration)
);

NBomberRunner
    .RegisterScenarios(scenario)
    .Run();
```

Notes:

- `Http.Send` returns a failed response for any status code outside 200–299, which is the failure rule this skill uses. Don't turn failures into successes.
- Checking `IsError` after each step and returning early stops the current iteration, so chained steps don't run with a missing id.
- Share one `HttpClient` across the whole test. Creating one per request exhausts sockets under load.
- Build query strings with `Uri.EscapeDataString` for each value, for example `$"{BaseUrl}/pets?limit={Uri.EscapeDataString(limit)}"`.

## Run once before or after the test

`.WithInit` runs once before the load starts, and `.WithClean` runs once after it ends. Use them for setup and cleanup endpoints (database reset, seeding, deleting test data) instead of calling those endpoints on every iteration. Make them fail loudly, so a broken setup stops the test immediately:

```csharp
.WithInit(async context =>
{
    // Recreates the database tables so every test run starts clean.
    var response = await httpClient.PutAsync($"{BaseUrl}/api/Databases", null);
    response.EnsureSuccessStatusCode();
})
.WithClean(async context =>
{
    var response = await httpClient.DeleteAsync($"{BaseUrl}/api/TestData");
    response.EnsureSuccessStatusCode();
})
```

## Check-only switch

Read `NBOMBER_CHECK_ONLY` and, when it's `1`, run a single iteration with no warm-up instead of the real load profile. Init and cleanup still run, so the check covers them too:

```csharp
var checkOnly = Environment.GetEnvironmentVariable("NBOMBER_CHECK_ONLY") == "1";

var scenario = Scenario.Create("pets_api", async context => { /* steps */ })
    .WithInit(async context => { /* setup */ });

scenario = checkOnly
    ? scenario
        .WithoutWarmUp()
        .WithLoadSimulations(Simulation.IterationsForConstant(copies: 1, iterations: 1))
    : scenario
        .WithWarmUpDuration(TimeSpan.FromSeconds(5))
        .WithLoadSimulations(
            Simulation.RampingInject(rate: TargetRate, interval: TimeSpan.FromSeconds(1), during: rampUp),
            Simulation.Inject(rate: TargetRate, interval: TimeSpan.FromSeconds(1), during: duration));
```

Run the check with `$env:NBOMBER_CHECK_ONLY = "1"; dotnet run -c Release` (PowerShell) or `NBOMBER_CHECK_ONLY=1 dotnet run -c Release` (bash). In PowerShell the variable stays set for that terminal, so clear it with `Remove-Item Env:NBOMBER_CHECK_ONLY` before running the real test.
## Load presets

Each preset runs the whole scenario (all steps) at the given rate.

```csharp
// Smoke: 1 per second for 30 seconds
.WithLoadSimulations(
    Simulation.Inject(rate: 1, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromSeconds(30))
)

// Load: ramp to 50/sec over 30 seconds, hold for 5 minutes
.WithLoadSimulations(
    Simulation.RampingInject(rate: 50, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromSeconds(30)),
    Simulation.Inject(rate: 50, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromMinutes(5))
)

// Stress: ramp to 200/sec over 1 minute, hold for 5 minutes
.WithLoadSimulations(
    Simulation.RampingInject(rate: 200, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromMinutes(1)),
    Simulation.Inject(rate: 200, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromMinutes(5))
)
```

For a custom profile, use the same two simulations with the user's rate, ramp-up and duration. Skip `RampingInject` if they don't want a ramp-up.

## Authentication snippets

Read every secret from an environment variable and fail fast if it's missing.

```csharp
// Bearer token
.WithHeader("Authorization", $"Bearer {token}")

// API key in a header (use the header name the user gives)
.WithHeader("X-API-Key", apiKey)

// API key in the query string (use the parameter name the user gives)
Http.CreateRequest("GET", $"{BaseUrl}/pets?api_key={Uri.EscapeDataString(apiKey)}")

// Basic auth
var basic = Convert.ToBase64String(Encoding.UTF8.GetBytes($"{username}:{password}"));
.WithHeader("Authorization", $"Basic {basic}")
```

## Scenario in an existing project

Put the scenario in its own class so the user can register it with their other scenarios:

```csharp
using NBomber.Contracts;
using NBomber.CSharp;
using NBomber.Http.CSharp;

public static class PetsApiScenario
{
    public static ScenarioProps Create(HttpClient httpClient)
    {
        return Scenario.Create("pets_api", async context =>
        {
            // steps as in the full example
            return Response.Ok();
        })
        .WithLoadSimulations(/* chosen preset */);
    }
}

// Registration, in the user's Program.cs:
// NBomberRunner.RegisterScenarios(PetsApiScenario.Create(httpClient)).Run();
```
