# NBomber Skills

[Claude Code](https://claude.com/claude-code) skills for [NBomber](https://nbomber.com), the load-testing framework for .NET.

| Skill | What it does |
|---|---|
| [nb-load-test-creator](nb-load-test-creator/) | Turns an OpenAPI/Swagger spec into a runnable C# NBomber load test that builds and passes a one-request check before you run the real load. |

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

## Requirements

- [Claude Code](https://claude.com/claude-code) (the skill runs shell commands, so it needs Claude Code rather than the claude.ai chat)
- [.NET SDK](https://dotnet.microsoft.com/download) (the `dotnet` CLI)
- Network access to nuget.org, the spec URL and the API under test
- Docker, only if the API you're testing runs its dependencies in containers

NBomber itself is free for personal use; organizations need an [NBomber license](https://nbomber.com). This skill is MIT-licensed, separately from NBomber.

## Installation

Skills live in a `skills` folder that Claude Code reads on startup. Clone this repo, copy the skill folder there, then delete the clone. Pick one of the two options below.

### Option 1: for all your projects

The skill goes into your personal skills folder (`~/.claude/skills/`). Run these from any folder.

macOS / Linux (bash):

```bash
git clone https://github.com/OlenaKostash/nbomber-skills.git
mkdir -p ~/.claude/skills
cp -r nbomber-skills/nb-load-test-creator ~/.claude/skills/
rm -rf nbomber-skills
```

Windows (PowerShell):

```powershell
git clone https://github.com/OlenaKostash/nbomber-skills.git
New-Item -ItemType Directory -Force "$HOME\.claude\skills" | Out-Null
Copy-Item -Recurse nbomber-skills\nb-load-test-creator "$HOME\.claude\skills\"
Remove-Item -Recurse -Force nbomber-skills
```

### Option 2: for one project only

The skill goes into the project's `.claude/skills/` folder. Run these **from the project's root folder** (the folder you open Claude Code in). Commit `.claude/skills/nb-load-test-creator`, and everyone who opens the project in Claude Code gets the skill.

macOS / Linux (bash):

```bash
git clone https://github.com/OlenaKostash/nbomber-skills.git
mkdir -p .claude/skills
cp -r nbomber-skills/nb-load-test-creator .claude/skills/
rm -rf nbomber-skills
```

Windows (PowerShell):

```powershell
git clone https://github.com/OlenaKostash/nbomber-skills.git
New-Item -ItemType Directory -Force ".claude\skills" | Out-Null
Copy-Item -Recurse nbomber-skills\nb-load-test-creator ".claude\skills\"
Remove-Item -Recurse -Force nbomber-skills
```

The last command deletes the cloned `nbomber-skills` folder. You only need the copy in `.claude/skills/`, and leaving the clone inside your project would add a second git repository to it.

### Check the install

Start a new Claude Code session (or restart Claude Code) so it picks up the skill, then type `/`. You should see `nb-load-test-creator` in the list.

## Usage

Ask for a load test in your own words, for example:

- "load test my API from its swagger"
- "how many requests per second can this endpoint handle?"
- "create an NBomber scenario for this service"

Claude loads the skill automatically. You can also start it directly with `/nb-load-test-creator`.

## Repository layout

```
nbomber-skills/
├── README.md                     this file
├── LICENSE
└── nb-load-test-creator/         the skill (copy this folder to install)
    ├── SKILL.md                  workflow Claude follows
    └── references/
        ├── nbomber.md            NBomber code patterns
        ├── openapi.md            reading OpenAPI 3.x and Swagger 2.0 specs
        └── troubleshooting.md    what to do when the one-request check fails
```

## License

[MIT](LICENSE)
