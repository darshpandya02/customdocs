---
title: "Quickstart"
linkTitle: "Quickstart"
weight: 2
description: >
    Bring up an IoT experiment, drive a device, collect the data, and release the hardware, in a handful of commands.
---

This is the **happy path**. It assumes you already have a SPHERE account with access
to an IoT project. If you don't, or you want each step explained, start with
[Using mrg-iot](https://mergetb.gitlab.io/testbeds/sphere/sphere-docs/docs/experimentation/iot-devices/using-mrg-iot/) instead.

{{% alert title="Beta" color="warning" %}}
IoT device support is currently in **beta**. Report issues and share feedback with the NEU IoT Facility.
{{% /alert %}}

## Prerequisites

- **`mrg-iot` installed** and on your `PATH`. See [Installation](https://mergetb.gitlab.io/testbeds/sphere/sphere-docs/docs/experimentation/iot-devices/installation/).
- A SPHERE / Merge Testbed account with access to an IoT project (`neuiot`, `iotbeta`, or `iotdev`)
- Optionally [VLC](https://www.videolan.org), if you want `mrg-iot` to open camera
  streams for you. Any RTSP-capable player works: `mrg-iot stream url` prints the
  addresses so you can open them in whatever you already use. See
  [Camera streams](https://mergetb.gitlab.io/testbeds/sphere/sphere-docs/docs/experimentation/iot-devices/using-mrg-iot/#camera-streams-mrg-iot-stream).

---

## 1. Log in

```sh
mrg-iot login
```

Your session token is cached at `~/.mrg-iot/session.json`, so you only do this once.

## 2. Pick a device

List what is free to reserve right now:

```sh
mrg-iot devices list --available
```

![mrg-iot devices list --available printing a PROJECT and DEVICE column for each unallocated device, ending with '... and 223 more'](devices-available.png#zoomable)

Device names look like `s-echodot-1`. Note one or two down for the next step.

{{% alert title="s- and m- names" color="info" %}}
Device names carry a prefix. `s-` is a device under test, the kind you send commands
to. `m-` is a **helper phone** (e.g. `m-googlepixel-1`), used to drive a device's
companion app; these are being brought online and will start appearing in this list.
`mrg-iot` already reports both, but only `s-` names accept commands today, and it
tells you so if you address an `m-` name.
{{% /alert %}}

## 3. Bring up the experiment

`exp setup` creates the experiment, allocates the hardware, brings it online,
creates an XDC, and attaches it. One command, no prompts:

```sh
mrg-iot exp setup myexp --devices s-echodot-1
```

![mrg-iot exp setup reporting the project, experiment, realization, and XDC it provisioned](exp-setup.png#zoomable)

{{% alert title="Names and durations" color="info" %}}
Experiment, realization, and XDC names must match `^[a-z][a-z0-9]*$`: lowercase
letters and digits only, starting with a letter, max 32 characters. The default
duration is `1w`, and **4 days is the minimum**.
{{% /alert %}}

Anything you leave out is derived: the project (if you belong to only one), the
experiment name (your username), the realization (`realiot`), and the XDC
(`<experiment>xdc`). So `mrg-iot exp setup --devices s-echodot-1` on its own works too.

## 4. Drive the device

Send commands with `cmd run`. Each one opens a session, runs, and exits:

```sh
mrg-iot cmd run "exp devices" --experiment myexp
```

![mrg-iot cmd run sending 'exp devices' and printing the device the experiment holds](cmd-run.png#zoomable)

Ask a device what it can do, then do it:

```sh
mrg-iot cmd run "s-echodot-1 commands" -e myexp
mrg-iot cmd run "s-echodot-1 click_button" -e myexp
```

Prefer a prompt you can keep typing into? `mrg-iot cmd shell -e myexp` opens one.

## 5. Collect the data and release the hardware

```sh
mrg-iot file list -e myexp                 # what the experiment produced
mrg-iot file download -e myexp             # the whole output directory, as a zip
mrg-iot exp teardown myexp                 # release everything
```

`file list` shows the share the experiment writes into. `output` holds the current
run, and `uploads` is where your own files land:

![mrg-iot file list showing the uploads and output directories with their modification times](file-list.png#zoomable)

To send a file the other way, into `uploads/`, use `file upload`:

```sh
mrg-iot file upload ./script.py -e myexp
```

`file remove` deletes one, and `file client` opens the share in a browser. See
[Get your data](https://mergetb.gitlab.io/testbeds/sphere/sphere-docs/docs/experimentation/iot-devices/using-mrg-iot/#5-get-your-data-mrg-iot-file) for the full set.

And `exp teardown` names each resource as it releases it:

![mrg-iot exp teardown reporting 'Torn down successfully' with the project, experiment, realization, and XDC it released](exp-teardown.png#zoomable)

{{% alert title="Nothing expires on its own while you're away" color="warning" %}}
An experiment you don't tear down keeps holding its hardware until its duration
runs out, and it counts against your quota. Run `exp teardown` when you're done.
It is safe to re-run and skips anything already gone.
{{% /alert %}}

---

## The whole thing

```sh
mrg-iot login
mrg-iot devices list --available
mrg-iot exp setup myexp --devices s-echodot-1
mrg-iot cmd run "s-echodot-1 click_button" -e myexp
mrg-iot file download -e myexp
mrg-iot exp teardown myexp
```

## One command instead of all of them

`mrg-iot run` does the same lifecycle as a guided flow: it asks which devices you
want, provisions them, drops you at the `spiot_ctl >` prompt, offers you the files
on the way out, and cleans up when you type `exit`.

```sh
mrg-iot run                # interactive
mrg-iot run --simplified   # interactive, but skips the optional prompts
```

It is the one command that asks questions. Every other command in these guides
runs unattended exactly as written.

## Next

- **[Using mrg-iot](https://mergetb.gitlab.io/testbeds/sphere/sphere-docs/docs/experimentation/iot-devices/using-mrg-iot/)**: the full walkthrough, including account setup, camera streams, and async commands.
- **[Troubleshooting & FAQ](https://mergetb.gitlab.io/testbeds/sphere/sphere-docs/docs/experimentation/iot-devices/troubleshooting/)**: when something goes wrong.
