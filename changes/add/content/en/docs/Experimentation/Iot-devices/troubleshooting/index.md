---
title: "Troubleshooting & FAQ"
linkTitle: "Troubleshooting & FAQ"
weight: 4
description: >
    Common errors with the mrg-iot CLI and the spiot_ctl control client, and how to resolve them.
---

This page collects the issues users hit most often, grouped by where they occur:
setup, login, resource conflicts, connection/tunnels, and device interaction. Each
entry lists the symptom you'll see and the fix.

Run the command with the global `--debug` flag. This writes verbose logs to
`~/.mrg-iot/debug.log`, and the background daemon writes to `~/.mrg-iot/daemon.log`.
These two files contain the underlying SSH, gRPC, and portal errors behind most
generic messages. These can give you a more detailed overview of any issue.

```sh
mrg-iot --debug run
cat ~/.mrg-iot/debug.log
```

## Exit codes

`mrg-iot` returns a distinct exit code per failure class — useful in scripts and CI:

| Code | Meaning |
|---|---|
| `0` | Success |
| `1` | General failure (login failed, resource not found, daemon already running) |
| `2` | Input validation failure (bad name/duration, argument error) |
| `3` | Creation failure (experiment / realization / XDC could not be created) |
| `4` | Readiness/timeout failure (XDC did not become ready in time) |
| `9` | Daemon failed to spawn |
| `130` | Interrupted with Ctrl-C |

---

## Installation & environment

### `mrg-iot: command not found` after install

`pipx` installs to `~/.local/bin`, which may not be on your `PATH`. Run:

```sh
pipx ensurepath
```

then open a new shell. Confirm with `mrg-iot --version`. Requires **Python 3.10+**.

### SSL / certificate errors on macOS

Symptom: portal calls fail with a `certificate verify failed` / SSL error when talking
to `grpc.sphere-testbed.net`. macOS Python often ships without a usable CA bundle.

{{% alert title="macOS SSL workaround" color="info" %}}
```sh
export SSL_CERT_FILE="$(python -m certifi)"
export REQUESTS_CA_BUNDLE="$SSL_CERT_FILE"
```
Add these to your shell profile to make them persistent.
{{% /alert %}}

### "The persistent-connection daemon is not supported on Windows."

The `connect` / `send` / `show video` / `download` daemon commands rely on Unix
features and do not run on Windows. Use `mrg-iot run` (interactive session) instead, or
run the daemon from WSL / Linux / macOS.

### Video viewer not found

The tool opens camera streams in **VLC**. If VLC is not installed or not on your `PATH`
you'll see:

```
VLC was not found; install it or add it to your PATH.
Streams remain available at rtsp://localhost:8554/<device>.
```

Install VLC from your OS package manager (it is **not** a pip package):

