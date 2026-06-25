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