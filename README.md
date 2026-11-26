# NBomber Skills

[Claude Code](https://claude.com/claude-code) skills for [NBomber](https://nbomber.com), the load-testing framework for .NET.

| Skill | What it does |
|---|---|
| [nb-load-test-creator](skills/nb-load-test-creator/) | Turns an OpenAPI/Swagger spec into a runnable C# NBomber load test that builds and passes a one-request check before you run the real load. |
| [nb-core](skills/nb-core/) | Writes and reviews any NBomber test, and checks the basic setup: Server GC, a useful warm-up, and no unnecessary `IsError` checks after steps. |
| [nb-http](skills/nb-http/) | Writes and reviews NBomber HTTP scenarios, and checks them for common HTTP mistakes: a hand-made `HttpClient`, hand-built JSON bodies, shared sessions and the obsolete `Http.Send`. Use it together with nb-core. |

## nb-load-test-creator

Point Claude at your API's OpenAPI spec and get a working load test in minutes, instead of writing the scenario, request bodies and auth handling by hand.

You stay in control at every step. Claude asks you:

1. which spec to use (a URL or a local `.json`, `.yaml` or `.yml` file; OpenAPI 3.x and Swagger 2.0),
2. which endpoints to cover, suggesting realistic flows such as *sign up → log in → add book → order that book*,
3. what to do with setup endpoints such as a database reset (run once before the test, not on every iteration),
4. the base URL, authentication and load profile (Smoke, Load, Stress or custom),
5. whether to create a new console project or add the scenario to an existing one.

It then generates one NBomber scenario with one step per endpoint, and builds it. With your permission, it runs the scenario once with a single request to check that every step works.

### What the generated test includes

- **Chained steps.** Ids and tokens from one step's response are used by the next steps, for example the id of a created item or the JWT from a login step.
- **Setup and cleanup.** Reset or seed endpoints run once in `WithInit`, and cleanup endpoints once in `WithClean`.
- **A check-only switch.** Set `NBOMBER_CHECK_ONLY=1` to run a single iteration instead of the full load profile.
- **No secrets in code.** Tokens, API keys and passwords are read from environment variables.
- **Latest packages.** `NBomber` and `NBomber.Http` are added at their latest stable versions.
- **Clear `TODO` markers** on every generated value you may want to replace with real data.

### Safety

Load tests send real traffic, so the skill:

- never runs the real load profile itself; you start it,
- asks before running even the one-request check,
- points out endpoints that change or delete data, and asks you to confirm you're targeting a test environment,
- doesn't start, stop or reconfigure your containers or services without asking.

### Example

> create a load test for my bookstore API, the swagger is at http://localhost:50762/swagger/v1/swagger.json

Claude lists the 7 operations in the spec, you pick the user flow and choose to run the database reset once before the test, and it generates:

```
POST /api/Users/singup → POST /api/Users/login → POST /api/Books
  → GET /api/Books → POST /api/Orders → POST /api/Users/logout
```

Each iteration signs up a new user, logs in with that user's token, adds a book and orders it. The load ramps up to 50 flows per second over 30 seconds, then holds for 5 minutes.

## nb-core

Use it when you write an NBomber test by hand, or to review one you already have, for any protocol. It works with a `.csproj` project and with a C# file-based app (`dotnet run app.cs`).

- **Writing a test.** The new code passes every check below.
- **Reviewing a test.** Claude reports each check as passed or failed. For each failed check, it gives the file and line and proposes the exact change. It doesn't change your code until you agree.

### What it checks

1. **Server garbage collection.** `ServerGarbageCollection` and `ConcurrentGarbageCollection` are on, in the `.csproj` (or a `Directory.Build.props`) or as `#:property` directives in a file-based app. A file-based app also needs `PublishAot=false`. With the default workstation GC, the load generator slows down, and the test measures the client instead of the system under test.
2. **The scenario has a useful warm-up.** The warm-up is at least 5 seconds (the default is 30 seconds). `WithoutWarmUp()` is only for testing and development runs. If the load starts very low, for example 1 request every 10 seconds, the default warm-up does almost nothing, so the skill proposes `WithWarmUpSimulations` (NBomber 6.7.0 or later).
3. **No unnecessary `IsError` checks after steps.** A failed step already ends the iteration, so `if (login.IsError) return login;` after `Step.Run` never runs. The skill doesn't report these checks when the scenario uses `.WithRestartIterationOnFail(shouldRestart: false)`.

## nb-http

Use it when you write an NBomber HTTP scenario by hand, or to review one you already have. It works with a `.csproj` project and with a C# file-based app (`dotnet run app.cs`). It works the same way as nb-core, and it also runs the nb-core checks, so install both.

### What it checks

1. **`HttpClient` comes from `Http.CreateDefaultClient()`**, not from `new HttpClient()` or `IHttpClientFactory`. This client is set up for load generation: up to 5000 connections per server, and pooled connections that are reused and then recycled. It needs .NET 8 or later.
2. **JSON request bodies use `WithJsonBody`**, not hand-written serialization. This check also finds a common bug: a `Content-Type` set with `WithHeader` is silently dropped, and the API answers 415 Unsupported Media Type.
3. **JSON responses use `SendRequest<T>`**, not hand-written parsing, and the fields that later steps use are checked.
4. **The `HttpClient` is reused**, not created on every iteration.
5. **Each virtual user has its own session**: its own token, or its own client when the session lives in cookies.
6. **Requests are sent with `SendRequest`**, not the obsolete `Http.Send`.
7. **Compression uses the NBomber.Http helpers**, not hand-written compression or decompression.

## Requirements

- [Claude Code](https://claude.com/claude-code) (the skills run shell commands and edit files, so they need Claude Code rather than the claude.ai chat)
- [.NET SDK](https://dotnet.microsoft.com/download) (the `dotnet` CLI)
- For nb-load-test-creator: network access to nuget.org, the spec URL and the API under test
- Docker, only if the API you're testing runs its dependencies in containers

NBomber itself is free for personal use; organizations need an [NBomber license](https://nbomber.com). These skills are MIT-licensed, separately from NBomber.

## Installation

Skills live in a `skills` folder that Claude Code reads on startup. Clone this repo, copy the skill folders there, then delete the clone. Pick one of the two options below.

The commands below install all skills. To install only some of them, remove the other folders from the copy command. nb-http needs nb-core.

### Option 1: for all your projects

The skills go into your personal skills folder (`~/.claude/skills/`). Run these from any folder.

macOS / Linux (bash):

```bash
git clone https://github.com/OlenaKostash/nbomber-skills.git
mkdir -p ~/.claude/skills
cp -r nbomber-skills/skills/nb-load-test-creator nbomber-skills/skills/nb-core nbomber-skills/skills/nb-http ~/.claude/skills/
rm -rf nbomber-skills
```

Windows (PowerShell):

```powershell
git clone https://github.com/OlenaKostash/nbomber-skills.git
New-Item -ItemType Directory -Force "$HOME\.claude\skills" | Out-Null
Copy-Item -Recurse nbomber-skills\skills\nb-load-test-creator, nbomber-skills\skills\nb-core, nbomber-skills\skills\nb-http "$HOME\.claude\skills\"
Remove-Item -Recurse -Force nbomber-skills
```

### Option 2: for one project only

The skills go into the project's `.claude/skills/` folder. Run these **from the project's root folder** (the folder you open Claude Code in). Commit `.claude/skills/`, and everyone who opens the project in Claude Code gets the skills.

macOS / Linux (bash):

```bash
git clone https://github.com/OlenaKostash/nbomber-skills.git
mkdir -p .claude/skills
cp -r nbomber-skills/skills/nb-load-test-creator nbomber-skills/skills/nb-core nbomber-skills/skills/nb-http .claude/skills/
rm -rf nbomber-skills
```

Windows (PowerShell):

```powershell
git clone https://github.com/OlenaKostash/nbomber-skills.git
New-Item -ItemType Directory -Force ".claude\skills" | Out-Null
Copy-Item -Recurse nbomber-skills\skills\nb-load-test-creator, nbomber-skills\skills\nb-core, nbomber-skills\skills\nb-http ".claude\skills\"
Remove-Item -Recurse -Force nbomber-skills
```

The last command deletes the cloned `nbomber-skills` folder. You only need the copy in `.claude/skills/`, and leaving the clone inside your project would add a second git repository to it.

### Check the install

Start a new Claude Code session (or restart Claude Code) so it picks up the skills, then type `/`. You should see `nb-load-test-creator`, `nb-core` and `nb-http` in the list.

## Usage

Ask in your own words. Claude loads the right skill automatically.

For nb-load-test-creator:

- "load test my API from its swagger"
- "how many requests per second can this endpoint handle?"
- "create an NBomber scenario for this service"

For nb-core:

- "check the setup of my NBomber project"
- "is the warm-up of my NBomber scenario long enough?"
- "do I need these IsError checks after each step?"

For nb-http:

- "review my NBomber test in LoadTests/Program.cs"
- "write an NBomber scenario that logs in and then gets the user profile"
- "why does my NBomber POST request get 415 Unsupported Media Type?"

You can also start a skill directly with `/nb-load-test-creator`, `/nb-core` or `/nb-http`.

## Repository layout

```
nbomber-skills/
├── README.md                     this file
├── LICENSE
└── skills/
    ├── nb-load-test-creator/     the load test creator skill (copy this folder to install)
    │   ├── SKILL.md              workflow Claude follows
    │   └── references/
    │       ├── nbomber.md        NBomber code patterns
    │       ├── openapi.md        reading OpenAPI 3.x and Swagger 2.0 specs
    │       └── troubleshooting.md  what to do when the one-request check fails
    ├── nb-core/                  the core skill (copy this folder to install)
    │   └── SKILL.md              checks for every NBomber test
    └── nb-http/                  the HTTP skill (copy this folder to install)
        └── SKILL.md              checks for NBomber HTTP scenarios
```

## License

[MIT](LICENSE)
