---
title: "Quickstart"
linkTitle: "Quickstart"
weight: 1
description: >
    The shortest path from a fresh install to interacting with a live IoT device.
---

This guide walks the **happy path**: install the tool, start an experiment with default
settings, and send a command to a device. For the full set of options, modes, and the
manual workflow, see [Device Creation](../device-creation/); for the complete control
command reference, see [Device Interaction](../device-interaction/).

{{% alert title="Beta" color="warning" %}}
IoT device support is currently in **beta**. Report issues and share feedback with the NEU IoT Facility.
{{% /alert %}}

## Prerequisites

- A Sphere / Merge Testbed account with access to an IoT project (e.g. `neuiot`)
- Python 3.10 or later, with [`pipx`](https://pipx.pypa.io) installed
- [VLC](https://www.videolan.org) — only needed for camera-equipped devices (the tool opens streams in VLC)

## 1. Install `mrg-iot`

```sh
pipx install mrg-iot
mrg-iot --version
```

{{% alert title="macOS SSL workaround" color="info" %}}
If portal calls fail with an SSL error on macOS, run:
```sh
export SSL_CERT_FILE="$(python -m certifi)"
export REQUESTS_CA_BUNDLE="$SSL_CERT_FILE"
```
{{% /alert %}}

## 2. Log in

```sh
mrg-iot login
```

Your session token is stored at `~/.mrg-iot/session.json`.

## 3. Run an experiment

`mrg-iot run --simplified` drives the full lifecycle — device selection, realization,
materialization, XDC creation, and tunnel setup — using sensible defaults derived from
your username. You are prompted only for the devices and the duration.

```sh
mrg-iot run --simplified
```

When prompted, select one or more devices by index or name:

![](05_Mrgiot_Auto_Select_Devices.png#zoomable)

{{% alert title="Recommendation" color="info" %}}
Use at least `4d` (4 days) for the duration. Expiry notifications are sent 3 days before XDC expiry and 1 day before realization expiry, so shorter durations leave no buffer.
{{% /alert %}}

When setup completes, camera feeds (if any) open automatically and the `spiot_ctl`
prompt becomes available.

## 4. Interact with a device

At the `spiot_ctl` prompt, list the devices in your experiment and send a command:

```
exp devices
```

![](01_Exp_Devices.png#zoomable)

```
s-echodot-1 click_button
```

## 5. Finish up

Type `exit` to end the session. `mrg-iot` offers to download your experiment files
(a `.tar.gz` archive containing logs and a `.pcap` capture) and then cleans up the
realization and XDC.

```
exit
```

## Detailed Steps

- **[Device Creation](../device-creation/)** — interactive, non-interactive, and daemon
  modes; the manual `mergexp` path; and per-resource management commands.
- **[Device Interaction](../device-interaction/)** — the full `exp` / `dev` / `query`
  command reference, async job handling, and automation with stored variables.
