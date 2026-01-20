---
title: "Device Interaction"
linkTitle: "Device Interaction"
weight: 2
description: >
    Learn how to incorporate and use IoT Devices provided by Northeastern as a part of experiments
---


{{% alert title="Attention" color="warning" %}} Support for IoT Devices is currently in beta, please report
any issues as you encounter them. Additionally, any feedback is greatly appreciated. - NEU IoT Facility {{% /alert %}}

{{% alert title="Note" color="info" %}} This guide uses the assumes the IoT Devices have been created. See 
[Device Creation](docs/Experimentation/Special_Resources/IoT_Devices/device_creation) for information on reserving
and materializing devices.
{{% /alert %}}

#### Experiment Manager Interaction

The IoT devices are being controlled by an experiment manager which handles IoT device commands from the user through
the spiot-client. If not already started, the spiot-client can be started in the XDC with the following:
```sh
./spiot_ctl
```


While running the spiot-client, you can run the following to get a list of IoT devices that are
being managed of the experiment:

```
exp devices
```

An experiment with devices `s-echodot-1`, `s-echodot-2`, and `s-echopop-1` would output the 
following:

![](01_Exp_Devices.png#zoomable)

To get app credentials for a device the following command can be used:
```
exp cred <device>
```

To get the credentials for `s-echodot-1` the command would be:
```
exp cred s-echodot-1
```

Which returns:

![](02_Exp_Cred_Device.png#zoomable)

In order to automate the execution of commands, the following commands implemented to provide support
important features relevant to automation:
```
exp sleep <duration in seconds>
exp read <filepath>
```

The experiment sleep command pauses execution of the experiment for a set amount of time, allowing for devices to
be given time to start up or for execution to be completed on an operation.

The experiment read command will read the specified file from the XDC and will send all the commands contained in
the file to be run automatically from a job queue.

#### Device Interaction

To get a list of commands supported by a device you can run the one of the two following commands:
```
dev <device> help
dev <device> commands
```

This will output a summary of all the commands and their descriptions that are supported by the device:

![](03_Dev_Name_Help.png#zoomable)

As mentioned in the description, more information about a command can be requested:
```
dev device <command> help
```

Note that commands can differ between devices, even if they are the same model. For example, the subject 
device `s-echopop-1` specifies targets for `click_button`, the description for this command can be queried 
using the following command:
```
dev s-echopop-1 click_button help
```

In comparison, the subject device `s-echodot-1` does not specify any target for `click_button`, the description
for this command can be queried using the same command:
```
dev s-echodot-1 click_button help
```

The results of running both of these commands would return:

![](04_Dev_Name_Command_Help.png#zoomable)

#### Command Queries

Some commands are asynchronously queued, these commands are given a job id that can be
store and used by experiment to get information about the command being executed.

Information can be queried using the following commands:

```
query state <device> <command> <id>
query get_result <device> <command> <id>
query wait_result <device> <command> <id>
```

The `query state` command gets the current state of the command, indicating whether it 
is complete, in progress, or canceled. To get the state of the `click_button` command on device
`s-echodot-1` with id `0` can be done using the following command:
```
query state s-echodot-1 click_button 0
```

If the result of the state command indicates that the command has completed then the `query get_result` command can be 
used to get the result of the command. To get the result of the same `click_button` command, the following command can 
be used:

```
query get_result s-echodot-1 click_button 0
```

This work workflow would be run as follows:

![](05_Query_State_and_Get_Result.png#zoomable)

Rather than manually checking the state before getting the value, the results the `wait_result`
command can be used to wait and get the result of the command as soon as it is available. To wait
and get the result of the `click_button` command with job id `0` for `s-echodot-1` the following
command would be used:

```
query wait_result s-echodot-1 click_button 0
```
Which would result in the following:

![](06_Query_Wait_Result.png#zoomable)

In order to better facilitate using query commands along with automating experiments, variables
can be used to store information from a command, that can be retrieved and used at a later time.
To do this the following command can be used:

```
dev <device> [-s <id or results>] <results> <command> [command args]
```

To execute and store the job id for a `click_button` command on `s-echodot-1` the following command
can be used:

```
dev s-echodot-1 -s id job click_button
```

This result can then be used in subsequent commands by prefixing the variable with an underscore,
using the previous example of storing `0` into a `job` variable would make the following two 
commands equivalent:
```
query wait_result s-echodot-1 click_button _job
query wait_result s-echodot-1 click_button 0
```

The usage of the variables would appear as follows:

![](07_Store_Load_Job_Id.png#zoomable)
