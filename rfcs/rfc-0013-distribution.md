# RFC-0013 — Distribution & Installation

- **Status:** Ratified
- **Version:** 1.0
- **References:** `foundation/charter.md` §7 (Non-Goals); `rfcs/rfc-0012-roadmap.md`; `rfcs/README.md`; implementation plan (`dist/`)

## Summary
RFC-0013 is the **authoritative distribution specification** for Tessera v1. It defines how the
runtime + SDK reach end users as a self-contained `dist/` tarball, how the `curl … | bash` bootstrap
verifies and installs it, and where the installed artifacts live. It is the canonical reference for
the tarball-based installer described in the implementation plan. Where this RFC and any other doc
disagree on distribution/installation behavior, this RFC takes precedence.

## 1. Purpose
Distribute Tessera v1 to end users **without publishing to PyPI**. The framework is delivered as a
self-contained source distribution inside a versioned tarball, installed locally from that tarball via
`uv`. No package index, no registry, no network dependency on a third-party repository at install
time beyond fetching the single tarball over TLS.

## 2. Motivation
The simplest trustworthy delivery is a single, checksum-pinned artifact:

```
curl -fsSL <repo>/dist/install.sh | bash
```

This one-line bootstrap downloads a release tarball, verifies its SHA256, extracts it, and runs the
installer. **No index is used** — there is no PyPI package, no `pip install tessera`, and no internal
index server. Motivation for avoiding an index:

- Eliminates a dependency on a package registry and its availability/trust chain.
- The artifact is fully self-contained: source, lockfile, and license ship together.
- Users get exactly the bytes the maintainer signed off on (pinned tag + inline checksum).
- Install is reproducible and inspectable — the bootstrap is tiny and the heavy logic lives in
  `install-modules/` inside the tarball, which is itself checksum-verified before execution.

## 3. Model
**sdist-in-tarball.** The release tarball contains the project source (a standard `pyproject.toml`
layout with `src/`). Installation builds/installs from that extracted source:

- `uv pip install .` is run from the **extracted source root**.
- It is a **non-editable, non-PyPI** install: `uv` resolves dependencies from the vendored `uv.lock`
  against the local source tree. No network PyPI query is required for the local packages; the lock
  pins every transitive dependency.
- Editable install (`-e`) is explicitly **forbidden** by this spec — the user install must be a
  closed, immutable artifact, not a live link back to a checkout.

## 4. Layout
Inside the tarball (and after extraction) the source layout is:

```
src/
├── tessera_runtime/     # the engine (was lib/ticket-management/)
└── tessera_sdk/         # the client SDK (was tessera/)
```

Packaging is driven by the root `pyproject.toml`, which builds both `tessera_runtime` and
`tessera_sdk`. The install exposes two console commands:

- `tessera` — the runtime CLI (Typer app over the runtime socket).
- `ticket` — the thin client CLI (SDK entry point).

The install prefix is named **`tessera/`** under the chosen user prefix (see §8). Both `tessera` and
`ticket` are symlinked into the user `bin` directory.

## 5. dist/ Manifest (exact)
The `dist/` directory (release staging) and the resulting tarball contain exactly:

**Included:**
- `pyproject.toml` — packaging for `src/tessera_runtime` + `src/tessera_sdk`.
- `uv.lock` — fully pinned dependency lock (reproducible, offline-resolvable).
- `LICENSE` — project license.
- `README.md` — quick-start / post-install guidance.
- `src/` — the engine and SDK source (`tessera_runtime/`, `tessera_sdk/`).
- `install-modules/` — the installer shell modules (see §6).
- `e2e/` — curated end-to-end smoke tests (see §9).

**Tarball artifacts (two files per release):**
- `tessera-<version>.tar.gz` — the source distribution tarball.
- `tessera-<version>.tar.gz.sha256` — the SHA256 checksum of the tarball.

**Excluded from the tarball:**
- `tests/` — except the curated `e2e/` subset.
- `formal-specifications/` — the docs repo is never shipped.
- `__pycache__/` and any compiled/cached artifacts.
- `.git/`, `.worktrees/`, `.hermes/`.
- Any `TicketsRepository/` or `.ticket-runtime/` runtime state directories.

`<version>` is the release version (e.g. `1.0.0`), matching the pinned tag in §11.

## 6. Installer Architecture
The installer is split into a tiny curl target and a set of modules shipped inside the tarball.

### 6.1 `dist/install.sh` (curl target — tiny)
The only thing `curl` fetches and pipes to `bash`. It is intentionally small and auditable:

- Hardcodes the **release base URL** (the `<repo>/dist/` location).
- Hardcodes the **pinned tag/version** (e.g. `v1.0.0`).
- Hardcodes the **expected tarball name** (`tessera-<version>.tar.gz`).
- Embeds the **inline SHA256** of that tarball.
- Downloads the tarball and its `.sha256` sidecar.
- **VERIFIES the checksum** and **refuses** (non-zero exit) on mismatch.
- Extracts to a temporary directory.
- `exec bash install-modules/main.sh` to hand off to the real installer.
- Supports flags:
  - `--check` — verify the download + checksum only; do not install.
  - `--dry-run` — run prechecks and print planned actions without mutating the system.

### 6.2 `dist/install-modules/`
- **`00-precheck.sh`** — Detect an available Python `>=3.12`. If absent, install `uv` (official
  installer, checksum-verified) and run `uv python install 3.12`. Detect/require `uv`. Perform a
  **Linux-only** platform check (abort on non-Linux — see §10).
- **`01-perms.sh`** — User-scope permission stub. **NO `sudo`** is ever invoked; the install is
  entirely within the user's home.
- **`02-scaffold.sh`** — Create the prefix `~/.local/share/tessera`. Run `uv pip install .` from the
  extracted source. Register the user systemd unit template
  `~/.config/systemd/user/tessera-runtime@.service`. Symlink `~/.local/bin/tessera` and
  `~/.local/bin/ticket` to the installed entry points.
- **`03-test.sh`** — Run `e2e/smoke_test.py`. A **non-zero exit aborts the install** (rollback via
  cleanup, see §9).
- **`04-cleanup.sh`** — Remove the tarball, bootstrap script, extraction directory, E2E artifacts,
  and any temporary repos. **NEVER removes the installed application.**
- **`main.sh`** — Sets `set -euo pipefail` and installs a `trap` to `04-cleanup.sh` on EXIT/error,
  then runs modules `00` → `04` in order.

## 7. Security
- The tarball is **SHA256-verified before any extraction or execution** of its contents. A mismatch
  is a hard refusal.
- Transport is **TLS-only** — the release base URL must be `https://`.
- The bootstrap pins an exact **tag/version** plus an **inline expected checksum**, so a compromised
  host serving a different tarball fails verification.
- **`sudo` is avoided entirely.** Every action stays in the user's home; nothing touches system
  directories. This removes the largest class of install-time privilege-escalation risk.
- (Future signing — see §12 — adds a second, independent verification layer on top of the checksum.)

## 8. Locations
All artifacts are user-scoped:

| Artifact | Path |
| --- | --- |
| Install prefix (app) | `~/.local/share/tessera/` |
| Runtime systemd unit (template) | `~/.config/systemd/user/tessera-runtime@.service` |
| CLI entry — `tessera` | `~/.local/bin/tessera` (symlink) |
| CLI entry — `ticket` | `~/.local/bin/ticket` (symlink) |

No files are written outside the user's home. `~/.local/bin` must be on `PATH` (standard on most
Linux distributions).

## 9. Post-install E2E
After scaffolding, `03-test.sh` runs `e2e/smoke_test.py`, which asserts:

- A lifecycle/event **hook fired** during a minimal runtime start.
- The environment variable **`TESSERA_TICKET_ID`** is set (proves the SDK→runtime path is wired).

If the smoke test returns non-zero, the installer aborts and `04-cleanup.sh` tears down the partial
install — the user is left with no half-installed `tessera` on `PATH`.

## 10. Platform Constraint
**Linux-only.** This is a Charter §7 Non-Goal for v1 (Windows/macOS explicitly out of scope). The
installer detects the platform in `00-precheck.sh` and **aborts with a clear message** on any
non-Linux OS. No macOS/Windows fallbacks are provided in v1.

## 11. Versioning
- The bootstrap pins to the repository tag **`v1.0.0`** for the v1.0 release (the `<version>` used in
  the tarball name and the inline SHA256).
- Upgrades are **idempotent reinstalls**: re-running the bootstrap with a newer pinned tag re-extracts,
  re-runs `uv pip install .` into the same prefix, and overwrites the previous install. Because the
  install is a closed, non-editable artifact, reinstalling is safe and deterministic; there is no
  side-by-side version state to reconcile.

## 12. Future (v1.1)
- **Signed tarballs** — GPG or sigstore (cosign) signatures verified in addition to (and ideally
  instead of relying solely on) the inline SHA256, giving a cryptographic identity to the release.
- **`--system` mode** — an opt-in mode that installs to a system prefix using `sudo`, with explicit
  user confirmation. v1.0 deliberately omits this; it is the only place `sudo` may ever appear.

## Rationale
A checksum-pinned, index-free tarball is the minimal trust surface for delivering a self-contained
Python application to Linux users. Shipping the source + lockfile and installing locally with `uv`
keeps the artifact reproducible and offline-installable, while the tiny auditable bootstrap plus an
in-tarball verified installer keeps the install path inspectable and privilege-free. This RFC freezes
that contract so the implementation plan and the installer code have a single source of truth.
