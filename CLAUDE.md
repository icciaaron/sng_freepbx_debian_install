# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

ICCI's fork of the official FreePBX Debian install script (`FreePBX/sng_freepbx_debian_install`). Single-file bash installer (`sng_freepbx_debian_install.sh`) that deploys a complete FreePBX 17 + Asterisk stack on Debian 12 (Bookworm) only. No build system, no tests, no CI — standalone shell script run as root on target machines.

## Git Remotes

- `origin` → `icciaaron/sng_freepbx_debian_install` (our fork — safe to push)
- `upstream` → `FreePBX/sng_freepbx_debian_install` (official — never push directly)

## Script Architecture

The script runs linearly with `set -e` and follows this phase order:

1. **Pre-flight checks** — OS version (bookworm only), root, 64-bit, FQDN, ASTVERSION validation, disk space (4GB), RAM (1GB), network connectivity, PID lock, script version check
2. **Repository setup** — Fix sources.list, block Debian 13/Trixie, add FreePBX GPG key + apt repo (HTTPS), set priority pins
3. **Kernel/DAHDI** (optional) — Kernel compatibility check, APT hooks for kernel management
4. **Dependency install** — 80+ packages via apt (Redis, Apache2, MariaDB, PHP 8.2, Node.js, etc.)
5. **System configuration** — asterisk user, TFTP, OpenSSL/Katana compat, Apache, PHP, Postfix hardening
6. **Asterisk install** — Base package + 15 component packages, sounds, version-switch utility
7. **FreePBX install** — ionCube, freepbx17 package, `fwconsole ma installlocal`, `fwconsole ma upgradeall`
8. **Post-install validation** (`set +e`) — Colored service checks, PHP version, module status, port assignments, pm2 processes, install summary

## Key Globals

| Variable | Default | Purpose |
|----------|---------|---------|
| `SCRIPTVER` | `1.15` | Script version, checked against GitHub raw |
| `ASTVERSION` | `22` | Asterisk major version; override with env var. Validated as integer. |
| `PHPVERSION` | `8.2` | PHP version used throughout |
| `LOG_FILE` | `/var/log/pbx/freepbx17-install-YYYY.MM.DD-HH.MM.SS.log` | All stderr redirected here via `exec 2>>` |
| `PKG_COUNTER` | `0` | Incremented by `pkg_install`, shown in output |
| `C_RED/GREEN/YELLOW/CYAN/WHITE/DIM/RESET` | ANSI codes | Color output; empty when stdout is not a terminal |

## Key Functions

- `pkg_install()` — Wrapper around `apt-get install`; shows counter `[N]`, exits on failure
- `install_asterisk(astver)` — Installs base Asterisk + all component packages for the given version
- `setup_repositories()` — Adds FreePBX apt repo (HTTPS) with GPG key, sets priority pins
- `create_post_apt_script()` — Generates `/usr/bin/post-apt-run` via heredoc (DAHDI kernel module upgrades)
- `check_kernel_compatibility()` — Generates `/usr/bin/kernel-check` (DAHDI kernel hold management)
- `show_help()` — Prints usage, all flags, env vars
- `msg_info/success/warn/error/step()` — Colored output helpers (cyan/green/yellow/red/white)
- `log()` / `message()` — Logging (log = file only, message = stdout + file)
- `terminate()` — EXIT trap; shows last 10 log lines on non-zero exit
- `errorHandler()` — ERR trap; colored failure box with recovery hints

## CLI Flags

`--help`, `--version`, `--dahdi` / `--dahdi-only`, `--noasterisk`, `--nofreepbx`, `--opensourceonly`, `--noaac`, `--nochrony`, `--dev`, `--testing`, `--skipversion`, `--debianmirror <URL>`, `--npmmirror <URL>`, `--disable-deb-update-v13`

## Working With This Script

- **Syntax check:** `bash -n sng_freepbx_debian_install.sh` — always run after edits
- **Test on Debian 12 VM** — no other way to validate. Use `--skipversion` when running modified versions.
- **`set -e` is active** throughout install phases. Validation phase switches to `set +e`.
- **Error handling relies on traps** — `ERR` trap calls `errorHandler`, `EXIT` trap calls `terminate`.
- **Generated scripts** — `create_post_apt_script()` and `check_kernel_compatibility()` write scripts to `/usr/bin/`. The post-apt-run uses a heredoc; kernel-check still uses echo-by-echo (candidate for future cleanup).
- **Color output** — Terminal-detected via `[ -t 1 ]`. All color vars are empty when piped. Use `msg_*` helpers for new output.

## ICCI Working Directory

`icci/` is gitignored — use it for notes, tracking docs, and scratch files that shouldn't go upstream. Currently contains `upstream-contributions.md` with the full PR plan, issue list, and testing checklist.

## Upstream Sync & Contributions

- Sync: `git fetch upstream && git merge upstream/master`
- Branch naming: `fix/description` or `feat/description`
- CLA required: https://oss-cla.sangoma.com/freepbx/sng_freepbx_debian_install
- Bug tracker: https://github.com/FreePBX/issue-tracker/issues
- Split changes into small, focused PRs. Test on Debian 12 VM before submitting.
