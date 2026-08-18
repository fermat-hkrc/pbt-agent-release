# Changelog / 更新日志

Every release is recorded here in English and Simplified Chinese. GitHub Release
notes are generated from the matching version entry; the two copies must not be
maintained independently.

每个版本均在此以英文和简体中文记录。GitHub Release notes 从对应版本条目自动生成，
不再单独维护另一份文案。

Entries before v0.1.7 predate this file and remain available in the Release
history. / v0.1.7 之前的版本早于本文件，仍可在 Release 历史中查看。

## 0.1.8 - 2026-08-18

### English

#### Added

- Added `pi-pbt mcp`, a stdio MCP server built on the official
  `@modelcontextprotocol/sdk`, so coding agents such as Claude Code and Codex
  can delegate campaigns while they keep developing. Six tools — `pbt_start`,
  `pbt_status`, `pbt_report`, `pbt_cancel`, `pbt_watch_start`,
  `pbt_watch_stop` — with campaign artifacts readable as
  `pbt://runs/<id>/...` MCP resources.
- Every delegated run snapshots one immutable commit into a detached git
  worktree with persistent state under `~/.pi-pbt/runs/` (`PI_PBT_RUNS_DIR`),
  a bounded queue (`PBT_MCP_MAX_CONCURRENT`), cancellation, and a captured
  `changes.patch`; the caller's checkout is never touched. Watches supersede:
  a newer commit cancels a pre-Review run, while a run already in Review
  finishes first and pending commits coalesce to the newest.
- Added `pbt_start`'s `include_uncommitted` option: the current uncommitted
  working-tree state is frozen into an immutable snapshot commit through a
  throwaway git index — the caller's index, HEAD, refs, stash, and files stay
  untouched — so code can be reviewed before any commit exists.
- Added this bilingual `CHANGELOG.md` as the single source of release notes:
  GitHub Release notes are generated from the tagged entry, and CI enforces
  that the version and both language sections stay in lockstep.

#### Changed

- Compacted the bundled PBT skills: `pbt-patterns` and `pbt-oracles` are now
  short executable protocols whose framework/generator/oracle/C++/research
  material loads from on-demand reference files, and `pbt-in-practice` became
  one of those references instead of a separately advertised skill.
- Historical `pbt-native/` and `pbt-out/` directories no longer select the
  harness: campaigns must probe the project-owned test framework first, record
  the probe in `PLAN.md`, and may write new fallback harness files only after
  that probe fails.
- Build-failure recovery is now scoped by project type: only campaigns
  detected as OpenHarmony are pointed at `openharmony-build-run` /
  `oh-closure.py`; ordinary repositories are steered to repair their own
  build, once per campaign.

#### Fixed

- The hook-run gate and the MCP run manager now judge campaigns by their
  artifacts and accept both `<out>/` and the SOP-literal `<out>/pbt-out/`
  layouts; a successful campaign could previously be reported as dead, and a
  child that died before Review could masquerade as `bugs_found`.
- Registered the bundled OAuth flows in the Bun single binary, fixing
  `/login` to xAI and the other OAuth providers
  (`Cannot find module './xai.js'`).
- Release downloads of the pinned `fd`/`rg` tools now retry transient network
  failures, and the release mirror no longer depends on a `gh` flag newer
  than the runners ship (`--clobber`), which had left a public release empty.

#### Embedded SDK

- Embedded pi `0.84.2`. `pi-pbt --version` reports
  `pi-pbt 0.1.8 (pi 0.84.2)`.

### 中文

#### 新增

- 新增 `pi-pbt mcp`:基于官方 `@modelcontextprotocol/sdk` 的 stdio MCP
  server,让 Claude Code、Codex 等 coding agent 把测试委派给 pi-pbt、同时继续
  开发。共六个工具 —— `pbt_start`、`pbt_status`、`pbt_report`、`pbt_cancel`、
  `pbt_watch_start`、`pbt_watch_stop`;campaign 产物可通过
  `pbt://runs/<id>/...` MCP resources 读取。
- 每次委派运行都把一个不可变 commit 快照进独立的 detached git worktree,状态
  持久化在 `~/.pi-pbt/runs/`(`PI_PBT_RUNS_DIR`),带串行队列
  (`PBT_MCP_MAX_CONCURRENT`)、取消能力和 `changes.patch` 捕获;调用方的
  工作树完全不被触碰。watch 支持 supersede:新 commit 会取消尚未进入 Review
  的运行;已在 Review 的运行先跑完,积压的 commit 合并为最新一个。
- `pbt_start` 新增 `include_uncommitted`:用一次性 git index 把当前未提交的
  工作树状态冻结成不可变快照 commit —— 调用方的 index、HEAD、refs、stash 和
  文件全部不动 —— 代码不用先 commit 也能提前审核。
- 新增本双语 `CHANGELOG.md` 作为 release notes 的唯一来源:GitHub Release
  notes 由对应版本条目自动生成,CI 强制版本号与中英两节保持同步。

#### 变更

- 精简内置 PBT skill:`pbt-patterns` 与 `pbt-oracles` 改为简短的可执行协议,
  framework/generator/oracle/C++/研究资料拆分为按需加载的 reference 文件;
  `pbt-in-practice` 不再单独注册为 skill,降为其中一份 reference。
