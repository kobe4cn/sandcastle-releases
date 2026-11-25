# Project config: `.sandcastle/sandcastle.config.ts`

[中文](config.zh-CN.md) | English

`onboard` generates it from what it detects. Only `install`, `isolation` and `gate` are required; everything else has a default.

```ts
import { defineProject } from "sandpipe";

export default defineProject({
  install: { onSandboxReady: "pnpm install --frozen-lockfile" },
  isolation: {
    // A separate test database or prefix per ticket, so parallel tickets don't wipe each other's tables. runId is the ticket number; the fallback check on main uses "main"
    env: (runId) => ({ DATABASE_URL: `postgres://test@host.containers.internal/shop_test_${runId}` }),
    // Reclaim this ticket's resources on the host after the sandbox closes (drop databases etc.). Failures are only logged
    cleanup: async (runId) => {},
  },
  gate: {
    script: ({ baseRef }) => "pnpm typecheck && pnpm lint && pnpm test",
    testFile: /test\/\S+\.test\.ts/,
    typecheck: "pnpm typecheck",
  },
});
```

`import "sandpipe"` doesn't need the package installed in the project; at runtime it resolves to the `sandpipe` you installed.

## Required

| Field | Description |
|---|---|
| `install.onSandboxReady` | Command that installs dependencies once the sandbox is ready (runs in a linux container) |
| `install.copyToWorktree` | Optional. Paths copied from the host into the worktree to avoid installing from scratch |
| `isolation.env(runId)` | Environment variables passed into the sandbox: a separate test database, key prefixes, etc. |
| `isolation.cleanup(runId)` | Optional. Reclaims this ticket's test resources after the sandbox closes |
| `gate.script({ baseRef })` | The merge gate, run in the sandbox with `sh -c`; exit code 0 means pass |
| `gate.testFile` | Regex for test file paths, used to pick failing files out of the gate output |
| `gate.typecheck` | The only command run after rebasing a PR that is waiting for review |

Gate script output conventions (optional; used to select tests by dependency and detect flaky failures):

- Print a line `GATE_TIER: selected …` to say only dependency-selected tests ran; without it the run counts as the full suite.
- When the environment variable `GATE_FULL=1` is set, the full suite must run (`baseline` and the fallback check on main set it).
- When failing files pass on a solo rerun, print `GATE_FLAKY`. List the rerun files before it and the summary of the first failure after it. The pipeline opens `flaky` tickets for those files.

## Optional

| Field | Default | Description |
|---|---|---|
| `autoMerge` | `true` | When `false`, even tickets that pass everything are left as PRs for a human to merge (onboard generates `false`, see onboarding-checklist.md) |
| `language` | `"zh"` | Language for issues, PRs and comments, `"zh"` or `"en"`: the script's fixed texts (PR bodies, comments, auto-opened tickets) and what the agent writes follow it, and the three acceptance-notes headings are checked in that language. The terminal output follows it too (set `SANDPIPE_LANG=zh\|en` to override; without a project config it follows `LANG`). Commit message language comes from the Commits section of PROJECT.md. onboard picks it from `--lang`, or else from the recent commit messages |
| `maxParallel` | 2 | How many tickets are implemented at once; the limit depends on the services the tests use and the machine load |
| `remote` / `baseBranch` | `origin` / `main` | |
| `mergeMethod` | `rebase` | |
| `mainCheckEvery` | 4 | After every this many merged tickets whose gate selected tests by dependency, run the full suite once on main |
| `implementMaxSessions` | 3 | Maximum fresh implement sessions per ticket |
| `reviseMax` | 2 | Maximum times the reviewer sends a ticket back |
| `contextBudget` | 150000 | Context budget per session (tokens) |
| `agentIdleTimeoutS` | 2400 | How long without output before the agent counts as stuck |
| `pollIntervalMs` / `maxIdlePolls` | 10 minutes / 12 | Scheduling interval; how many idle polls in a row end the run |
| `labels` | see below | Rename labels |
| `sandbox(env)` | podman | Swap the sandbox implementation (a `@ai-hero/sandcastle` SandboxProvider) |
| `integrity` | none | Consistency check before merging: `{ name, command(head, baseRef), broken(baseRef) }`. For example, checking whether the database migration chain forked: a rebase can have no text conflict and still break the chain |
| `appendOnlyFiles` | `[]` | Append-only index files. Several tickets appending to one doesn't count as changing the same file, so they can run in parallel |
| `acceptancePrep(branch)` | none | For tickets that change pages, fixed setup steps at the top of the PR's acceptance notes |
| `beforeSchedule()` | none | Maintenance run on the host at the start of every scheduling pass |
| `texts` | generic | Bodies of auto-opened tickets and inline prompts |

## Labels

| Default name | Meaning |
|---|---|
| `Sandcastle` | Picked by a human; the pipeline only handles tickets with this label |
| `sandcastle:prepped` | prep confirmed the ticket can be implemented |
| `sandcastle:pr-open` | PR opened, waiting for human review |
| `sandcastle:stuck` | Needs a human; the reason is in an issue comment |
| `sandcastle:decided` | The agent made a design decision on a human's behalf; remove it after you've reviewed it |
| `flaky` | Tests that only passed on a rerun in the gate; the pipeline opens these tickets automatically |
| `sandcastle:main-broken` | The fallback full run on main failed; the pipeline opens this ticket automatically and puts it first |
| `ready-for-agent` / `ready-for-human` | Triage after splitting tickets |

`sandpipe doctor --fix` creates any missing labels.
