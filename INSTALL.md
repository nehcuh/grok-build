# Grok — Windows Distribution & Installation Guide

> This guide is for **end users** who receive a pre-built `grok.exe`.
> If you want to build from source, see [`BUILDING_ON_WINDOWS.md`](BUILDING_ON_WINDOWS.md).

---

## What you need

| Requirement | Notes |
|-------------|-------|
| Windows 10 (1903+) or Windows 11 | x86-64 only |
| PowerShell 5.1+ | Included in all supported Windows versions |
| DeepSeek API key | [platform.deepseek.com/api_keys](https://platform.deepseek.com/api_keys) |
| Internet access | For model API calls at runtime |

**No runtime libraries required.** The binary statically links the MSVC C runtime — no Visual C++ Redistributable needed.

---

## Distribution package

Distribute these two files together:

```
grok.exe        ← the binary (~110 MB, self-contained)
install.ps1     ← optional installer script
```

---

## Installation

### Option A — Automated (recommended)

Place `grok.exe` and `install.ps1` in the same folder, then run:

```powershell
.\install.ps1
```

The script will:
1. Copy `grok.exe` → `%USERPROFILE%\.grok\bin\grok.exe`
2. Add `%USERPROFILE%\.grok\bin` to your user `PATH` (permanent)
3. Prompt for your DeepSeek API key and write `~/.grok/config.toml`
4. Verify the installation by printing the version

**Parameters:**

```powershell
# Use a different source binary
.\install.ps1 -GrokExe "C:\Downloads\grok.exe"

# Pre-set API key via environment variable (no prompt)
$env:DEEPSEEK_API_KEY = "sk-..."
.\install.ps1

# Choose a different default model
.\install.ps1 -Model deepseek-v4-flash
```

---

### Option B — Manual

1. **Copy binary:**
   ```powershell
   New-Item -ItemType Directory -Force "$env:USERPROFILE\.grok\bin"
   Copy-Item grok.exe "$env:USERPROFILE\.grok\bin\grok.exe"
   ```

2. **Add to PATH** (permanent, current user):
   ```powershell
   $p = [System.Environment]::GetEnvironmentVariable("PATH","User")
   [System.Environment]::SetEnvironmentVariable("PATH","$p;$env:USERPROFILE\.grok\bin","User")
   ```
   Restart your terminal after this step.

3. **Write config** — create `%USERPROFILE%\.grok\config.toml` with the content
   shown in the [Configuration](#configuration) section below.

---

## Configuration

The config file lives at `%USERPROFILE%\.grok\config.toml` (`~/.grok/config.toml`).

### Minimal config (DeepSeek V4)

```toml
[auth]
preferred_method = "api_key"   # skip xAI OIDC login

[models]
default = "deepseek-v4-pro"

[model.deepseek-v4-pro]
model          = "deepseek-v4-pro"
base_url       = "https://api.deepseek.com/v1"
api_backend    = "chat_completions"
env_key        = "DEEPSEEK_API_KEY"   # reads from environment variable
context_window = 900000

[model.deepseek-v4-flash]
model          = "deepseek-v4-flash"
base_url       = "https://api.deepseek.com/v1"
api_backend    = "chat_completions"
env_key        = "DEEPSEEK_API_KEY"
context_window = 900000
```

### API key options

You can supply the key in two ways:

**A. Environment variable** (recommended — key never written to disk):
```powershell
# Current session only:
$env:DEEPSEEK_API_KEY = "sk-..."

# Permanent (survives reboots):
[System.Environment]::SetEnvironmentVariable("DEEPSEEK_API_KEY","sk-...","User")
```

**B. Directly in config.toml** (simpler, but key is stored in plaintext):
```toml
[model.deepseek-v4-pro]
...
api_key = "sk-..."
```

---

## First run

```powershell
grok
```

grok will start the TUI. You should see the prompt within a few seconds. If you
see a login screen instead of a prompt, check that your `config.toml` contains
`[auth] preferred_method = "api_key"` and that `DEEPSEEK_API_KEY` is set.

### Useful commands

| Action | Command |
|--------|---------|
| One-shot prompt | `grok -p "Hello"` |
| Switch model | `/model deepseek-v4-flash` inside TUI |
| List available models | `grok models` |
| Print version | `grok --version` |
| Open help | `grok --help` |

---

## Uninstall

```powershell
# Remove binary
Remove-Item "$env:USERPROFILE\.grok\bin\grok.exe"

# Remove from PATH
$p = [System.Environment]::GetEnvironmentVariable("PATH","User") -split ";" |
     Where-Object { $_ -ne "$env:USERPROFILE\.grok\bin" }
[System.Environment]::SetEnvironmentVariable("PATH",($p -join ";"),"User")

# Optionally remove all grok data (sessions, config, cache)
# WARNING: this deletes your entire session history.
# Remove-Item -Recurse "$env:USERPROFILE\.grok"
```

---

## Troubleshooting

### `grok` is not recognised after install

The PATH change takes effect in **new** terminal windows. Close and reopen your
terminal, or run:
```powershell
$env:PATH += ";$env:USERPROFILE\.grok\bin"
```

### Login screen appears instead of prompt

Check that `~/.grok/config.toml` exists and contains:
```toml
[auth]
preferred_method = "api_key"
```
And that at least one model entry has `env_key` or `api_key` set.

### Connection error / 401 Unauthorized

- Verify the key is set: `Write-Output $env:DEEPSEEK_API_KEY`
- Check your balance at [platform.deepseek.com](https://platform.deepseek.com)
- Confirm the base URL is `https://api.deepseek.com/v1`

### Model not found

Run `grok models` to list all configured models. Make sure the `[model.<id>]`
section in config.toml exists and the `model` field matches the API model ID
exactly (e.g. `deepseek-v4-pro`, not `deepseek-v4`).

---

## Known limitations (Windows)

Because this project targets Linux/macOS as its primary platform, some features
do not function on Windows at runtime even though the binary compiles:

- **PTY / interactive shell sessions** — local terminal execution uses Unix PTY APIs; shell integration is limited.
- **Unix socket IPC** — the leader–follower process model uses Unix domain sockets internally.
- **`/proc` filesystem diagnostics** — Linux-specific; gracefully skipped.

Core features — AI conversations, file editing, web search, MCP tools, session
history — work normally.
