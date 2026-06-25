---
title: "Device Creation"
linkTitle: "Device Creation"
weight: 2
description: >
    Reserve, materialize, and connect to Sphere IoT devices using the mrg-iot CLI or the Merge portal directly.
---

New to IoT devices on Sphere? Start with the [Quickstart](content/en/docs/Experimentation/Iot-devices/quickstart/) for the
shortest path. This page is the complete reference for every creation workflow.

## Overview

IoT devices in Sphere are provisioned as nodes within a **realization** attached to an **XDC** (Experiment Development Container). Once a realization is active and the XDC is connected, SSH tunnels expose three services to your laptop:

| Port | Service |
|------|---------|
| `8554` | RTSP video stream proxy (camera-equipped devices) |
| `9000` | Experiment file download server |
| `17000` | ExperimentControl gRPC channel |

There are two ways to reach this state: the **automated path** using `mrg-iot` (recommended), and the **manual path** via the Merge portal or CLI with hand-crafted experiment models.

---

## Prerequisites

- A Sphere / Merge Testbed account with access to an IoT project (e.g. `neuiot`)
- Python 3.10 or later
- `pipx` installed (`pip install --user pipx` then `pipx ensurepath`)
- [VLC](https://www.videolan.org) for viewing RTSP camera streams (the tool launches VLC automatically)

---

## Automated Experiment Creation with mrg-iot

### Installation

Install `mrg-iot` from PyPI into an isolated environment:

```sh
pipx install mrg-iot
```

Verify the installation:

```sh
mrg-iot --version
```

{{% alert title="macOS SSL workaround" color="info" %}}
If portal calls fail with an SSL error, run:
```sh
export SSL_CERT_FILE="$(python -m certifi)"
export REQUESTS_CA_BUNDLE="$SSL_CERT_FILE"
```
{{% /alert %}}

### Authentication

Save your Sphere credentials locally before running experiments:

```sh
mrg-iot login
```

This stores a session token at `~/.mrg-iot/session.json` (mode `0600`). To remove stored credentials:

```sh
mrg-iot logout
```

### Running an Experiment

`mrg-iot run` drives the complete experiment lifecycle — from device selection through cleanup — in a single command. It supports three modes:

#### Interactive Mode (default)

Prompts for every option, including experiment name, description, network name, realization name, XDC name, and duration:

```sh
mrg-iot run
```

At startup you are asked whether to use the simplified setup:

![](02_Mrgiot_Auto_Mode_Select.png#zoomable)

Each value is confirmed before the model is compiled and pushed:

![](07_Mrgiot_Manual_Experiment_Setup.png#zoomable)

#### Simplified Interactive Mode

Uses sensible defaults for optional fields and only prompts for inputs that cannot be inferred. Default values are derived from your username:

| Field | Default value |
|---|---|
| Experiment name | `<username>` |
| Experiment description | `Experiment generated using mrg-iot tool by <username>` |
| Network name | `mrg-iot-net` |
| Realization name | `realiot` |
| XDC name | `<username>xdc` |

```sh
mrg-iot run --simplified
```

#### Non-Interactive (Scripted) Mode

Supplies all values via flags; no prompts are shown. Suitable for CI pipelines and reproducible workflows:

```sh
mrg-iot run --non-interactive \
  --project neuiot \
  --devices s-echodot-1 s-googlenest-1 \
  --exp-name myexp \
  --exp-desc "Smart-speaker capture run" \
  --realization realiot \
  --duration 4d \
  --xdc myxdc \
  --commands-file /path/to/commands.txt
```

**Available flags for `mrg-iot run`:**

| Flag | Short | Description |
|---|---|---|
| `--non-interactive` | `-n` | Disable all prompts; requires all flags to be set |
| `--simplified` | `-s` | Interactive mode with default values pre-filled |
| `--project <name>` | | Target project (e.g. `neuiot`) |
| `--devices <d> [<d>...]` | | One or more device names |
| `--exp-name <name>` | | Experiment name |
| `--exp-desc <text>` | | Experiment description |
| `--realization <name>` | | Realization name |
| `--duration <duration>` | | Experiment duration (minimum: 4 days) |
| `--xdc <name>` | | XDC name |
| `--network <name>` | `-net` | Model network name (default: `mrg-iot-net`) |
| `--commands-file <path>` | `-cf` | Path to a newline-delimited file of commands to run automatically |
| `--download-files` | `-df` | Automatically download the experiment archive on exit |
| `--delete-xdc` | `-dx` | Delete the XDC during cleanup |
| `--delete-exp` | `-de` | Delete the experiment during cleanup |
| `--debug` | `-d` | Write verbose logs to stderr and `debug.log` |

### Input Validation Rules

The CLI validates inputs before contacting the portal and exits with code `2` on failure:

| Field | Rules |
|---|---|
| Names (experiment, realization, XDC, network) | Lowercase letters and digits only; must start with a letter; max 32 characters |
| Description | Letters, digits, spaces, commas, periods, hyphens; max 256 characters |
| Duration | Minimum **4 days**. Formats: `1w`, `4d`, `1w2d3h`, `1 week 2 days` |

{{% alert title="Recommendation" color="info" %}}
Use at least `4d` (4 days) as the duration. Expiry notifications are sent 3 days before XDC expiry and 1 day before realization expiry, so shorter durations leave no buffer.
{{% /alert %}}

### Step-by-Step Walkthrough

The following describes what happens when you run `mrg-iot run`:

**1. Login**

If no valid session token exists, you are prompted for your Sphere username and password.

![](03_Mrgiot_Auto_Login.png#zoomable)

**2. Project Selection**

If your account belongs to more than one project, you are presented with a numbered list. Enter the index or name of the project to use.

![](04_Mrgiot_Auto_Projects.png#zoomable)

**3. Device Selection**

All devices available in your project are listed with index numbers. Select devices by index or by name. You may add multiple devices across several prompts. Once finished, enter an empty line or confirm to proceed.

![](05_Mrgiot_Auto_Select_Devices.png#zoomable)

Invalid or unknown device names are ignored, and your selection is confirmed before continuing:

![](06_Mrgiot_Auto_Select_Devices_Out.png#zoomable)

**4. Model Compilation and Push**

`mrg-iot` builds a Python experiment model, compiles it, and pushes it to the Merge portal. In simplified mode this happens automatically; in customized mode you are first prompted for the experiment name, description, and network name.

![](07_Mrgiot_Auto_Compile.png#zoomable)

**5. Duration Selection**

Enter how long the realization and XDC should remain active, in `<N>w <N>d <N>h <N>m <N>s` format. Example: `0w 4d 0h 0m 0s`.

![](08_Mrgiot_Auto_Select_Duration.png#zoomable)

**6. Realization and Materialization**

The experiment is realized (resources are reserved) and then materialized (devices are booted and configured).

**7. XDC Creation and Attachment**

An XDC is created and the realization is attached to it. `mrg-iot` waits until the XDC is ready before proceeding.

![](09_Mrgiot_Manual_xdc_Name.png#zoomable)

![](10_Mrgiot_Experiment_Start_Up.png#zoomable)

**8. Tunnel Setup**

SSH tunnels are established from your laptop to the XDC, forwarding ports `8554`, `9000`, and `17000`.

**9. Camera Streams**

If any selected device has an accessible camera, a VLC window opens automatically for each camera feed. (If VLC is not installed, the streams are skipped but remain available at `rtsp://localhost:8554/<handler>`.)

![](11_Mrgiot_Camera_Stream.png#zoomable)

**10. Interactive Session**

The `spiot_ctl` prompt becomes available. See [Device Interaction](content/en/docs/Experimentation/Iot-devices/device-interaction/) for the full command reference.

### Post-Experiment Cleanup

Type `exit` at the `spiot_ctl` prompt to end the session. `mrg-iot` then:

1. **Offers to download experiment files.** Files are packaged as a `.tar.gz` archive named:
   ```
   <realization>.<experiment>.<project>.tar.gz
   ```
   With simplified setup and username `jsmith` on project `neuiot`:
   ```
   realiot.jsmith.neuiot.tar.gz
   ```

   ![](12_Mrgiot_Experiment_Download.png#zoomable)

2. **Cleans up resources.** In simplified mode the XDC is deleted automatically after the realization is detached and relinquished. In customized mode you are prompted whether to delete the XDC and then the experiment.

   ![](13_Mrgiot_Manual_Post_Download.png#zoomable)

To extract the downloaded archive:

```sh
tar -xvf realiot.jsmith.neuiot.tar.gz
```

![](14_Mrgiot_Extract_Download.png#zoomable)

The archive contains at minimum a log file and a `.pcap` network capture. Additional files (screenshots, command outputs) are included based on the commands you ran during the session.

---

## Persistent Daemon Mode

For workflows where you need to drive an experiment from multiple terminal sessions or scripts without keeping `mrg-iot run` open, use the daemon-backed connection mode.

### Starting the Daemon

```sh
mrg-iot connect
```

This resolves the enclave interactively (or via flags), spawns a background daemon, and returns immediately. The daemon writes a connection descriptor to `~/.mrg-iot/connection.json` and a Unix socket to `~/.mrg-iot/connection.sock`.

Optional flags to skip prompts:

```sh
mrg-iot connect \
  --project neuiot \
  --experiment myexp \
  --realization realiot \
  --xdc myxdc
```

### Sending Commands

```sh
mrg-iot send "exp devices"
mrg-iot send "s-echodot-1 click_button"
mrg-iot send "exp cred s-echodot-1" "exp sleep 10"   # multiple commands, executed in order
```

### Viewing Camera Streams

```sh
mrg-iot show video
```

Opens a viewer window for each accessible camera in the active experiment.

### Downloading Experiment Files

```sh
mrg-iot download traffic
mrg-iot download traffic --output /path/to/output.tar.gz
```

### Stopping the Daemon

```sh
mrg-iot disconnect
```

---

## Manual Experiment Creation

{{% alert title="Note" color="info" %}}
For experiments that include non-IoT nodes, the manual path is required. For IoT-only experiments the automated path with `mrg-iot` is strongly recommended.

Consult the Merge Testbed Hello World documentation ([CLI](/docs/experimentation/hello-world/) / [Web Interface](/docs/experimentation/hello-world-gui/)) for foundational experiment creation concepts.
{{% /alert %}}

### 1. Write the Experiment Model

IoT devices are reserved as **metal nodes**. Specify the device name in the `host` parameter. The following example creates three nodes connected in a chain:

```python
from mergexp import *

net = Network("Example", addressing==ipv4, routing==static)

a = net.node("a", metal==True, host=="s-echodot-1")
b = net.node("b", metal==True, host=="s-echodot-2")
c = net.node("c", metal==True, host=="s-echopop-1")

net.connect([a, b])
net.connect([b, c])

experiment(net)
```

Compile and push the model, then realize and materialize through the Merge portal or CLI.

### 2. Connect to the XDC with Port Forwarding

Connect to the XDC while establishing the required tunnels:

```sh
mrg xdc ssh <xdc_name> -L 9000:192.168.254.1:9000 -L 8554:192.168.254.1:8554
```

This forwards the file download server (port `9000`) on the host machine (IP `192.168.254.1`) to your local machine. For the RTSP video stream, add `-L 8554:192.168.254.1:8554`.

### 3. Download and Run the Control Client

Inside the XDC, download the `mrg-iot` tool:

```sh
pip install mrg-iot
```

Start the client:

```sh
mrg-iot ctl
```

### 4. Access Camera Streams

While connected with the tunnel active, open a stream on your local device with VLC:

```sh
vlc rtsp://localhost:8554/<handler_name>
```

{{% alert title="Note" color="info" %}}
Install VLC from your OS package manager (it is not a pip package) and ensure it is on your `PATH`. The stream URL works in any RTSP-capable player if you prefer a different viewer.
{{% /alert %}}

### 5. Download Experiment Files

While the XDC tunnel is active, download files on your local device:

```sh
curl -O -J http://localhost:9000
```

---

## Resource Management Reference

`mrg-iot` exposes fine-grained subcommands for managing individual resources outside of the full experiment workflow.

### Experiments

```sh
mrg-iot exp list [--project <project>]
mrg-iot exp info --project <project> --name <name>
mrg-iot exp create --project <project> --name <name> [--description <desc>] \
    [--network <net>] [--realization <rz>] [--duration <d>]
mrg-iot exp delete --project <project> --name <name>
```

### Realizations

```sh
mrg-iot realization list   --project <project> --experiment <exp>
mrg-iot realization info   --project <project> --experiment <exp> --name <rz>
mrg-iot realization status --project <project> --experiment <exp> --name <rz>
mrg-iot realization create --project <project> --experiment <exp> \
    --devices <d1> [<d2>...] [--name <rz>] [--network <net>] [--duration <d>]
mrg-iot realization relinquish --project <project> --experiment <exp> --name <rz>
```

`realization` may be abbreviated as `rz`.

### Materializations

```sh
mrg-iot materialization list          --project <project> --experiment <exp>
mrg-iot materialization info          --project <project> --experiment <exp> --realization <rz>
mrg-iot materialization status        --project <project> --experiment <exp> --realization <rz>
mrg-iot materialization create        --project <project> --experiment <exp> --realization <rz>
mrg-iot materialization dematerialize --project <project> --experiment <exp> --realization <rz>
```

`materialization` may be abbreviated as `mtz`.

### XDCs

```sh
mrg-iot xdc list [--project <project>]
mrg-iot xdc info   --project <project> --name <name>
mrg-iot xdc create --project <project> [--name <name>] [--duration <d>] [--no-wait]
mrg-iot xdc create --project <project> --experiment <exp> --realization <rz>
mrg-iot xdc attach --project <project> --name <name> --experiment <exp> --realization <rz>
mrg-iot xdc detach --project <project> --name <name> --experiment <exp> --realization <rz>
mrg-iot xdc delete --project <project> --name <name>
```

---

## In-XDC Interactive Client

`mrg-iot ctl` is an interactive client intended to be run **from inside the XDC**, connecting directly to ExperimentControl over WireGuard rather than through an SSH tunnel. No portal login is required.

```sh
mrg-iot ctl              # connects to 192.168.254.1:17000 (default)
```

This opens the same `spiot_ctl` prompt used by `mrg-iot run`. See [Device Interaction](content/en/docs/Experimentation/Iot-devices/quickstart/device-interaction/) for the full command reference.

Command history is saved to `~/.spiot_history` (up to 500 entries).
