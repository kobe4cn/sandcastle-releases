# 新项目接入检查清单

中文 | [English](onboarding-checklist.md)

从 `onboard` 到放手让 Sandcastle 自动合并，按顺序走；某一步没过就停下修，不要跳。
命令都在目标项目根目录跑；配置放在别处时先 `export SANDCASTLE_PROJECT_DIR=<目录>`。

## 0. 生成配置

- [ ] `sandpipe onboard`，生成 `.sandcastle/`，agent 补全 PROJECT.md
- [ ] 人看一遍 `PROJECT.md`：去掉所有 `<!-- 待补` 标记，命令、目录、约定都和项目实际一致
- [ ] 人看一遍 `sandcastle.config.ts`：
  - `install.onSandboxReady` 在 linux 容器里能装好依赖
  - `isolation.env` 让每张票用独立的库名或前缀（并行时互不清表）；有外部服务时配上 `cleanup`
    - onboard 已按 `.env.example` / `.env` 里指向本机的变量生成初稿（都没有时按 compose 服务猜键名）：运行时从项目根 `.env` 读、换主机、数据库各自加 `_i<runId>`；核对有没有漏掉的库或中间件、有没有不该进沙箱的
    - config 读项目根的 `.env` / `.env.test` 时，在 `isolation.hostEnvFiles` 里写上，并在宿主机准备好这些文件（通常被 gitignore，新克隆的仓库没有；`sandpipe doctor` 会警告）
    - 有网页界面时，PROJECT.md 要有「浏览器验收」，dev server 要的变量也要在 `isolation.env` 里（用独立的库），写法见 [config 的「浏览器验收」](config.zh-CN.md#浏览器验收)；没有这一节 `sandpipe doctor` 会警告；onboard 拼不全时会把缺的信息列进缺口清单。沙箱镜像不管有没有网页界面都装了浏览器，后来加前端只需补这一节
  - `gate.script` 就是你愿意据此合并的门槛；有迁移时考虑配 `integrity`
  - 保持 onboard 生成的 `autoMerge: false`、`maxParallel: 1`
- [ ] `cp .sandcastle/.env.example .sandcastle/.env`，填好流水线用的 token（`CLAUDE_CODE_OAUTH_TOKEN`，`claude setup-token` 生成，或 `ANTHROPIC_API_KEY`）；流水线只用这里面的 token，与本机 Claude Code 登录无关
- [ ] 跑 `sandpipe doctor --fix` 建镜像（第一次要几分钟）

## 1. doctor 全过

- [ ] `sandpipe doctor --fix`：没有 ✗。⚠ 要逐条看过，确认可以接受

## 2. 补文档（建议）

- [ ] 看 `sandpipe doctor` 的「项目知识」检查。已有项目还没有 `.sandcastle/OPEN-QUESTIONS.md` 的，先跑 `sandpipe onboard --questions-only`
- [ ] 在 Claude Code 里运行 `/fill-project-docs`（早先接入的项目要先跑 `sandpipe update` 才能装上这个 skill）
- [ ] 把术语表和 ADR 推到 main（`baseBranch`），沙箱才读得到；skill 只提交、不推送
- [ ] 这一步是建议：只做涉及事实的票可以跳过，做需要意图的票之前要补完

## 3. 门槛基线

- [ ] `sandpipe baseline`：在沙箱里对 `origin/main` 跑一次完整门槛，必须通过
  - 运行期间每 5 分钟打印一次心跳，完整输出实时写在 `.sandcastle/logs/gate-*.log`（开头会打印路径），可以 `tail -f` 跟着看
  - 没过：要么 main 本来就坏，要么 `gate.script` / `isolation` / `install` 在沙箱里不对。修好再跑，不要带着红的基线做票
  - 通过但提示有偶发失败：记下 `GATE_FLAKY` 列出的文件，先修，或者暂时从门槛里排除
- [ ] 记下这次门槛用了多久（输出里的 `GATE_TIME`），作为之后判断门槛是否变慢的参照

## 4. 金丝雀票（只开 PR）

- [ ] 开一张几乎不会失败的小票，比如改一处文案或补一条注释，打上 `Sandcastle` 标签
- [ ] `SANDCASTLE_ONLY=<票号> sandpipe`
- [ ] 核对：
  - prep 写了 `## Implementation guide`，标签按状态机走（见 docs/config.zh-CN.md 的「标签」）
  - PR 正文有 Review、Gate、用量；改了页面时有验收说明
  - 因为 `autoMerge: false`，PR 留着等人合并，评论写明原因
  - `.sandcastle/logs/` 里没有异常；沙箱、worktree 和测试库都清干净了
- [ ] 人审过后手动合并

## 5. 真实小票（人工合并）

- [ ] 挑一张真实的小票（验收标准不超过 3 条），同样用 `SANDCASTLE_ONLY` 跑
- [ ] 人审 PR：改动是否在票的范围内，测试是否真的覆盖了验收标准，reviewer 的结论你是否同意
- [ ] 如果涉及迁移、页面、跨模块，再各跑一张这种类型的票
- [ ] 连续几张票（建议 3 张以上）reviewer 给 merge 时你也都会合，才进下一步

## 6. 放开

- [ ] `sandcastle.config.ts` 去掉 `autoMerge: false`（默认就是自动合并）
- [ ] `maxParallel` 逐步调高；上限看测试依赖的服务（比如数据库连接数）和机器负载
- [ ] 前几轮去掉 `SANDCASTLE_ONLY` 跑全部带标签的票时，留意：兜底完整测试是否通过，有没有自动开出 `main-broken` / `flaky` 票

## 回退

出问题时把 `autoMerge` 改回 `false`、`maxParallel` 改回 `1`，回到第 4、5 步。
