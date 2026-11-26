# 项目配置：`.sandcastle/sandcastle.config.ts`

中文 | [English](config.md)

`onboard` 按探测结果生成。必填的只有 `install`、`isolation`、`gate`，其余都有默认值。

```ts
import { defineProject, readHostEnv, runHostScript, toContainerHost, withDbSuffix } from "sandpipe";

export default defineProject({
  install: { onSandboxReady: "pnpm install --frozen-lockfile" },
  isolation: {
    // 每张票独立的测试库或前缀，并行的票互不清表。runId 是票号，main 上的兜底检查是 "main"
    // .env 里是 DATABASE_URL=postgres://test@localhost/shop_test，进沙箱的是 postgres://test@host.containers.internal/shop_test_i<runId>
    env: (runId) => ({
      DATABASE_URL: withDbSuffix(toContainerHost(readHostEnv(".env").DATABASE_URL ?? "postgres://test@localhost/shop_test"), `_i${runId}`),
    }),
    // 关沙箱后在宿主机上回收这张票的资源（删库等）。失败只记日志
    cleanup: async (runId) => {
      await runHostScript("node", ["scripts/drop-test-db.js", `_i${runId}`], `#${runId} 回收失败`);
    },
  },
  gate: {
    script: ({ baseRef }) => "pnpm typecheck && pnpm lint && pnpm test",
    testFile: /test\/\S+\.test\.ts/,
    typecheck: "pnpm typecheck",
  },
});
```

`import "sandpipe"` 不需要项目里装这个包，运行时会指向你装的 `sandpipe`。

## 必填

| 字段 | 说明 |
|---|---|
| `install.onSandboxReady` | 沙箱建好后装依赖的命令（在 linux 容器里跑） |
| `install.copyToWorktree` | 可选。从宿主机拷进 worktree 的路径，省掉从头安装 |
| `isolation.env(runId)` | 进沙箱的环境变量：独立的测试库、key 前缀等 |
| `isolation.cleanup(runId)` | 可选。关沙箱后回收这张票的测试资源 |
| `isolation.hostEnvFiles` | 可选。config 从宿主机读的 env 文件（相对项目根，比如 `[".env", ".env.test"]`），`sandpipe doctor` 检查它们存在，缺了警告。不写时 doctor 扫 config 源码里引号包着的 `.env` / `.env.*` 路径（不算 `.sandcastle/` 下的和 `*.example` / `*.sample`） |
| `gate.script({ baseRef })` | 合并门槛，在沙箱里用 `sh -c` 跑，退出码 0 算通过 |
| `gate.testFile` | 测试文件路径的正则，用来从门槛输出里认出失败的文件 |
| `gate.typecheck` | 等人审的 PR rebase 之后只跑这条命令 |

门槛脚本的输出约定（可选，用来按依赖选测试、识别偶发失败）：

- 输出一行 `GATE_TIER: selected …`，表示只跑了按依赖选出的测试；没有这一行就算完整测试。
- 环境变量 `GATE_FULL=1` 时必须跑完整测试（`baseline` 和 main 上的兜底检查会设）。
- 失败的文件单独重跑后通过时，输出 `GATE_FLAKY`。它前面列出重跑过的文件，后面是第一次失败的摘要。流水线会为这些文件开 `flaky` 票。
- 输出按纯文本逐行解析，ANSI 转义码会被去掉。门槛默认设了 `NO_COLOR=1` / `FORCE_COLOR=0`，想要颜色就在 `gate.env` 里设 `NO_COLOR: ""`；门槛脚本自己 grep 失败行时不要依赖颜色码。

## 隔离辅助函数

`sandpipe` 导出，给 `isolation.env` / `cleanup` 用。config 在宿主机上被加载，它们都跑在宿主机，相对路径按项目根算。

| 函数 | 做什么 |
|---|---|
| `readHostEnv(files, projectRoot?)` | 读宿主机上的一个或多个 env 文件（比如 `[".env", ".env.test"]`）并合并，后面的覆盖前面的。不存在的文件跳过，不报错 |
| `toContainerHost(value)` | 把 `localhost` / `127.0.0.1` 换成 `host.containers.internal`，沙箱才连得上宿主机上的服务 |
| `withDbSuffix(url, suffix)` | 给库名或前缀加后缀（比如 `` `_i${runId}` ``）。保留 query（`?authSource=admin`）和结尾的 `/`；不含 `://` 的值直接拼接。URL 没有库名时抛错，不让测试打到开发库 |
| `runHostScript(command, args, label)` | 在宿主机上跑回收脚本。成功返回 stdout；失败打一行警告（`label` 加 stderr 前 500 字）并返回 `undefined`，从不抛错 |

## 浏览器验收

有网页界面的项目，PROJECT.md 写了「浏览器验收」一节，agent 才会在沙箱里起 dev server、用浏览器点一遍改过的页面；没有这一节（只有 HTML 注释也算没有），改页面的票只能留 PR 给人看，`sandpipe doctor` 会警告。

