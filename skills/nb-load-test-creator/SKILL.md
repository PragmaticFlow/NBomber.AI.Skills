---
name: nb-load-test-creator
description: Creates runnable NBomber load tests in C# from an OpenAPI/Swagger spec (URL or local .json, .yaml or .yml file; OpenAPI 3.x and Swagger 2.0). Use it whenever the user wants a load, performance, stress, smoke, spike or soak test for an HTTP or REST API, mentions NBomber, or wants to turn a spec into test scenarios, even if they don't say "NBomber" or "load test". Typical requests include "load test my API", "how many requests per second can this endpoint handle", "stress test localhost:5000 from its swagger", "benchmark these endpoints" and "create an NBomber scenario for this service". The user picks the endpoints, auth and load profile; the skill generates one scenario with a step per endpoint, chaining ids between steps.
license: MIT
compatibility: Requires Claude Code with shell access, the .NET SDK (dotnet CLI), and network access to nuget.org, the OpenAPI spec URL and the API under test. Docker is only needed if the tested API depends on containers.
metadata:
  author: NBomber
  version: 1.1.0
  category: testing
  tags: [load-testing, performance-testing, nbomber, openapi, swagger, csharp, dotnet]
---

# Load Test Creator

Turn an API description into a runnable NBomber load test in C#. The user chooses the source, picks the endpoints to cover, and gets one NBomber scenario with one step per endpoint.

Work through the steps below in order, and ask the user at each point marked **Ask**. Load tests send real traffic, so the user should stay in control of what gets called, how hard, and with which credentials.

## Step 1: Choose the metadata source

**Ask** which source to build the test from. Present the supported sources as a list:

1. OpenAPI / Swagger specification (supported)

More sources (for example HAR files or Postman collections) may be added later. If the user asks for one that isn't supported yet, say so and offer OpenAPI instead.

## Step 2: Get the OpenAPI specification

**Ask** for either:

- a URL to the spec (JSON or YAML), or
- a path to a local spec file (`.json`, `.yaml`, `.yml`).

For a URL, download the raw file to the scratchpad or a temp folder (for example with `curl -L -o spec.json <url>` or PowerShell `Invoke-WebRequest -Uri <url> -OutFile spec.json`) and read it from there. Don't rely on a web-fetch tool that summarizes pages, because you need the exact paths, parameters and schemas.

Accept both OpenAPI 3.x (`"openapi": "3.x.x"`) and Swagger 2.0 (`"swagger": "2.0"`). They store the same information in different places. Read [references/openapi.md](references/openapi.md) for how to find the base URL, parameters, bodies, example values and security schemes in each version.

If the download fails or the file isn't a valid spec, tell the user what went wrong and ask for another URL or file.

## Step 3: Let the user pick endpoints

List every operation in the spec as a numbered list, grouped by tag if the spec uses tags:

```
Pets
  1. GET    /pets            List all pets
  2. POST   /pets            Create a pet
  3. GET    /pets/{petId}    Get a pet by id
  4. DELETE /pets/{petId}    Delete a pet
Store
  5. GET    /store/inventory Get inventory counts
```

**Ask** which ones to cover. Accept numbers, ranges (`1-3`), or `all`. For long specs, show the list anyway; don't silently trim it.

You can offer a few suggested selections as options (for example a realistic user flow, `all`, or read-only endpoints), plus a way to type a custom selection. Numbers alone mean nothing to the user without scrolling back to the list, so every option, including `all`, must spell out what it covers:

- **Label:** a short name plus the numbers, in the order the steps will run, for example `User flow: 5,6,1,2,4,7`. For the option that covers everything, the label is exactly `all (1-N)`, where N is the number of endpoints, for example `all (1-7)`.
- **Description:** name every endpoint the option covers, as a short plain-language action, joined with arrows in the order the steps will run. The `all` option is no exception: list every endpoint, never just "all endpoints". For example, `all (1-7)` gets `Sign up → log in → add book → list books → order that book → log out → reset database`. Put any warning after the flow, for example `Includes reset database on every iteration, which will likely wipe data other iterations are using.`

Use the same action names in every option, so the user can compare options at a glance.

