<div align="center">

<h1>
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://media.x.ai/v1/website/spacexai-symbol-white-transparent-0c31957f.png">
    <source media="(prefers-color-scheme: light)" srcset="https://media.x.ai/v1/website/spacexai-symbol-black-transparent-6435cf42.png">
    <img alt="SpaceXAI logo" src="https://media.x.ai/v1/website/spacexai-symbol-black-transparent-6435cf42.png" width="96">
  </picture>
  <br>
  Grok Build (<code>grok</code>) — Windows Edition
</h1>

**Grok Build** is SpaceXAI's terminal-based AI coding agent. It runs as a
full-screen TUI that understands your codebase, edits files, executes shell
commands, searches the web, and manages long-running tasks — interactively,
headlessly for scripting/CI, or embedded in editors via the Agent Client
Protocol (ACP).

**This fork adds native Windows (x86_64-pc-windows-msvc) build support and
configures the agent to run on [DeepSeek V4](https://platform.deepseek.com)
without requiring an xAI account.**

[Windows Quick Start](#-windows-quick-start) ·
[Installation](#installation) ·
[Configuration](#configuration) ·
[Building from source](#building-from-source-windows) ·
[What changed](#what-changed-in-this-fork) ·
[Original project](#original-project)

![Grok Build TUI](https://media.x.ai/v1/website/universe-tui-screenshot-6f7a0837.png)

</div>

---

## ⚡ Windows Quick Start

> **Prerequisites:** Windows 10 (1903+) · PowerShell 5.1+ ·
> [DeepSeek API key](https://platform.deepseek.com/api_keys)

```powershell
# 1. Download the release package (grok.exe + install.ps1)
#    Then run the installer — it handles everything below automatically:
.\install.ps1

# 2. Set your API key (permanent):
[System.Environment]::SetEnvironmentVariable("DEEPSEEK_API_KEY", "sk-...", "User")

# 3. Launch (open a new terminal after install to pick up PATH):
grok
```

The installer copies `grok.exe` to `~/.grok/bin/`, adds it to your user PATH,
and writes a ready-to-use `~/.grok/config.toml`. **No xAI login, no Visual C++
redistributable, no runtime dependencies.**

---

## Installation

### What's in the distribution package

```
grok.exe      ← self-contained binary (~110 MB, statically linked CRT)
install.ps1   ← PowerShell installer
```

### Automated install (recommended)

Place both files in the same folder and run:

```powershell
.\install.ps1
```

The script will:

1. Copy `grok.exe` → `%USERPROFILE%\.grok\bin\grok.exe`
2. Add `%USERPROFILE%\.grok\bin` to your user `PATH` (permanent, no admin needed)
3. Prompt for your DeepSeek API key and write `~/.grok/config.toml`
4. Confirm the install with `grok --version`

**Installer parameters:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `-GrokExe` | `grok.exe` next to the script | Path to the binary |
| `-ApiKey` | `$env:DEEPSEEK_API_KEY` | DeepSeek API key (prompts if unset) |
| `-Model` | `deepseek-v4-pro` | Default model ID |

```powershell
# Pre-supply the API key to skip the prompt:
$env:DEEPSEEK_API_KEY = "sk-..."
.\install.ps1

# Use Flash instead of Pro as the default:
.\install.ps1 -Model deepseek-v4-flash
```

### Manual install

```powershell
# 1. Copy binary
New-Item -ItemType Directory -Force "$env:USERPROFILE\.grok\bin"
Copy-Item grok.exe "$env:USERPROFILE\.grok\bin\grok.exe"

# 2. Add to PATH (permanent)
$p = [System.Environment]::GetEnvironmentVariable("PATH", "User")
[System.Environment]::SetEnvironmentVariable("PATH", "$p;$env:USERPROFILE\.grok\bin", "User")

# 3. Create ~/.grok/config.toml  (see Configuration section below)
```

### Uninstall

```powershell
Remove-Item "$env:USERPROFILE\.grok\bin\grok.exe"
$p = [System.Environment]::GetEnvironmentVariable("PATH","User") -split ";" |
     Where-Object { $_ -ne "$env:USERPROFILE\.grok\bin" }
[System.Environment]::SetEnvironmentVariable("PATH", ($p -join ";"), "User")
# Optional — removes all sessions and config:
# Remove-Item -Recurse "$env:USERPROFILE\.grok"
```

---

## Configuration

The config file lives at `%USERPROFILE%\.grok\config.toml`.
The installer creates it automatically; you can edit it at any time.

### Full config reference

```toml
# ── Authentication ────────────────────────────────────────────────────────────
# Force API-key auth — no xAI login prompt.
[auth]
preferred_method = "api_key"

# ── Default model ─────────────────────────────────────────────────────────────
[models]
default = "deepseek-v4-pro"   # change to "deepseek-v4-flash" for lower cost

# ── DeepSeek V4 Pro (frontier reasoning / agentic coding) ────────────────────
# 1.6 T parameters (49 B active), 1 M context window.
[model.deepseek-v4-pro]
model          = "deepseek-v4-pro"
base_url       = "https://api.deepseek.com/v1"
name           = "DeepSeek V4 Pro"
api_backend    = "chat_completions"
env_key        = "DEEPSEEK_API_KEY"   # read from environment (recommended)
# api_key      = "sk-..."            # or embed directly (plaintext)
context_window = 900000

# ── DeepSeek V4 Flash (fast / low-cost) ──────────────────────────────────────
# 284 B parameters (13 B active). Reasoning closely approaches Pro.
[model.deepseek-v4-flash]
model          = "deepseek-v4-flash"
base_url       = "https://api.deepseek.com/v1"
name           = "DeepSeek V4 Flash"
api_backend    = "chat_completions"
env_key        = "DEEPSEEK_API_KEY"
context_window = 900000
```

### Choosing between Pro and Flash

| | DeepSeek V4 Pro | DeepSeek V4 Flash |
|-|-----------------|-------------------|
| Best for | Complex coding, reasoning, long-context agents | Fast iteration, high-volume, simple tasks |
| Parameters | 1.6 T / 49 B active | 284 B / 13 B active |
| Context | 1 M tokens | 1 M tokens |
| Input price (cache miss) | $1.74 / M tok | $0.14 / M tok |
| Output price | $3.48 / M tok | $0.28 / M tok |

### Setting the API key

**Recommended — environment variable (key never written to disk):**

```powershell
# Current session only:
$env:DEEPSEEK_API_KEY = "sk-..."

# Permanent (survives reboots, affects all new terminals):
[System.Environment]::SetEnvironmentVariable("DEEPSEEK_API_KEY", "sk-...", "User")
```

**Alternative — embed in config.toml:**

```toml
[model.deepseek-v4-pro]
api_key = "sk-..."   # stored in plaintext; ensure the file is not world-readable
```

---

## Usage

```powershell
grok                          # launch TUI
grok -p "Explain this file"   # one-shot headless prompt
grok models                   # list configured models
grok --version                # print version
grok --help                   # full flag reference
```

**Inside the TUI:**

| Action | Command |
|--------|---------|
| Switch model | `/model deepseek-v4-flash` |
| New session | `/new` |
| Open help | `?` |
| Quit | `Ctrl+C` or `/quit` |

---

## Building from source (Windows)

See **[BUILDING_ON_WINDOWS.md](BUILDING_ON_WINDOWS.md)** for the full guide.
Summary:

1. Install Rust (rustup) with the MSVC toolchain.
2. Download [protoc v29.3 for Windows](https://github.com/protocolbuffers/protobuf/releases/download/v29.3/protoc-29.3-win64.zip) and register it in `~/.cargo/config.toml`:
   ```toml
   [env]
   PROTOC = "C:\\tools\\protoc-29.3\\bin\\protoc.exe"
   ```
3. Build:
   ```powershell
   cargo build -p xai-grok-pager-bin           # debug
   cargo build -p xai-grok-pager-bin --release  # optimised
   cargo build -p xai-grok-pager-bin --profile release-dist  # distribution (thin LTO)
   ```

All necessary linker flags (PDB suppression, 64 MB main-thread stack) are
already in `.cargo/config.toml` — no extra environment variables required.

---

## What changed in this fork

This branch (`windows/build-support`) adds the following on top of the upstream
[xai-org/grok-build](https://github.com/xai-org/grok-build) `main`:

| File | Change |
|------|--------|
| `crates/build/xai-proto-build/src/lib.rs` | Cross-platform protoc dependency scan: replaces `/dev/stdout` with a temp file and `/dev/null` with `NUL` on Windows |
| `.cargo/config.toml` | Windows MSVC flags: `debuginfo=0`, `/DEBUG:NONE` (avoids `LNK1318`), `/STACK:67108864` (avoids `STATUS_STACK_OVERFLOW`) |
| `install.ps1` | PowerShell end-user installer |
| `INSTALL.md` | Distribution and installation guide |
| `BUILDING_ON_WINDOWS.md` | Source-build guide for Windows |
| `.gitignore` | Ignore stray `NUL` file created by protoc on Windows |

---

## Troubleshooting

**`grok` not found after install**
The PATH change takes effect in new terminals. Run `$env:PATH += ";$env:USERPROFILE\.grok\bin"` in the current session, or open a new one.

**Login screen appears instead of prompt**
Verify `~/.grok/config.toml` contains `[auth] preferred_method = "api_key"` and at least one model has `env_key` or `api_key` set.

**401 Unauthorized / connection error**
Run `Write-Output $env:DEEPSEEK_API_KEY` to confirm the key is set. Check your balance at [platform.deepseek.com](https://platform.deepseek.com).

**`RUSTFLAGS` env var breaks the build**
When `RUSTFLAGS` is set in the environment, it *replaces* (not supplements) the flags in `.cargo/config.toml`, silently dropping `/STACK` and `/DEBUG:NONE`. Clear it before building:
```powershell
Remove-Item Env:RUSTFLAGS -ErrorAction SilentlyContinue
```

---

## Known limitations (Windows runtime)

| Feature | Status |
|---------|--------|
| AI conversations, file editing, web search, MCP | ✅ Works |
| Session history, config, model switching | ✅ Works |
| PTY / interactive shell sessions | ⚠️ Unix PTY APIs; limited on Windows |
| Leader–follower IPC | ⚠️ Uses Unix domain sockets internally |
| `/proc` diagnostics | ⚠️ Linux-only; gracefully skipped |

---

## Original project

This fork is based on **[xai-org/grok-build](https://github.com/xai-org/grok-build)**.
Full documentation at [docs.x.ai/build/overview](https://docs.x.ai/build/overview).
User guide: [`crates/codegen/xai-grok-pager/docs/user-guide/`](crates/codegen/xai-grok-pager/docs/user-guide/).

## Repository layout

| Path | Contents |
|------|----------|
| `crates/codegen/xai-grok-pager-bin` | Composition-root; builds the `xai-grok-pager` binary |
| `crates/codegen/xai-grok-pager` | TUI: scrollback, prompt, modals, rendering |
| `crates/codegen/xai-grok-shell` | Agent runtime + leader/stdio/headless entry points |
| `crates/codegen/xai-grok-tools` | Tool implementations (terminal, file edit, search, …) |
| `crates/codegen/xai-grok-workspace` | Host filesystem, VCS, execution, checkpoints |
| `crates/common/`, `crates/build/` | Shared leaf crates |
| `third_party/` | Vendored Mermaid diagram stack |

## License

First-party code is licensed under the **Apache License, Version 2.0** — see [`LICENSE`](LICENSE).

Third-party and vendored code remains under its original licenses:
[`THIRD-PARTY-NOTICES`](THIRD-PARTY-NOTICES) ·
[`crates/codegen/xai-grok-tools/THIRD_PARTY_NOTICES.md`](crates/codegen/xai-grok-tools/THIRD_PARTY_NOTICES.md) ·
[`third_party/NOTICE`](third_party/NOTICE)
