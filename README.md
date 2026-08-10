# pi-pbt — releases

[中文](#中文) · [Installation guide](docs/installation.md) · [安装指南](docs/installation.zh.md)

pi-pbt is an AI agent that runs **property-based testing** campaigns on a
codebase. Point it at a Python, Rust, Go, Java, or C++ repository and it reads
the source, works out what must always be true of it, writes and runs tests that
check those rules against large numbers of generated inputs, and reports the bugs
it finds with a minimal reproducer for each.

**This repository distributes the released binaries.** The source lives in a
separate repository; here you get the executables and the installation guide.

## Download

One self-contained executable — no Node.js, no Bun, nothing to install
alongside it.

| Platform | Download | Checksum |
|---|---|---|
| Linux x64 | [`pi-pbt-linux-x64.zip`](https://github.com/fermat-hkrc/pbt-agent-release/releases/latest/download/pi-pbt-linux-x64.zip) | [`.sha256`](https://github.com/fermat-hkrc/pbt-agent-release/releases/latest/download/pi-pbt-linux-x64.zip.sha256) |
| Linux arm64 (aarch64) | [`pi-pbt-linux-arm64.zip`](https://github.com/fermat-hkrc/pbt-agent-release/releases/latest/download/pi-pbt-linux-arm64.zip) | [`.sha256`](https://github.com/fermat-hkrc/pbt-agent-release/releases/latest/download/pi-pbt-linux-arm64.zip.sha256) |
| macOS Apple Silicon | [`pi-pbt-macos-arm64.zip`](https://github.com/fermat-hkrc/pbt-agent-release/releases/latest/download/pi-pbt-macos-arm64.zip) | [`.sha256`](https://github.com/fermat-hkrc/pbt-agent-release/releases/latest/download/pi-pbt-macos-arm64.zip.sha256) |

Not sure which Linux build you need? `uname -m` — `x86_64` takes the x64 file,
`aarch64` the arm64 one. Older versions are on the
[releases page](https://github.com/fermat-hkrc/pbt-agent-release/releases).

```bash
sha256sum -c pi-pbt-linux-x64.zip.sha256    # optional integrity check
unzip pi-pbt-linux-x64.zip                  # yields ./pi-pbt, already executable
sudo install -Dm755 pi-pbt /usr/local/bin/pi-pbt
pi-pbt --help
```

Then configure a model and start a run — see the
**[installation guide](docs/installation.md)**, which also covers the toolchains
the language under test needs, CI and git-hook integration, and the live
dashboard.

Found a problem? Open an [issue](https://github.com/fermat-hkrc/pbt-agent-release/issues).

---

## 中文

pi-pbt 是一个做**性质测试**(property-based testing)的 AI agent。把它指向一个
Python / Rust / Go / Java / C++ 仓库,它会读源码,推断出「这段代码无论输入什么都
必须成立的规律」,据此写测试并用大量自动生成的输入去跑,最后把找到的 bug 连同
最小复现一起报出来。

**本仓库只发布构建产物**:源码在另一个仓库,这里提供可执行文件和安装文档。

下载对应平台的压缩包(见上表),然后:

```bash
sha256sum -c pi-pbt-linux-x64.zip.sha256    # 可选:校验完整性
unzip pi-pbt-linux-x64.zip                  # 解出 ./pi-pbt,已带可执行权限
sudo install -Dm755 pi-pbt /usr/local/bin/pi-pbt
pi-pbt --help
```

只有一个自包含的可执行文件,不需要 Node.js、Bun 或任何依赖。不确定该下哪个
Linux 版本就看 `uname -m`:`x86_64` 用 x64,`aarch64` 用 arm64。

接下来配置模型、跑第一次测试,见 **[安装指南](docs/installation.zh.md)** —— 里面
还写了被测语言需要的工具链、CI 与 git hook 接入方式,以及实时看板。

有问题请提 [issue](https://github.com/fermat-hkrc/pbt-agent-release/issues)。