If the selection includes operations that change or delete data (`POST`, `PUT`, `PATCH`, `DELETE`), point out that the load test will run them many times, and confirm the user is targeting a test environment rather than production.

### Setup and cleanup endpoints

Some endpoints prepare or reset the system instead of serving users: recreating the database, seeding data, clearing caches, deleting test data. Spot them by names, paths, summaries and descriptions with words like reset, seed, init, setup, prepare, database, migrate, clean or purge (for example `PUT /api/Databases`). Running one of these on every iteration usually wipes data that other iterations are using. Skipping it can break the test entirely, for example when the API only creates its tables when that endpoint is called.

For each one you spot, whether or not the user selected it, **Ask** what to do with it:

- **Run once before the test** (in `.WithInit`). Recommended for reset, setup and seed endpoints.
- **Run once after the test** (in `.WithClean`). Recommended for cleanup endpoints.
- **Include in the load**, like any other step.
- **Skip it.**

Say what the endpoint appears to do, and why you recommend that option.

## Step 4: Base URL

Take the base URL from the spec (`servers` in 3.x, `host` + `basePath` + `schemes` in 2.0). **Ask** the user to confirm it or give a different one. Ask outright if the spec has several servers, a relative server URL, or server variables.

## Step 5: Check what the API needs to work

A load test is only useful if the API works in the first place. When the API's source code is on the user's machine (the base URL is `localhost`, or the user points to the project), **Ask** for the project folder if you don't know it, then check:

- **Dependencies.** Find the databases, queues and other services the API uses: connection strings in `appsettings*.json` or `.env` files, and services in `docker-compose.yml`. Check that each one is running (`docker ps`, `docker compose ps`).
- **Ports.** Check that the port in each connection string belongs to the right service. Another container or a locally installed server may already hold it, for example a different Postgres on 5432. Then the API connects to the wrong database and fails with errors like "password authentication failed".
- **Setup on startup.** Check whether the API creates its tables or seeds data when it starts. If not, find what does (a setup endpoint, a migration command, a SQL script). If it's an endpoint, handle it as described in "Setup and cleanup endpoints" in Step 3. Otherwise, add it to the handover notes as something to run before the test.

Report what you found. Don't start, stop or reconfigure the user's containers or services without asking. They may belong to other projects.

If the API isn't on the user's machine, skip this step. The one-request check in Step 10 will catch problems instead.

## Step 6: Authentication

**Ask** how the test should authenticate, every time, whatever the spec says. Offer:

- No authentication
- Bearer token
- API key (in a header or a query parameter, with its name)
- Basic authentication (username and password)

If the spec declares security schemes, mention them as a hint, but let the user decide.

Never write secrets into the code. The generated test reads them from environment variables (for example `API_TOKEN`, `API_KEY`, `API_USERNAME`, `API_PASSWORD`) and stops with a clear message if one is missing. Tell the user which variables to set.

## Step 7: Load profile

**Ask** which load profile to use, offering these presets plus a custom option:

| Preset | What it does |
|---|---|
| Smoke | 1 request/sec for 30 seconds. Checks everything works. |
| Load | Ramp up to 50 requests/sec over 30 seconds, then hold 50/sec for 5 minutes. |
| Stress | Ramp up to 200 requests/sec over 1 minute, then hold 200/sec for 5 minutes. |
| Custom | The user gives rate, ramp-up time and duration. |

Each "request" here is one full scenario run, meaning all selected steps. Write the chosen values as named settings at the top of the generated file so they're easy to change. [references/nbomber.md](references/nbomber.md) shows the matching NBomber load simulations.

## Step 8: Where to put the code

**Ask** whether to:

- create a **new C# console project** (suggest a name such as `<ApiTitle>LoadTest`), or
- add the scenario to an **existing project** (ask for its path).

### Dependencies: always the latest version

Every package the generated test depends on (`NBomber`, `NBomber.Http`, and anything else you add) must be at its latest stable version. Use a beta or other prerelease version only if the user asks for it.

