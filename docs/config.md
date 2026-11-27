# Project config: `.sandcastle/sandcastle.config.ts`

[中文](config.zh-CN.md) | English

`onboard` generates it from what it detects. Only `install`, `isolation` and `gate` are required; everything else has a default.

```ts
import { defineProject, readHostEnv, runHostScript, toContainerHost, withDbSuffix } from "sandpipe";

export default defineProject({
  install: { onSandboxReady: "pnpm install --frozen-lockfile" },
  isolation: {
    // A separate test database or prefix per ticket, so parallel tickets don't wipe each other's tables. runId is the ticket number; the fallback check on main uses "main"
    // .env has DATABASE_URL=postgres://test@localhost/shop_test; the sandbox gets postgres://test@host.containers.internal/shop_test_i<runId>
    env: (runId) => ({
      DATABASE_URL: withDbSuffix(toContainerHost(readHostEnv(".env").DATABASE_URL ?? "postgres://test@localhost/shop_test"), `_i${runId}`),
    }),
    // Reclaim this ticket's resources on the host after the sandbox closes (drop databases etc.). Failures are only logged
    cleanup: async (runId) => {
      await runHostScript("node", ["scripts/drop-test-db.js", `_i${runId}`], `#${runId} cleanup failed`);
    },
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
| `isolation.hostEnvFiles` | Optional. Env files the config reads on the host (relative to the project root, e.g. `[".env", ".env.test"]`); `sandpipe doctor` warns when any is missing. When absent, doctor scans the config source for quoted `.env` / `.env.*` paths (ignoring ones under `.sandcastle/` and `*.example` / `*.sample`) |
| `gate.script({ baseRef })` | The merge gate, run in the sandbox with `sh -c`; exit code 0 means pass |
| `gate.testFile` | Regex for test file paths, used to pick failing files out of the gate output |
| `gate.typecheck` | The only command run after rebasing a PR that is waiting for review |
| `gate.services` | Optional. Services the gate depends on, `[{ name, host, port }]` (e.g. `[{ name: "postgres", host: "localhost", port: 5432 }]`). `sandpipe run` connects to each over TCP once at startup and exits with an error naming the service and `host:port` if any is unreachable; `sandpipe doctor` prints one ok / fail line per service. `host` is an address the host machine reaches directly |

Gate script output conventions (optional; used to select tests by dependency and detect flaky failures):

- Print a line `GATE_TIER: selected …` to say only dependency-selected tests ran; without it the run counts as the full suite.
- When the environment variable `GATE_FULL=1` is set, the full suite must run (`baseline` and the fallback check on main set it).
- When failing files pass on a solo rerun, print `GATE_FLAKY`. List the rerun files before it and the summary of the first failure after it. The pipeline opens `flaky` tickets for those files.
- When a service the gate depends on is unreachable (non-zero exit code and an output line matching `ECONNREFUSED`, `getaddrinfo ENOTFOUND` / `EAI_AGAIN`, `Cannot connect to the Docker daemon` / `Cannot connect to Podman`), the terminal prints that line, no fix session is opened, it doesn't count as a failure, and the whole step reruns later; if it's still unreachable once retries run out, the ticket is marked stuck with a comment naming that line. Add the project's own connection errors with the optional `gate.envErrorPattern` (a regex, e.g. `/MongoServerSelectionError/`). `No test files found` and ordinary assertion failures don't count.
- Output is parsed line by line as plain text, with ANSI escape codes stripped. The gate sets `NO_COLOR=1` / `FORCE_COLOR=0` by default; to get colors back, set `NO_COLOR: ""` in `gate.env`. When the gate script greps its own failure lines, do not rely on color codes.

## Isolation helpers

Exported from `sandpipe` for `isolation.env` / `cleanup`. The config is imported on the host, so they run on the host; relative paths resolve against the project root.

| Function | What it does |
|---|---|
| `readHostEnv(files, projectRoot?)` | Reads one or more host env files (e.g. `[".env", ".env.test"]`) and merges them, later files overriding earlier ones. Missing files are skipped, never an error |
| `toContainerHost(value)` | Replaces `localhost` / `127.0.0.1` with `host.containers.internal`, so the sandbox can reach services on the host |
| `withDbSuffix(url, suffix)` | Appends a suffix (e.g. `` `_i${runId}` ``) to the database name or prefix. Keeps the query (`?authSource=admin`) and a trailing `/`; values without `://` are concatenated as is. Throws when the URL has no database name, instead of letting tests hit the dev database |
| `runHostScript(command, args, label)` | Runs a cleanup script on the host. Returns stdout on success; on failure logs one warning (`label` plus the first 500 characters of stderr) and returns `undefined`. Never throws |

## Browser check

