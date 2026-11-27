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

## Before you start

How good Sandcastle's output is depends on how much of your intent is written down. You don't need to have it all ready: onboard helps you fill in what's missing. It writes the gaps it finds to an open questions list (projects onboarded before the list existed get one with `onboard --questions-only`), and you answer them with `/fill-project-docs`.

Project knowledge comes in three kinds:

| Kind | Can it be read from the code? | Who fills it in | Examples |
|---|---|---|---|
| Fact | Yes | The onboard agent writes it into PROJECT.md | Directory layout, commands, test framework, services the tests depend on |
| Convention | Partly | The agent drafts it, a human confirms | Code style, how tests are written, commit conventions |
| Intent | No | A human, answering in `/fill-project-docs` | Business rules, domain terms, red lines, reasons behind a design |

Tickets that only touch facts (fixing a bug, adding a field the way existing ones are added) work without these docs. Tickets that need intent that isn't written down stop and wait for a human to fill it in.

## Install

```bash
npm i -g sandpipe
# or
pnpm add -g sandpipe   # first time with pnpm: run `pnpm setup` and reopen the terminal, so its global bin directory is on PATH
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

Two unrelated credentials are involved: `sandpipe onboard` completes `.sandcastle/PROJECT.md` by running `claude -p` locally under your own Claude Code login, while the pipeline's sandboxed agents use only `CLAUDE_CODE_OAUTH_TOKEN` (from `claude setup-token`) or `ANTHROPIC_API_KEY` in `.sandcastle/.env`. A local login-expiry warning doesn't affect the pipeline.

Then label issues with `Sandcastle` and run:

```bash
SANDCASTLE_ONLY=12 sandpipe    # only #12; without it, all labelled tickets are processed
```

New projects only open PRs by default and don't auto-merge (`autoMerge: false`), one ticket at a time. Follow [docs/onboarding-checklist.md](docs/onboarding-checklist.md) through a canary ticket and real small tickets, and loosen the limits once you're confident.

Auto-merge requires both the gate and the checks on the PR to pass. If your CI covers things the gate doesn't, consider adding them to `gate.script`; if your CI is unreliable, set `mergeRequires: { checks: "none" }` (see [docs/config.md](docs/config.md)).

## Generated files

| File | Purpose |
|---|---|
| `.sandcastle/PROJECT.md` | Project knowledge for the agent: overview, commands, testing, standards, red lines, conflict handling, commit conventions |
| `.sandcastle/sandcastle.config.ts` | How to run: installing dependencies, test isolation, the merge gate. Fields are described in [docs/config.md](docs/config.md) |
| `.sandcastle/models.json` | Which model each phase (prep / implement / review …) uses |
| `.sandcastle/.env` | Tokens, not committed |
| `.sandcastle/Containerfile` | Sandbox image |
| `.claude/` | Context budget hook, screenshot-viewing subagent, `/prep-ticket` and `/fill-project-docs` skills |

## Commands

| Command | Purpose |
|---|---|
| `sandpipe [run]` | Run the pipeline |
| `sandpipe onboard [--no-agent] [--model <name>] [--lang zh\|en] [--verbose]` | Onboard a new project; existing files under `.sandcastle/` are not overwritten, and for this tool's files under `.claude/` that differ from the template it shows the diff and asks to keep, overwrite or save as `.new` (non-interactive: saves `.new`); the output is a short summary per stage (detect / files / checks), `--verbose` prints the full details |
| `sandpipe onboard --questions-only [--model <name>] [--lang zh\|en]` | Only write the open questions list `.sandcastle/OPEN-QUESTIONS.md`, for projects onboarded before it existed; touches no other file |
| `sandpipe doctor [--fix]` | Pre-run checks; exit code 1 when anything shows ✗ |
| `sandpipe baseline` | Run the full gate once against `origin/main`, report only |
| `sandpipe update` | Refresh this tool's files under `.claude/` to the current version, asking the same way as onboard when one differs from the template; refreshes the global skill too if installed |
| `sandpipe skill` | Install `/prep-ticket` into `~/.claude/skills/` so it works in repos without sandcastle (it shadows the project copy; `update` refreshes it too) |
| `sandpipe upgrade` | Install the latest version |
| `sandpipe --version` | |

Only GitHub is supported for now; GitLab isn't done yet.
