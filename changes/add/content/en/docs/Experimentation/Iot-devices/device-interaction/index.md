---
title: "Device Interaction"
linkTitle: "Device Interaction"
weight: 3
description: >
    Issue commands to IoT devices, query async job results, and automate experiment workflows using the spiot_ctl control interface.
---

{{% alert title="Prerequisite" color="info" %}}
This guide assumes IoT devices have already been reserved and an active session is running. See [Device Creation](../device-creation) for setup instructions, or the [Quickstart](../quickstart) for the short path.
{{% /alert %}}

## Overview

All device interaction goes through the **ExperimentControl** gRPC channel, exposed locally on port `17000` via an SSH tunnel. The `spiot_ctl` client (started automatically by `mrg-iot run`) connects to this channel and presents an interactive prompt.

The control interface organizes commands into three namespaces:

| Namespace | Purpose |
|---|---|
| `exp` | Experiment-level operations (list devices, get credentials, automation helpers) |
| `dev` | Send commands to a specific IoT device |
| `query` | Poll and retrieve results for asynchronous device commands |

---

## Starting the Control Client

### Via mrg-iot run (recommended)

The `spiot_ctl` prompt is opened automatically at the end of the `mrg-iot run` setup flow. No additional steps are required.

### Via mrg-iot ctl (inside the XDC)

If you are already inside the XDC, you can connect to ExperimentControl directly over WireGuard:

```sh
mrg-iot ctl              # default host: 192.168.254.1
```

---

## Experiment Commands

### List Devices

List all IoT devices currently managed by the experiment:

```
exp devices
```

Example output for an experiment with `s-echodot-1`, `s-echodot-2`, and `s-echopop-1`:

