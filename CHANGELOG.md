# 更新记录

## 0.2.1

- 在项目外运行不再像崩溃：`sandpipe` / `run` / `baseline` 找不到 `.sandcastle/sandcastle.config.ts`（或其他没接住的错误）时只打一行 `✗ 提示`、退出码 1，不再先打一整行压缩后的源码和堆栈；要完整堆栈设 `SANDPIPE_DEBUG=1`（e2e A3 发现）
- `sandpipe onboard` 不在 git 仓库根目录时直接报错、不写任何文件；之前在非 git 目录里会照样生成配置，远端写成 `fatal: not a git repository…`、基线猜成 master（e2e A3 发现）
- `sandpipe update` 在没接入的目录（没有 `.sandcastle/sandcastle.config.ts`）里报错、不写任何文件；之前会在当前目录建出 `.claude/` 和 `.sandcastle/`（e2e A3 发现）
- 发版：npm 发布关掉 provenance。源码仓库公开后 npm 会自动生成 provenance，要求 `repository.url` 指向源码仓库，而 `repository` 有意指向 releases 仓库，0.2.0 因此只发到了 GitHub Release、没发到 npm；0.2.0 的内容都在 0.2.1 里

## 0.2.0

- 英文终端和 PR 评论里等检查的分钟数按单复数写：`up to 1 minute`、`after 1 minute`，不再写成 `1 minutes`（e2e I3 发现）
- 等 PR 上的检查时，过了宽限期 PR 上仍然没有任何检查（仓库没有 CI）、按只看门槛合并的，终端多打一行 `· #n 的 PR #m 上没有检查（没有 CI），只按门槛合并`，不再在 ⧗ 之后直接合并、看不出原因；`waitForChecks` 这种情况返回 `{ ok: true, noChecks: true }`（e2e I8 发现）
- 运行结束的「全部完成」只在还有带 `sandcastle:pr-open` 的 open 票时才提醒去审，没有就不提（查询失败时照旧提醒）（e2e 发现）
- 门槛输出不再带 ANSI 颜色码：跑门槛（首次门槛、修复后、rebase 后、baseline、main 兜底完整测试，都走 `runGate`）时环境默认设 `NO_COLOR=1`、`FORCE_COLOR=0`，`isolation.env` 和 `gate.env` 里给了同名变量时以它们为准（新导出纯函数 `gateEnv`，`src/gate-progress.ts`）；宿主机一侧统一去掉转义码（`startGateProgress` 的 `onLine`，以及 provider 没回调 onLine 时兜底的 stdout），所以 `.sandcastle/logs/gate-*.log`、PR 正文的 Gate 一节、评论、flaky 的失败文件解析和票正文、fix 提示词都是纯文本。想要颜色就在 `gate.env` 里设 `NO_COLOR: ""`。config 文档（中英）的输出约定补上这一条（closes #21）。
- 内部：合并收尾（等 PR 上的检查、合并、分支保护 / rebase 补救、留 PR，以及等检查时让出空位与宿主机锁）从 `src/main.ts` 抽到新模块 `src/merge-flow.ts`，依赖由参数注入，补单测 `test/merge-flow.test.ts`。行为、终端输出和 PR 评论不变（closes #31）。
- `doctor` 新增「合并条件」一项：`autoMerge: true`、`mergeRequires.checks` 为 `"none"`、而仓库的 `.github/workflows` 里有 `.yml` / `.yaml` 时警告「合并只看门槛、不等 CI」，提示 CI 覆盖了门槛没覆盖的东西时加进 `gate.script` 或把 `checks` 改回 `"all"` / `"required"`；其余情况给通过并写明当前合并条件（如「门槛 + PR 上所有检查」）。只看本地 workflow 文件，不查分支保护；只警告，`--fix` 不处理（closes #30）。
- 自动合并时 `gh pr merge` 被仓库的分支保护挡住（要求审批、必需检查、合并队列等）不再当成「没能直接合并」去 rebase 补救、白跑一遍门槛：直接留 PR（`sandcastle:pr-open`），评论写明是分支保护挡住的、附 gh 报错前 300 字，提示人去合并或调整分支保护 / `mergeRequires`（中英文案 `reasonBranchProtection`）；rebase 补救里的合并重试碰到同样的报错也不再继续重试。只按 gh 的报错文本判断（新导出 `isBranchProtectionError`，`src/merge-checks.ts`），不调 API 查保护规则（closes #29）。
- **默认行为变了**：`autoMerge: true` 时，自动合并现在要门槛和 PR 上的 forge 检查（CI）都通过。已开 `autoMerge` 的项目升级后会按默认的 `mergeRequires: { checks: "all", timeoutMinutes: 30 }` 等 PR 上的检查再合并；想保持旧行为（只看门槛）要显式设 `mergeRequires: { checks: "none" }`。检查失败或等了 `timeoutMinutes` 分钟还没跑完时不合并，留 PR（`sandcastle:pr-open`），评论列出是哪些检查和链接（中英文案 `reasonChecksFailed` / `reasonChecksTimeout`）；PR 上没有检查不阻塞。rebase 补救 force-push 后先重新等检查再合并。等检查时放开宿主机锁，这张票不占实现的空位（调度可以开新票），但仍算在做：不会被重新开工，改的文件照样占着。forge 没实现 `prChecks` 时每次运行在终端警告一次，按没有检查合并。README（中英）、config 文档（中英）的可选字段表加 `mergeRequires`，新增 ADR 0002（closes #28）。
- 合并前等 forge 检查的底层（还没接进 land，行为不变）：新配置项 `mergeRequires: { checks?: "none" | "required" | "all"; timeoutMinutes?: number }`，默认 `{ checks: "all", timeoutMinutes: 30 }`；Forge 新增**可选**方法 `prChecks(id, requiredOnly)`（新导出类型 `PrCheck`），GitHub 实现用 `gh pr checks --json name,bucket,link`（`skipping` 算通过、`cancel` 算失败，没有检查返回空数组），自己写的 forge 不实现时按没有检查处理；新模块 `src/merge-checks.ts` 提供 `checksVerdict` 与可注入时钟的 `waitForChecks`（每 30 秒轮询，有失败立刻返回，超时返回 `why: "timeout"`，空列表宽限 60 秒后当作没有 CI）（closes #27）。
- 票正文写着「要先由人决定」的待决定项（「方案 A / B」「是否修订 ADR」之类）、但正文和人的评论里都没写明选了哪个时，prep 不再从 `ready-for-agent` 标签推断已经决定：打 `sandcastle:stuck` + `ready-for-human`，评论列出选项和建议；agent 写的 `<!-- sandcastle:decided -->` 评论不算人的决定。DECISIONS 规则补上：碰到「必须停下」（含项目红线，比如修订 ADR）的，不管票带什么标签都交给人，票正文列出但没写明选了哪个的待决定项也归这一类；只有票正文或人的评论明确要求时才算已经决定，标签和 agent 写的 decided 评论都不算。reviewer 看到 decided 评论选的正是票正文的待决定项、或内容碰了「必须停下」时，写进「需要人判断」，票留 PR、不自动合并。只改提示词，不新增标签（closes #18）。
- 没配浏览器验收的项目，改了页面的 PR 不再在「准备」段写「本票没改页面，不用起 dev server」：review 提示词要求 reviewer 在摘要后单独输出机器可读的 `<browser-check>` 标签（`none` / `done` / `missing` 三选一）（配不配浏览器验收都输出，和摘要里「浏览器验收」那一行对应）；没有 `.browser-check/` 但标签是 `done` 或 `missing` 的票也算改了页面（新增 `pageChangedByReview`，`src/pr-body.ts`）。改了页面但项目没配 `acceptancePrep` 时，准备段写「本票改了页面，需要起 dev server（项目没配准备步骤）」（英文同步）。有 `.browser-check/` 或配了 `acceptancePrep` 的票行为不变（closes #17）。
- 一张票都没开工就空转到上限退出时，不再打印「全部完成」，改为说明原因：列出最后一次调度里被 blocker 挡住的票（`#n 等 #m、#k`）；没有被挡的票时说明没有可开工的票（没有带 `Sandcastle` 标签、还没开工的票，或者都在等人审 PR / 带 `sandcastle:stuck`）。开工过票时照旧打印「全部完成」。原因文案由新增的 `idleSummary`（`src/guide.ts`）生成（closes #26）。
- `sandpipe doctor` 新增「Blocked by」检查：列出带 `Sandcastle` 标签的 open 票，逐个确认 `## Blocked by` 里引用的票在本仓库是否存在；有不存在的引用时警告并列出 `#n → #m`，提示改正 Blocked by，接入时就能发现，不用等调度打 stuck。引用都存在或没有这类票时通过；列票失败时整项警告，单个 blocker 查询出错（不是不存在）时跳过它。不会不通过，`--fix` 不处理（closes #25）。
- 调度区分「blocker 不存在」和「还开着」：`## Blocked by` 里的票号在本仓库查不到时，prep 和 plan 打印「跳过 #n：blocker #m 不存在或无权访问」，给票打 `sandcastle:stuck` 并留言说明（票号写错或从别的仓库复制来的，改正后去掉标签即可），不再当成 open 一直跳过；网络、权限等其他查询错误仍按 open 处理，不打 stuck。`Forge.issueState` 的返回类型加宽为 `"open" | "closed" | "missing"`（`missing` = 平台确认编号不存在），自定义 forge 不用改照样能编译；GitHub 实现在 gh 报 `Could not resolve to an issue or pull request` 时返回 `missing`（closes #24）。
- 有网页界面的项目补「浏览器验收」有指引了：config 文档（中英）新增「浏览器验收」一节，说明 dev server 在沙箱里要的变量放进 `isolation.env`、用 `withDbSuffix` 分出独立的库、验收数据怎么建，并附 PROJECT.md 这一节的通用写法；接入清单（中英）和 onboard 提示词指向它。`sandpipe doctor` 新增「浏览器验收」检查：探测到网页界面但 PROJECT.md 没有这一节时警告并指向文档，有时通过，没有网页界面时不出现；「PROJECT.md」一项不再附带浏览器验收的说明（closes #15）。
- onboard 生成缺口清单 `.sandcastle/OPEN-QUESTIONS.md`（提交进仓库），取代原来只打印到终端、只存在 `logs/` 里的拿不准的点。每条一行 `- [类别] 问题 → 写到哪`，类别是 `规范` / `意图`（英文项目 `convention` / `intent`）；清单已存在就不碰。写完校验格式，结尾提示剩几条，以及在 Claude Code 里运行 `/fill-project-docs`。
- onboard 的 agent 可写的文件从只有 PROJECT.md 改成一个列表（PROJECT.md 和缺口清单）。
- 探测文档时认 `GLOSSARY.md`。
- `sandpipe onboard --questions-only`：给早于缺口清单接入的项目补一份 `.sandcastle/OPEN-QUESTIONS.md`。agent 读现有 PROJECT.md 和代码找缺口，只许写这份清单，不生成、不改动其他文件；清单已存在时提示删掉后重跑。
- `sandpipe doctor` 新增「项目知识」检查，读缺口清单：还有 N 条时警告，提示在 Claude Code 里运行 `/fill-project-docs`；清单不存在时警告，提示运行 `sandpipe onboard --questions-only`；清单为空时通过；格式错时警告并指出行号。这项检查不会不通过；没有 PROJECT.md 时不做。
- 新增 `/fill-project-docs` skill（装在 `.claude/skills/fill-project-docs/`，附 NOTICE）：在 Claude Code 里和人逐条补完缺口清单，一次问一个问题，能从代码查到的事实不问人；答案按类别写进 PROJECT.md 对应的节、术语表（`CONTEXT.md` / `GLOSSARY.md`，都没有就新建 `GLOSSARY.md`）或 ADR，写完从清单删掉这一条；之后可选梳理核心术语，跑 `sandpipe doctor` 自检，人确认后提交。已接入的项目要跑 `sandpipe update` 才装得上；`sandpipe doctor` 会在它缺失或被改过时警告。
- 缺口清单的「写到哪」不再指向 PROJECT.md 里不存在的节：onboard（含 `--questions-only`）的提示词列出可用的节名（中文项目列中文那套，英文项目列英文那套），写明不能新建节，放不进的写最接近的那一节，清单示例也按项目语言给（英文项目的示例用英文类别和节名）；onboard 写完校验清单时，目标节名不认识的条目打印警告并给出行号（只警告，不算格式错误）。
- prep 遇到缺意图信息的票（业务规则、领域术语、红线、设计原因在票、注入的项目知识和代码里都找不到）不再猜着做：打 `sandcastle:stuck` + `ready-for-human`，评论写明缺什么、应该写进哪份文档（PROJECT.md 的哪一节、术语表或 ADR），提示在 Claude Code 里运行 `/fill-project-docs` 补上后去掉标签。不新增标签，缺口清单不注入提示词（closes #7）。
- 中英文 README 新增「前提」一节：产出质量取决于写下来的意图，缺的部分 onboard 会带你补；用一张表列出事实、规范、意图三类项目知识能否从代码推出、由谁补，并说明只涉及事实的票（修 bug、照已有的样子加字段）没有这些文档也能做。中英文接入清单在 doctor 和门槛基线之间加「补文档（建议）」一步：看 doctor 的「项目知识」检查（老项目先跑 `sandpipe onboard --questions-only`），运行 `/fill-project-docs`，把术语表和 ADR 推到 main；原来的第 2–5 步顺延为第 3–6 步（closes #8）。
- 门槛长时间运行时有进度了：每次门槛开始打印 `📄 #<票号> 门槛输出实时写进 <路径>`，输出逐行实时写进 `.sandcastle/logs/gate-<票号>-<时间戳>.log`（同一张票重跑另开新文件），运行中每 5 分钟打印一次心跳 `⏳ #<票号> 门槛已跑 X 分钟，输出见 <路径>`（日志写不进时警告一次，门槛照跑）。baseline、run（含修复后重跑、rebase 补救）和 main 兜底都生效。档位（`GATE_TIER`）、`GATE_FLAKY`、`GATE_TIME` 仍按完整输出判断（closes #10）。
- config 新增可选字段 `isolation.hostEnvFiles`：声明 config 从宿主机读的 env 文件（比如 `.env`、`.env.test`）。`sandpipe doctor` 新增「宿主机 env 文件」检查，其中有文件在项目根下不存在时警告并提示从 `.env.example` 复制或向同事要，都在时通过；没声明时扫 config 源码里引号包着的 `.env` / `.env.*` 路径（不算 `.sandcastle/` 下的和 `*.example` / `*.sample`），已有的 config 不改也能报出缺失；什么都没找到时不出现这一项（closes #11）。
- `sandpipe onboard` 遇到 `.claude/` 下已存在、内容和模板不同的本工具文件（hook、子代理、skill 及附属文件）不再静默跳过：显示 `git diff --no-index --stat`，让人选 `[k]` 保留 / `[o]` 覆盖 / `[n]` 另存为 `<文件>.new` / `[d]` 看完整 diff；内容相同的显示「无需改动」不问；非交互时一律另存 `.new` 并警告，提示合并后删掉。`sandpipe update` 对这些文件按同一规则处理，不再静默覆盖；全局 skill 和 `.sandcastle/tsconfig.json`、`.sandcastle/.gitignore` 照旧直接刷新（closes #12）。
- `sandpipe` 导出 isolation 辅助函数，`sandcastle.config.ts` 不用再手写样板：`readHostEnv(files, projectRoot?)` 按顺序读宿主机 env 文件并合并（缺文件跳过），`toContainerHost(value)` 把 `localhost` / `127.0.0.1` 换成 `host.containers.internal`，`withDbSuffix(url, suffix)` 给库名 / 前缀加后缀（保留 query 和结尾的 `/`，非 URL 直接拼接，URL 没有库名时抛错），`runHostScript(command, args, label)` 在宿主机上跑回收脚本（失败只警告一行、返回 `undefined`）；另导出常量 `CONTAINER_HOST`。duckdb_crm 的 `TEST_MONGO_URL` / `TEST_MYSQL_URL` 如果没写库名，原来会变成 `/_i<票号>`，现在 `isolation.env` 直接报错（报错里遮掉密码）。`examples/duckdb_crm` 的配置改用它们，`docs/config`（中英）新增「隔离辅助函数」一节，开头的示例改用它们。
- `sandpipe onboard` 生成能直接跑的 `isolation.env` 初稿：扫项目根的 `.env.example` / `.env.sample` / `.env.template` / `.env`，凡是值指向本机（`localhost` / `127.0.0.1`）的变量（几个库、什么中间件都算）和 `*_DB_NAME` 这类库名变量各生成一行；每次运行用 `readHostEnv(".env")` 读项目根的 `.env`，把主机换成 `host.containers.internal`，数据库类（URL scheme 是 postgres / mysql / mongodb 等，或单独的库名变量）各自加 `_i<票号>`，`.env` 里没有的键不传，其他变量（API key 等）不碰。env 文件里没扫到时按 compose 服务猜键名（postgres / mysql → `DATABASE_URL`、mongodb → `MONGODB_URI`、redis → `REDIS_URL`、s3 → `S3_ENDPOINT`、elasticsearch → `ELASTICSEARCH_URL`、rabbitmq → `RABBITMQ_URL`、kafka → `KAFKA_BROKERS`、clickhouse → `CLICKHOUSE_URL`，键名撞了改用 `<服务>_URL`）。cleanup 给一段用 `runHostScript` 的注释示例。两样都没有时生成的配置不变。onboard 的 agent 会核对这些变量是否齐、键名对不对，不一致时在建议里写出改法；中英文接入清单补了一句（closes #23）。
- 提交历史太少时提交信息的语言不再默认英文：detect 新增 `commitLanguageBasis`，取到的提交标题少于 5 条且项目文档文字足够（中文字符加英文字母不少于 200）时按文档的主要语言定（`docs`），文档文字不够时仍按那几条提交（`few-commits`），≥5 条照旧（`commits`）。onboard 给了 `--lang` 且不是按足够多的提交判断时，提交语言跟随 `--lang`（`lang`）。不是按提交判断时，PROJECT.md 草稿「提交」一节注明提交历史只有 N 条、语言按什么推断；onboard 提示词写明推断结果直接沿用、不列进缺口清单，清单示例的「提交信息用中文还是英文」换成「分支命名有没有约定」（closes #14）。

