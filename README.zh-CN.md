# sandpipe

中文 | [English](README.md)

把 GitHub issue 无人值守地做成 PR、过测试后合并的流水线，基于 [`@ai-hero/sandcastle`](https://github.com/mattpocock/sandcastle)。

流水线按顺序做这些事：

1. 拆票，写实现指引；
2. 按依赖和改动的文件排期；
3. 在 podman 沙箱里让 Claude Code 实现，由另一个会话审查，必要时打回重改；
4. 跑你定义的测试门槛；
5. 全部通过就合并，否则留 PR 给人看。

## 需要

- Node ≥ 22.18
- 已登录的 `gh`（`gh auth login`）
- podman（macOS 上还要 `podman machine start`）
- Claude Code 的凭据：订阅用户可用 `claude setup-token` 生成

## 前提

Sandcastle 的产出质量取决于你的意图有多少写了下来。不用事先备齐：缺的部分 onboard 会带你补。它把找到的缺口写进缺口清单（早于清单接入的项目用 `onboard --questions-only` 补一份），你再用 `/fill-project-docs` 逐条回答。

项目知识分三类：

| 类别 | 能否从代码推出 | 由谁补 | 例子 |
|---|---|---|---|
| 事实 | 能 | onboard 的 agent 写进 PROJECT.md | 目录结构、命令、测试框架、测试依赖的服务 |
| 规范 | 部分能 | agent 起草，人确认 | 代码风格、测试写法、提交约定 |
| 意图 | 不能 | 人在 `/fill-project-docs` 里回答 | 业务规则、领域术语、红线、设计原因 |

只涉及事实的票（修 bug、照已有的样子加字段）没有这些文档也能做。需要的意图没写下来的票会停下来，等人补上。

## 安装

```bash
npm i -g sandpipe
# 或
pnpm add -g sandpipe   # 第一次用 pnpm 装全局包：先跑 `pnpm setup` 再重开终端，把它的全局 bin 目录加进 PATH
```

也可以直接装 GitHub Release 上的包：`npm i -g https://github.com/kobe4cn/sandcastle-releases/releases/latest/download/sandpipe.tgz`。

以后升级：`sandpipe upgrade`，然后在各项目里跑 `sandpipe update`。

## 接入一个项目

在项目根目录：

```bash
sandpipe onboard          # 探测项目，生成 .sandcastle/，让 claude 读代码写好 PROJECT.md
cp .sandcastle/.env.example .sandcastle/.env    # 填 token
podman build -t sandcastle:<项目名> -f .sandcastle/Containerfile .sandcastle   # onboard 最后会打印准确的命令
sandpipe doctor --fix     # 逐项检查，顺带建好标签
sandpipe baseline         # 在沙箱里对 main 跑一次完整门槛
```

然后给 issue 打上 `Sandcastle` 标签，运行：

```bash
SANDCASTLE_ONLY=12 sandpipe    # 只做 #12；不设就处理全部带标签的票
```

新项目默认只开 PR、不自动合并（`autoMerge: false`），一次只做一张票。按 [docs/onboarding-checklist.zh-CN.md](docs/onboarding-checklist.zh-CN.md) 走完金丝雀票、真实小票，确认没问题后再放开。

自动合并要门槛和 PR 上的检查都通过。CI 覆盖了门槛没覆盖的东西时，可以考虑把它们加进 `gate.script`；CI 不可靠时设 `mergeRequires: { checks: "none" }`（见 [docs/config.zh-CN.md](docs/config.zh-CN.md)）。

## 生成的文件

| 文件 | 作用 |
|---|---|
| `.sandcastle/PROJECT.md` | 给 agent 看的项目知识：概况、命令、测试、规范、红线、冲突处理、提交约定 |
| `.sandcastle/sandcastle.config.ts` | 怎么跑：装依赖、测试隔离、合并门槛。字段说明见 [docs/config.zh-CN.md](docs/config.zh-CN.md) |
| `.sandcastle/models.json` | 各阶段（prep / implement / review …）用哪个模型 |
| `.sandcastle/.env` | token，不提交 |
| `.sandcastle/Containerfile` | 沙箱镜像 |
| `.claude/` | 上下文预算 hook、截图查看子代理、`/prep-ticket` 和 `/fill-project-docs` skill |

## 命令

| 命令 | 作用 |
|---|---|
| `sandpipe [run]` | 跑流水线 |
| `sandpipe onboard [--no-agent] [--model <名>] [--lang zh\|en]` | 接入新项目，`.sandcastle/` 下已有的文件不覆盖；`.claude/` 下本工具的文件和模板不同时显示差异，问保留、覆盖还是另存为 `.new`（非交互时另存 `.new`） |
| `sandpipe onboard --questions-only [--model <名>] [--lang zh\|en]` | 只生成缺口清单 `.sandcastle/OPEN-QUESTIONS.md`，给早于清单接入的项目；不动其他文件 |
| `sandpipe doctor [--fix]` | 跑之前的检查；有 ✗ 时退出码为 1 |
| `sandpipe baseline` | 对 `origin/main` 跑一次完整门槛，只报告 |
| `sandpipe update` | 把 `.claude/` 下本工具的文件刷成当前版本，和模板不同的按 onboard 的规则询问；装过全局 skill 的话一起刷新 |
| `sandpipe skill` | 把 `/prep-ticket` 装到 `~/.claude/skills/`，没接入 sandcastle 的仓库也能用（同名时它盖过项目里的那份，`update` 会顺带刷新） |
| `sandpipe upgrade` | 安装最新版本 |
| `sandpipe --version` | |

现在只支持 GitHub，GitLab 还没做。
