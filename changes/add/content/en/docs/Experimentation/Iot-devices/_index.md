---
title: "IoT Devices"
linkTitle: "IoT Devices"
weight: 6
description: >
    Reserve, materialize, and interact with physical IoT devices provided by Northeastern as part of your experiments.
---

{{% alert title="Beta" color="warning" %}}
Support for IoT devices is currently in **beta**. Please report any issues you encounter, and share feedback with the NEU IoT Facility.
{{% /alert %}}

SPHERE lets you incorporate real IoT devices — smart speakers, cameras, and similar
hardware — into your experiments. Devices are provisioned as nodes within a
**realization** attached to an **XDC** (Experiment Development Container). Once the
realization is active and the XDC is connected, SSH tunnels expose the device services
to your laptop, and you drive the devices through an interactive control client.

The `mrg-iot` command-line tool automates this entire lifecycle, and the `spiot_ctl`
control interface is used to issue commands to the devices once they are running.

## Which page do I need?

- **[Quickstart](quickstart)** — Start here. The shortest path from install to
  interacting with a device, in a handful of commands.
- **[Device Creation](device-creation)** — The full reference for reserving,
  materializing, and connecting to devices with `mrg-iot`, including the automated,
  daemon, and manual workflows.
- **[Device Interaction](device-interaction)** — The complete command reference for
  the `spiot_ctl` control interface: sending commands, querying async job results,
  and automating workflows.
- **[Troubleshooting & FAQ](troubleshooting)** — Common errors and how to resolve
  them: cleaning up leftover resources, login and connection problems, validation
  rules, and device-command errors.