- 历史遗留的 `pbt-native/`、`pbt-out/` 目录不再决定 harness 位置:campaign
  必须先实测项目自有测试框架并把探测结果记入 `PLAN.md`,探测失败后才允许
  新建 fallback harness 文件。
- 构建失败的引导按项目类型区分:只有确认为 OpenHarmony 的 campaign 才会被
  指向 `openharmony-build-run` / `oh-closure.py`;普通仓库被引导修复自己的
  构建,且每个 campaign 只提示一次。

#### 修复

- hook-run gate 与 MCP run manager 改为按产物判定结果,并同时兼容 `<out>/`
  与 SOP 字面的 `<out>/pbt-out/` 两种布局;此前成功的 campaign 可能被误判为
  "无 REPORT.md",Review 前死亡的子进程可能伪装成 `bugs_found`。
- 在 Bun 单文件二进制中注册内置 OAuth 流程,修复 `/login` 到 xAI 等 OAuth
  服务商时的 `Cannot find module './xai.js'`。
- release 打包下载固定版本的 `fd`/`rg` 时会对瞬时网络故障重试;镜像脚本不再
  依赖 runner 上不存在的 `gh --clobber` 参数(该问题曾导致公开 release 空无
  一物)。

#### 内嵌 SDK

- 内嵌 pi `0.84.2`。`pi-pbt --version` 输出
  `pi-pbt 0.1.8 (pi 0.84.2)`。

## 0.1.7 - 2026-08-12

### English

#### Security

- Disabled Bun's automatic loading of a scanned repository's `bunfig.toml` in
  every compiled target. A repository-controlled `preload` could previously run
  before pi-pbt's entry point and tool guards. The binary smoke test now verifies
  this against a hostile fixture.

#### Added

- Added `quick`, `standard`, and `thorough` campaign effort tiers, covering time
  budgets, property counts, generated-input counts, deepening rounds, and oracle
  requirements. Select them with `--effort` or `PBT_EFFORT`.
- Added `pi-pbt kea` for config-driven HarmonyOS application testing with Kea2
  and a connected device.
- Added pinned `fd` and `rg` binaries to every release archive, so pi-pbt can run
  on systems where neither search tool is installed.

#### Changed

- Campaigns must use the repository's existing test harness and a real
  language-specific PBT framework instead of creating a parallel harness or a
  hand-written random loop.
- Release artifacts are three DEFLATE-compressed zip archives: Linux x64, Linux
  arm64, and macOS arm64. Each extracts to `pi-pbt-<platform>/` with the binary,
  bundled search tools, notices, and executable permissions intact.

#### Fixed

- Asset extraction now honors the run's `$HOME` under Bun instead of polluting
  the operator's real home directory.
- Binary smoke testing now refuses to validate a stale `dist/pi-pbt`.
- `watch` now forwards coverage mode to campaigns, and dashboard startup errors
  identify the failed port or installation step.
- Release packaging copies staged tools across filesystems and keeps package
  output separate from cross-compiled binaries, avoiding the `EXDEV` and path
  collision failures encountered while cutting this release.

#### Embedded SDK

- Embedded pi `0.84.1`. `pi-pbt --version` reports
  `pi-pbt 0.1.7 (pi 0.84.1)`.

### 中文

#### 安全修复

- 所有编译目标均已禁用 Bun 自动加载被扫描仓库中的 `bunfig.toml`。此前仓库可通过
  `preload` 在 pi-pbt 入口和工具守卫启动前执行代码。binary smoke test 现已使用恶意
  fixture 对该场景做回归验证。

#### 新增

- 新增 `quick`、`standard`、`thorough` 三档 campaign effort，分别约束时间预算、
  性质数量、生成输入数量、深化轮次和 oracle 要求。可通过 `--effort` 或
  `PBT_EFFORT` 选择。
- 新增 `pi-pbt kea`，通过配置文件、Kea2 和已连接设备对 HarmonyOS 应用执行测试。
- 每个平台的 release 压缩包均内置经过校验并固定版本的 `fd` 和 `rg`，系统未安装这
  两个搜索工具时也可运行 pi-pbt。

#### 变更

- campaign 必须使用仓库已有的测试框架和对应语言的真实 PBT framework，不再另建
  平行 harness，也不得手写随机循环冒充性质测试。
- release 产物改为三个使用 DEFLATE 压缩的 zip：Linux x64、Linux arm64 和 macOS
  arm64。每个压缩包解出 `pi-pbt-<platform>/`，其中包含主程序、搜索工具、notice，
  并保留可执行权限。

#### 修复

- Bun 下的资源解压现会遵循本次运行的 `$HOME`，不再污染操作者真实的 home 目录。
- binary smoke test 不再误验旧的 `dist/pi-pbt`。
- `watch` 会把 coverage mode 传给 campaign；dashboard 启动失败时会指出具体端口或
  安装步骤。
- release 打包改为跨文件系统复制暂存工具，并将打包输出与交叉编译二进制分离，避免
  本次发版过程中遇到的 `EXDEV` 和路径冲突失败。

#### 内嵌 SDK

- 内嵌 pi `0.84.1`。`pi-pbt --version` 输出
  `pi-pbt 0.1.7 (pi 0.84.1)`。
