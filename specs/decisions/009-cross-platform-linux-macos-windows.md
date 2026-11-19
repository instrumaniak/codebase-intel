# ADR 009 — Cross-platform support (Linux/macOS/Windows) is first-class, abstracted behind one module

## Status: Accepted

**Note (2026-11-19, pre-implementation spec review):** written during the review that made the mandate; supersedes the original "Linux first-class, macOS best-effort, Windows unsupported until demanded" position.

## Date: 2026-11

## Context

Original scope: Linux first-class, macOS best-effort, Windows unsupported until demanded. The OSS release (v0.4) targets strangers running `cargo install` and prebuilt binaries; Windows is a large share of dev machines; and the platform-specific surface in the design turned out to be small:

1. File locking (`<db>.lock`)
2. Thread/IO priority for the indexing worker
3. Data-directory resolution
4. RSS measurement in tests
5. The build/release/CI matrix

Everything else (tree-sitter, rusqlite, sqlite-vec, gix, blake3, tokio, `ignore`, `rmcp`) is already cross-platform. Linux-first left a whole class of "works on my machine" failures for the v0.4 audience and would have made Windows an untested accidental target.

## Decision

**Linux, macOS and Windows (x86_64) are all first-class from v0.1.** CI runs fmt + clippy + tests on `ubuntu-latest`, `macos-latest`, `windows-latest`; `cargo-dist` ships binaries for `x86_64-unknown-linux-gnu`, `aarch64-unknown-linux-gnu`, `x86_64-apple-darwin`/`aarch64-apple-darwin`, `x86_64-pc-windows-msvc`. Platform-specific behavior is confined to:

- `util.rs` — platform data dir via `directories::BaseDirectories::data_dir()`; background-class priority setter (Unix: per-thread `nice(19)`, plus best-effort `ioprio_set`/`setiopolicy_np`; Windows: `SetThreadPriority(THREAD_MODE_BACKGROUND_BEGIN)`)
- the walker — symlink/junction handling and its confinement test (symlink test is `#[cfg(unix)]` only; Windows confinement covered by canonicalization unit tests)
- `cargo test`'s `deps` test fails on any `#[cfg(unix)]`/`#[cfg(windows)]` outside those two modules (§11)

File locking uses **std `std::fs::File::try_lock`** — advisory, nonblocking, released on handle close, `flock(LOCK_EX|LOCK_NB)` on Unix and `LockFileEx … FAIL_IMMEDIATELY` on Windows, one code path, zero new dependencies. (Requires Rust ≥ 1.89, satisfied by MSRV = current stable − 2 as of Nov 2026. On Windows the lock file is opened read+write, never append-only.) MSRV stays `stable − 2` pinned by `rust-toolchain.toml`.

## Reasons

- The incremental cost is: one `directories` call, one priority call per OS, `sysinfo` as a dev-dependency, and a third CI runner. Small next to the install friction removed for the v0.4 audience.
- std file locking gives identical semantics on both OS families (advisory, nonblocking, released on close) with zero dependency risk — no lock crate to audit or version-pin.
- One tested code path beats three slightly-divergent ones; platform divergence is the failure mode this buys out of.
- Dogfood remains Linux (author machine); Windows gets CI coverage, not dogfood parity — documented, and honest.

## Consequences

- CI minutes grow (~1.5–2× on the PR matrix); a red Windows CI blocks merge like any other platform — no silent "best-effort" leg.
- WAL and mmap require local disks on every platform; `CODEBASE_INTEL_HOME` pointed at a network drive is unsupported (§4.1); NFS/SMB are not supported hosts for the index.
- Case-insensitive filesystems (default Windows & macOS) are tolerated: repo-relative paths stored exactly as walked; the `blake3(canonical_root)` hash is machine-specific by construction (indexes are per-machine artifacts).
- Long paths (>260) rely on std wide-character APIs; `\\?`-prefixed canonicalization is a documented follow-up if dogfooding demands it.
- 32-bit targets unsupported everywhere (no mmap); `aarch64-pc-windows-msvc` not shipped initially — revisit at OSS release.
- Release matrix grows to 5 binaries; `cargo-dist` handles the packaging.

## Revisit Trigger

- Windows CI becomes infrastructure-flaky and blocks merges repeatedly — demote Windows to build-only and re-open this ADR.
- aarch64-windows demand appears — extend the release matrix.
- Linux/macOS-vs-Windows behavior divergence is discovered in dogfooding that testing missed — file new ADRs per divergence, never silent workarounds.
