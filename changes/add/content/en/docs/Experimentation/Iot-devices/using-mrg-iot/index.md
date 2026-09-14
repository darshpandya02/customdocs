---
title: "Using mrg-iot"
linkTitle: "Using mrg-iot"
weight: 3
description: >
    The complete workflow: account setup, installation, running an experiment, interacting with devices, collecting the data, and releasing the hardware.
---

This guide takes you end to end: account, tool, experiment, interaction, data, cleanup.
If you already have an account and just want the commands, use the
[Quickstart](https://mergetb.gitlab.io/testbeds/sphere/sphere-docs/docs/experimentation/iot-devices/quickstart/) instead.

{{% alert title="Beta" color="warning" %}}
IoT device support is currently in **beta**. Report issues and share feedback with the NEU IoT Facility.
{{% /alert %}}

Everything below is driven by `mrg-iot`, which runs on your own machine. Each
command that talks to a running experiment opens its own SSH session to the XDC,
forwards the ports it needs, does its work, and closes again. There is no daemon
and nothing cached between invocations, so any command here can be scripted
exactly as written.

---

## 1. Create a SPHERE account

### 1a. SPHERE portal account

IoT devices are reserved through the SPHERE portal, so you need a portal account
before anything else. The process (registering an identity, choosing a username,
and having an administrator approve the account) is the same as for any other
SPHERE experiment and is covered in full here:

> **[Research Accounts →](https://mergetb.gitlab.io/testbeds/sphere/sphere-docs/docs/experimentation/getting-started/)**

In short: sign up at [launch.sphere-testbed.net](https://launch.sphere-testbed.net)
(or with `mrg register` from the CLI), wait for approval, and note the **username**
and **password** you chose. `mrg-iot` logs in with exactly those credentials.

{{% alert title="Note" color="info" %}}
You do **not** need to install the `mrg` CLI or set up an SSH key to use IoT devices.
`mrg-iot` talks to the portal itself and fetches the SSH key it needs for the XDC
automatically.
{{% /alert %}}

### 1b. IoT project access

An approved portal account is not enough on its own. Your account also has to be a
member of an **IoT project**. These are the projects that own the IoT hardware:

| Project | Use |
|---|---|
| `neuiot` | The production IoT inventory at Northeastern. |
| `iotbeta` | Pre-release devices and features. |
| `iotdev` | Facility development and testing. |

Most researchers want `neuiot`. Request membership through the portal at
[launch.sphere-testbed.net](https://launch.sphere-testbed.net), or ask the NEU IoT
Facility to add you. Once you are a member, this command lists devices instead of
an error:

```sh
mrg-iot devices list --available --project neuiot
```

{{% alert title="If you belong to exactly one project" color="info" %}}
`--project` is optional, because `mrg-iot` uses your only project automatically. It is
required only when you belong to several, and the error message lists which ones
you can pick from.
{{% /alert %}}

---

## 2. Log in

{{% alert title="Install the tool first" color="info" %}}
This guide assumes `mrg-iot` is already on your `PATH`. If it isn't, see
[Installation](https://mergetb.gitlab.io/testbeds/sphere/sphere-docs/docs/experimentation/iot-devices/installation/), then come back here.
{{% /alert %}}

```sh
mrg-iot login                                  # prompts for username and password
mrg-iot login -u myuser                        # prompts for the password only
mrg-iot login -u myuser --password-stdin < f   # reads the password from stdin
mrg-iot logout                                 # clears the stored session
```

![mrg-iot login --help showing the --username/-u and --password-stdin options](login-help.png#zoomable)

The session is cached at `~/.mrg-iot/session.json` (mode `0600`). Every
authenticated command re-validates the stored token and only re-prompts if the
portal *rejects* it, so you rarely log in more than once. A brief network outage
does not cost you the session: if the portal simply can't be reached, the cached
token is kept.

### Unattended login (scripts and CI)

Set credentials in the environment instead, and skip `login` entirely:

```sh
# Either an existing bearer token...
export MRG_IOT_TOKEN=<token>
export MRG_IOT_USERNAME=<username>   # optional; used for display and SSH key lookup

# ...or a username and password, exchanged for a token normally.
export MRG_IOT_USERNAME=<username>
export MRG_IOT_PASSWORD=<password>
```

`MRG_IOT_TOKEN` takes precedence and skips the login call. `MRG_IOT_USERNAME` and
`MRG_IOT_PASSWORD` must both be set to take effect.

{{% alert title="There is no --password flag" color="warning" %}}
Deliberately so: on Linux any user can read another process's arguments from
`/proc/<pid>/cmdline`, and on every platform the command lands in shell history and
CI logs. Use `--password-stdin` or `MRG_IOT_TOKEN`.
{{% /alert %}}

---

## 3. Run an experiment

### 3a. Choose your devices: `mrg-iot devices`

The `devices` group answers two questions: what is free to reserve, and what is in
an experiment you already have.

![mrg-iot devices --help showing the list and info subcommands](devices-help.png#zoomable)

Before you can reserve anything you need device names. `devices list --available`
asks the portal which devices no experiment currently holds:

```sh
mrg-iot devices list --available
```

![mrg-iot devices list --available printing the project and device name for each free device](devices-available.png#zoomable)

Device names are of the form `<prefix>-<model>-<n>`, e.g. `s-echodot-1`. This is the
one device command that opens no session. It is pure inventory, and it hides devices
that are already allocated, including the ones your own experiment holds.

The prefix says what kind of thing the name refers to:

| Prefix | Meaning |
|---|---|
| `s-` | A device under test, e.g. `s-echodot-1`. These are what you reserve and send commands to. |
| `m-` | A **helper phone**, e.g. `m-googlepixel-1`: a handset used to drive a device's companion app. These are being brought online and will start appearing alongside the `s-` names. |
| `h-` | A **handler**, e.g. `h-camera-2`: the testbed hardware that acts on a device or watches it. You never address these directly; cameras are named by [`mrg-iot stream list`](#camera-streams-mrg-iot-stream) instead. |

`mrg-iot` already reads both `s-` and `m-` names out of the experiment's banner, so
`devices list` and `devices info` will report helper phones as soon as they are in
your experiment. Only `s-` names are routed for commands today, and addressing an
`m-` name returns a message saying so rather than failing silently.

Once an experiment is running, the same command without `--available` names the
devices *that experiment* is about:

```sh
mrg-iot devices list --experiment myexp
```

![mrg-iot devices list showing the devices in a running experiment](devices-list.png#zoomable)

And `devices info` prints what the experiment publishes about each one: VLAN, MAC,
addresses, and the camera watching it, if any.

```sh
mrg-iot devices info -e myexp                       # every device
mrg-iot devices info -e myexp -d s-echodot-1        # just one
```

![mrg-iot devices info listing VLAN, MAC, device IP, gateway, and camera per device](devices-info.png#zoomable)

### 3b. Run the experiment: `mrg-iot exp`

The `exp` group owns the whole lifecycle. Its verbs pair up: `create`/`delete`,
`deploy`/`undeploy`, and `setup`/`teardown` for both halves at once.

![mrg-iot exp --help showing the list, status, create, setup, deploy, undeploy, teardown, and delete subcommands with what each one does](exp-help.png#zoomable)

`exp setup` does the whole provisioning sequence in one command: it creates the
experiment and its network model, realizes it (allocates the hardware),
materializes it (brings the hardware online), creates an XDC, and attaches the two.

```sh
mrg-iot exp setup myexp --devices s-echodot-1 s-googlenest-1
```

![mrg-iot exp setup reporting the project, experiment, realization, and XDC it provisioned](exp-setup.png#zoomable)

Check where it got to at any time:

```sh
mrg-iot exp status myexp
```

![mrg-iot exp status showing an experiment in the deployed state](exp-status.png#zoomable)

The state advances **created → realized → materialized → deployed**. `deployed` is
the one you can send commands in.

#### What you can leave out

Only two things are ever required: the project (and only if you belong to more
than one) and, on the commands that build a model, the device list. Everything else
is derived rather than prompted for.

| Argument | Default |
|---|---|
| `--project`, `-p` | Your only project. Required when you have several. |
| `--devices`, `-d` | No default. Required by `exp create` and `exp setup`, which build the model from it. |
| `--experiment`, `-e` / `NAME` | **Your username**, so user `jsmith` gets experiment `jsmith`. |
| `--xdc`, `-x` | **`<experiment>xdc`**, so `exp setup myexp` provisions `myexpxdc`. |
| `--realization`, `-r` | **`realiot`** |
| `--duration`, `-dur` / `--xdc-duration`, `-xdur` | **`1w`** (minimum `4d`) |
| `--network`, `-net` | **`mrg-iot-net`** |
| `--description`, `-desc` | `Experiment created with mrg-iot by <username>` |

So the shortest useful invocation is:

```sh
mrg-iot exp setup --devices s-echodot-1
```

Every one of those defaults is still overridable by its flag. `exp setup --help`
lists them with the value each falls back to:

![mrg-iot exp setup --help listing NAME, --project, --devices, --description, --network, --realization, --duration, --xdc, and --xdc-duration with their defaults](exp-setup-help.png#zoomable)

If a derived name points at something that doesn't exist, the error says which name
it tried and which flag overrides it. The command never stops to ask.

{{% alert title="Naming and duration rules" color="warning" %}}
Names you choose (experiment, realization, XDC, network) must match `^[a-z][a-z0-9]*$`:
start with a lowercase letter, lowercase letters and digits only, **no hyphens,
underscores, spaces, or capitals**, max 32 characters. This does *not* apply to
device identifiers like `s-echodot-1`; those come from the testbed and you pass them
through as-is.

Durations must be **at least 4 days**. `1w`, `4d`, `1w2d3h`, and `1 week` are all
accepted; anything shorter is rejected with exit code `2`.
{{% /alert %}}

#### Step by step, if you prefer

`exp setup` is `exp create` plus `exp deploy`. Splitting them is useful when the
experiment should outlive your terminal, or when several people share one enclave:

```sh
mrg-iot exp create myexp --devices s-echodot-1   # experiment + model; nothing allocated
mrg-iot exp deploy myexp                         # realize → materialize → xdc → attach
mrg-iot exp list                                 # yours; --all for everyone's
```

![mrg-iot exp list showing a table of experiments with their project, name, creator, and description](exp-list.png#zoomable)

{{% alert title="NAME goes before --devices" color="info" %}}
`--devices` is greedy, so `exp create -d d1 d2 myexp` reads as three devices and no
name. Write `exp create myexp -d d1 d2`. The flag takes either spelling, `-d d1 d2`
or `-d d1,d2`, and drops duplicates.
{{% /alert %}}

Splitting them also means the hardware and the experiment record can be released
separately later. See [§6](#6-stop-the-experiment-mrg-iot-exp).

---

## 4. Interact with the devices

### 4a. Choose an interaction

Commands to devices are written in the `spiot_ctl` control language: a device name,
a command, and its arguments. Capabilities differ between devices, even between two
of the same model, so ask the device itself rather than assuming:

```sh
mrg-iot cmd run "s-echodot-1 commands" -e myexp        # what this device supports
mrg-iot cmd run "s-echodot-1 click_button help" -e myexp   # how one command is used
```

To browse what the hardware can do before you have an experiment running, use the
device catalog at
**[devices.iot.sphere-testbed.net](https://devices.iot.sphere-testbed.net)**, which
lists every device in the testbed along with its details.

The experiment-level commands are the same everywhere:

| Command | Description |
|---|---|
| `exp devices` | List the devices in the experiment. |
| `exp cred <device>` | Get a device's application credentials. |
| `exp sleep <seconds>` | Pause for N seconds. |
| `exp read <file>` | Read a command file on the XDC and queue every line in it. |
| `exp clear <variable>` | Forget a stored variable. |
| `<device> help` / `<device> commands` | The commands that device supports. |
| `<device> <command> help` | Detailed usage for one device command. |
| `query state <device> <command> <id>` | Where a queued job has got to. |
| `query get_result <device> <command> <id>` | The result of a finished job. |
| `query wait_result <device> <command> <id>` | Block until a job finishes, then return its result. |

A sample of what device commands look like across the inventory. The exact set
depends on the device:

| Device kind | Commands |
|---|---|
| Smart plugs | `power_on`, `power_off`, `get_power_consumption`, `power_by_timing` |
| Button pushers | `click_button [timeout]`, `hold_button <duration>` |
| Speakers | `play_tts "<text>"`, `play_audio "<path>"`, `start_capture`, `end_capture` |
| TV / streaming | `remote_press <button>`, `atv_remote <button>`, `roku_remote <button>` |
| Android devices | `tap <x> <y>`, `swipe <direction>`, `type "<text>"`, `home`, `adb_shell <cmd>` |
| IR blasters | `send_ir <button>`, `hold_ir <button> <duration>`, `ir_cmds` |
| Robot arms / plotters | `move_to <x> <y>`, `press_button <position>`, `turn_dial <dial> <degrees>`, `list_positions` |

{{% alert title="The CLI does not pre-check device commands" color="info" %}}
The experiment validates its own input, so a missing or out-of-range argument comes
back from the experiment rather than from `mrg-iot`. (`mrg-iot` still validates its
*own* arguments: experiment names, durations, and so on.)
{{% /alert %}}

### 4b. Execute the interaction: `mrg-iot cmd`

The `cmd` group has one subcommand per way of waiting: `run` waits for the result,
`async-run` doesn't and leaves you a handle, `wait`/`status` pick that handle up
later, and `shell`/`ctl` give you a prompt.

![mrg-iot cmd --help showing the run, async-run, wait, status, tasks, shell, and ctl subcommands](cmd-help.png#zoomable)

#### Run and wait

`cmd run` sends a batch of commands and prints each result:

```sh
mrg-iot cmd run "exp devices" -e myexp
```

![mrg-iot cmd run sending 'exp devices' and printing the device the experiment holds](cmd-run.png#zoomable)

```sh
mrg-iot cmd run "s-echodot-1 click_button" "s-echodot-1 power_off" -e myexp
```

A device command either answers immediately or is **queued**: the experiment takes
the job, hands back an id, and works in the background. `cmd run` finishes the job
for you: when a reply says "queued" it sends `query wait_result` and prints the real
result, so one command in means one result out however long the device takes.

#### Run without waiting

`cmd async-run` prints what came straight back and records the job as a **task** with
a handle of its own (`t1`, `t2`, and so on) which you pick up in a later invocation:

```sh
mrg-iot cmd async-run "s-echodot-1 hold_button 10" -e myexp
# Command hold_button is queued
# ✔ t1: s-echodot-1 hold_button (job 3)

mrg-iot cmd tasks          # everything on record (local; opens no session)
mrg-iot cmd status t1      # where it has got to
mrg-iot cmd wait t1        # block, then print the result
```

A handle carries the project, experiment, realization, and XDC its job was started
in, which is what makes `mrg-iot cmd wait t1` a complete command, with no flags
needed.
Job ids restart at `0` in every experiment, so they are never what you type.

Tasks are forgotten when the experiment they belong to is torn down, because a
redeployed experiment starts counting job ids from `0` again.

#### An interactive prompt

```sh
mrg-iot cmd shell -e myexp
```

This is one foreground session you keep open and type into, at the `spiot_ctl >`
prompt. Type `exit` (or Ctrl-C / Ctrl-D) to leave. History is saved to
`~/.spiot_history`.

If you are already **inside an XDC**, none of the SSH machinery is needed. The
experiment is reachable directly over WireGuard:

```sh
mrg-iot cmd ctl              # default host 192.168.254.1
mrg-iot cmd ctl 10.0.0.5     # override the ExperimentControl host
```

#### Storing a result and reusing it

At the prompt (and in files run by `exp read`), `-s` captures part of a command's
reply into a variable, which you then reference with a leading underscore:

```
s-echodot-1 -s id job click_button
query wait_result s-echodot-1 click_button _job
```

`-s` takes a **reference** and then a **variable name**. The reference is `id`,
`result`, or `state`.

### Camera streams: `mrg-iot stream`

Camera-equipped devices are proxied over RTSP. Three subcommands:

![mrg-iot stream --help showing the open, list, and url subcommands](stream-help.png#zoomable)

```sh
mrg-iot stream list -e myexp    # name the cameras
mrg-iot stream open -e myexp    # VLC viewers; runs until you close them
mrg-iot stream url  -e myexp    # RTSP URLs; keeps forwarding until Ctrl+C
```

![mrg-iot stream list printing the RTSP address of the experiment's camera](stream-list.png#zoomable)

The port in a stream URL is assigned per session, so `stream list` prints a
`<port>` placeholder. Run `stream url` for the real addresses, or `stream open` to
just watch them.

![A VLC window playing a live camera feed from an IoT device](camera-stream.png#zoomable)

Because a stream is continuous, the command has to keep running to hold its tunnel
open. Closing the command closes the feed. Prefer another player? Pipe the URLs to
it:

```sh
mrg-iot stream url -e myexp 2>/dev/null | xargs vlc
```

{{% alert title="stdout carries data, stderr carries commentary" color="info" %}}
Every `mrg-iot` command follows the usual Unix split: tables, URLs, and verbatim
output from the experiment go to **stdout**; `✔`/`✖`/`▲` lines, progress, and
prompts go to **stderr**. So a redirect collects the data and the narration still
reaches your terminal.
{{% /alert %}}

---

## 5. Get your data: `mrg-iot file`

The experiment writes its capture and logs to a file server on the XDC, which
`mrg-iot` reaches through a tunnel. The `file` group is how you move things in and
out of it:

![mrg-iot file --help showing the download, upload, list, remove, and client subcommands](file-help.png#zoomable)

See what is there:

```sh
mrg-iot file list -e myexp              # the root
mrg-iot file list output -e myexp       # or any directory in it
```

![mrg-iot file list showing the uploads and output directories in the experiment's share](file-list.png#zoomable)

Download one file, or the whole `output` directory when you name nothing:

```sh
mrg-iot file download -e myexp                          # → ~/Downloads/myexp.output.zip
mrg-iot file download output/capture.pcap -e myexp      # → ~/Downloads/myexp.output.capture.pcap
mrg-iot file download capture.pcap -o /tmp/run.pcap -e myexp
```

A download is named `<experiment>.<remote path>`, with every `/` turned into a `.`.
That keeps two facts a bare basename would lose, namely which experiment the file
came from and which directory it sat in, so `output/capture.pcap` and
`traces/capture.pcap` no longer overwrite each other. Pass `--output`/`-o` for a
name of your own.

Uploading and deleting work in the root only; uploads always land in `uploads/`:

```sh
mrg-iot file upload ./script.py -e myexp
mrg-iot file remove capture.pcap -e myexp
```

Or browse the share yourself:

```sh
mrg-iot file client -e myexp    # opens the web client in a browser, until Ctrl+C
```

{{% alert title="Names are the file server's, not your shell's" color="info" %}}
There is no globbing, and a path is relative to the share's root, never to your
machine. Run `mrg-iot file list` and use the name it prints.
{{% /alert %}}

The output archive contains at minimum a log file and a `.pcap` network capture.
Anything your commands produced, such as screenshots and command output, is in there too.

---

## 6. Stop the experiment: `mrg-iot exp`

`exp teardown` is the inverse of `exp setup`: it detaches and deletes the XDC,
deletes the materialization and realization (releasing the hardware), and deletes
the experiment record.

```sh
mrg-iot exp teardown myexp
```

![mrg-iot exp teardown reporting the experiment, realization, and XDC it released](exp-teardown.png#zoomable)

To release the hardware but keep the experiment so you can redeploy it later:

```sh
mrg-iot exp undeploy myexp    # detach → delete xdc → delete materialization → delete realization
mrg-iot exp deploy myexp      # ...and bring it back when you need it
```

Both report each resource they removed, and a resource that is already gone is
reported as *skipped* rather than as a failure, so re-running either one is safe
and still exits `0`.

{{% alert title="Nothing is cleaned up for you" color="warning" %}}
If you interrupt provisioning with Ctrl-C, or finish without tearing down, the
experiment, realization, materialization, and XDC are **kept** so you can reconnect.
They hold your hardware and count against your quota until their duration expires.
Run `exp teardown` when you are genuinely done.
{{% /alert %}}

Download anything you want to keep **before** tearing down. The share goes away
with the experiment, and so do any task handles left by `cmd async-run`.

---

## What lives where on your machine

Everything user-scoped is under `~/.mrg-iot/`:

| File | Purpose |
|---|---|
| `session.json` | Your cached login: `{"username", "token"}` (mode `0600`). |
| `debug.log` | Verbose log, appended across runs, rotated at 5 MB (3 kept). Attach this to bug reports. |
| `tasks.json` | Jobs left running by `cmd async-run`, so `cmd wait` / `cmd status` / `cmd tasks` can name one later. |

There is no cached *connection* state: which enclave a command targets comes from
its flags, or from the defaults above, every time.

## Next

- **[Troubleshooting & FAQ](https://mergetb.gitlab.io/testbeds/sphere/sphere-docs/docs/experimentation/iot-devices/troubleshooting/)**: symptoms, causes, and fixes.
- **[Quickstart](https://mergetb.gitlab.io/testbeds/sphere/sphere-docs/docs/experimentation/iot-devices/quickstart/)**: the same flow, condensed to a handful of commands.
