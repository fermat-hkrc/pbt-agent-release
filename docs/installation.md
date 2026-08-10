# Installing PBT Agent

[中文版 / Chinese version](installation.zh.md)

pi-pbt is an AI agent: point it at a repository and it reads the code, works out
**what should always be true of it**, writes tests that check those rules against
large numbers of generated inputs, and reports the bugs it finds with a minimal
reproducer for each. That style of testing is called **property-based testing**
(PBT).

Installing it means downloading **one executable file**: no Node.js, no Bun, no
dependencies to install, and nothing to put next to it.

## 1. System requirements

pi-pbt has no dependencies of its own. But it **really compiles and runs the
test code it writes**, so the toolchain for the language under test must be
present:

| You test | Install first |
|---|---|
| Any repo | `git` (it reads the commit history and diffs) |
| C / C++ | `cmake` ≥ 3.16, a C++17 compiler (`clang`/`g++`), `make` or `ninja`. The test libraries (GoogleTest/RapidCheck) are downloaded by CMake, so the first build needs network access. |
| Python | `python3` ≥ 3.9 with `pip` (`hypothesis` is installed into the environment it finds) |
| Rust | `cargo` (`proptest` is added as a dev-dependency) |
| Go | `go` toolchain |
| Java | JDK + Maven/Gradle (jqwik) |
| OpenHarmony components (cross-compiled to arm) | optional `qemu-user` (`qemu-arm`), to run arm binaries on an x86 machine |

Debian/Ubuntu example for C++ targets:

```bash
sudo apt-get install -y git cmake ninja-build clang
```

Arch Linux:

```bash
sudo pacman -S --needed git cmake ninja clang
```

## 2. Download and install

Obtain the archive for your platform from wherever you received pi-pbt (a
Releases page, an internal mirror, or a direct handoff). Each ships with a
`.sha256` checksum alongside it. The published files are:

| Platform | File | Download |
|---|---|---|
| Linux x64 | `pi-pbt-linux-x64.zip` | ~37 MiB |
| Linux arm64 (aarch64) | `pi-pbt-linux-arm64.zip` | ~37 MiB |
| macOS Apple Silicon | `pi-pbt-macos-arm64.zip` | ~26 MiB |

Not sure which Linux one you need? `uname -m` — `x86_64` takes the x64 file,
`aarch64` the arm64 one.

Every archive extracts to a single executable named `pi-pbt` (about 100 MiB —
most of that is the embedded Bun runtime, which is what makes the binary
self-contained). `unzip` restores the executable bit, so there is no `chmod`
step on Linux/macOS:

```bash
sha256sum -c pi-pbt-<platform>.zip.sha256   # optional integrity check
unzip pi-pbt-<platform>.zip                 # yields ./pi-pbt
sudo install -Dm755 pi-pbt /usr/local/bin/pi-pbt
```

(Extracting on Windows drops Unix permissions — as it does for any archive — so
if the file travelled through a Windows machine, `chmod +x pi-pbt` first.)

It does not matter where the real file lives; a symlink onto your `PATH` works
too.

The arm64 and macOS builds are cross-compiled on an x64 Linux machine; they are
validated by format (the release job asserts each is really an aarch64 ELF /
Mach-O arm64), not executed on the target. Command-line use is unaffected; if
the interactive interface misbehaves, build from source on that machine
(`npm run build:binary`).

macOS additionally needs the Gatekeeper quarantine cleared:

```bash
xattr -d com.apple.quarantine /usr/local/bin/pi-pbt
```

Verify:

```bash
pi-pbt --help          # prints usage
pi-pbt --list-models   # lists available models once a provider is configured
```

On first run pi-pbt creates its own config directory `~/.pi-pbt/agent/` (login
credentials, model config, history) — deliberately separate from an upstream
`pi` install's `~/.pi/agent/`, so the two do not interfere.

## 3. Configure a model

### First, the part that matters most: use the strongest model you have

This matters more than any other setting. The hardest step of the whole process
is **coming up with the properties** — working out, from the code, what must
hold no matter what input it gets. There is no lookup answer for that; it is
pure reasoning:

