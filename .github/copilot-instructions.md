# Copilot Instructions for ProxmoxVE Helper-Scripts

## What This Repository Is

A collection of shell scripts that create and manage LXC containers/VMs on a Proxmox VE host. There is no application code, no build system, and no test suite — the "product" is the scripts themselves plus shared function libraries sourced at runtime from `raw.githubusercontent.com`.

## Scope: What Changes Are Accepted Here

**This repo only accepts bug fixes, improvements, and features for scripts that already exist here.**

- ✅ Fix a bug in `ct/<app>.sh` or `install/<app>-install.sh`
- ✅ Improve update logic in an existing `update_script()` function
- ❌ Add new `ct/<app>.sh` + `install/<app>-install.sh` pairs — these belong in [ProxmoxVED](https://github.com/community-scripts/ProxmoxVED)
- ❌ Create or edit files in `json/` — metadata is managed via PocketBase externally

## Architecture: Two-File-Per-App Pattern

Every app is two files linked only by naming convention (not any explicit reference):

| File | Runs on | Purpose |
|---|---|---|
| `ct/<AppName>.sh` | Proxmox host | Declares `APP` + `var_*` defaults, creates the LXC, defines `update_script()` |
| `install/<appname>-install.sh` | Inside the LXC | Does the actual `apt`/binary/service install |

**The link:** `build.func`'s `variables()` derives the install filename via `NSAPP=$(echo "${APP,,}" | tr -d ' ')`. So `APP="Adguard"` in `ct/adguard.sh` resolves to `install/adguard-install.sh`. No other pointer exists between the files.

**Docker-based exception:** Some apps (e.g. `coolify`, `dockge`, `komodo`) have `ct/<name>.sh` but **no** `install/<name>-install.sh`. Their install/update logic lives in `tools/addon/<name>.sh`.

## Execution Flow

1. User runs `bash -c "$(curl -fsSL .../ct/app.sh)"` on the Proxmox host.
2. `ct/app.sh` sources `misc/build.func` over HTTP, sets `APP`/`var_*`, calls `variables`, `color`, `catch_errors`, then `start`.
3. `start()` checks `command -v pveversion`:
   - **Present (host):** run the whiptail-driven install menu → `build_container()`
   - **Absent (inside container):** call `update_script()` defined in the same `ct/*.sh` file
4. `build_container()` runs `pct create`, exports `FUNCTIONS_FILE_PATH` env var (the fetched in-container func library), then runs `install/<app>-install.sh` via `lxc-attach`.
5. Inside the container, `install/<app>-install.sh` sources `$FUNCTIONS_FILE_PATH` via `source /dev/stdin <<<"$FUNCTIONS_FILE_PATH"`.

## Shared Function Libraries (`misc/*.func`)

- **`build.func`** — host orchestrator: `variables()`, `start()`, `build_container()`, `description()`, whiptail menus
- **`core.func`** — colors, `msg_info`/`msg_ok`/`msg_error`, `silent()` wrapper (the `$STD` variable), spinner, `cleanup_lxc`
- **`tools.func`** — in-container helpers: `fetch_and_deploy_gh_release`, `setup_nodejs`, `setup_postgresql`, `setup_mariadb`, `setup_mysql`, `setup_php`, `setup_rust`, `setup_java`, `setup_uv`, `download_gpg_key`, `manage_tool_repository`, and many more
- **`install.func`** / **`alpine-install.func`** — in-container bootstrap: `setting_up_container`, `network_check`, `update_os`, `motd_ssh`, `customize`
- **`error_handler.func`** — `catch_errors` trap/signal handling
- **`vm-core.func`**, **`cloud-init.func`** — VM-flavored analogues used by `vm/*.sh`

## Key Conventions

### Standard install script header (Debian/Ubuntu)
```bash
#!/usr/bin/env bash
source /dev/stdin <<<"$FUNCTIONS_FILE_PATH"
color
verb_ip6
catch_errors
setting_up_container
network_check
update_os
```

### Standard ct/ script structure
```bash
#!/usr/bin/env bash
source <(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/misc/build.func)
APP="MyApp"
var_tags="${var_tags:-mytag}"
var_cpu="${var_cpu:-1}"
var_ram="${var_ram:-512}"
var_disk="${var_disk:-4}"
var_os="${var_os:-debian}"
var_version="${var_version:-12}"
var_unprivileged="${var_unprivileged:-1}"

header_info "$APP"
variables
color
catch_errors

function update_script() { ... }

start
build_container
description
```

### Suppressing noisy install output
Use `$STD` (expands to `silent` in quiet mode, nothing in verbose) to wrap commands:
```bash
$STD apt-get install -y nginx
```

### Interactive prompts must support unattended deploys
Always check for an existing variable before prompting, and export it from `ct/<app>.sh`:
```bash
# In ct/app.sh
export var_admin_user="${var_admin_user:-}"

# In install/app-install.sh
if [[ -z "${var_admin_user:-}" ]]; then
  read -r -p "${TAB3}Admin username: " var_admin_user
fi
var_admin_user="${var_admin_user:-admin}"
```

### GitHub release deployments
Use `fetch_and_deploy_gh_release` from `tools.func`:
```bash
# Binary (single file)
fetch_and_deploy_gh_release "myapp" "org/myapp" "binary"

# Tarball with specific file pattern
fetch_and_deploy_gh_release "myapp" "org/myapp" "prebuild" "latest" "/opt/myapp" "myapp_linux_$(arch_resolve).tar.gz"

# With explicit version
fetch_and_deploy_gh_release "myapp" "org/myapp" "prebuild" "1.2.3" "/opt/myapp" "myapp-linux-amd64.tar.gz"
```

## Code Standards

- Shebang: `#!/usr/bin/env bash`
- Quote all variables: `"$VAR"` not `$VAR`
- Lowercase variable names
- Lowercase, hyphen-separated filenames
- Never hardcode credentials
- PRs should only touch files relevant to the change

## Linting

No CI shellcheck enforcement. Local validation uses `.shellcheckrc`:
```
disable=SC2034,SC1091,SC2155,SC2086,SC2317,SC2181,SC2164
```
Recommended VS Code extensions: `bmalehorn.shell-syntax`, `timonwong.shellcheck`, `foxundermoon.shell-format`.

## Debugging/Testing

The only way to test a script is to run it against a real Proxmox VE host. Use `dev_mode` flags (comma-separated):

```bash
dev_mode="trace,keep" bash -c "$(curl -fsSL https://raw.githubusercontent.com/YOUR_FORK/ProxmoxVE/YOUR_BRANCH/ct/myapp.sh)"
```

| Flag | Effect |
|---|---|
| `trace` | `set -x` for maximum verbosity |
| `keep` | Don't delete the container if the build fails |
| `pause` | Pause at key points before customization |
| `breakpoint` | Drop to a shell at hardcoded `breakpoint` calls |
| `logs` | Save build logs to `/var/log/community-scripts/` |
| `motd` | Force-update the Message of the Day |
| `net` | Log every engine fetch (HTTP status, duration, URL) |
| `timing` | Time each step, list slowest at end |