For a project with a web UI, the agent starts the dev server in the sandbox and clicks through the changed pages in a browser only when PROJECT.md has a Browser check section. Without it (a section holding only HTML comments counts as missing), tickets that change pages are left as PRs for a human; when a web UI is detected or `browser.pagePaths` is set, both `sandpipe run` at startup and `sandpipe doctor` warn. With it, a ticket that changes pages (the diff hits `browser.pagePaths`, there is a `.browser-check/`, or the reviewer's `browser-check` is `done` / `missing`) only auto-merges when: there is at least one screenshot; `RESULT.md` exists, has no ❌ / ⚠ in its list / table rows, and every screenshot it references exists; the reviewer's `browser-check` is `done` (the reviewer looks at the screenshots itself, gives a per-step `<browser-verdict>` against the ticket's Browser check, and only says `done` when every step is ✅ and agrees with `RESULT.md`), and the reviewer's per-step `browser-verdict` is present with no ❌ / ❓ rows (a `done` without it, or with such rows, blocks even when `RESULT.md` is all ✅). Anything missing leaves the PR open, with each item listed in the terminal and a PR comment. The check is taken at the end of implement: if revise or fix later changes files matching `browser.pagePaths` (diff from the commit at the check to HEAD), the screenshots are out of date and the PR is left open; after a resume (no recorded commit) or when that diff can't be read, it's treated as out of date too. If the agent resolving conflicts in the rebase fallback touches pages (conflicting files plus files newly appearing in the diff), the branch isn't pushed and the PR is left open. Changes that don't touch pages keep the check valid. The browser check is not redone automatically; a human rechecks. When pages changed but the ticket's implementation guide has no `- **Browser check**` item, the PR's Browser check section and a PR comment suggest adding a prep rule. The PR's Browser check section shows the reviewer's verdict first (the reviewer's `<browser-verdict>`, or a note that none was given), then the implementer's self-check (`RESULT.md`).

The Containerfile onboard generates installs playwright-cli and Chromium whether or not the project has a web UI. onboard first reads the project's existing acceptance material (README, test notes, old `.sandcastle` prompts, scripts under `scripts/` that create acceptance data); if it can piece together the four steps below it writes this section, otherwise it lists what's missing (dev server start command and port, how to create acceptance data, how to log in and with which account, the variables the dev server needs) in the open questions list. When a backend project adds a frontend: add this section to PROJECT.md (following the example below) and set `browser.pagePaths` (see Optional); the sandbox already has the browser.

- **Put the dev server's variables in `isolation.env`**: every variable the dev server reads in the sandbox (database, session secret, object storage, etc.) must be in what `isolation.env` returns, or it won't start. Read them from the host env files with `readHostEnv` from the previous section, and pass values that point at local services through `toContainerHost`.
- **Use a separate database**: as for tests, add a `` `_i${runId}` `` suffix to the database name with `withDbSuffix`, so acceptance data never touches the dev database and parallel tickets don't wipe each other's tables (see the example at the top).
- **Acceptance data**: make the first step of this section the command that creates data (seed script, accounts); fixed setup steps for the human's acceptance go in `acceptancePrep` (see Optional).
- **How to write it**: spell out commands, port and accounts in four steps: prepare data, start the server in the background and wait until it responds, log in, check the log. If you aren't sure it can start in the sandbox, leave the section out for now; doctor keeps reminding you.

Example (replace the commands, port and accounts with the project's own):

```markdown
## Browser check

1. Prepare: `pnpm db:seed` (creates the `admin@example.com` / `password` account).
2. Start the server: `pnpm dev --port 3000 > /tmp/dev.log 2>&1 &`, and wait until `curl -s -o /dev/null http://localhost:3000/login` succeeds.
3. Log in: `playwright-cli open http://localhost:3000/login`, log in with the account above; it only counts once the address leaves `/login`.
4. To find errors, `grep -n -i error /tmp/dev.log`.
```

## Optional

| Field | Default | Description |
|---|---|---|
| `autoMerge` | `true` | When `false`, even tickets that pass everything are left as PRs for a human to merge (onboard generates `false`, see onboarding-checklist.md) |
| `mergeRequires` | `{ checks: "all", timeoutMinutes: 30 }` | What auto-merge waits for besides the gate: `checks` is `"all"` (every check on the PR must pass), `"required"` (only required checks) or `"none"` (the gate alone, for projects whose CI is unreliable). A PR with no checks doesn't block the merge. When a check fails, or checks are still running after `timeoutMinutes` minutes, the PR is left open (`sandcastle:pr-open`) with a comment listing the checks. While waiting, the ticket doesn't hold an implementation slot |
| `language` | `"zh"` | Language for issues, PRs and comments, `"zh"` or `"en"`: the script's fixed texts (PR bodies, comments, auto-opened tickets) and what the agent writes follow it, and the three acceptance-notes headings are checked in that language. The terminal output follows it too (set `SANDPIPE_LANG=zh\|en` to override; without a project config it follows `LANG`). Commit message language comes from the Commits section of PROJECT.md. onboard picks it from `--lang`, or else from the recent commit messages; with fewer than 5 commits, from the main language of the project docs (the PROJECT.md draft notes that it was inferred) |
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
| `sandbox(env)` | podman | Swap the sandbox implementation (a `@ai-hero/sandcastle` SandboxProvider). Pass `env` into the container as-is: it blanks the Claude credentials so third-party profile sessions can't read them |
| `integrity` | none | Consistency check before merging: `{ name, command(head, baseRef), broken(baseRef) }`. For example, checking whether the database migration chain forked: a rebase can have no text conflict and still break the chain |
| `appendOnlyFiles` | `[]` | Append-only index files. Several tickets appending to one doesn't count as changing the same file, so they can run in parallel |
| `acceptancePrep(branch)` | none | For tickets that change pages, fixed setup steps at the top of the PR's acceptance notes |
| `browser.pagePaths` | `[]` | Regexes for page file paths (relative to the repo root), e.g. `[/^app\/routes\//]`; a ticket touching a matching file counts as changing pages. onboard writes a default for the detected frontend framework (next, remix / react-router, nuxt / vue, svelte, astro; other projects with a web UI get tsx / jsx / vue / svelte / css under `src/`), and leaves it out without a web UI |
| `protectedPaths` | `[]` | Regexes for protected paths (relative to the repo root, no leading `/`, matching `git diff --name-only` output), e.g. `[/^\.github\//, /^migrations\//]`. If a ticket's final diff (after review / revisions / fix) touches a matching file it isn't auto-merged; the PR is left open and the comment lists each "file — rule matched". When the agent resolved conflicts during a rebase, the diff is re-read and checked again before pushing. With rules configured, an unreadable diff also leaves the PR open. A ticket can allow specific files with a `- **Protected**: \`path\`, \`dir/\`` item in its implementation guide (relative to the repo root; a trailing `/` covers everything under that directory, otherwise it must equal the file in the diff): allowed hits don't block the merge but are listed in a `## Protected paths` section of the PR body (in the update comment when the PR already exists; allowed files newly touched while resolving rebase conflicts get a separate comment). A leading `./` or `/` is stripped; when protected paths leave the PR open, the comment lists allowed entries that matched no file. `onboard` writes defaults from what it detects: snapshots (`__snapshots__/` or `.snap`) give `/(^\|\/)__snapshots__\//` and `/\.snap$/`, Playwright snapshot directories give `/-snapshots\//`, `.github/workflows/` gives `/^\.github\/workflows\//`, a detected migration directory gives `/^<dir>\//`; nothing detected means no line. `sandpipe doctor` lists the current rules. Lockfiles are not a default (every ticket that adds a dependency changes them); add them yourself if needed (see `docs/adr/0003-protected-paths.md`) |
| `trustedAuthors` | `[]` | Extra logins whose tickets count as trusted, e.g. bot accounts not installed as a GitHub App. A ticket's author is trusted when it's a bot, is in this list, or has admin / write permission on the repo (maintain counts as write); tickets from untrusted authors without `sandcastle:trusted` aren't prepped or started: they get `needs-triage` and one comment. Comment markers (agent decisions, review passed, reprep, limit wait) and a PR's `/sandcastle` or Request changes also count only from trusted authors. Agents in the sandbox only read comments whose `author_association` is OWNER / MEMBER / COLLABORATOR or from bots; this list doesn't apply there |
| `beforeSchedule()` | none | Maintenance run on the host at the start of every scheduling pass |
| `texts` | generic | Bodies of auto-opened tickets and inline prompts |

## Labels

| Default name | Meaning |
|---|---|
| `Sandcastle` | Picked by a human; the pipeline only handles tickets with this label |
| `sandcastle:prepped` | prep confirmed the ticket can be implemented |
| `sandcastle:pr-open` | PR opened, waiting for human review |
| `sandcastle:reviewed` | Review passed; the gate is running. If the run is interrupted and the branch is unchanged, the next run goes straight to the gate instead of reimplementing and re-reviewing. Created by the pipeline on first use |
| `sandcastle:merging` | PR opened and the gate passed; waiting for checks / merging. If the run is interrupted, the next run resumes waiting instead of reimplementing. Created by the pipeline on first use |
| `sandcastle:stuck` | Needs a human; the reason is in an issue comment |
| `sandcastle:decided` | The agent made a design decision on a human's behalf; remove it after you've reviewed it |
| `flaky` | Tests that only passed on a rerun in the gate; the pipeline opens these tickets automatically |
| `sandcastle:main-broken` | The fallback full run on main failed; the pipeline opens this ticket automatically and puts it first |
| `ready-for-agent` / `ready-for-human` | Triage after splitting tickets |
| `needs-triage` | The ticket's author lacks write access; waiting for a maintainer to confirm it (`labels.needsTriage`). Created by the pipeline on first use |
| `sandcastle:trusted` | A maintainer confirmed this ticket can go to Sandcastle (`labels.trusted`; adding a label needs at least triage access). Created by the pipeline on first use |

`sandpipe doctor --fix` creates any missing labels.
