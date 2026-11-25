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
  - `gate.script` is a gate you're willing to merge on; consider `integrity` if there are migrations
  - Keep the `autoMerge: false` and `maxParallel: 1` that onboard generated
- [ ] `cp .sandcastle/.env.example .sandcastle/.env` and fill in the tokens
- [ ] Build the image with the command onboard printed at the end (`podman build -t sandcastle:<project> …`)

## 1. doctor passes

- [ ] `sandpipe doctor --fix`: no ✗. Read every ⚠ and confirm it's acceptable

## 2. Gate baseline

- [ ] `sandpipe baseline`: runs the full gate once against `origin/main` in a sandbox, and it must pass
  - If it fails: either main is already broken, or `gate.script` / `isolation` / `install` is wrong in the sandbox. Fix it and rerun; don't work tickets on a red baseline
  - If it passes but reports flaky failures: note the files listed by `GATE_FLAKY` and fix them first, or exclude them from the gate for now
- [ ] Note how long the gate took (`GATE_TIME` in the output), as a reference for spotting a slower gate later

## 3. Canary ticket (PR only)

- [ ] Open a small ticket that can hardly fail, such as changing a piece of copy or adding a comment, and label it `Sandcastle`
- [ ] `SANDCASTLE_ONLY=<number> sandpipe`
- [ ] Check:
  - prep wrote `## Implementation guide`, and the labels follow the state machine (see "Labels" in docs/config.md)
  - The PR body has Review, Gate and Usage; acceptance notes are there when pages changed
  - Because of `autoMerge: false`, the PR is left for a human to merge, with a comment saying why
  - Nothing odd in `.sandcastle/logs/`; the sandbox, worktree and test databases are all cleaned up
- [ ] Merge by hand after a human review

## 4. Real small tickets (merged by hand)

- [ ] Pick a real small ticket (at most 3 acceptance criteria) and run it the same way with `SANDCASTLE_ONLY`
- [ ] Review the PR: does the change stay within the ticket, do the tests really cover the acceptance criteria, do you agree with the reviewer's verdict
- [ ] If migrations, pages or cross-module changes are involved, run one ticket of each kind
- [ ] Move on only after several tickets in a row (3 or more suggested) where the reviewer said merge and you would have merged too

## 5. Let go

- [ ] Remove `autoMerge: false` from `sandcastle.config.ts` (auto-merge is the default)
- [ ] Raise `maxParallel` gradually; the limit depends on the services the tests use (such as database connections) and machine load
- [ ] For the first few runs without `SANDCASTLE_ONLY` over all labelled tickets, watch whether the fallback full run passes and whether `main-broken` / `flaky` tickets get opened

## Rolling back

When something goes wrong, set `autoMerge` back to `false` and `maxParallel` back to `1`, and return to steps 3 and 4.