| Platform | Command |
|---|---|
| macOS | `brew install --cask vlc` |
| Debian/Ubuntu | `sudo apt-get install vlc` |
| Fedora | `sudo dnf install vlc` |
| Windows | Download from [videolan.org](https://www.videolan.org) |

Confirm with `vlc --version`. The stream stays available regardless — you can open
`rtsp://localhost:8554/<handler>` in any RTSP-capable player while the tunnel is active.

---

## Authentication & sessions

Your session is stored at `~/.mrg-iot/session.json` (mode `0600`).

### "Login failed"

The portal rejected the attempt. Causes, in order of likelihood:
- **Wrong username or password** — re-run `mrg-iot login` and re-enter.
- **Portal unreachable** — check network access to `grpc.sphere-testbed.net:443`.
- **Account/project access** — if login succeeds but you see "No accessible projects",
  your account isn't yet attached to an IoT project; contact the NEU IoT Facility.

### "Already logged in as `<user>`. Run 'mrg-iot logout' first to switch accounts."

A valid session already exists. To switch accounts:

```sh
mrg-iot logout
mrg-iot login
```

### Token expired

Tokens are validated before each command. When one expires, the tool clears the stored
session and re-prompts for credentials automatically — just log in again when asked.

### "No logged-in session found" on logout

You're already logged out; nothing to do.

---

## Resource conflicts & cleanup

This is the most common class of issue: a previous run left an experiment, realization,
materialization, or XDC behind, and a new run collides with it or you simply want to
clean up.

{{% alert title="Resources persist by design" color="warning" %}}
If you interrupt a run with **Ctrl-C** (exit code `130`), or don't pass `--delete-xdc` /
`--delete-exp`, the experiment, realization, materialization, and XDC are **kept** so you
can reconnect and reuse them. They are *not* cleaned up automatically. Delete them
manually (below) when you're truly done, or they count against your quota.
{{% /alert %}}

### An experiment / XDC / realization with that name already exists

You have two options:

1. **Delete it via the CLI** (see the cleanup commands below), then re-run, **or**
2. **Delete it through the portal** at
   [launch.sphere-testbed.net](https://launch.sphere-testbed.net), **or**
3. Simply choose a **different name** on the next run.

### Cleanup commands per resource

Each command needs `--project` plus the relevant resource flags. Run them in this order —
detach/dematerialize/relinquish before deleting the parent:

```sh
# 1. Detach the XDC from the realization (if attached)
mrg-iot xdc detach --project neuiot --name myxdc --experiment myexp --realization realiot

# 2. Delete the XDC
mrg-iot xdc delete --project neuiot --name myxdc

# 3. Tear down the materialization
mrg-iot materialization dematerialize --project neuiot --experiment myexp --realization realiot

# 4. Relinquish the realization
mrg-iot realization relinquish --project neuiot --experiment myexp --name realiot

# 5. Delete the experiment
mrg-iot exp delete --project neuiot --name myexp
```

To see what currently exists before cleaning up:

```sh
mrg-iot exp list --project neuiot
mrg-iot xdc list --project neuiot
mrg-iot realization list --project neuiot --experiment myexp
```

`realization` may be abbreviated `rz`; `materialization` may be abbreviated `mtz`.

### "Failed to create experiment `<name>` ..." from the portal

The portal rejected creation (often a name collision or a transient error). Re-run the
tool; if it persists, delete the conflicting resource (above) or contact a system
administrator. The full portal error is appended to the message and in `debug.log`.

### "Invalid `<field>`: Must start with a letter, lowercase letters and numbers only, no spaces"

Names you choose — **experiment, realization, XDC, and network names** — must match
`^[a-z][a-z0-9]*$`:

- start with a lowercase letter
- lowercase letters and digits only — **no hyphens, underscores, spaces, or capitals**
- max 32 characters

{{% alert title="Note" color="info" %}}
This rule applies to names **you** pick. It does **not** apply to device identifiers like
`s-echodot-1`, which already contain hyphens — those come from the testbed and you pass
them as-is to `--devices`.
{{% /alert %}}

Descriptions allow letters, digits, spaces, commas, periods, and hyphens (max 256 chars).

### "The realization duration `<value>` is invalid ... at least 4 days"

Durations must be **at least 4 days**. Accepted formats include `4d`, `1w`, `1w2d3h`,
or `0w 4d 0h 0m 0s`. Expiry notices go out 3 days before XDC expiry and 1 day before
realization expiry, so 4 days is the practical minimum.

---

## Connection, tunnels & the daemon

The daemon stores its state in `~/.mrg-iot/`: `connection.json` (descriptor),
`connection.sock` (CLI ↔ daemon socket), and `daemon.log` (diagnostics).

### "A connection daemon is already running (pid=`<pid>`). Run 'mrg-iot disconnect' first."

Stop the existing daemon, then reconnect:

```sh
mrg-iot disconnect
mrg-iot connect ...
```

If the daemon crashed and left a stale `connection.json`, the next `connect` clears it
automatically once it sees the PID is dead.

### "No active connection. Run 'mrg-iot connect' first."

You ran `send`, `show video`, or `download` without a live daemon. Start one with
`mrg-iot connect` (or use an interactive `mrg-iot run` session instead).

### "Daemon failed to start; see ~/.mrg-iot/daemon.log for details." (exit 9)

The daemon spawned but exited before it was ready (default wait: 120s). Open
`~/.mrg-iot/daemon.log` — the real cause is almost always one of:

- **SSH could not reach the XDC** (`SSHException`, `Tunnel channel open failed`) — the XDC
  isn't fully ready yet; wait and retry.
- **A local tunnel port is busy** (`Address already in use` on `8554`, `9000`, or `17000`).
  See below.
- **Portal/network error** resolving the experiment.

### "Could not connect after 5 attempts" / "after 10 attempts"

The tool reached the XDC but couldn't reach the **ExperimentControl** gRPC server on
port `17000`. (`mrg-iot ctl` retries 5×; `run`/`connect` retry 10×, 3s apart.) Usually the
experiment is still materializing or just crashed. Confirm the materialization and XDC are
ready, then retry:

```sh
mrg-iot materialization status --project neuiot --experiment myexp --realization realiot
mrg-iot xdc info --project neuiot --name myxdc
```

### Local port already in use (8554 / 9000 / 17000)

A previous session (or another app) is holding the port the tunnel needs to bind locally.
Find and stop it:

```sh
lsof -i :17000      # or :8554 / :9000
kill <pid>
```

`mrg-iot disconnect` will also release ports held by a stale daemon.

### "Tunnel channel open failed" in daemon.log

The XDC is reachable over SSH but the forwarded service isn't up yet — typically the
experiment isn't materialized or the XDC isn't fully ready. Wait, confirm status (above),
and retry `connect`.

---

## Waiting & timeouts

| Operation | Wait before timeout | What it's waiting for |
|---|---|---|
| `xdc create` (without `--no-wait`) | ~5 minutes | XDC status → ready |
| `run` / `connect` | ~30s (10 × 3s) | ExperimentControl reachable on `:17000` |
| `ctl` (inside XDC) | ~15s (5 × 3s) | ExperimentControl over WireGuard |
| `connect` daemon spawn | 120s | Daemon writes `connection.json` |

If `xdc create` times out (exit `4`), the infrastructure is likely busy. Create without
waiting and poll the status yourself:

```sh
mrg-iot xdc create --project neuiot --name myxdc --no-wait
mrg-iot xdc info --project neuiot --name myxdc
```

---

## Device interaction (spiot_ctl)

These appear at the interactive prompt or via `mrg-iot send`.

### "Command: `<cmd>` not found"

The device doesn't support that command, or the device name is wrong. List the device's
actual commands first — capabilities can differ even between two devices of the same model:

```
s-echodot-1 help
s-echodot-1 click_button help
```

### "Invalid action type, specify a device (prefixed with 's-'), exp, or query ..."

Every command must start with a device id (`s-...`), `exp`, or `query`. Check the leading
token — e.g. `s-echodot-1 click_button`, not `echodot-1 click_button`.

### "Insufficient arguments provided, (minimum 2)" / "COMMAND INCORRECT ARGS"

The command needs more arguments. Use `<device> <command> help` to see required
parameters. Note that one device may require a target (e.g. a specific button) where
another accepts none.

### "Variable `<name>` not stored in memory."

You referenced a stored variable (prefixed with `_`) that was never set. Store it first
with the `-s` flag, then reference it with the underscore:

```
s-echodot-1 -s job id click_button
query wait_result s-echodot-1 click_button _job
```

### `query state` / `query get_result` with a bad job ID

The job ID must be a number; a non-numeric id is rejected. Querying a job ID that doesn't
exist returns an empty result. `get_result` returns nothing until the job's state is
`complete` — use `query wait_result` to block until it finishes, or poll `query state`
first.

### "Incorrect number of arguments, expected only one device" / "Failed to get device" (on `exp cred`)

`exp cred` takes exactly one device name, and that device must exist in the experiment.
Run `exp devices` to confirm the exact name.

---

## Video streams

### "Camera ... is unreachable" at session start

The backend probes each camera before proxying it. If a camera is offline or not yet
booted, that feed is skipped with an "unreachable"/"timeout" message. Other streams are
unaffected. Give the device time to finish booting and reconnect, or verify the device
actually has a camera handler (`<device> help`).

### Stream won't play

- Make sure the session/tunnel is active and **VLC** is installed (see *Installation*).
- Open the stream directly: `rtsp://localhost:8554/<handler>` (daemon mode:
  `mrg-iot show video`).
- The handler name comes from the device; check `<device> help` for camera handlers.

---

## Downloading experiment files

### "'`<path>`' already exists, would you like to delete it before downloading"

The destination archive already exists. Answer `y` to overwrite, or specify a different
target:

```sh
mrg-iot download traffic --output /path/to/another.tar.gz
```

### Download fails / "Failed to start file server"

The file server (port `9000`) couldn't serve the experiment's data store — usually the
store name is wrong or the directory no longer exists. Confirm the experiment is still
active and the `9000` tunnel is up, then retry. The archive (`<realization>.<experiment>.<project>.tar.gz`)
always contains a log and a `.pcap`; extract with `tar -xvf <archive>.tar.gz`.

---

## Still stuck?

1. Verify state from the portal at
   [launch.sphere-testbed.net](https://launch.sphere-testbed.net).
2. Re-run with `--debug` and read `~/.mrg-iot/debug.log` and `~/.mrg-iot/daemon.log`.
3. Report the issue (with the relevant log lines) to the NEU IoT Facility — IoT support
   is in beta and feedback is welcomed.
