---
title: "Troubleshooting & FAQ"
linkTitle: "Troubleshooting & FAQ"
weight: 3
description: >
    Common mrg-iot errors, with the symptom, cause, and fix, plus answers to the questions that come up most.
---

Issues are grouped by where they occur: install, login, provisioning, connections,
device commands, streams, and files. If you don't find yours, jump to
[Still stuck?](#still-stuck).

## First: turn on tracing

Tracing is controlled by the environment, so it is a
property of your shell session rather than of one invocation:

```sh
MRG_IOT_TRACE=1 mrg-iot exp status myexp
```

That raises the log level to DEBUG and echoes every record to stderr. To keep your
terminal readable, send the trace to a file instead:

```sh
MRG_IOT_TRACE=1 MRG_IOT_TRACE_FILE=/tmp/mrg.log mrg-iot exp status myexp
```

Logs are **always** written to `~/.mrg-iot/debug.log` regardless (appended across
runs, rotated at 5 MB, 3 kept). Attach that file and the output of `mrg-iot --version`
to any bug report.

## Exit codes

Each code names **what** failed, not where in a sequence it failed. The order
follows the pipeline, so a higher code means the run got further:

| Code | Meaning |
|---|---|
| `0` | Success |
| `1` | General failure: it asked the portal and the answer was no |
| `2` | Input this tool rejected before calling the portal (bad name, duration under 4 days, unresolvable project) |
| `3` | The experiment could not be created |
| `4` | The model could not be built or pushed |
| `5` | Realization failed |
| `6` | Materialization failed |
| `7` | The XDC could not be created or never became ready |
| `8` | Attaching the XDC to the realization failed |
| `9` | The session (SSH / tunnels / ExperimentControl) could not be established |
| `10` | Cleanup failed |
| `130` | Interrupted with Ctrl-C |

---

## Installation & environment

### `mrg-iot: command not found` after install

`pipx` installs to `~/.local/bin`, which may not be on your `PATH`:

```sh
pipx ensurepath
```

Open a new shell and confirm with `mrg-iot --version`. Requires **Python 3.9+**.

### SSL / certificate errors on macOS

Symptom: portal calls fail with `certificate verify failed` when talking to
`grpc.sphere-testbed.net`. macOS Python often ships without a usable CA bundle.

```sh
export SSL_CERT_FILE="$(python -m certifi)"
export REQUESTS_CA_BUNDLE="$SSL_CERT_FILE"
```

Add both lines to your shell profile to make them persistent. `mrg-iot` detects this
case and prints the same two lines when it happens.

### "Could not resolve a testbed hostname"

DNS could not resolve `grpc.sphere-testbed.net` or `jump.sphere-testbed.net`. Check
your network connection, and any VPN or split-DNS configuration.

### Installing the dev tools fails on Python 3.9

The runtime supports 3.9+, but the `dev` extras (black, pylint, isort) require
**3.10+**. On a 3.9 environment install runtime dependencies only: `pip install -e .`
or `pip install -r requirements/prod.txt`.

---

## Authentication & sessions

Your session lives at `~/.mrg-iot/session.json` (mode `0600`).

### "Login failed"

The portal rejected the attempt. In order of likelihood:

- **Wrong username or password**: re-run `mrg-iot login`.
- **Account not yet approved**: a registered identity is not the same as an
  approved portal account. See
  [Research Accounts](https://mergetb.gitlab.io/testbeds/sphere/sphere-docs/docs/experimentation/getting-started/).
- **Portal unreachable**: check access to `grpc.sphere-testbed.net:443`.

### You log in fine, but no projects are listed

Your account isn't a member of an IoT project yet. Request membership in `neuiot`
(or `iotbeta` / `iotdev`) through the portal or from the NEU IoT Facility. See
[Getting Started §1b](https://mergetb.gitlab.io/testbeds/sphere/sphere-docs/docs/experimentation/iot-devices/getting-started/#1b-iot-project-access).

### "Already logged in as `<user>`"

To switch accounts:

```sh
mrg-iot logout
mrg-iot login
```

### A command won't stop asking for credentials in a script

If credentials are needed, none are cached, and stdin is not a terminal, `mrg-iot`
exits with an explanation rather than hanging. Supply them through the environment:

```sh
export MRG_IOT_TOKEN=<token>                 # preferred in shared CI; revocable
# or
export MRG_IOT_USERNAME=<user> MRG_IOT_PASSWORD=<pass>
```

A rejected password fails immediately rather than retrying, since there is nothing
to re-prompt.

---

## Provisioning & resource conflicts

### The name already exists

An earlier run left the experiment, realization, or XDC behind. Either release it:

```sh
mrg-iot exp teardown myexp     # undeploy + delete, in one command
mrg-iot exp undeploy myexp     # release the hardware, keep the experiment record
```

…or pick a different name, or delete it through the portal at
[launch.sphere-testbed.net](https://launch.sphere-testbed.net).

`teardown` and `undeploy` are safe to re-run: a resource that is already gone is
reported as *skipped*, not as a failure, and the command still exits `0`.

Which verb releases what is in the group's own help. `undeploy` stops before
deleting the experiment record, `teardown` doesn't:

![mrg-iot exp --help showing that undeploy detaches, deletes the xdc, materialization and realization, while teardown does all of that and deletes the experiment](exp-help.png#zoomable)

To see what exists first:

```sh
mrg-iot exp list               # your experiments; --all for everyone's
mrg-iot exp status myexp       # created / realized / materialized / deployed
mrg-iot xdc list
```

{{% alert title="Resources persist by design" color="warning" %}}
Interrupting with Ctrl-C (exit `130`), or finishing `mrg-iot run` without
`--delete-xdc` / `--delete-exp`, **keeps** everything so you can reconnect. Nothing
is cleaned up automatically; it holds hardware and counts against your quota until
its duration expires.
{{% /alert %}}

### "Invalid `<field>`: Must start with a letter, lowercase letters and numbers only, no spaces"

Names **you** pick (experiment, realization, XDC, network) must match
`^[a-z][a-z0-9]*$`:

- start with a lowercase letter
- lowercase letters and digits only, with no hyphens, underscores, spaces, or capitals
- max 32 characters

This does **not** apply to device identifiers like `s-echodot-1`; those come from the
testbed and you pass them through as-is to `--devices`.

Descriptions allow letters, digits, spaces, commas, periods, and hyphens (max 256).

### "The duration is invalid … at least 4 days"

The minimum is **4 days**. `4d`, `1w`, `1w2d3h`, and `1 week` are all accepted; the
default is `1w`. Expiry notices go out 3 days before XDC expiry and 1 day before
realization expiry, so 4 days is also the practical minimum for getting any warning.

### `exp create -d d1 d2 myexp` reserves three devices and no experiment name

`--devices` is greedy, so it swallows the name. Put `NAME` **before** the flag:

```sh
mrg-iot exp create myexp -d d1 d2
```

Leaving `--devices` off entirely is the easier mistake to spot, because argparse
refuses it outright rather than building a model from nothing:

![mrg-iot exp create myexp printing 'error: the following arguments are required: --devices/-d' above the usage line](exp-create-nodevices.png#zoomable)

### The XDC never becomes ready (exit `7`)

`xdc create` waits up to ~5 minutes. If the infrastructure is busy, create without
waiting and poll it yourself:

```sh
mrg-iot xdc create --name myxdc --no-wait
mrg-iot xdc status --name myxdc
```

### A named default points at nothing

Errors of the form "no such experiment `jsmith`" usually mean a *derived* default was
used: the experiment defaults to your username, the XDC to `<experiment>xdc`, and the
realization to `realiot`. The message names what it tried and which flag overrides it.

---

## Connections, sessions & tunnels

{{% alert title="There is no daemon any more" color="info" %}}
`mrg-iot connect`, `disconnect`, `send`, `show video`, and `download traffic` have
been **removed**. Each command now opens its own SSH session for as long as it runs
and closes it afterwards. If an older version left `connection.json`,
`connection.sock`, or `daemon.log` in `~/.mrg-iot/`, they are dead files and safe to
delete.

| Removed | Use instead |
|---|---|
| `mrg-iot send …` / `mrg-iot cmd send …` | `mrg-iot cmd run …` (or `cmd async-run` for the old don't-wait behaviour) |
| `mrg-iot ctl` | `mrg-iot cmd ctl` |
| `mrg-iot show video` | `mrg-iot stream open` |
| `mrg-iot download traffic` | `mrg-iot file download` |
| `mrg-iot connect` / `disconnect` | `mrg-iot cmd shell`, or just run each command on its own |
{{% /alert %}}

If a script or a habit still reaches for one of those, `mrg-iot --help` is the
current surface. Anything not in it no longer resolves:

![The mrg-iot help output, listing the cmd, devices, exp, file, login, run, and stream command groups](overview.png#zoomable)

### "SSH to the xdc failed"

The XDC is not reachable. Confirm the experiment is actually deployed and the XDC is
attached, then retry:

```sh
mrg-iot exp status myexp
```

If it says `materialized` rather than `deployed`, the XDC isn't attached yet.

### "Could not connect after N attempts"

`mrg-iot` reached the XDC but not the **ExperimentControl** server on port `17000`,
usually because the experiment is still materializing, or it crashed. Check
`exp status`, wait, and retry.

### A local port is already in use

This is no longer a thing to fix. The local end of every tunnel is an OS-assigned
ephemeral port, which is also why you can run `stream open` in one terminal,
`cmd shell` in another, and `file download` in a third without collisions. Only the
*remote* ports are fixed (`8554`, `9001`, `17000`).

### A command hangs, then times out

Re-run with `MRG_IOT_TRACE=1` and watch stderr, or read `~/.mrg-iot/debug.log`, for
the underlying gRPC or SSH error. Device commands are given up to 5 minutes.

### "The portal rejected a request"

A gRPC error came back from the portal. The message includes the portal's own text.
If it persists, contact a testbed administrator with the relevant lines from
`debug.log`.

---

## Device commands

These come back from the experiment, not from `mrg-iot`, because the experiment
validates its own input.

### "Invalid action type, specify a device (prefixed with 's-'), exp, or query"

Every command must start with a device id (`s-…`), `exp`, or `query`. Check the
leading token: `s-echodot-1 click_button`, not `echodot-1 click_button`.

{{% alert title="The old `dev <device> <command>` form is gone" color="info" %}}
The experiment now answers it with "Invalid action type". `mrg-iot` rewrites that
spelling for you, but anything sending raw commands another way needs updating to
`<device> <command>`.
{{% /alert %}}

### "Command: `<cmd>` not found"

The device doesn't support that command, or the device name is wrong. Capabilities
differ even between two devices of the same model, so ask the device:

```sh
mrg-iot cmd run "s-echodot-1 commands" -e myexp
mrg-iot cmd run "s-echodot-1 click_button help" -e myexp
```

### "Insufficient arguments provided, (minimum 2)"

A command needs at least a category and a name: `exp devices`, not `devices`. For
device commands, `<device> <command> help` shows the required parameters.

### "Variable `<name>` not stored in memory"

You referenced a variable (leading `_`) that was never set. Store it first. The `-s`
flag takes a **reference** and then a **variable name**, where the reference is `id`,
`result`, or `state`:

```
s-echodot-1 -s id job click_button
query wait_result s-echodot-1 click_button _job
```

### `query` returns nothing

`query` commands take exactly three arguments and the job id must be digits.
`get_result` returns nothing until the job's state is `complete`, so use
`query wait_result` to block until it finishes, or poll `query state` first. In
practice you rarely type these: `cmd run` sends `wait_result` for you, and
`cmd wait` / `cmd status` are the same verbs addressed by task handle.

### A task handle stopped working

Handles are forgotten when their experiment is torn down (`exp undeploy`,
`exp teardown`, `exp delete`), because a redeployed experiment starts counting job
ids from `0` again and a surviving handle would ask about one job and be answered
about another. A handle that is no longer on record fails immediately and locally;
no session is opened to find that out:

![mrg-iot cmd wait t99 printing 'No task t99. `mrg-iot cmd tasks` lists the ones on record.'](cmd-wait-notask.png#zoomable)

`mrg-iot cmd tasks` lists what is still on record.

### `cmd tasks` shows no state for each job

By design, since showing it would cost one round trip per row.
`mrg-iot cmd status t1` is the command that asks.

---

## Camera streams

### `stream open` does nothing, or errors

VLC must be installed and on your `PATH`. Confirm with `vlc --version`; `pip` and
`pipx` cannot install it.

| Platform | Command |
|---|---|
| macOS | `brew install --cask vlc` |
| Debian / Ubuntu | `sudo apt-get install vlc` |
| Fedora | `sudo dnf install vlc` |
| Windows | [Download from videolan.org](https://www.videolan.org/vlc/) |

To use a different player, `mrg-iot stream url` prints the RTSP addresses and keeps
forwarding them until Ctrl+C.

### VLC connects but shows nothing

Check for an editor forwarding the same port. VS Code's automatic port forwarding
binds `127.0.0.1:<port>` and shadows the tunnel for anything connecting to
`localhost`. Remove the entry from its **Ports** panel, or set
`"remote.autoForwardPorts": false`.

### `stream list` prints `<port>` instead of a number

That is the literal output. The port is assigned per session, so there is no number
to print until a session exists. Run `stream url` for the real addresses, or
`stream open` to watch them.

### The stream closed when the command exited

Expected. A stream is continuous, so the command has to keep running to hold its
tunnel open. Closing the command closes the feed.

### "Camera … is unreachable"

The backend probes each camera before proxying it; one that is offline or still
booting is skipped and the others are unaffected. Give the device time, or confirm
it actually has a camera. `mrg-iot devices info` pairs each device with its camera,
and `mrg-iot stream list` names them.

---

## Files

### `file download` can't find your file

Names are the file server's, not your shell's: no globbing, and a path is relative to
the share's root, never to your machine. List it first and use the name printed:

```sh
mrg-iot file list -e myexp
mrg-iot file list output -e myexp
```

### The downloaded file has a strange name

That is the naming scheme: `<experiment>.<remote path>`, with every `/` turned into a
`.`, so `output/capture.pcap` from experiment `myexp` arrives as
`~/Downloads/myexp.output.capture.pcap`. It keeps the experiment and the source
directory in the name so two files with the same basename don't overwrite each other.
Pass `--output`/`-o` for a name of your own.

### `file upload` put the file somewhere unexpected

Uploads always go to `uploads/`, because the share's root is read-only for the
experiment's file-server user. `upload` and `remove` work in the root only.

### `file download` with no name gives me a `.zip`

Also expected: `output` can only come back through the web client's zip endpoint, so
it lands as `<experiment>.output.zip`. Name a single file to get it verbatim.

### Nothing was captured

The output archive always contains at least a log file and a `.pcap`. Anything else
depends on the commands you ran. If the archive is missing entirely, confirm the
experiment was materialized long enough for capture to start.

---

## FAQ

**Do I need an SSH key, or the `mrg` CLI?**
No. `mrg-iot` talks to the portal itself and fetches the SSH key it needs for the XDC
automatically.

**Can I drive one experiment from several terminals?**
Yes. Sessions are cheap and independent, because every local tunnel port is
ephemeral. Run
`stream open` in one, `cmd shell` in another, and `file download` in a third.

**Does anything run in the background between commands?**
No. Each command opens its session, works, and closes. The only thing kept on disk is
your login, the debug log, and task records from `cmd async-run`.

**How do I script this without any prompts?**
Everything except `mrg-iot run` is already non-interactive: anything you omit is
derived rather than asked for. For `run` itself, pass `--non-interactive`.

**Why did my experiment name default to my username?**
That's the rule: `--experiment` defaults to your username, `--xdc` to
`<experiment>xdc`, and `--realization` to `realiot`.

**What is the minimum reservation?**
4 days. The default is 1 week.

**Which projects have IoT hardware?**
`neuiot` (production), `iotbeta` (pre-release), `iotdev` (facility development).

---

## Still stuck?

1. Check the experiment's state from the portal at
   [launch.sphere-testbed.net](https://launch.sphere-testbed.net), or with
   `mrg-iot exp status <name>`.
2. Re-run with `MRG_IOT_TRACE=1` and read `~/.mrg-iot/debug.log`.
3. Report the issue to the NEU IoT Facility with those log lines and the output of
   `mrg-iot --version`. IoT support is in beta and feedback is welcomed.
