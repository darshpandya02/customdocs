---
title: "Installation"
linkTitle: "Installation"
weight: 1
description: >
    Install the mrg-iot command-line tool, and the optional video player used for camera streams.
---

Everything you do with SPHERE IoT devices is driven by one tool,
[`mrg-iot`](https://pypi.org/project/mrg-iot/), which runs on your own machine. This
page installs it. Once it is on your `PATH`, continue with the
[Quickstart](https://mergetb.gitlab.io/testbeds/sphere/sphere-docs/docs/experimentation/iot-devices/quickstart/)
or the full
[Using mrg-iot](https://mergetb.gitlab.io/testbeds/sphere/sphere-docs/docs/experimentation/iot-devices/using-mrg-iot/)
guide.

{{% alert title="Beta" color="warning" %}}
IoT device support is currently in **beta**. Report issues and share feedback with the NEU IoT Facility.
{{% /alert %}}

## Requirements

| Requirement | Notes |
|---|---|
| **Python 3.9+** | 3.9 through 3.12 are supported and CI-tested. |
| **Network access** | To the portal (`grpc.sphere-testbed.net:443`) and the SSH jump host (`jump.sphere-testbed.net:2022`). |
| **[VLC](https://www.videolan.org/vlc/)** | Optional. Only needed for `stream open`, which launches viewers for you. Any RTSP-capable player works instead. It is not a pip package, so install it from your OS package manager. |

A SPHERE account is **not** needed to install the tool, only to use it. Account setup
is covered in
[Using mrg-iot §1](https://mergetb.gitlab.io/testbeds/sphere/sphere-docs/docs/experimentation/iot-devices/using-mrg-iot/#1-create-a-sphere-account).

## Install `mrg-iot`

The supported install path is [`pipx`](https://pipx.pypa.io/), which puts `mrg-iot`
on your `PATH` in its own isolated environment:

```sh
pipx install mrg-iot
```

Or into an ordinary virtual environment:

```sh
python3 -m venv .venv && source .venv/bin/activate
pip install mrg-iot
```

## Verify

```sh
mrg-iot --version
mrg-iot --help
```

`--help` is worth reading once: it lists every command group and, at the bottom, what
each name defaults to if you don't pass its flag. That is what makes every command in
these guides scriptable exactly as written.

![The mrg-iot help output, listing its command groups, then the derived defaults for project, devices, experiment, xdc, realization, network, and duration](overview.png#zoomable)

{{% alert title="`mrg-iot: command not found`" color="info" %}}
`pipx` installs to `~/.local/bin`, which may not be on your `PATH`. Run
`pipx ensurepath`, open a new shell, and try `mrg-iot --version` again.
{{% /alert %}}

## Install a video player (optional)

Camera-equipped devices are proxied over RTSP. `mrg-iot stream open` launches **VLC**
viewers for you, so install it if you want that. It is only needed for that one
command: any RTSP-capable player works, and
[`mrg-iot stream url`](https://mergetb.gitlab.io/testbeds/sphere/sphere-docs/docs/experimentation/iot-devices/using-mrg-iot/#camera-streams-mrg-iot-stream)
prints the addresses so you can open them in whatever you already use.

| Platform | Command |
|---|---|
| macOS | `brew install --cask vlc` |
| Debian / Ubuntu | `sudo apt-get install vlc` |
| Fedora | `sudo dnf install vlc` |
| Windows | [Download from videolan.org](https://www.videolan.org/vlc/) |

Confirm with `vlc --version`. Neither `pip` nor `pipx` can install VLC.

## macOS: certificate errors

macOS Python often ships without a usable CA bundle, so portal calls fail with
`certificate verify failed`. Export the certifi one:

```sh
export SSL_CERT_FILE="$(python -m certifi)"
export REQUESTS_CA_BUNDLE="$SSL_CERT_FILE"
```

Add both lines to your shell profile to make it stick. `mrg-iot` detects this case
and prints the same two lines when it happens.

## Next

- **[Quickstart](https://mergetb.gitlab.io/testbeds/sphere/sphere-docs/docs/experimentation/iot-devices/quickstart/)**: log in and drive a real device in a handful of commands.
- **[Using mrg-iot](https://mergetb.gitlab.io/testbeds/sphere/sphere-docs/docs/experimentation/iot-devices/using-mrg-iot/)**: the complete workflow, starting with account setup.
- **[Troubleshooting & FAQ](https://mergetb.gitlab.io/testbeds/sphere/sphere-docs/docs/experimentation/iot-devices/troubleshooting/)**: if the install or the first command fails.
