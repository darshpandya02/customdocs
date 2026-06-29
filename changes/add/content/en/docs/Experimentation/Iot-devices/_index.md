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

Sphere lets you incorporate real IoT devices — smart speakers, cameras, and similar
hardware — into your experiments. Devices are provisioned as nodes within a
**realization** attached to an **XDC** (Experiment Development Container). Once the
realization is active and the XDC is connected, SSH tunnels expose the device services
to your laptop, and you drive the devices through an interactive control client.

The `mrg-iot` command-line tool automates this entire lifecycle, and the `spiot_ctl`
control interface is used to issue commands to the devices once they are running.

## Key concepts

If you are new to the Merge Testbed, these terms appear throughout the IoT guides:

| Term | Meaning |
|---|---|
| **Experiment** | The model describing the devices and the network connecting them. |
| **Realization** | A reservation of physical resources that satisfies an experiment's model. |
| **Materialization** | The act of booting and configuring the reserved devices so they are live. |
| **XDC** (Experiment Development Container) | The container you connect to in order to reach your experiment's devices. |
| **Enclave** | The isolated network segment a materialized experiment runs in; `mrg-iot` resolves it for you when connecting. |
| **ExperimentControl** | The gRPC service (port `17000`) that relays your commands to the devices. |