- A **strong model** produces properties with real teeth ("encoding then decoding
  must return the original value", "no interleaving of calls can drive the balance
  negative"). Only properties like that can catch real bugs.
- A **weak model** produces empty ones ("calling it does not crash"). The tests
  pass, the report looks fine, and nothing was found — and it is hard to tell
  from the output that nothing was really tested.

Deciding what counts as the correct answer, and working out whether a failure is
a bug in the code or in the test, lean on reasoning just as heavily. So **do not
economize here**: reasoning you save is bugs you miss.

### How to configure it

pi-pbt uses [pi](https://github.com/earendil-works/pi) as its engine, so models
and providers are configured exactly as in pi — with one difference: **the
config directory is `~/.pi-pbt/agent/` instead of `~/.pi/agent/`**. Wherever
pi's docs say `~/.pi/agent/<file>`, read `~/.pi-pbt/agent/<file>`.

The two quickest paths:

**API key via environment variable** — works immediately:

```bash
export ANTHROPIC_API_KEY=sk-ant-...   # or OPENAI_API_KEY, GEMINI_API_KEY, ...
pi-pbt --list-models
```

**Interactive login** — start `pi-pbt`, then use `/login` to sign in (including
subscription accounts) and `/model` (Ctrl+L) to pick a model.

For everything else, follow pi's documentation directly:

- [Providers & authentication](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/providers.md) — built-in providers, API keys, subscription login
- [Models & `models.json`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/models.md) — adding custom models or providers that speak an OpenAI/Anthropic/Google-compatible API (file goes in `~/.pi-pbt/agent/models.json`)
- [Custom providers](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/custom-provider.md) — custom APIs or OAuth

Example `~/.pi-pbt/agent/models.json` for a self-hosted OpenAI-compatible proxy:

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

## 4. Start testing

### Interactive

`cd` into the repository under test and start it:

```bash
cd /path/to/your/repo
pi-pbt
```

Then just ask:

```text
Run property-based testing on this repository: identify the best targets, write
properties, run them, and produce bug reports.
```

It works through four steps — scan the code for targets, decide what to test,
write and run the tests, review the results — and writes what it produces to
`pbt-out/`: the plan (`PLAN.md`), the list of properties (`PROPERTIES.md`), a
summary (`REPORT.md`), and one `bug_reports/*.md` per confirmed bug, each with a
minimal reproducer.

### From the command line (CI, scripts)

```bash
pi-pbt -p "Run property-based testing on this repository; write results to pbt-out/."
```

`-p` prints the run and exits; it never waits for input, so it is safe under CI
runners, git hooks, and `nohup`.

You do not pick a mode per project type: there is one entry point, and it works
the rest out itself. When the target does not build with a plain
`cmake`/`cargo`/`pytest` — a component of a large OS or of a monorepo — it stands
that component up first; OpenHarmony C++ components follow a dedicated build-and-
run path (real compiled artifacts, run under qemu-arm). It decides from the
repository's own contents (e.g. an `@ohos/` `bundle.json`), automatically.

To pin the entry point explicitly in a script, lead the first message with
`/skill:pbt-workflow`:

```bash
pi-pbt -p "/skill:pbt-workflow 对当前仓库做性质测试(PBT),目标是找出 bug。产物写到 pbt-out/。"
```

### CI / git-hook integration

To test **one specific commit**, use this subcommand:

```bash
pi-pbt hook-run <sha> --repo /path/to/repo --lang zh
```

It reports its verdict as an exit code, so it drops straight into CI as a check:
`1` when bugs were found (`bug_reports/` non-empty), `2` when no `REPORT.md` was
produced (it died or timed out), `0` on a clean pass.

### Watching a repo: test every new commit

Two ways, depending on whether you want to *watch* it work.

**Interactive — in front of you (recommended whenever you have pi-pbt open).**
In an interactive `pi-pbt` session, say:

```
/skill:pbt-watch
```

It watches the current repo (or one you name) in the background and, the moment
a new commit lands, runs the whole thing **right there in front of you** —
finding targets, planning, writing and running tests, reviewing — then keeps
watching for the next commit. Watching does not tie up the session: the command
returns immediately and you can keep asking for other things meanwhile; after
each round it carries on by itself, with no command to type to resume. Nothing
is pushed off somewhere you cannot see it, and you can just tell it to stop. For
an OpenHarmony module, a pre-built source environment is detected and reused
automatically. (This also shows up live in the dashboard, below.)

**Unattended — on a machine nobody is watching.** On a CI mirror, a server with
no pi-pbt open, or where a git hook cannot be installed, run it resident in the
background instead. It checks for new commits on a timer and tests each one it
finds, with the results going to its log:

```bash
# watch local commits on the current repo, every 30s (put it in tmux/systemd)
pi-pbt watch --repo /path/to/repo --lang zh --provider xai-oauth --model grok-4.5

# watch pushes to the remote tracking branch instead
pi-pbt watch --repo /path/to/repo --fetch --branch master --interval 60 --lang zh --provider xai-oauth --model grok-4.5
```

The starting point is the latest commit at startup (commits that already existed
are not tested retroactively); every new commit is then tested, with the verdict
(the exit codes above) recorded in the log. All `hook-run` flags (`--out`,
`--workdir`, `--spec`, `--lang`, `--scan-root`, `--provider`, `--model`,
`--tui`) pass through. `--tui` (or `PBT_HOOK_TUI=1`) runs it in the full
interactive interface in the same terminal (good for demos, not for CI: it stops
and waits for `/quit` after each round).

### Environment variables

| Variable | Effect |
|---|---|
| `PBT_LANG=zh` | work in Chinese and write everything it produces in Chinese; also `--lang zh` on subcommands |
| `PBT_SCAN_ROOT=/path` | the repo to scan, for setups where the working directory is a clean copy (git hooks, CI) |
| `PBT_OH_WORKSPACE=/path` | a pre-built full OpenHarmony source environment (source tree + toolchain + built dependencies) to reuse, instead of working out how to build the component standalone. **Detected automatically** when the repo sits inside such an environment (a parent directory with both `.repo/` and `out/`) — set it explicitly only to override; the startup log prints which one is in use |
| `PBT_HOOK_TUI=1` | same as `hook-run --tui` / `watch --tui`: run in the interactive interface (needs a terminal; waits for `/quit`) |

## 5. Dashboard: watch it work, live

pi-pbt can serve a web dashboard showing every run on the machine as it happens
— whether you started it by hand, a git hook did, or a repo watch did — with the
conversation, token usage and produced files, plus the history of past runs. No
extra npm install is required; pi-pbt installs and runs it itself:

```bash
pi-pbt dashboard install       # one-time: pins the dashboard package under
                               # ~/.pi-pbt/dashboard and wires the bridge
                               # extension into ~/.pi-pbt/agent/extensions/
pi-pbt dashboard --port 8000   # run resident (put it in tmux/systemd)
```

For example:

```bash
tmux new-session -d -s dash 'pi-pbt dashboard --port 8000 >> ~/pbt-dash.log 2>&1'
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8000/   # expect 200 (takes ~20s)
```

Then open **http://localhost:8000** in a browser. On a remote host, tunnel the
port first: `ssh -N -f -L 8000:localhost:8000 <host>`.

How it works: the install step wires things up once, so **every pi-pbt run on
the machine shows up automatically** — one launched by a git hook, by CI, or by
a repo watch appears the moment it starts (named `pbt_hook_<sha>`), with no
per-run configuration. History is read from `~/.pi-pbt/agent/sessions/` (not
`~/.pi/agent/sessions/`) — pi-pbt keeps its own config and history separate from
a plain `pi` install.

Notes:
- The sidebar groups runs by **pinned folders** — pin the repo directories you
  care about (folder menu in the UI) or their runs are only reachable through
  the full list.
- Verify it is alive with `curl localhost:8000` (some hosts don't show the
  ports in `ss`/`netstat`).
- After updating pi-pbt, run `pi-pbt dashboard install` again to refresh the
  separately installed, exact-pinned dashboard package.
- Only one dashboard can run per user: it occupies the chosen HTTP port and
  the bridge gateway port **9999**. `--port` changes only the HTTP port. Stop
  the existing instance before restarting; `pi-pbt dashboard` reports either
  occupied port before invoking the upstream server.

## 6. Troubleshooting

- **It improvises instead of following its built-in workflow** — the built-in
  workflow files did not load (the startup log shows `skills=0`). Delete the
  cache directory `~/.pi-pbt/cache/` and run again to let it rebuild; if it is
  still 0, the home directory is most likely not writable or the disk is full.
- **`command not found: cmake` partway through** — install the toolchain for
  your target language (§1) and re-run. It will not skip the build and pretend
  the code was tested.
- **It hangs in CI** — it should not: it never waits for input and force-exits
  when done. If you see a hang, file an issue with the log.
- **macOS blocks the binary** — run the `xattr -d com.apple.quarantine` step
  from §2.
