# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

Proxmox VE Helper-Scripts: a large collection of shell scripts that let users spin up LXC containers and VMs on a Proxmox VE host with a single `bash -c "$(curl -fsSL ...)"` command. There is no application code, no package.json, and no build system — the "product" is the scripts themselves plus shared function libraries they source at runtime over HTTPS from `raw.githubusercontent.com`.

## Critical Rule: No New Scripts In This Repo

**Never add new `ct/*.sh` + `install/*-install.sh` pairs to this repository.** New scripts belong exclusively in the sibling repo [community-scripts/ProxmoxVED](https://github.com/community-scripts/ProxmoxVED), where they're tested against a real Proxmox host before a maintainer promotes them here. `.github/workflows/close-new-script-prs.yml` auto-closes PRs that add new scripts directly to this repo.

This repo (ProxmoxVE) only accepts **bug fixes, improvements, and features for scripts that already exist here**. If asked to "add support for App X," check whether `ct/<app>.sh` already exists first — if not, the correct answer is that it belongs in ProxmoxVED, not here.

Also out of scope for this repo: the `json/` metadata directory has been removed and is gitignored (`json/`, `json/*.bak`, `json/*.tmp`). Script/website metadata (name, description, logo, tags, `app_vars`) is now managed externally via PocketBase/the website, not via files in this repo — do not create or edit `json/` files. Likewise, the AI-assistance PR template section references `AGENTS.md` and `.github/agents/pve-script-creator.agent.md` — both live in ProxmoxVED, not here.

## No Build/Lint/Test Pipeline

There is no test suite, no CI shellcheck/shfmt enforcement, and nothing to `npm install` or `make`. `.github/workflows/` is entirely about PR process automation, not static analysis: `close-new-script-prs.yml` and `close-invalid-pr-template.yml` gate new scripts and PR template compliance, `auto-update-app-headers.yml` regenerates ASCII art headers in `tools/headers/` when `ct/**.sh`, `tools/**.sh`, or `vm/**.sh` change, `push-json-to-pocketbase.yml` syncs metadata to the website's backend, and the rest handle labeling/changelogs/stale-issue cleanup.

The only local validation tooling is a root `.shellcheckrc` (`disable=SC2034,SC1091,SC2155,SC2086,SC2317,SC2181,SC2164`) used by the recommended VS Code ShellCheck extension — it is not invoked by any script or CI job. Recommended extensions (`.vscode/extensions.json`): `bmalehorn.shell-syntax`, `timonwong.shellcheck`, `foxundermoon.shell-format`, `editorconfig.editorconfig`.

There is no runnable "test a script" command — the only realistic way to validate a change is to run it against a real (or throwaway) Proxmox VE host, e.g. `bash -c "$(curl -fsSL https://raw.githubusercontent.com/<fork>/ProxmoxVE/<branch>/ct/<app>.sh)"`.

## Architecture: The Two-File-Per-App Pattern

Every app is (normally) implemented as **two separate files linked only by a naming convention**, not by any explicit reference:

- `ct/<AppName>.sh` — runs on the **Proxmox host**. Declares `APP="<AppName>"` and `var_*` defaults (cpu/ram/disk/os/version/unprivileged/tags), sources `misc/build.func` over HTTP, then calls `variables`, `color`, `catch_errors`, `start`, `build_container`, `description`. Also defines `update_script()`, which the *same file* re-executes later **inside the container** to handle in-place updates (see execution flow below).
- `install/<appname>-install.sh` — runs **inside the freshly created LXC**. Sources the function library the host already fetched (via `$FUNCTIONS_FILE_PATH`, see below) and does the actual `apt`/binary/GitHub-release install.

The link between the two files is purely string derivation done in `variables()` (in `misc/build.func`): `NSAPP=$(echo "${APP,,}" | tr -d ' ')`, then `var_install="${NSAPP}-install"`. So `APP="Adguard"` in `ct/adguard.sh` resolves to `install/adguard-install.sh` — there is no other pointer between the two files. Some apps have a second entry point `ct/alpine-<name>.sh` that reuses the *same* `install/<name>-install.sh`, dispatched via `var_os=alpine`.

**Exception (a growing minority of apps):** newer Docker-based apps such as `coolify`, `dockge`, `dokploy`, `komodo`, `runtipi`, `overseerr`, `minio` have `ct/<name>.sh` but **no** `install/<name>-install.sh` — install/update logic instead lives in `tools/addon/<name>.sh`. Check for this before assuming the standard pairing when investigating or modifying one of these apps.

## Shared Function Libraries (`misc/*.func`)

`ct/*.sh` and `install/*.sh` are thin — nearly all behavior comes from libraries sourced at runtime:

- `core.func` — colors, `header_info`, `msg_info`/`msg_ok`/`msg_error`, the `silent()` wrapper (behind the `$STD` variable — expands to nothing when verbose, `"silent"` when quiet, to suppress noisy install output), spinner, `cleanup_lxc`.
- `build.func` (largest file) — the host-side orchestrator: `variables()` (derives `NSAPP`/`var_install`), `start()` (decides install vs. update, see below), `build_container()` (`pct create`s the LXC, exports `FUNCTIONS_FILE_PATH`, `lxc-attach`es the install script), `description()` (writes the container's Proxmox HTML description linking to `community-scripts.org/scripts/<slug>`), whiptail Default/Advanced menus.
- `install.func` / `alpine-install.func` — in-container helpers for Debian/Ubuntu vs. Alpine (`setting_up_container`, `network_check`, `update_os`, `motd_ssh`, `customize`). Which one gets fetched into the container depends on `var_os`.
- `tools.func` (second largest) — reusable install-time helpers, e.g. `fetch_and_deploy_gh_release`, retrying `apt` install, keyring/repo management. Most `install/*-install.sh` files lean heavily on this rather than reimplementing installs.
- `error_handler.func` — `catch_errors` trap/signal handling and failure telemetry.
- `api.func` — opt-in anonymous telemetry to `telemetry.community-scripts.org` (gated by a `DIAGNOSTICS` flag).
- `cloud-init.func`, `vm-core.func` — Cloud-Init config and the VM-flavored analogue of `core.func`, used by `vm/*.sh` instead of the LXC path.

## End-to-End Execution Flow

1. User runs `bash -c "$(curl -fsSL .../ct/adguard.sh)"` on the Proxmox host.
2. `ct/adguard.sh` sources `misc/build.func` over HTTP, sets `APP`/`var_*`, calls `variables`, `color`, `catch_errors`, then `start`.
3. `start()` sources `tools.func` and checks `command -v pveversion`: present → this is the host, run the normal whiptail-driven `install_script` (fresh container creation); absent → this is a re-run **inside an already-built container** (e.g. via the container's `update` command), so it instead calls the script's own `update_script()`. **The same `ct/*.sh` file is both the installer and the updater**, branching on where it's executing.
4. `build_container()` runs `pct create` with `var_cpu`/`var_ram`/`var_disk`/`var_os`/`var_version`/`var_unprivileged`, downloads the matching in-container func library (`install.func` or `alpine-install.func`) and exports it as the `FUNCTIONS_FILE_PATH` env var, then fetches `install/${var_install}.sh` and runs it via `lxc-attach -n "$CTID" -- bash -c "$_install_script"`.
5. Inside the container, `install/<app>-install.sh` sources `$FUNCTIONS_FILE_PATH` via `source /dev/stdin <<<"$FUNCTIONS_FILE_PATH"` (avoids a second network round trip) and performs the actual install using `msg_info`/`msg_ok`/`msg_error` pairs and `$STD`-wrapped commands.
6. Back on the host, `build_container()` checks a failure flag, then `ct/<app>.sh` calls `description()` to write the HTML description onto the container and prints the completion banner (IP, URL).

## Answering-a-Prompt-Up-Front Convention

Install scripts must be deployable unattended, so interactive prompts must check for an existing value first:

```bash
if [[ -z "${var_admin_user:-}" ]]; then
  read -r -p "${TAB3}Admin username: " var_admin_user
fi
var_admin_user="${var_admin_user:-admin}"
```

Naming: `var_<something>`, the same namespace as container-config `var_*` variables. Critically, the variable **must also be exported from `ct/<app>.sh`** (`export var_admin_user="${var_admin_user:-}"`) — `lxc-attach` only forwards variables that were actually exported, so an unexported `var_*` silently never reaches the container's install script.

## `dev_mode` Debug Flags

The only debugging tooling in this repo is the `dev_mode` env var, comma-combinable, set before invoking a `ct/*.sh` URL:

```bash
dev_mode="trace,keep" bash -c "$(curl -fsSL https://raw.githubusercontent.com/cgonzalezbermudez/ProxmoxVE/main/ct/myapp.sh)"
```

| Flag | Effect |
| --- | --- |
| `trace` | `set -x` for maximum verbosity |
| `keep` | don't delete the container if the build fails |
| `pause` | pause execution at key points before customization |
| `breakpoint` | drop to a shell at hardcoded `breakpoint` calls |
| `logs` | save detailed build logs to `/var/log/community-scripts/` |
| `motd` | force-update the Message of the Day |
| `net` | log every engine fetch (HTTP status, duration, URL) |
| `timing` | time each step, list the slowest at the end |

`dryrun` was removed — it only intercepted the `silent()` wrapper, so `pct create` still ran and the container was still built regardless.

## Code Standards (from CONTRIBUTING.md)

- Shebang `#!/usr/bin/env bash` on every script.
- Quote all variables (`"$VAR"`, not `$VAR`).
- Filenames: lowercase, hyphen-separated.
- Variable names: lowercase.
- One script per service.
- PRs should touch only the files relevant to the change — no unrelated modifications bundled in.

## Repo Layout

- `ct/` — host-side "create/update container" scripts (one per app).
- `install/` — in-container install scripts (one per app).
- `vm/` — VM creation scripts (Cloud-Init based, use `vm-core.func`/`cloud-init.func` instead of the LXC libraries).
- `misc/` — the shared `.func` libraries described above, plus `misc/images/`.
- `tools/` — `tools/pve` (host maintenance), `tools/addon` (post-install add-ons, notably the install path for Docker-based apps with no `install/*-install.sh`), `tools/copy-data` (migration helpers), `tools/headers` (auto-generated ASCII art banners — don't hand-edit, they're regenerated by CI).
- `turnkey/` — `turnkey.sh`, a generic installer for TurnKey Linux appliances.
