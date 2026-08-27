# snap-zizmor

**[zizmor](https://github.com/zizmorcore/zizmor) as a [Snap](https://snapcraft.io) package.**

zizmor is a static analysis tool for GitHub Actions — it finds (and explains how to fix) common security issues in typical GitHub Actions CI/CD setups, such as template injection, excessive permissions and unpinned actions. This repo packages it as a strictly-confined Snap.

## Install

[![Get it from the Snap Store](https://snapcraft.io/en/dark/install.svg)](https://snapcraft.io/zizmor)

```bash
sudo snap install zizmor
```

The snap and the app share the same name, so the command is just `zizmor` — no alias needed.

## Usage

Audit a checkout, a single workflow, or standard input:

```bash
zizmor .
zizmor path/to/workflow.yml
cat workflow.yml | zizmor -
```

Audit a repository on GitHub without cloning it (requires online mode, below):

```bash
zizmor owner/repo
```

### Online vs offline mode

zizmor runs **offline** by default: it only reads the files you point it at. Setting `GH_TOKEN`, `GITHUB_TOKEN` or `ZIZMOR_GITHUB_TOKEN` switches it to online mode, which enables audits that query the GitHub API (and is what remote `owner/repo` inputs need). Setting `ZIZMOR_OFFLINE` forces offline mode regardless of any token.

### Exit codes

| Code | Meaning |
| ---- | ------- |
| `0` | Audit succeeded, nothing to report |
| `1` | Error during the audit |
| `2` | Argument parsing failure |
| `3` | No inputs were collected |
| `11` / `12` / `13` / `14` | Findings, with the highest at informational / low / medium / high severity |

## Interfaces

The snap runs under **strict confinement** and declares:

| Interface | Notes |
| --------- | ----- |
| `home` | auto-connected; this is what lets zizmor read repositories under `$HOME` |
| `network` | auto-connected; only used in online mode |
| `removable-media` | **not** auto-connected — `sudo snap connect zizmor:removable-media` to audit a checkout under `/media` or `/mnt` |

> **Note on paths.** The `home` interface excludes top-level hidden directories in `$HOME` (and `~/snap`), so a checkout in e.g. `~/.local/src/repo` is not readable. Only the first path component matters — a repository's own `.github/workflows` is fine, as long as the repository itself is not inside a hidden directory.

## How it works

The snap builds a **pinned upstream release tag** (currently `v1.29.0`), recorded in `snap/snapcraft.yaml`. Renovate raises a pull request whenever zizmorcore/zizmor publishes a new release, so upgrades are reviewed rather than picked up silently on the next build; release candidates (`-rc`) are excluded by a rule in [`renovate.json`](renovate.json).

Only the `crates/zizmor` crate of upstream's workspace is built, via `cargo install --locked`, so the build uses upstream's committed `Cargo.lock` rather than re-resolving dependencies. Upstream's `rust-toolchain.toml` is deliberately ignored — it pins an exact toolchain plus `rustfmt`/`clippy` components this build never uses, and honouring it would silently change the compiler on every upstream bump — so the build uses the stable toolchain, comfortably above zizmor's MSRV.

### Build gates

Every build exercises the **packed tree**, not the build tree, so `apps.zizmor.command` is proven to resolve as well:

- `zizmor --version` must report exactly the pinned release, tying the binary back to the tag.
- A workflow whose `run:` step interpolates `github.event.issue.title` must produce a `template-injection` finding with a findings exit code (11–14).
- A benign workflow must exit `0`, so the gate also fails on an audit that fires indiscriminately.
- The binary audits this repo's own two workflows and must find nothing.

Any of these failing aborts the build instead of shipping an unexercised binary.

## Building locally

Requires `snapcraft` (`sudo snap install snapcraft --classic`) and a working LXD:

```bash
snapcraft
snapcraft lint zizmor_*.snap
```

## Continuous integration

- [`.github/workflows/snap.yml`](.github/workflows/snap.yml) — builds the snap on every push and pull request, then lints it with `snapcraft lint` in the same job. This is a **merge gate only**; it never publishes. Building **and** publishing to `latest/edge` is handled by [snapcraft.io](https://snapcraft.io) on its own schedule, and this repo is intentionally never tagged — the release tags live upstream.
- [`.github/workflows/zizmor.yml`](.github/workflows/zizmor.yml) — audits this repo's own workflows with zizmor. Dogfooding: the tool packaged here is also the tool guarding this repo.

## Disclaimer

This is a **community-maintained Snap**, not an official zizmorcore project — approved by upstream in [zizmorcore/zizmor#1760](https://github.com/zizmorcore/zizmor/issues/1760). Report issues with the packaging here; report issues with the tool itself [upstream](https://github.com/zizmorcore/zizmor/issues).

## Credits

- **[zizmorcore/zizmor](https://github.com/zizmorcore/zizmor)** — the static analysis tool this Snap packages.
- **[barryprice](https://github.com/barryprice)** — Snap packaging and maintenance.

License: MIT, as declared in `snap/snapcraft.yaml`.
