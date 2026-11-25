# 项目配置：`.sandcastle/sandcastle.config.ts`

中文 | [English](config.md)

`onboard` 按探测结果生成。必填的只有 `install`、`isolation`、`gate`，其余都有默认值。

```ts
import { defineProject } from "sandpipe";

export default defineProject({
  install: { onSandboxReady: "pnpm install --frozen-lockfile" },
  isolation: {
    // 每张票独立的测试库或前缀，并行的票互不清表。runId 是票号，main 上的兜底检查是 "main"
    env: (runId) => ({ DATABASE_URL: `postgres://test@host.containers.internal/shop_test_${runId}` }),
    // 关沙箱后在宿主机上回收这张票的资源（删库等）。失败只记日志
    cleanup: async (runId) => {},
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
| `gate.script({ baseRef })` | 合并门槛，在沙箱里用 `sh -c` 跑，退出码 0 算通过 |
| `gate.testFile` | 测试文件路径的正则，用来从门槛输出里认出失败的文件 |
| `gate.typecheck` | 等人审的 PR rebase 之后只跑这条命令 |

门槛脚本的输出约定（可选，用来按依赖选测试、识别偶发失败）：

- 输出一行 `GATE_TIER: selected …`，表示只跑了按依赖选出的测试；没有这一行就算完整测试。
- 环境变量 `GATE_FULL=1` 时必须跑完整测试（`baseline` 和 main 上的兜底检查会设）。
- 失败的文件单独重跑后通过时，输出 `GATE_FLAKY`。它前面列出重跑过的文件，后面是第一次失败的摘要。流水线会为这些文件开 `flaky` 票。

## 可选

| 字段 | 默认 | 说明 |
|---|---|---|
| `autoMerge` | `true` | `false` 时全部通过的票也只留 PR 等人合并（onboard 生成 `false`，见 onboarding-checklist.zh-CN.md） |
| `language` | `"zh"` | issue、PR、评论用的语言，`"zh"` 或 `"en"`：脚本写的固定文字（PR 正文、评论、自动开的票）和 agent 写的内容都跟着变，验收说明的三个小标题也按语言检查。终端输出也跟着变（设 `SANDPIPE_LANG=zh\|en` 可以单独指定；没有项目配置时看 `LANG`）。提交信息的语言看 PROJECT.md「提交」一节。onboard 取 `--lang`，没给时按最近的提交信息判断 |
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
