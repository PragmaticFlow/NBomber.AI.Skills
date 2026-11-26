# NBomber Skills

[Agent Skills](https://agentskills.io) for [NBomber](https://nbomber.com), the load-testing framework for .NET.

The skills use the open `SKILL.md` format, so they work with any AI coding agent that supports Agent Skills, for example Claude Code, Codex, Cursor or GitHub Copilot.

| Skill | What it does |
|---|---|
| [nb-core](skills/nb-core/) | Writes and reviews any NBomber test, and checks the basic setup. Use it when you write an NBomber test by hand, or to review one you already have, for any protocol. It works with a `.csproj` project and with a C# file-based app (`dotnet run app.cs`). |
| [nb-http](skills/nb-http/) | Writes and reviews NBomber HTTP scenarios, and checks them for common HTTP mistakes. Use it together with nb-core. |

## Installation

Each agent reads skills from its own `skills` folder. The easiest way to put them there is the [skills.sh](https://skills.sh) CLI. You can also copy the folders by hand.

nb-http needs nb-core, so install both.

### With skills.sh (recommended)

You need [Node.js](https://nodejs.org) 22.20 or later. Run this **from the project's root folder** (the folder you open your agent in):

```bash
npx skills add PragmaticFlow/NBomber.AI.Skills
```

Other useful commands:

| Command | What it does |
|---|---|
| `npx skills add PragmaticFlow/NBomber.AI.Skills --list` | Lists the skills in this repo without installing them. |
| `npx skills check` | Shows if updates are available. |
| `npx skills update` | Updates the installed skills. |
| `npx skills remove nb-core nb-http` | Removes the skills. |

### Manual install

Clone this repo, copy the skill folders into your agent's skills folder, then delete the clone. Pick one of the two options below.

### Check the install

Start a new agent session (or restart the agent) so it picks up the skills. Then ask the agent which skills it has: it should list `nb-core` and `nb-http`. In Claude Code, you can also type `/` and find them in the list.

## Usage

Ask in your own words. The agent loads the right skill automatically.

For nb-core:

- "check the setup of my NBomber project"
- "is the warm-up of my NBomber scenario long enough?"

For nb-http:

- "review my NBomber test in LoadTests/Program.cs"
- "write an NBomber scenario that logs in and then gets the user profile"
- "why does my NBomber POST request get 415 Unsupported Media Type?"

In agents that support slash commands, such as Claude Code, you can also start a skill directly with `/nb-core` or `/nb-http`.

## License

[MIT](LICENSE)