- Never write a version number from memory, from this skill's files, or from another project. Add packages with `dotnet add package <Name>`, which installs the latest stable version. To see the latest version, run `dotnet package search <Name> --exact-match`, or check `https://api.nuget.org/v3-flatcontainer/<name-in-lowercase>/index.json`.
- The code examples in [references/nbomber.md](references/nbomber.md) show the NBomber 6 API. If the latest version has changed the API, the build in Step 10 will fail. Fix the code to match the latest version, not the other way round. Never pin an older package just to make the examples compile.

### New or existing project

For a new project: run `dotnet new console -n <Name>`, then `dotnet add package NBomber` and `dotnet add package NBomber.Http`, and write `Program.cs`.

For an existing project: check which versions of `NBomber` and `NBomber.Http` it references.

- If a package is missing, add it with `dotnet add package`.
- If a package is older than the latest stable version, tell the user which version the project has and which is latest. Then **Ask** before upgrading it with `dotnet add package <Name>`, because other code in that project may depend on the old version. If the user says no, keep their version and mention it in the handover.

Add the scenario in its own file (for example `LoadTests/<ApiTitle>Scenario.cs`) as a static method returning `ScenarioProps`, and tell the user how to register it with `NBomberRunner.RegisterScenarios(...)`. Don't overwrite an existing `Program.cs` without asking.

## Step 9: Generate the scenario

Build one NBomber scenario with one `Step.Run` per selected endpoint. [references/nbomber.md](references/nbomber.md) has the full code pattern. Its key points:

- **Order and chaining.** Put steps in a sensible order: create (`POST`) first, then read/update (`GET`, `PUT`, `PATCH`) on the created item, then `DELETE` last. When a step creates something, read its id from the response and use it for the `{id}` path parameters in later steps. If a path parameter can't come from an earlier step, use an example or generated value and mark it with a `// TODO` comment.
- **Request data.** Use example values from the spec first. If there are none, generate valid values from the schema (type, format, enum, min/max, required fields) and mark generated values with `// TODO: generated value, replace with real data if needed`. Make fields that must be unique (emails, usernames, ids) unique per run, for example by adding `Guid.NewGuid()`.
- **Failures.** Any response outside 200–299 counts as a failed step. `Http.Send` from NBomber.Http already reports non-2xx responses as failures. After each step, stop the iteration if it failed, so later chained steps don't run with missing ids.
- **Naming.** Name steps after the operation, for example `"POST /pets"`, so the NBomber report is easy to read.
- **Setup and cleanup.** Put endpoints chosen to run once before the test in `.WithInit`, and those chosen to run once after it in `.WithClean`. Make the init code fail loudly (for example `EnsureSuccessStatusCode()`) so a broken setup stops the test instead of producing thousands of step errors.
- **Check-only switch.** Add a `NBOMBER_CHECK_ONLY` environment variable switch that replaces the load simulations with a single iteration and turns off warm-up. It's used for the one-request check in Step 10, and the user can reuse it later. [references/nbomber.md](references/nbomber.md) shows the code.

## Step 10: Build, check and hand over

Run `dotnet build` to make sure the code compiles, and fix any errors.

### One-request check

**Ask** the user for permission to run the scenario once, with a single request: the init step (if any), one pass through all steps, and the cleanup step (if any). Explain that this is harmless traffic and shows whether every step works before a real load test. If the API runs on the user's machine, remind them to start it first.

Run it with the check-only switch on, for example `$env:NBOMBER_CHECK_ONLY = "1"; dotnet run -c Release` in PowerShell, or `NBOMBER_CHECK_ONLY=1 dotnet run -c Release` in bash. In an existing project that registers other scenarios, make sure only this scenario runs, or ask the user first.

Never run the real load profile yourself. It sends heavy traffic to the user's API, so leave that to them.

### If the check fails

Read [references/troubleshooting.md](references/troubleshooting.md) and follow it. It explains how to tell whether the test or the API is at fault, and what to do in each case. Never change the test to hide a server-side error.

### Hand over

Tell the user:

- where the files are,
- which endpoints are covered and in what order, and what runs once before or after the test,
- which environment variables to set,
- which values are generated and marked `TODO`,
- the result of the one-request check,
- how to run the real test (`dotnet run -c Release`), how to run the check again (with `NBOMBER_CHECK_ONLY=1`), and that NBomber writes its reports to a `reports` folder.