![](01_Exp_Devices.png#zoomable)

### Get Device Credentials

Retrieve application credentials for a specific device:

```
exp cred <device>
```

Example:

```
exp cred s-echodot-1
```

![](02_Exp_Cred_Device.png#zoomable)

Credentials include any device-specific login or API tokens needed to interact with the device's native application.

### Pause Execution

Pause the experiment for a set duration (in seconds). This is useful for allowing devices time to start up or for spacing out automated command sequences:

```
exp sleep <seconds>
```

Example — pause for 30 seconds:

```
exp sleep 30
```

### Run Commands from a File

Read a file from the XDC filesystem and execute each line as a command, in order, via the job queue:

```
exp read <filepath>
```

This is the primary mechanism for automating experiment workflows. Each line in the file is treated as a single command.

---

## Device Commands

### List Available Commands

Display all commands supported by a device, with short descriptions:

```
<device> help
<device> commands
```

Example:

```
s-echodot-1 help
```

![](03_Dev_Name_Help.png#zoomable)

The output lists each command name alongside a brief description of what it does.

{{% alert title="Note" color="info" %}}
Commands can differ between devices even if they are the same model. A device may support additional targets or parameters that its sibling does not. Always query a specific device rather than assuming commands are identical across the same model.
{{% /alert %}}

### Get Help for a Specific Command

Display detailed information about a single command, including accepted parameters:

```
<device> <command> help
```

Examples — comparing `click_button` across two devices:

```
s-echopop-1 click_button help
s-echodot-1 click_button help
```

`s-echopop-1` may list specific button targets as required parameters, while `s-echodot-1` may accept no arguments for the same command. The output for each device reflects its actual capabilities.

### Execute a Device Command

Send a command to a device:

```
<device> <command> [args...]
```

Examples:

```
s-echodot-1 click_button
s-echopop-1 click_button action_button
s-echodot-1 power_off
```

Commands execute either synchronously (result returned immediately) or asynchronously (a job ID is returned for polling). Device help output indicates which mode applies.

---

## Command Queries

Many device commands are executed **asynchronously**. When you run such a command, it is placed in a job queue and assigned a numeric **job ID**. You can use the `query` namespace to inspect the state of running or completed jobs.

### Check Command State

Get the current state of an async command (`pending`, `in_progress`, `complete`, or `canceled`):

```
query state <device> <command> <id>
```

Example:

```
query state s-echodot-1 click_button 0
```

### Retrieve Command Result

Fetch the result of a completed command. Only meaningful once the state is `complete`:

```
query get_result <device> <command> <id>
```

Example:

```
query get_result s-echodot-1 click_button 0
```

A common pattern is to first check state, then retrieve the result:

```
query state s-echodot-1 click_button 0
query get_result s-echodot-1 click_button 0
```

![](05_Query_State_and_Get_Result.png#zoomable)

### Wait for Result

Block until the command completes, then return the result immediately. This eliminates the need to manually poll state:

```
query wait_result <device> <command> <id>
```

Example:

```
query wait_result s-echodot-1 click_button 0
```

Use `wait_result` in scripts and automated workflows to ensure sequential dependencies are satisfied before proceeding.

---

## Storing and Reusing Job IDs

When automating sequences of commands, you often need to capture the job ID returned by one command and pass it to a subsequent `query` command. The `-s` flag stores a value from the command output into a named variable:

```
<device> -s <variable_name> <field> <command> [args...]
```

| Argument | Description |
|---|---|
| `-s <variable_name>` | Name of the variable to store the value into |
| `<field>` | Which field to capture: `id` (job ID) or `results` |
| `<command>` | The device command to execute |

### Example: Storing a Job ID

Execute `click_button` on `s-echodot-1` and store the returned job ID in a variable named `job`:

```
s-echodot-1 -s job id click_button
```

![](07_Store_Load_Job_Id.png#zoomable)

### Example: Using a Stored Variable

Reference a stored variable by prefixing it with an underscore (`_`). The following two commands are equivalent when `job` holds the value `0`:

```
query wait_result s-echodot-1 click_button _job
query wait_result s-echodot-1 click_button 0
```

This pattern is especially useful in files executed via `exp read`, where job IDs from earlier commands need to be referenced later in the same script.

### Full Automation Example

```
s-echodot-1 -s job id click_button
query wait_result s-echodot-1 click_button _job
exp sleep 5
s-echodot-1 -s job id click_button
query wait_result s-echodot-1 click_button _job
```

---

## Video Streams

Camera-equipped devices expose RTSP streams proxied through port `8554`. Access them with VLC:

```sh
vlc rtsp://localhost:8554/<handler_name>
```

When using `mrg-iot run`, accessible camera feeds are opened automatically in VLC windows at session start.

When using the daemon mode, open streams on demand:

```sh
mrg-iot show video
```

{{% alert title="Note" color="info" %}}
VLC must be installed and on your `PATH` (install it from your OS package manager — it is not a pip package). If VLC is missing, the tool prints a notice and the stream stays available at the URL above for any RTSP-capable player.
{{% /alert %}}

---

## Daemon Mode Command Reference

When an experiment session is running as a background daemon (started with `mrg-iot connect`), use `mrg-iot send` to issue commands without an interactive prompt:

```sh
mrg-iot send "<command>"
```

Multiple commands are executed in order:

```sh
mrg-iot send "exp devices" "exp cred s-echodot-1"
```

All `exp`, `dev`, and `query` commands are supported identically to the interactive prompt.

### Example Script

```sh
mrg-iot send "exp devices"
mrg-iot send "s-echodot-1 -s job id click_button"
mrg-iot send "query wait_result s-echodot-1 click_button _job"
mrg-iot send "exp sleep 10"
mrg-iot send "s-echodot-1 power_off"
```

---

## Downloading Experiment Files

### From the Daemon

```sh
mrg-iot download traffic
mrg-iot download traffic --output /path/to/output.tar.gz
```

### From Inside the XDC (manual)

While connected with the `9000` tunnel active:

```sh
curl -O -J http://192.168.254.1:9000
```

The downloaded archive is named:

```
<realization>.<experiment>.<project>.tar.gz
```

Extract it with:

```sh
tar -xvf <archive>.tar.gz
```

The archive always contains a log file and a `.pcap` network capture. Additional output files (screenshots, command results) are included based on the commands executed during the session.

---

## Full Command Reference

### exp namespace

| Command | Description |
|---|---|
| `exp devices` | List all devices in the current experiment |
| `exp cred <device>` | Get application credentials for a device |
| `exp sleep <seconds>` | Pause execution for the given number of seconds |
| `exp read <filepath>` | Read and execute commands from a file on the XDC |

### device namespace

| Command | Description |
|---|---|
| `<device> help` | List all commands supported by the device |
| `<device> commands` | Alias for `<device> help` |
| `<device> <command> help` | Show detailed help for a specific command |
| `<device> <command> [args...]` | Execute a command on the device |
| `<device> -s <var> id <command> [args...]` | Execute and store the returned job ID |
| `<device> -s <var> results <command> [args...]` | Execute and store the returned result |

### query namespace

| Command | Description |
|---|---|
| `query state <device> <command> <id>` | Get the current state of an async command |
| `query get_result <device> <command> <id>` | Retrieve the result of a completed command |
| `query wait_result <device> <command> <id>` | Block until the command completes and return its result |

Variables stored with `-s` are referenced in subsequent commands by prefixing the variable name with `_` (e.g. `_job`).
