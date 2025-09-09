---
title: "Device Creation"
linkTitle: "Device Creation"
weight: 1
description: >
    Incorporate IoT Devices provided by Northeastern as a part of experiments, using an automated client or the Merge portal
---


{{% alert title="Attention" color="warning" %}} Support for IoT Devices is currently in beta, please report
any issues as you encounter them. Additionally, any feedback is greatly appreciated. - NEU IoT Facility {{% /alert %}}

# Automated Experiment Creation

{{% alert title="Attention" color="warning" %}} This guide works only for experiments where
only IoT Devices are being materialized.
{{% /alert %}}

{{% alert title="Note" color="info" %}} This guide uses the ["mrg-iot"](https://drive.google.com/file/d/1kJtzUpEiVm7dGaDi3IiqHKefdlsQoUXd/view?usp=drive_link) tool,
to manage device model creation, materialization, reservation, and activation. Additionally, 
the tool automatically will open camera feeds for devices that support them.
{{% /alert %}}

Once downloaded, the mrg-iot tool can be run using the following command:
```sh
./mrg-iot <username> <password> <project> "<list of devices>" [--delete]
```

Where username and password are your Merge username and password, the project is the
IoT facility you will be commissioning devices from; which is accessible on the resources page
followed by a list of devices you would like to reserve. Optionally, a delete flag can be
included to indicate that you want mrg-iot to clean up after completion and to delete the
experiment and all components.

The user `j.smith` with the password `hunter27` who wants to commission `s-echodot-1` and 
`s-echodot-2` from the project `sphereiot` and does not want to clean up devices would run:
```sh
./mrg-iot j.smith hunter27 sphereiot "s-echodot-1, s-echodot-2"
```

This will automatically generate the materialization and XDC, then realize the experiment 
and attach the XDC to it. Once connected, the XDC will connect to the server and open camera 
feeds for all devices that are part of the experiment. Once this is done there will be a command 
line interface for experiment interaction. 

Once the experiment is complete you can run the following command twice to exit both the command 
line and the mrg-iot tool. If you exit from the experiment and want to return to the command line 
you can return to the command line interface by running:
```sh
python3 spiot-ctl
```

After experiment completion, files can be downloaded by running:
```sh
curl -O -J http://172.30.0.1:9000
```
From inside the XDC.

# Manual Experiment Creation

{{% alert title="Attention" color="warning" %}}
Experiment creation follows the same systems used for the reservation of other nodes, for documentation
on this system see the "Hello World" 
([Command Line Interface](/docs/experimentation/hello-world),
[Web Interface](/docs/experimentation/hello-world-gui)) documentation for more information.
{{% /alert %}}

{{% alert title="Note" color="info" %}}
It is recommended to us FFplay for watching rtsp streams from devices, as users have previously experienced
issues watching streams using VLC
{{% /alert %}}

When creating the experiment, in order incorporate IoT devices into the host parameter of a node, and specifying
it is a metal node. As an example, a user wanting to create an experiment with nodes `a`, `b`, and `c` where the 
devices are `s-echodot-1`, `s-echodot-2`, and `s-echopop-1` respectively. In this example where `a` is connected
to `b` which is connected to `c`, which is not connected to `a`. The user would create the following model:

```python
from mergexp import *

# Create a network topology object
net = Network("Example", addressing==ipv4, routing==static)

a = net.node("a", metal==True, host=="s-echodot-1")
b = net.node("b", metal==True, host=="s-echodot-2")
c = net.node("c", metal==True, host=="s-echopop-1")

net.connect([a,b])
net.connect([b,c])

experiment(net)
```

Once the experiment is realized and attached to an XDC, connect to it using the following mrg command:
```sh
mrg xdc ssh <xdc_name> -L 9000:172.30.0.1:9000 -L 8554:172.30.0.1:8554
```

This will connect to the XDC while linking to the RTSP restream and file server. Which can be used for
interacting with the experiment while in progress. The following command will download the spiot-client
which is used for direct experiment communication.
```sh
curl https://gitlab.com/-/snippets/4862725/raw/main/spiot_ctl | tr -d '\r' > spiot_ctl; chmod +x ./spiot_ctl
```

Which can then be run:
```sh
./spiot_ctl
```

For accessing the remote stream you can use the data provided by the command line interface:
```sh
ffplay rtsp://localhost:8554/<handler_name>
```

To download experiment files, while linked to the XDC run:
```sh
curl -O -J http://localhost:9000
```

