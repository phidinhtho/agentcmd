# Agent Commander — releases

Download site for the **Agent Commander server** — the `agentcmd` binary that
runs on your own computer, drives the Claude Code CLI inside your project
folders, and talks to the Agent Commander mobile app over a TLS certificate the
app pins at pairing time.

This repository holds **release artifacts only** — no source code. Every
release is built and published by hand, one tag at a time.

- Product site — <https://agentcmd.app>
- Install guide, in English and Vietnamese — <https://agentcmd.app/support>
- Every build ever published — [Releases](../../releases)

---

## Install

**macOS, Linux, WSL** — in a terminal:

```bash
curl -fsSL https://github.com/phidinhtho/agentcmd/releases/latest/download/install.sh | bash
```

**Windows** — in PowerShell:

```powershell
irm https://github.com/phidinhtho/agentcmd/releases/latest/download/install.ps1 | iex
```

No `sudo`, no administrator rights, and nothing written outside your own user
folder. The one exception on Windows is a firewall rule, which the installer
prints for you to run yourself — see [Windows notes](#windows-notes).

Run the same line again later and it upgrades in place: it compares versions,
stops the service, swaps the binary, starts it again, and **puts the old binary
back** if the new one does not come up within 30 seconds.

## Before you install

- **The Claude Code CLI, installed and signed in.** The server does not talk to
  Anthropic directly — it runs your CLI. Without it the installer stops *before
  downloading anything* and prints how to get it.
  On Windows this must be the **native `claude.exe`** (`irm https://claude.ai/install.ps1 | iex`);
  the npm `claude.cmd` shim will not work, because the server starts the CLI
  directly rather than through a shell.
- **Windows: Git for Windows** (`winget install Git.Git`). The CLI's Bash tool
  needs it. The installer only warns if it is missing.
- **Nothing else at runtime.** No Node, no database server, no process manager.
  The server is one binary and one SQLite file.

## Supported targets

| Target | Package | Companion desktop app |
|---|---|---|
| `darwin_arm64` | `agentcmd-<version>-darwin_arm64.tar.gz` | menu bar app (macOS 13+) |
| `darwin_amd64` | `agentcmd-<version>-darwin_amd64.tar.gz` | menu bar app (macOS 13+) |
| `linux_amd64` | `agentcmd-<version>-linux_amd64.tar.gz` | none — use the CLI |
| `linux_arm64` | `agentcmd-<version>-linux_arm64.tar.gz` | none — use the CLI |
| `windows_amd64` | `agentcmd-<version>-windows_amd64.zip` | system tray app |

A target is only present in a release if that release was built for it; the
`packages` object in `manifest.json` is the authoritative list, and the
installer says so plainly rather than failing when a target is absent. The same
goes for the companion apps: each is built only on the operating system and
architecture it runs on, so a given release carries the one its build machine
could produce. The `menubar` field of each entry in `manifest.json` tells you
whether that particular package has one.

Notes on the edges:

- **WSL** is served by the Linux packages. Enable systemd in `/etc/wsl.conf`
  before installing, or the background service cannot be installed.
- **Windows on ARM** is not built. The installer will install the x64 package
  and warn you that it runs under emulation.
- **Alpine and other musl systems** are refused: the embedded sidecar is built
  against glibc.

## What the installer does

1. Detects your OS and architecture, and checks the tools it needs are present.
2. Reads `manifest.json` from the release, downloads the package named there,
   and **verifies its SHA-256 before unpacking a single byte**.
3. Puts `agentcmd` where your account can run it. On Windows it adds that
   directory to your user `PATH`; on macOS and Linux it tells you the line to
   add to your shell profile if `~/.local/bin` is not on your `PATH` already.
4. Installs the companion desktop app if the package carries one.
5. Hands over to `agentcmd setup`, which creates the data directory and a
   migrated database, generates the TLS key and certificate the app will pin,
   writes an environment file only you can read, installs and starts the
   background service, and finishes by printing a **pairing QR code** valid for
   ten minutes.

All installation logic lives in `setup`, inside the binary. The scripts only do
the four things that must happen before a binary exists: download, verify,
place, unquarantine.

## Where things end up

| | macOS, Linux, WSL | Windows |
|---|---|---|
| Binary | `~/.local/bin/agentcmd` | `%LOCALAPPDATA%\Programs\agentcmd\agentcmd.exe` |
| Companion app | `~/Applications/Agent Commander.app` | `%LOCALAPPDATA%\Programs\agentcmd\Agent Commander.exe` |
| Data directory | `~/agentcmd-engine` | `%USERPROFILE%\agentcmd-engine` |
| Configuration | `~/.config/agentcmd/.env` | `%APPDATA%\agentcmd\.env` |
| Service logs | `~/agentcmd-engine/logs/service.{out,err}.log` | `%USERPROFILE%\agentcmd-engine\logs\service.{out,err}.log` |
| Background service | launchd agent, or systemd `--user` | Task Scheduler task `org.thopd.agentcmd` |

The data directory must be on a local disk. Setup refuses OneDrive folders,
network drives, UNC paths and `\\wsl$`, because SQLite cannot lock files there.

## Options

`irm … | iex` cannot take arguments, so on Windows the options are environment
variables set before the pipe:

```powershell
$env:AC_INSTALL_VERSION = 'v1.2.0'   # pin a version
$env:AC_INSTALL_PREFIX  = 'D:\apps\agentcmd'
$env:AC_INSTALL_NO_TRAY = '1'        # skip the tray app
$env:AC_INSTALL_NO_SETUP = '1'       # place the binary only
irm https://github.com/phidinhtho/agentcmd/releases/latest/download/install.ps1 | iex
```

On macOS and Linux the same options are flags after `bash -s --`, and the
environment variables work too:

```bash
curl -fsSL …/install.sh | bash -s -- v1.2.0          # pin a version
curl -fsSL …/install.sh | bash -s -- --prefix ~/bin
curl -fsSL …/install.sh | bash -s -- --no-setup      # place the binary only
curl -fsSL …/install.sh | bash -s -- --no-menubar
```

`--force` (`$env:AC_INSTALL_FORCE`) is what lets you reinstall the same version
or go back to an older one; without it a downgrade is refused.

## Installing without a network, and checking by hand

Download the package for the target machine plus `manifest.json` from the
release, copy them across, then:

```bash
./install.sh --from agentcmd-<version>-<os>_<arch>.tar.gz
```

```powershell
.\install.ps1 -From .\agentcmd-<version>-windows_amd64.zip -Sha256 <hex>
```

Every release also carries `SHA256SUMS` for checking by hand:

```bash
shasum -a 256 -c SHA256SUMS      # macOS
sha256sum -c SHA256SUMS          # Linux
```

```powershell
Get-FileHash .\agentcmd-<version>-windows_amd64.zip -Algorithm SHA256
```

## Upgrading

Re-run the install line, or let the binary do it:

```
agentcmd update --check     # is there a newer release? (writes a cache, no install)
agentcmd update             # download, verify, swap, restart — roll back if it fails
```

`update` is the only command in the binary that reaches the network besides the
push relay, and it only runs when you ask for it. It hands the actual swap to
the `install.sh` / `install.ps1` inside the downloaded package, so there is one
implementation of "replace the binary", not two.

Your database, keys, environment file and already-paired devices are untouched
by an upgrade.

## Uninstalling

```bash
curl -fsSL https://github.com/phidinhtho/agentcmd/releases/latest/download/uninstall.sh | bash
```

```powershell
irm https://github.com/phidinhtho/agentcmd/releases/latest/download/uninstall.ps1 | iex
```

That stops and removes the service, the binary and the companion app, and
leaves your data alone.

To delete the database, keys and environment file as well, add `--purge` on
macOS and Linux (`… | bash -s -- --purge`) or set `$env:AC_UNINSTALL_PURGE = '1'`
before the pipe on Windows. Either way it asks you to type `YES` first, from
your terminal — so it cannot happen in a script with nobody watching.

## Windows notes

- **Windows 11, or Windows 10 version 1809 or later, x64.** Windows 11 is what
  releases are tested on.
- **The service is a Task Scheduler task**, `org.thopd.agentcmd`, triggered when
  you sign in. It runs under your own interactive token with least privilege —
  no stored password, no UAC prompt, no console window. It lives inside your login
  session: signing out ends it, signing back in starts it. `--system`,
  `--cron` and `--run-as` do not exist there; Windows has exactly one scope.
- **The firewall rule is the one step that needs an administrator window.** A
  windowless background process never triggers the usual "allow this app?"
  prompt, so without a rule your phone simply cannot reach the server over the
  local network — silently. Setup prints the exact `netsh advfirewall` line;
  run it once in PowerShell opened with *Run as administrator*. Machines that
  pair over Tailscale, or listen only on `127.0.0.1`, do not need it, which is
  why `agentcmd doctor` treats a missing rule as a warning rather than an error.
- **Nothing is code-signed yet.** The installer clears the Mark-of-the-Web from
  what it unpacks, so going through the script is smooth; if you download a zip
  and open it by hand, expect SmartScreen to call it an unknown publisher.
- **The first run can be slow.** Defender scans the ~84 MB sidecar the moment it
  is unpacked. Subsequent starts are fast.
- **One user per machine.** The task name is machine-global, so a second user
  account on the same machine cannot install its own service.

## After installing

```
agentcmd doctor      # re-checks everything and tells you how to fix what is wrong
agentcmd status      # service, engine, cost, paired devices, disk
agentcmd auth show   # a fresh pairing QR code, valid for ten minutes
```

## What is in a release

| Asset | For |
|---|---|
| `agentcmd-<version>-<os>_<arch>.tar.gz` / `.zip` | the packages listed in the table above |
| `agentcmd-menubar-<version>-darwin_<arch>.zip` | the macOS app on its own (already inside the matching darwin package) |
| `agentcmd-tray-<version>-windows_amd64.zip` | the Windows tray app on its own (already inside the windows package) |
| `manifest.json` | machine-readable: version, commit, date, and per-target file name, SHA-256, size |
| `SHA256SUMS` | the same checksums, for `shasum -c` by hand |
| `install.sh`, `uninstall.sh` | what `curl … \| bash` fetches |
| `install.ps1`, `uninstall.ps1` | what `irm … \| iex` fetches |

Two URL shapes, and nothing else:

```
https://github.com/phidinhtho/agentcmd/releases/latest/download/<asset>
https://github.com/phidinhtho/agentcmd/releases/download/<tag>/<asset>
```

The first always points at the newest published release; the second is frozen.
Assets of a published tag are never overwritten — a fix is always a new tag. A
release found to be broken is marked pre-release, which makes `latest` fall back
to the one before it while leaving pinned installs working.

## What you are trusting

Running the install line means trusting TLS to `github.com` and trusting the
owner of this repository — the same shape of trust as any other one-line
installer.

The SHA-256 in `manifest.json` protects you from a truncated download or a CDN
serving the wrong file. It does **not** protect you from someone who takes over
this repository; that would need signed releases, which these are not yet.

The scripts send nothing anywhere. There is no telemetry and no install counter
beyond GitHub's own download numbers. They never touch your pairing code — only
`agentcmd setup` and `agentcmd auth show` print that, on your own terminal.

## Support

Questions and bug reports: <https://agentcmd.app/support>.

Please do not include your server address, your pairing code, or conversation
content in a report.

## License

The `agentcmd` binary and the companion apps are released under the MIT license
— see [LICENSE](LICENSE).
