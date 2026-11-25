# sandpipe

[中文](README.zh-CN.md) | English

An unattended pipeline that turns GitHub issues into PRs and merges them once tests pass, built on [`@ai-hero/sandcastle`](https://github.com/mattpocock/sandcastle).

The pipeline runs these steps in order:

1. Split tickets and write an implementation guide;
2. Schedule by dependencies and the files each ticket changes;
3. Have Claude Code implement in a podman sandbox, review it in a separate session, and send it back for changes when needed;
4. Run the test gate you define;
5. Merge when everything passes; otherwise leave the PR for a human.

## Requirements

- Node ≥ 22.18
- A logged-in `gh` (`gh auth login`)
- podman (on macOS also `podman machine start`)
- Claude Code credentials: subscribers can generate one with `claude setup-token`

## Install

```bash
npm i -g sandpipe
# or
pnpm add -g sandpipe
```

You can also install the package from the GitHub Release: `npm i -g https://github.com/kobe4cn/sandcastle-releases/releases/latest/download/sandpipe.tgz`.

To upgrade later: `sandpipe upgrade`, then run `sandpipe update` in each project.

## Onboard a project

In the project root:

```bash
sandpipe onboard          # detect the project, generate .sandcastle/, let claude read the code and write PROJECT.md
cp .sandcastle/.env.example .sandcastle/.env    # fill in tokens
podman build -t sandcastle:<project> -f .sandcastle/Containerfile .sandcastle   # onboard prints the exact command at the end
sandpipe doctor --fix     # check everything, creating labels along the way
sandpipe baseline         # run the full gate once against main in a sandbox
```

Then label issues with `Sandcastle` and run:

```bash
SANDCASTLE_ONLY=12 sandpipe    # only #12; without it, all labelled tickets are processed
```

New projects only open PRs by default and don't auto-merge (`autoMerge: false`), one ticket at a time. Follow [docs/onboarding-checklist.md](docs/onboarding-checklist.md) through a canary ticket and real small tickets, and loosen the limits once you're confident.

## Generated files

| File | Purpose |
|---|---|
| `.sandcastle/PROJECT.md` | Project knowledge for the agent: overview, commands, testing, standards, red lines, conflict handling, commit conventions |
| `.sandcastle/sandcastle.config.ts` | How to run: installing dependencies, test isolation, the merge gate. Fields are described in [docs/config.md](docs/config.md) |
| `.sandcastle/models.json` | Which model each phase (prep / implement / review …) uses |
| `.sandcastle/.env` | Tokens, not committed |
| `.sandcastle/Containerfile` | Sandbox image |
| `.claude/` | Context budget hook, screenshot-viewing subagent, `/prep-ticket` skill |

## Commands

| Command | Purpose |
|---|---|
| `sandpipe [run]` | Run the pipeline |
| `sandpipe onboard [--no-agent] [--model <name>] [--lang zh\|en]` | Onboard a new project; existing files are not overwritten |
| `sandpipe doctor [--fix]` | Pre-run checks; exit code 1 when anything shows ✗ |
| `sandpipe baseline` | Run the full gate once against `origin/main`, report only |
| `sandpipe update` | Refresh this tool's files under `.claude/` to the current version |
| `sandpipe upgrade` | Install the latest version |
| `sandpipe --version` | |

Only GitHub is supported for now; GitLab isn't done yet.
