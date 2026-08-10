# 安装 PBT Agent

[English version / 英文版](installation.md)

pi-pbt 是一个 AI agent:你把它指向一个代码仓库,它自己读代码、想出这段代码
**应该始终成立的规律**,写出测试去大量随机地验证这些规律,把发现的 bug 连同
最小复现一起写成报告。这种测法叫**性质测试**(property-based testing,PBT)。

安装就是下载**一个可执行文件**:不需要装 Node.js、Bun 或任何依赖,也不需要
在它旁边放别的文件。

## 1. 系统要求

pi-pbt 自己没有依赖。但它会**真实编译并运行它写出来的测试代码**,所以被测
语言的工具链必须先装好:

| 被测目标 | 需要预装 |
|---|---|
| 任何仓库 | `git`(要读提交历史和改动) |
| C / C++ | `cmake` ≥ 3.16、C++17 编译器(`clang`/`g++`)、`make` 或 `ninja`。测试库(GoogleTest/RapidCheck)由 CMake 自动下载,首次构建需要网络。 |
| Python | `python3` ≥ 3.9 + `pip`(会在探测到的环境里安装 `hypothesis`) |
| Rust | `cargo`(会加 `proptest` 开发依赖) |
| Go | `go` 工具链 |
| Java | JDK + Maven/Gradle(jqwik) |
| OpenHarmony 组件(交叉编译到 arm) | 可选 `qemu-user`(`qemu-arm`),用于在 x86 机器上运行 arm 程序 |

Debian/Ubuntu(C++ 目标)示例:

```bash
sudo apt-get install -y git cmake ninja-build clang
```

Arch Linux:

```bash
sudo pacman -S --needed git cmake ninja clang
```

## 2. 下载与安装

从你获取 pi-pbt 的渠道(Releases 页面、内部镜像或直接分发)取得对应平台的
压缩包,每个都附带 `.sha256` 校验文件。已发布的文件为:

| 平台 | 文件 | 下载体积 |
|---|---|---|
| Linux x64 | `pi-pbt-linux-x64.zip` | 约 37 MiB |
| Linux arm64(aarch64) | `pi-pbt-linux-arm64.zip` | 约 37 MiB |
| macOS Apple Silicon | `pi-pbt-macos-arm64.zip` | 约 26 MiB |

不确定该拿哪个 Linux 版本?跑 `uname -m` —— 显示 `x86_64` 用 x64 那个,显示
`aarch64` 用 arm64 那个。

每个压缩包解出来都是一个名为 `pi-pbt` 的可执行文件(约 100 MiB,其中绝大部分
是内嵌的 Bun 运行时 —— 正是它让这个二进制自包含)。`unzip` 会恢复可执行位,
所以在 Linux/macOS 上不需要 `chmod`:

```bash
sha256sum -c pi-pbt-<platform>.zip.sha256   # 可选的完整性校验
unzip pi-pbt-<platform>.zip                 # 解出 ./pi-pbt
sudo install -Dm755 pi-pbt /usr/local/bin/pi-pbt
```

(在 Windows 上解压会丢掉 Unix 权限位 —— 任何压缩格式都一样 —— 所以如果文件
中转过 Windows 机器,先 `chmod +x pi-pbt`。)

放哪都行,做个符号链接到 `PATH` 也可以。

arm64 和 macOS 版都是在 x64 Linux 上交叉编译出来的,只做了格式校验(发布流程
会断言产物确实是 aarch64 ELF / Mach-O arm64),没有在目标机器上实跑。命令行
方式不受影响;若交互式界面表现异常,请在那台机器上从源码构建
(`npm run build:binary`)。

macOS 还需清除 Gatekeeper 隔离标记:

```bash
xattr -d com.apple.quarantine /usr/local/bin/pi-pbt
```

验证:

```bash
pi-pbt --help          # 打印用法
pi-pbt --list-models   # 配好模型服务商后列出可用模型
```

首次运行时 pi-pbt 会创建自己的配置目录 `~/.pi-pbt/agent/`(存放登录凭据、
模型配置和历史记录),与 `pi` 自己的 `~/.pi/agent/` 有意分开、互不影响。

## 3. 配置模型

### 先说最重要的:用你手里最强的模型

这一条比其他任何配置都重要。整个流程里最难的一步是**提炼性质** —— 从代码里
想明白"这段代码无论输入什么都应该满足什么"。这一步没有标准答案,全靠推理:

- **强模型**能提炼出真正有约束力的性质(比如"编码再解码必须还原成原值"
  "无论怎么并发调用,余额都不会变成负数"),这类性质才可能揪出真 bug。
- **弱模型**只会写出"调用了不崩溃就算过"这种空性质。测试全绿、报告漂亮,
  但一个 bug 也找不到 —— 而且你很难从结果看出它其实什么都没测。