## 0.1.4

- npm 页面上 README 的相对链接（中文版、配置说明、接入清单）不再 404：`package.json` 的 `repository` 指向公开的 releases 仓库，发版时把 README 和文档同步过去。
- 发版 workflow 的 `actions/checkout`、`actions/setup-node` 升到 v7（不再跑在已弃用的 Node 20 上）。

## 0.1.3

- 用户手册中英两份：`README.md`、`docs/config.md`、`docs/onboarding-checklist.md` 是英文，同名的 `.zh-CN.md` 是中文。内部文档移到 `docs/internal/`。
- onboard 生成的文件（`sandcastle.config.ts`、`.gitignore`、`.env.example`、`Containerfile`）注释都是中英两行。
- 终端输出（流水线日志、doctor、onboard、`--help`、报错）按项目配置的 `language` 显示中文或英文；没有项目配置时看 `LANG`，`SANDPIPE_LANG=zh|en` 可以强制指定。
- 代码注释中英对照（声明文件里的说明也是两种语言），`npm test` 会检查。
- `sandpipe onboard --lang zh|en`：PROJECT.md 草稿（章节标题、待补标记）和配置里的 `language` 按它写，没给时按提交信息判断。

## 0.1.2

- 新增配置 `language: "zh" | "en"`（默认 `zh`）：issue、PR、评论用什么语言写。onboard 按提交历史判断并写进配置。
- `sandpipe --help` 打印用法。
- prep 提示词说明宿主机工作目录里有未提交改动和被忽略的文件，指引以基线分支为准；进度评论第一行写 `模型档案：<档案名>`。
- 带类型声明（`dist/types/`，package.json 的 `types`）。onboard 和 update 生成 `.sandcastle/tsconfig.json`，用 paths 把 `sandpipe` 指到本机装的这份工具，不是 Node 的项目也不用装任何依赖，编辑器里有类型提示，也没有告警。这个文件含本机路径，所以加进了 `.sandcastle/.gitignore`（老项目跑 `sandpipe update` 补上）；doctor 会检查它是不是指向当前版本。
- PROJECT.md 的章节标题中英文都认（`## Overview`、`## Commands`、`## Testing`、`## Commits`……），同一节只能写一次。
- 门槛用时不到一分钟时按秒显示。

## 0.1.1

- 改名为 `sandpipe`，发布到 npm：`npm i -g sandpipe` 或 `pnpm add -g sandpipe`。`upgrade` 按安装方式用 npm 或 pnpm 升级。
- `doctor` 新增两项检查：`.claude/`、`.gitignore` 是否已推送到主分支；`GH_TOKEN` 能否读写本仓库的 issue、读 PR。
- Python 项目：`[tool.mypy]` 配了 `files` 时，类型检查命令不再加 ` .`。

## 0.1.0

第一个发布版本。

- 命令：`onboard`（接入新项目）、`doctor`（跑前检查，`--fix` 建标签）、`baseline`（main 上跑一次完整门槛）、`run`（跑流水线）、`update`（刷新项目 `.claude/` 下的文件）、`upgrade`（装最新版）。
- 平台：GitHub（`gh` CLI）。GitLab 还没做。
- 新项目默认 `autoMerge: false`、`maxParallel: 1`，接入步骤见 `docs/onboarding-checklist.md`。
