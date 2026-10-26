# Jerry Deluge for Feral Hosting (Rootless / Shared Slot)

A dedicated, **separate Deluge 1.3.15 instance** for Feral Hosting shared slots. This helper is designed for accounts that do not have root access, `sudo`, APT, or system-wide systemd.

It builds a private runtime containing Python 2.7.18, OpenSSL 1.1.1w, patched libtorrent 1.1.14, and Deluge 1.3.15 under your home directory. It then runs a separate daemon and WebUI with private configuration, credentials, ports, and download folders.

> **Important:** This is not Feral's official Deluge installer. Install Deluge 1.x through Feral's own software manager first. The helper reads the provider configuration as a starting point, but does not replace or edit the provider-managed Deluge runtime, `~/.config/deluge`, its processes, or its data. Always run this helper as your normal Feral SSH user—never as root or with `sudo`.

## Contents

- [What it does](#what-it-does)
- [Requirements and limitations](#requirements-and-limitations)
- [Install](#install)
- [Open the WebUI and connect to the daemon](#open-the-webui-and-connect-to-the-daemon)
- [Commands](#commands)
- [Ports and paths](#ports-and-paths)
- [Update the helper](#update-the-helper)
- [Troubleshooting](#troubleshooting)
- [Uninstall](#uninstall)
- [Security notes](#security-notes)

## What it does

- Builds a private Deluge 1.3.15 runtime without root, APT, or systemd.
- Uses Python 2.7.18, OpenSSL 1.1.1w, and patched libtorrent 1.1.14.
- Starts a separate `deluged` process and a separate Deluge WebUI.
- Uses a separate RPC port, incoming BitTorrent port, and WebUI port.
- Keeps the configuration, authentication file, logs, and download directories under your home directory.
- Synchronizes the private Deluge 1.3.15 Connection Manager hostlist with the configured RPC port when starting. This helps repair stale entries left over from a previous port.
- Does not automatically edit Feral's reverse-proxy configuration. A proxy URL works only if that route is configured for your slot.

## Requirements and limitations

1. A Feral shared slot with SSH access.
2. Deluge 1.x installed using Feral's software manager, so `~/.config/deluge/core.conf` exists.
3. A complete clone of this repository. The helper depends on the runtime builder, source/patch lock files, patch, and runtime validation script; downloading only `bin/feral-deluge` is not sufficient.
4. A working compiler/toolchain and the required development headers available to your account. The build can fail if the provider restricts compilers, build processes, or development headers.
5. Enough quota, free storage, memory, and build time to compile the private runtime.

The script does not run `sudo`, `apt`, or systemd commands. It cannot install missing system packages or bypass Feral's restrictions. The install may not work on every shared slot.

Deluge 1.3.15 and Python 2.7 are legacy, end-of-life software. Use a strong password and expose the WebUI only through a trusted, authenticated route.

## Install

### 1. Install provider Deluge first

In the Feral Hosting web manager, install Deluge 1.x if it is not already installed. Do not remove it after setting up this helper; the helper uses its configuration as a starting point.

Check that the provider config exists:

```bash
ls -l ~/.config/deluge/core.conf
```

Do not copy Jerry configuration over `~/.config/deluge`.

### 2. Clone the full repository

Run these commands as your regular Feral user:

```bash
cd ~
git clone https://github.com/race-in/jerry-custom-setup.git
cd ~/jerry-custom-setup
```

If you already cloned it, update it using the [update instructions](#update-the-helper).

### 3. Install Jerry's private instance

Recommended starting command for a shared slot:

```bash
bash bin/feral-deluge install --jobs 1
```

The helper checks the provider configuration, chooses free ports, builds the private runtime if it is not already installed, creates Jerry's separate profile, and starts the private daemon and WebUI. Compilation may take a while.

You can optionally choose ports yourself:

```bash
bash bin/feral-deluge install \
  --jobs 1 \
  --rpc-port 28602 \
  --incoming-port 45000 \
  --web-port 36876
```

Choose ports that are available on your slot. All three ports must be different; the RPC port must also differ from the provider Deluge RPC port. The script checks whether the selected ports can be bound at install time. Do not copy the example port numbers blindly if they are already in use.

The installer prints the WebUI password and RPC credentials at the end. **Save them in a password manager immediately.** The WebUI password is not printed again by the normal `connection` command.

### 4. Verify the instance

```bash
bash bin/feral-deluge status
bash bin/feral-deluge connection
```

The status command should report Jerry's daemon and WebUI as running. The connection command prints the WebUI address, RPC port, RPC username, and RPC password. Treat that output as secret; do not post it in public logs or screenshots.

## Open the WebUI and connect to the daemon

### WebUI address

The installer prints a direct address similar to:

```text
http://<feral-hostname>:<web-port>/
```

That address is only usable if the provider permits access to that port from your client. The helper does **not** create a public reverse proxy or open provider firewalls.

If your Feral slot has a configured reverse-proxy route, use the URL supplied by that setup. For example, a route may look like:

```text
https://<feral-hostname>/<your-username>/deluge2/
```

The exact hostname and path depend on your account and proxy configuration. Do not assume that the example route exists on every slot. If the direct port is inaccessible, use an already configured trusted proxy or ask Feral support which access method is supported.

Log in with the **WebUI password printed during installation**.

### Connection Manager / daemon connection

The WebUI and daemon are separate processes. The daemon RPC endpoint is intended to be local-only for normal use.

1. Run `bash bin/feral-deluge connection` in SSH.
2. Open the WebUI's Connection Manager.
3. If needed, add or select host `127.0.0.1:<RPC-port>` (for example, `127.0.0.1:28602`).
4. Use the RPC username and RPC password printed by the command.
5. Connect to that host.

The helper synchronizes its private `hostlist.conf.1.2` and `web.conf` with the RPC port during startup. If a stale host entry is present, start/restart should repair the private configuration and restart the private WebUI when necessary.

## Commands

Run commands from the repository directory, as your normal Feral user:

| Command | Purpose |
| --- | --- |
| `bash bin/feral-deluge help` | Show the command synopsis. |
| `bash bin/feral-deluge install --jobs 1` | Build/install the private runtime and profile, or start an already-managed profile. |
| `bash bin/feral-deluge status` | Show whether Jerry's private daemon and WebUI are running. |
| `bash bin/feral-deluge connection` | Show the WebUI address and RPC connection details, including the RPC password. Keep the output private. |
| `bash bin/feral-deluge start` | Start missing private processes; also sync the private RPC host configuration. |
| `bash bin/feral-deluge stop` | Stop only Jerry's private WebUI and daemon. |
| `bash bin/feral-deluge restart` | Restart only Jerry's private WebUI and daemon. |
| `bash bin/feral-deluge reset-password` | Interactively set a new Jerry WebUI password. |
| `bash bin/feral-deluge uninstall` | Remove Jerry's private profile and stop its processes, while keeping download data. |
| `bash bin/feral-deluge uninstall --purge-data --yes` | Remove the private profile and permanently delete the dedicated Jerry data directory. |

The helper does not manage the provider's Deluge process. Do not use broad commands such as `pkill -f deluged` on a shared host; they can affect other instances.

### Change ports

Ports are selected automatically when not supplied:

- RPC: a free port in the range 20000–32000.
- Incoming BitTorrent: a free port in the range 40000–60000.
- WebUI: a free port in the range 32001–39000.

These are search ranges, not guarantees that every port is externally reachable. Incoming connectivity depends on Feral's network rules. The RPC endpoint should remain local; do not expose it publicly.

If you need to change ports after installation, back up the private configuration first and review the helper's port validation and profile behavior. Do not change only one config value manually: the daemon, WebUI, and Connection Manager hostlist must agree.

## Ports and paths

| Item | Default location / behavior |
| --- | --- |
| Repository | `~/jerry-custom-setup` |
| Private runtime | `~/.local/opt/deluge-1.3.15-jerry` |
| Private configuration | `~/.config/deluge-jerry` |
| Download data | `~/private/deluge-jerry/data` |
| Watch folder | `~/private/deluge-jerry/watch` |
| Torrent files | `~/private/deluge-jerry/torrents` |
| Completed folder | `~/private/deluge-jerry/completed` |
| Daemon log | `~/.config/deluge-jerry/deluged.log` |
| WebUI log | `~/.config/deluge-jerry/deluge-web.log` |
| WebUI launcher log | `~/.config/deluge-jerry/deluge-web-launch.log` |
| RPC authentication | `~/.config/deluge-jerry/auth` |
| WebUI configuration | `~/.config/deluge-jerry/web.conf` |
| Daemon configuration | `~/.config/deluge-jerry/core.conf` |
| Connection Manager host list | `~/.config/deluge-jerry/hostlist.conf.1.2` |

Configuration and credential files are private to your Unix account. Do not publish their contents; the `auth` file contains RPC credentials.

## Update the helper

From your existing clone:

```bash
cd ~/jerry-custom-setup
git status --short
git pull --ff-only origin main
bash -n bin/feral-deluge
```

If `git pull` refuses because `bin/feral-deluge` has local changes, do not immediately run `git reset --hard`. Inspect and back up the local file first. On older Git versions, `git stash push` may not be supported; `git stash save "message"` is the older syntax, but a file backup is still recommended.

Then restart Jerry's private instance:

```bash
bash bin/feral-deluge restart
bash bin/feral-deluge status
```

The private compiled runtime is reused when it is already installed. The helper's startup routine synchronizes the private WebUI/hostlist RPC settings.

## Troubleshooting

### WebUI opens but says disconnected

Check the private daemon and WebUI:

```bash
bash bin/feral-deluge status
ss -lnt 2>/dev/null | grep -E ':(<RPC_PORT>|<WEB_PORT>)\\b'
tail -n 40 ~/.config/deluge-jerry/deluged.log
tail -n 40 ~/.config/deluge-jerry/deluge-web.log
```

Replace `<RPC_PORT>` and `<WEB_PORT>` with the ports shown by `bash bin/feral-deluge connection` and `status`.

Confirm that `core.conf`'s `daemon_port`, `web.conf`'s `default_daemon`, `web.conf`'s `daemon` list, and `hostlist.conf.1.2` all point to the same RPC port. The hostlist entry should normally use `127.0.0.1`, not the provider's RPC port.

Try:

```bash
bash bin/feral-deluge restart
tail -n 40 ~/.config/deluge-jerry/deluge-web.log
```

A successful connection is logged as `Connected to daemon at 127.0.0.1:<RPC_PORT>`. An old warning for a stale port at the beginning of a log may be from a previous startup; inspect the newest timestamps after restart.

### WebUI gives HTTP 404 on the proxy path

The direct backend and reverse-proxy route are different. Test the direct backend root using the actual WebUI port:

```bash
WEB_PORT="$(grep -m1 '"port"' ~/.config/deluge-jerry/web.conf | sed 's/[^0-9]//g')"
curl -sS -o /dev/null -w 'HTTP %{http_code}\\n' "http://127.0.0.1:${WEB_PORT}/"
```

A direct backend root response of HTTP 200 confirms the WebUI HTTP server is responding. Requesting the proxy prefix (such as `/<username>/deluge2/`) directly on the backend may return 404; the reverse proxy is responsible for routing that prefix. If the public URL fails, check the proxy route and the base path used by the WebUI.

### Port already in use

Choose a different port with the install options, ensuring the three ports differ and the RPC port differs from provider Deluge's RPC port. If a profile already exists, do not rerun a fresh install over its configuration without reviewing the current state.

### Runtime build fails

Read the first failing build step and the relevant output. Confirm available quota/storage and whether the slot permits compilation and provides development headers. The script intentionally does not use sudo/APT to install packages. If the host restricts compilation, contact Feral support; repeatedly rerunning the build will not remove a provider restriction.

### Lost the WebUI password

Run:

```bash
bash bin/feral-deluge reset-password
```

Enter and confirm a new password when prompted. This changes Jerry's WebUI password only; it does not reset the provider Deluge password or the RPC password.

### Gather diagnostics

```bash
bash bin/feral-deluge status
bash bin/feral-deluge connection
tail -n 50 ~/.config/deluge-jerry/deluged.log
tail -n 50 ~/.config/deluge-jerry/deluge-web.log
```

Before sharing output publicly, redact the RPC password, WebUI credentials, host/account details, and any private paths you do not want to disclose.

## Uninstall

To remove Jerry's private config and stop its processes while **keeping downloaded data**:

```bash
bash bin/feral-deluge uninstall
```

To also delete the dedicated Jerry data directory permanently:

```bash
bash bin/feral-deluge uninstall --purge-data --yes
```

The purge command deletes `~/private/deluge-jerry` including the helper's data, watch, torrent, and completed folders. Back up anything you need first. The private compiled runtime under `~/.local/opt/deluge-1.3.15-jerry` is retained by the helper so it can be reused or removed manually later. Uninstall does not remove or change Feral's provider-managed Deluge.

## Security notes

- Never run this helper with `sudo` or as root.
- Keep the WebUI password and RPC credentials private.
- Avoid exposing the RPC port to the internet.
- Use only a trusted, authenticated WebUI proxy or the provider-supported access method.
- Deluge 1.3.15 and Python 2.7 are end-of-life legacy software; limit exposure and keep the private instance isolated.