- **dev server 要的变量放进 `isolation.env`**：dev server 在沙箱里读的变量（数据库、会话密钥、对象存储等）都要出现在 `isolation.env` 的返回值里，少一个就起不来。用上一节的 `readHostEnv` 从宿主机 env 文件读出来，指向本机服务的值过一遍 `toContainerHost`。
- **用独立的库**：和测试一样，库名用 `withDbSuffix` 加上 `` `_i${runId}` `` 后缀，验收数据不会碰到开发库，也不会和并行的票互相清表（写法见开头的示例）。
- **验收数据**：在 PROJECT.md 这一节第一步写建数据的命令（种子脚本、建账号）；给人验收时的固定准备步骤写在 `acceptancePrep`（见「可选」）。
- **写法**：按「准备数据、后台起服务并等它能访问、登录、查日志」四步写清命令、端口和账号。拿不准沙箱里能不能起来就先不写，doctor 会一直提示。

示例（命令、端口、账号换成项目自己的）：

```markdown
## 浏览器验收

1. 准备：`pnpm db:seed`（建 `admin@example.com` / `password` 账号）。
2. 起服务：`pnpm dev --port 3000 > /tmp/dev.log 2>&1 &`，等 `curl -s -o /dev/null http://localhost:3000/login` 成功。
3. 登录：`playwright-cli open http://localhost:3000/login`，用上面的账号登录，地址离开 `/login` 才算登录成功。
4. 查错时 `grep -n -i error /tmp/dev.log`。
```

## 可选

| 字段 | 默认 | 说明 |
|---|---|---|
| `autoMerge` | `true` | `false` 时全部通过的票也只留 PR 等人合并（onboard 生成 `false`，见 onboarding-checklist.zh-CN.md） |
| `mergeRequires` | `{ checks: "all", timeoutMinutes: 30 }` | 自动合并除了门槛还要等什么：`checks` 取 `"all"`（PR 上的检查全部通过）、`"required"`（只看必需检查）或 `"none"`（只看门槛，给 CI 不可靠的项目用）。PR 上没有检查时不阻塞合并。有检查失败、或等了 `timeoutMinutes` 分钟还没跑完时留 PR（`sandcastle:pr-open`），评论列出是哪些检查。等的时候这张票不占实现的空位 |
| `language` | `"zh"` | issue、PR、评论用的语言，`"zh"` 或 `"en"`：脚本写的固定文字（PR 正文、评论、自动开的票）和 agent 写的内容都跟着变，验收说明的三个小标题也按语言检查。终端输出也跟着变（设 `SANDPIPE_LANG=zh\|en` 可以单独指定；没有项目配置时看 `LANG`）。提交信息的语言看 PROJECT.md「提交」一节。onboard 取 `--lang`，没给时按最近的提交信息判断，提交少于 5 条时按项目文档的主要语言（PROJECT.md 草稿会注明是推断的） |
| `maxParallel` | 2 | 同时实现的票数，上限看测试依赖的服务和机器负载 |
| `remote` / `baseBranch` | `origin` / `main` | |
| `mergeMethod` | `rebase` | |
| `mainCheckEvery` | 4 | 按依赖选测试的票每合并几张，对 main 跑一次完整测试 |
| `implementMaxSessions` | 3 | 一张票最多开几次全新的 implement 会话 |
| `reviseMax` | 2 | reviewer 最多打回几次 |
| `contextBudget` | 150000 | 单个会话的上下文预算（token） |
| `agentIdleTimeoutS` | 2400 | agent 多久没输出算卡死 |
| `pollIntervalMs` / `maxIdlePolls` | 10 分钟 / 12 | 调度间隔；连续空转多少次后结束 |
| `labels` | 见下 | 改标签名 |
| `sandbox(env)` | podman | 换沙箱实现（`@ai-hero/sandcastle` 的 SandboxProvider） |
| `integrity` | 无 | 合并前的一致性检查：`{ name, command(head, baseRef), broken(baseRef) }`。比如检查数据库迁移链有没有分叉：rebase 没有文本冲突，迁移链也可能断 |
| `appendOnlyFiles` | `[]` | 只追加的索引文件。几张票同时追加不算改同一个文件，可以并行 |
| `acceptancePrep(branch)` | 无 | 改了页面的票，PR「验收说明」最前面固定的准备步骤 |
| `beforeSchedule()` | 无 | 每次调度开头在宿主机上跑的维护 |
| `texts` | 通用版 | 自动开的票和内联提示词的正文 |

## 标签

| 默认名 | 含义 |
|---|---|
| `Sandcastle` | 人工选中，流水线只处理带这个标签的票 |
| `sandcastle:prepped` | prep 已确认可以实现 |
| `sandcastle:pr-open` | 已开 PR，等人审 |
| `sandcastle:stuck` | 需要人介入，原因写在 issue 评论里 |
| `sandcastle:decided` | agent 替人拍板过设计决定，等人事后确认，看过后去掉 |
| `flaky` | 门槛里重跑才通过的测试，流水线自动开票 |
| `sandcastle:main-broken` | main 上的兜底完整测试失败，流水线自动开票，排在最前 |
| `ready-for-agent` / `ready-for-human` | 拆票后的分流 |

`sandpipe doctor --fix` 会建好缺的标签。
