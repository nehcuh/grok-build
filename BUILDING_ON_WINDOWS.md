# Building on Windows

This document describes how to build this project natively on Windows (x86_64 MSVC toolchain).

> **Note:** The primary development and release targets are Linux and macOS.
> Windows support is not officially tested in CI. These instructions reflect
> known workarounds for building locally and may require adjustment as the
> codebase evolves.

## Prerequisites

### Rust toolchain

Install the Rust toolchain via [rustup](https://rustup.rs/). This project pins
a specific stable channel in `rust-toolchain.toml`; rustup will download it
automatically on first build.

The MSVC toolchain is required. Make sure the Visual Studio Build Tools
(or a full Visual Studio installation) are present, including:

- MSVC C++ x64 build tools
- Windows 10/11 SDK

### Protocol Buffers compiler (protoc)

The project ships `bin/protoc` as a [DotSlash](https://dotslash-cli.com/)
manifest. DotSlash only supports macOS and Linux, so the `bin/protoc` file
cannot be executed on Windows.

You must supply a Windows-native `protoc.exe` instead.

**Step 1 — Download protoc v29.3 for Windows:**

```
https://github.com/protocolbuffers/protobuf/releases/download/v29.3/protoc-29.3-win64.zip
```

> Use version **29.3** to match the digest pinned in `bin/protoc`.

**Step 2 — Extract to a stable location, for example:**

```
C:\tools\protoc-29.3\
  └─ bin\
      └─ protoc.exe
  └─ include\
      └─ google\protobuf\...
```

**Step 3 — Register the path in your user-level Cargo config
(`%USERPROFILE%\.cargo\config.toml`), creating the file if it does not exist:**

```toml
[env]
PROTOC = "C:\\tools\\protoc-29.3\\bin\\protoc.exe"
```

The `PROTOC` environment variable is checked first by `xai-proto-build`'s
`find_protoc` logic and bypasses the unusable DotSlash wrapper entirely.

---

## Build configuration fixed in `.cargo/config.toml`

The project-level `.cargo/config.toml` already includes the following entries
for the Windows MSVC targets, so no extra flags are needed at the command line:

```toml
[target.x86_64-pc-windows-msvc]
rustflags = [
    "-C", "force-unwind-tables=yes",
    "-C", "target-feature=+crt-static",
    # Avoids MSVC linker LNK1318 "PDB LIMIT" error on very large binaries.
    "-C", "debuginfo=0",
    "-C", "link-arg=/DEBUG:NONE",
    # Avoids STATUS_STACK_OVERFLOW (0xC00000FD) at startup.
    "-C", "link-arg=/STACK:67108864",
]
```

**Why these flags are needed:**

| Flag | Reason |
|------|--------|
| `debuginfo=0` | The main binary (`xai-grok-pager`) is large enough that MSVC's PDB writer hits an internal page-count limit (`LNK1318 LIMIT (12)`) when debug information is included. Disabling debug info sidesteps this limit. |
| `/DEBUG:NONE` | Tells `link.exe` to skip PDB file creation entirely, preventing the linker from even attempting to write one. |
| `/STACK:67108864` | Windows default main-thread stack is 1 MB (vs 8 MB on Linux/macOS). This application has deep call chains at startup, before the tokio runtime and its worker threads start, causing `STATUS_STACK_OVERFLOW`. This flag raises the PE header stack reserve to 64 MB. |

If you need to use a Windows debugger, replace the `debuginfo=0` and
`/DEBUG:NONE` flags with `-C link-arg=/PDBPAGESIZE:16384` to raise the PDB
page size instead of disabling the file altogether.

> **Important:** Do **not** set the `RUSTFLAGS` environment variable manually.
> When `RUSTFLAGS` is set, it *replaces* (not supplements) the `rustflags`
> from `config.toml`, so platform-specific flags like `/STACK` and
> `/DEBUG:NONE` will be silently dropped, leading to linker errors or
> stack overflows at runtime.

---

## Source patch required: `xai-proto-build`

The build helper crate `crates/build/xai-proto-build` originally passed
`/dev/stdout` and `/dev/null` as protoc output paths, which are valid only on
Unix. The working tree includes a cross-platform fix:

- Dependency output is written to a `tempfile::NamedTempFile` and read back,
  instead of piped through `/dev/stdout`.
- The null discard device is selected at compile time (`NUL` on Windows,
  `/dev/null` elsewhere).
- Path separators in the dependency output are normalized before filtering, so
  Windows backslashes do not break the `google/protobuf/` well-known-type skip
  rule.

This fix is backward-compatible; Linux and macOS builds are unaffected.

---

## Building

Once the prerequisites above are in place, a plain `cargo build` works:

```powershell
cargo build --workspace
```

A full workspace debug build takes roughly 4–6 minutes on a modern machine
(most of that is the first-time dependency compilation; incremental rebuilds
are much faster).

---

## Known limitations

- **PTY / terminal features** — Much of the terminal-interaction code uses Unix
  PTY APIs (`openpty`, `forkpty`, etc.). These code paths compile but cannot
  function on Windows. Integration tests that rely on PTY sessions will fail or
  be skipped.
- **Unix socket IPC** — The leader/follower IPC uses Unix domain sockets. The
  server binary will not run correctly on Windows.
- **`/proc` filesystem access** — Several Linux-specific diagnostics reference
  `/proc`. These are best-effort and will degrade gracefully at runtime.

In short: the binaries compile, but runtime behavior is limited to the
platform-independent subset of functionality.