拿什么当"正确答案"来判断结果对错、以及失败之后定位是代码的错还是测试的错,
同样吃推理。所以**别为了省钱在这里用小模型**:省下的推理会直接变成漏掉的 bug。

### 怎么配

pi-pbt 用 [pi](https://github.com/earendil-works/pi) 作为底层引擎,模型和
服务商的配置方式与 pi 完全一致 —— 唯一区别:**配置目录是 `~/.pi-pbt/agent/`
而不是 `~/.pi/agent/`**。pi 文档里凡是写 `~/.pi/agent/<文件>` 的地方,请读作
`~/.pi-pbt/agent/<文件>`。

最快的两条路:

**用环境变量里的 API key** —— 立即可用:

```bash
export ANTHROPIC_API_KEY=sk-ant-...   # 或 OPENAI_API_KEY、GEMINI_API_KEY 等
pi-pbt --list-models
```

**交互式登录** —— 启动 `pi-pbt` 后,用 `/login` 登录(支持订阅账号),
`/model`(Ctrl+L)选模型。

其余细节直接看 pi 的官方文档:

- [服务商与登录](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/providers.md) —— 内置服务商、API key、订阅账号登录
- [模型与 `models.json`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/models.md) —— 添加自定义模型或服务商(兼容 OpenAI/Anthropic/Google 接口的都行;文件放 `~/.pi-pbt/agent/models.json`)
- [自定义服务商](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/custom-provider.md) —— 自定义接口或 OAuth

自建 OpenAI 兼容中转的 `~/.pi-pbt/agent/models.json` 示例:

```json
{
  "providers": {
    "myproxy": {
      "api": "openai-completions",
      "baseUrl": "https://my-proxy.example.com/v1",
      "apiKey": "sk-...",
      "models": [
        { "id": "claude-opus-4-8", "name": "Claude Opus 4.8", "reasoning": true,
          "contextWindow": 1000000, "maxTokens": 32000 }
      ]
    }
  }
}
```

## 4. 开始测

### 交互式

`cd` 进被测仓库,直接启动:

```bash
cd /path/to/your/repo
pi-pbt
```

然后直接提要求:

```text
对当前仓库做性质测试(PBT):找出最值得测的目标,写性质并运行,产出 bug 报告。
```

它会依次完成四步 —— 扫描代码找目标、定测试计划、写测试并跑、复核结果 ——
把产物写到 `pbt-out/` 目录:测试计划 `PLAN.md`、性质清单 `PROPERTIES.md`、
总结 `REPORT.md`,以及每个确认的 bug 一份 `bug_reports/*.md`(附最小复现)。

### 命令行方式(CI、脚本)

```bash
pi-pbt -p "对当前仓库做性质测试(PBT),产物写到 pbt-out/。"
```

`-p` 跑完即退出,不会卡住等输入 —— 在 CI、git hook、`nohup` 后台下都安全。

你不需要按项目类型挑选什么模式:入口只有一个,后面的事它自己判断。目标无法
用简单的 `cmake`/`cargo`/`pytest` 构建时(比如大型操作系统或 monorepo 里的
一个组件),它会自己把这个组件先立起来;OpenHarmony 的 C++ 组件则走专门的
编译与运行方式(真实编译产物 + qemu-arm 运行)。判断依据是仓库内容本身
(例如 `@ohos/` 的 `bundle.json`),全自动。

若想在脚本里把入口写死,首条消息以 `/skill:pbt-workflow` 开头:

```bash
pi-pbt -p "/skill:pbt-workflow 对当前仓库做性质测试(PBT),目标是找出 bug。产物写到 pbt-out/。"
```

### CI / git hook 集成

针对**单个提交**测一遍,用这个子命令:

```bash
pi-pbt hook-run <sha> --repo /path/to/repo --lang zh
```

它用退出码表示结论,可以直接当 CI 的一道检查:发现 bug(`bug_reports/`
非空)退 `1`;没产出 `REPORT.md`(中途挂了或超时)退 `2`;干净通过退 `0`。

### 盯着仓库:每来一个新提交就测一遍

两种方式,取决于你要不要**亲眼看到**它测。

**交互式 —— 在你眼前跑(推荐,只要你开着 pi-pbt)。** 在交互式 `pi-pbt`
里说:

```
/skill:pbt-watch
```

它在后台盯着当前仓库(或你指定的仓库),一旦有新提交落地,就**在你眼前**
把整套测试跑一遍(找目标、定计划、写测试、跑、复核,全程可见),跑完继续
盯下一个提交。盯着的过程不占用会话:命令立刻返回,期间你照常提别的要求;
一轮测完它自己继续,不需要你敲任何命令恢复。全程不会甩到后台看不见的地方;
不用时让它停下来即可。OpenHarmony 模块预先准备好的源码环境会自动识别并复用。
(这个会话也会同步显示在下面的 dashboard 里。)

**无人值守 —— 没人盯着的机器。** CI 镜像、没开 pi-pbt 的服务器、或装不了
git hook 的场景,改成常驻后台运行:它定时检查新提交,每来一个就测一遍,
结果打在日志里:

```bash
# 监控本地新 commit,每 30s 轮询(放 tmux/systemd 驻留)
pi-pbt watch --repo /path/to/repo --lang zh --provider xai-oauth --model grok-4.5

# 改为监控远端分支的新推送
pi-pbt watch --repo /path/to/repo --fetch --branch master --interval 60 --lang zh --provider xai-oauth --model grok-4.5
```

起点是启动那一刻的最新提交(此前已有的提交不会补测);之后每来一个新提交
测一遍,结论(上面那套退出码)打在日志里。`hook-run` 的全部参数
(`--out`/`--workdir`/`--spec`/`--lang`/`--scan-root`/`--provider`/`--model`/`--tui`)原样透传。`--tui`(或 `PBT_HOOK_TUI=1`)会在同一个终端里用完整的交互式界面跑(适合演示,别用在 CI:每轮结束后它会停下来等你 `/quit`)。

### 环境变量

| 变量 | 作用 |
|---|---|
| `PBT_LANG=zh` | 全程用中文思考、并用中文写所有产物;子命令也可用 `--lang zh` |
| `PBT_SCAN_ROOT=/path` | 要扫描的仓库路径。用于工作目录是一份干净副本的场景(git hook / CI) |
| `PBT_OH_WORKSPACE=/path` | 预先准备好的完整 OpenHarmony 源码环境(源码 + 编译工具链 + 已编译好的依赖),直接复用而不是从头推导怎么单独构建。仓库位于这样的环境内部时(某个上级目录同时有 `.repo/` 和 `out/`)会**自动识别**,只有要覆盖时才需要显式设置;启动日志会打印实际用的是哪个 |
| `PBT_HOOK_TUI=1` | 等同 `hook-run --tui` / `watch --tui`:用交互式界面跑(需要终端;结束后等你 `/quit`) |

## 5. 网页面板:实时看它在干什么

pi-pbt 可以起一个网页面板,实时展示这台机器上正在跑的每一次运行(你手动开的、
git hook 触发的、盯仓库触发的都算)—— 对话过程、token 用量、产物,以及历史
记录。不需要额外装 npm 包,pi-pbt 自己负责安装和运行:

```bash
pi-pbt dashboard install       # 一次性:把 dashboard 包锁版安装到
                               # ~/.pi-pbt/dashboard,并把 bridge 扩展接入
                               # ~/.pi-pbt/agent/extensions/
pi-pbt dashboard --port 8000   # 驻留运行(放 tmux/systemd)
```

例如:

```bash
tmux new-session -d -s dash 'pi-pbt dashboard --port 8000 >> ~/pbt-dash.log 2>&1'
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8000/   # 期望 200(约 20s 启动)
```

然后浏览器打开 **http://localhost:8000**。远程主机先做端口转发:
`ssh -N -f -L 8000:localhost:8000 <host>`。

工作原理:install 那步做了一次性接线,之后**这台机器上每一次 pi-pbt 运行都会
自动出现在面板里** —— git hook、CI、盯仓库拉起的运行一开始就能看到(名字形如
`pbt_hook_<sha>`),不用逐次配置。历史记录读自 `~/.pi-pbt/agent/sessions/`
(不是 `~/.pi/agent/sessions/`)—— pi-pbt 的配置和记录与普通 `pi` 安装互不干扰。

注意事项:
- 侧边栏按**收藏的目录**分组 —— 把关心的仓库目录收藏上(界面里的 folder
  菜单),否则它的记录只能从完整列表里翻。
- 用 `curl localhost:8000` 验证是否活着(有些机器上 `ss`/`netstat` 看不到端口)。
- 更新 pi-pbt 后,再次执行 `pi-pbt dashboard install`,以刷新这个独立安装且精确
  锁版的 dashboard 包。
- 每个用户只能运行一个 dashboard:它会占用网页端口以及 bridge gateway 的
  **9999** 端口。`--port` 只改变网页端口。重启前先停止旧实例;`pi-pbt dashboard`
  会在调用上游服务之前报告任一被占用的端口。

## 6. 常见问题

- **它不按内置流程走,自由发挥** —— 内置流程文件没能就绪(启动日志里
  `skills=0`)。删掉缓存目录 `~/.pi-pbt/cache/` 再跑一次让它重建;若仍为 0,
  多半是家目录不可写或磁盘满。
- **中途报 `command not found: cmake`** —— 按 §1 装好目标语言的工具链后重跑。
  它不会绕过编译假装测过。
- **在 CI 里卡住不动** —— 不该发生:它不会等输入,跑完会强制退出。若确实卡住,
  请带日志提 issue。
- **macOS 拦住不让运行** —— 执行 §2 里的 `xattr -d com.apple.quarantine`。
