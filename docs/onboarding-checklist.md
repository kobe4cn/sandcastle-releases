# New project onboarding checklist

[中文](onboarding-checklist.zh-CN.md) | English

Go in order, from `onboard` to letting Sandcastle merge on its own. If a step fails, stop and fix it; don't skip ahead.
Run every command in the target project's root; if the config lives elsewhere, first `export SANDCASTLE_PROJECT_DIR=<dir>`.

## 0. Generate the config

- [ ] `sandpipe onboard` generates `.sandcastle/`, and the agent fills in PROJECT.md
- [ ] A human reads `PROJECT.md`: remove every `<!-- TODO` marker; commands, directories and conventions match the project
- [ ] A human reads `sandcastle.config.ts`:
  - `install.onSandboxReady` installs dependencies in a linux container
  - `isolation.env` gives each ticket its own database name or prefix (so parallel tickets don't wipe each other's tables); set `cleanup` when there are external services
    - onboard already generated a draft from the variables in `.env.example` / `.env` that point at local services (guessing key names from compose services when there are none): at run time it reads them from the project root's `.env`, swaps the host and adds `_i<runId>` to each database; check for missing databases or middleware and for anything that shouldn't reach the sandbox
    - When the config reads `.env` / `.env.test` from the project root, list them in `isolation.hostEnvFiles` and make sure they exist on the host (they're usually gitignored, so a fresh clone lacks them; `sandpipe doctor` warns)
    - With a web UI, PROJECT.md needs a Browser check section and the dev server's variables must be in `isolation.env` (with a separate database); see [Browser check in config](config.md#browser-check). Without it `sandpipe doctor` warns; when onboard can't piece it together it lists the missing details in the open questions list. The sandbox image has the browser whether or not there's a web UI, so a frontend added later only needs this section
  - `gate.script` is a gate you're willing to merge on; consider `integrity` if there are migrations
  - Keep the `autoMerge: false` and `maxParallel: 1` that onboard generated
- [ ] `cp .sandcastle/.env.example .sandcastle/.env` and fill in the token the pipeline uses (`CLAUDE_CODE_OAUTH_TOKEN` from `claude setup-token`, or `ANTHROPIC_API_KEY`); the pipeline reads only this file, independent of your local Claude Code login
- [ ] Build the image with the command from onboard's next steps (`podman build -t sandcastle:<project> …`)

## 1. doctor passes

- [ ] `sandpipe doctor --fix`: no ✗. Read every ⚠ and confirm it's acceptable

## 2. Fill in project docs (recommended)

- [ ] Look at the "Project knowledge" check in `sandpipe doctor`. If an existing project has no `.sandcastle/OPEN-QUESTIONS.md` yet, run `sandpipe onboard --questions-only` first
- [ ] Run `/fill-project-docs` in Claude Code (projects onboarded earlier need `sandpipe update` first to install the skill)
- [ ] Push the glossary and ADRs to main (`baseBranch`) so the sandbox can read them; the skill only commits, it doesn't push
- [ ] This step is recommended, not required: you can skip it if you only run tickets that touch facts, but finish it before running tickets that need intent

## 3. Gate baseline

- [ ] `sandpipe baseline`: runs the full gate once against `origin/main` in a sandbox, and it must pass
  - While it runs, a heartbeat is printed every 5 minutes, and the full output is written live to `.sandcastle/logs/gate-*.log` (the path is printed at the start); `tail -f` it to follow along
  - If it fails: either main is already broken, or `gate.script` / `isolation` / `install` is wrong in the sandbox. Fix it and rerun; don't work tickets on a red baseline
  - If it passes but reports flaky failures: note the files listed by `GATE_FLAKY` and fix them first, or exclude them from the gate for now
- [ ] Note how long the gate took (`GATE_TIME` in the output), as a reference for spotting a slower gate later

## 4. Canary ticket (PR only)

- [ ] Open a small ticket that can hardly fail, such as changing a piece of copy or adding a comment, and label it `Sandcastle`
- [ ] `SANDCASTLE_ONLY=<number> sandpipe`
- [ ] Check:
  - prep wrote `## Implementation guide`, and the labels follow the state machine (see "Labels" in docs/config.md)
  - The PR body has Review, Gate and Usage; acceptance notes are there when pages changed
  - Because of `autoMerge: false`, the PR is left for a human to merge, with a comment saying why
  - Nothing odd in `.sandcastle/logs/`; the sandbox, worktree and test databases are all cleaned up
- [ ] Merge by hand after a human review

## 5. Real small tickets (merged by hand)

- [ ] Pick a real small ticket (at most 3 acceptance criteria) and run it the same way with `SANDCASTLE_ONLY`
- [ ] Review the PR: does the change stay within the ticket, do the tests really cover the acceptance criteria, do you agree with the reviewer's verdict
- [ ] If migrations, pages or cross-module changes are involved, run one ticket of each kind
- [ ] Move on only after several tickets in a row (3 or more suggested) where the reviewer said merge and you would have merged too

## 6. Let go

- [ ] Remove `autoMerge: false` from `sandcastle.config.ts` (auto-merge is the default)
- [ ] Raise `maxParallel` gradually; the limit depends on the services the tests use (such as database connections) and machine load
- [ ] For the first few runs without `SANDCASTLE_ONLY` over all labelled tickets, watch whether the fallback full run passes and whether `main-broken` / `flaky` tickets get opened

## Rolling back

When something goes wrong, set `autoMerge` back to `false` and `maxParallel` back to `1`, and return to steps 4 and 5.
