# gpiosvr
An asyncio-based service translating GPIO signals into Linux kernel key events exposed via `/dev/input/event*`.

Developed and tested for Raspberry Pi variants, namely Pi 3B, 4B and Pico 1 connected via USB. On Pi 3B and Pi 4B, [pigpiod](https://abyz.me.uk/rpi/pigpio/pigpiod.html) is used as a backend for evaluating GPIO edges. As it does not work on Pi 5B any more, [libgpiod](https://libgpiod.readthedocs.io/en/latest/) is currently adapted as an alternative backend.

For the adoption of Raspberry Pico, the C daemon from [picod project](https://abyz.me.uk/picod/index.html) is used. By contrast, the client-side Python module of `picod`was completely re-written and extended. Eventually, the GPIOs of a Raspberry Pico, connected to a Linux machine via USB 2.0 can be controlled largely as if they were native GPIOs.

For a minimal setup, one central instance of `gpio_server.py` running on a Linux machine is required. Its main task is to translate pre-configured signals received on corresponding GPIOs into key events. Additionally, a server instance can be accessed from CLI tools via a Unix socket. In a more advanced setup, multiple instances of `gpio_server.py` may run on one Linux machine. In this case, each instance is exclusively attached to one available GPIO chip. For example, on a Raspberry Pi 4B, this might be the native GPIO chip of the 40 pin header, and additionally a Raspberry Pico connected via USB 2.0. Both instances might then be used by clients in the same manner, as long as the desired target is addressed properly.

Since *gpiosvr* is based on asyncio, and asyncio in turn is backed by a C implementation in CPython, the processing runs fast enough for most use cases, even on single board computers.

For apidocs, see [https://xoocoon.github.io/gpiosvr](https://xoocoon.github.io/gpiosvr/).

## Signal definition language

Various signal transmitters can be used with a `gpio_server.py` instance. For translating pulsed and other signals into key events, a `gpio_server.py` instance requires protocol definitions in one or more JSON files. The following is an example for capturing a pulsating door bell signal. On the hardware side, whenever the door bell rings, a rectifier and Z diode emits pulses to GPIO 25. With the following JSON declarations, a `KEY_SOUND` key press is emulated.

```
{
  "evdevName": "pi-sig-injector",
  "gpios": {
    "25": "door-bell"
  },
  "signalProtocols": {
    "door-bell": {
      "basePulse_μs": 9000,
      "lengthTarget_basePulseCount": 5,
      "codeStartBit": 1,
      "preamble_ms": 10,
      "postamble_ms": 12,
      "tolerancePercentage": 30,
      "debounce_μs": 600,
      "signalListener": {
        "mayInject": false
      },
      "keys": {
        "0x1f": {
          "keyName": "KEY_SOUND"
        }
      }
    }
  }
}
```

These declarations state that, whenever at least 5 pulses with a length of 9000 microseconds each, are received on GPIO 25 with a timing tolerance of plus/minus 30%, a pair of a key down and a key up event for `KEY_SOUND` shall be propagated via the corresponding `/dev/input/event*` device. Of course, such emulated "key presses" can be processed by another instance on the same Linux machine, triggering virtually any action. 

To this end, *gpiosvr* features an additional service `key_monitor.py`. It may be deployed in one or more instances, each grabbing and processing an arbitrary subset of key events. Unless such a key monitoring service is deployed, the effect is no different to really pressing a key on a keyboard. This opens up a versatile way for decoupling signal transmitters and receivers in an ecosystem of Linux deamons, optionally orchestrated by systemd.

Another signal source might be an IR remote control. In this case, the processing service can be omitted entirely when there is already a media center software processing the translated key presses. The following is an example for capturing the pulses of an MCE-compatible remote control.

```
{
  "evdevName": "pi-ir-injector",
  "gpios": {
    "19": "Media Center Edition"
  },
  "irProtocols": {
    "Media Center Edition": {
      "baseProtocol": "RC6_MCE",
      "address": "0x800f",
      "irListener": {
        "mayInject": true
      },
      "keys": {
        "0x0411": {
          "keyName": "KEY_VOLUMEDOWN"
        },
        "0x0410": {
          "keyName": "KEY_VOLUMEUP"
        }
      }
    }
  }
}
```

By contrast to the previous example, the configuration relates to a specific IR protocol handler instead of a generic signal handler. This demonstrates that *gpiosvr* can be extended by individual protocol handlers and corresponding JSON configurations.

More complete examples are included in the repo under `templates/`:

- `pi_signal_config.json` includes an example for a light sensor, simply translating any GPIO edge into a pair of key down and key up events.
- `pico_signal_button_config.json` includes an example for translating push button presses, including repeated key events as long as a button is held down.
- `pi_signal_ir_config.json` includes an example for mapping the hex codes produced by a RC6_MCE IR handler to key names.

For a reference of supported configuration keys, please see the documentation of the [signal](https://xoocoon.github.io/gpiosvr/signal.html#configuration-classes) module.

## Key definitions

The key definitions used by *gpiosvr* correspond to the constants used by the Linux kernel, as defined in [input-event-codes.h](https://github.com/torvalds/linux/blob/master/include/uapi/linux/input-event-codes.h).

In the examples above, these are `KEY_SOUND` and `KEY_VOLUMEUP` / `KEY_VOLUMEDOWN`, respectively.

## Key processing language

`key_monitor.py` executes pre-defined commands on the Linux machine whenever a specific key event is triggered. The following is an example for commands to execute whenever the button F12 is pressed (`shortPressCommands`) or held down (`longPressCommands`).

```
"keys": {
    "KEY_F12": {
      "shortPressCommands": [
        [
          "/usr/bin/curl",
          "--netrc-file",
          "/mnt/encrypted/rest.netrc",
          "https://mediabox.local:6863/standby"
        ]
      ],
      "longPressCommands": [
        [
          "/usr/bin/curl",
          "--netrc-file",
          "/mnt/encrypted/rest.netrc",
          "https://mediabox.local:6863/shutdown"
        ]
      ],
      "ledColor": "orange"
    }
}
```

The example also shows that an indicator LED can be aligned with key events. If you come to the conclusion that `key_monitor.py` can be used independently of `gpio_server.py`, you are right. Imagine a little keypad connected to your single board computer. As the keypad already produces genuine key events in the Linux kernel, you do not need a `gpio_server.py` instance in the first place. It is sufficient to set up a `key_monitor.py` instance along with a configuration file like the one above in order to execute pre-defined commands.

More complete examples are included in the repo under `templates/`:

- `key_rc6-mce_config.json` includes an example for controlling a home theater installation via an IR remote control.

For a reference of supported configuration keys, please see the documentation of the [key](https://xoocoon.github.io/gpiosvr/key.html#configuration-classes) module.

# Installation

## Package dependencies

The following packages need to be installed:

- [ctlshell](https://github.com/xoocoon/ctlshell) package
- [pigpio](https://abyz.me.uk/rpi/pigpio/download.html), if *pigpiod* shall be used as a backend for Raspberry Pi up to 4B
- [libgpiod](https://libgpiod.readthedocs.io/en/latest/python_api.html) package, if *libgpiod* shall be used as a backend for Raspberry Pi up to 5B – **Not yet integrated.**
- [evdev](https://pypi.org/project/evdev) package, for handling key events
- [pyserial](https://pypi.org/project/pyserial) package, if Raspberry Pico shall be controlled via USB 2.0
- [python-systemd](https://github.com/systemd/python-systemd) for running `gpio_server.py` and/or `key_monitor.py` as systemd service(s)

Except for the first one, on a Debian-based system, the packages can be installed as follows:

```
sudo apt update && sudo apt install -y pigpiod gpiod python3-libgpiod python3-evdev python3-serial python3-systemd
```

## Linux prerequisites

The Linux kernel on the host machine needs to run the `uinput` kernel module. It is by default included in most Linux distributions. You can check its availability at runtime with the following commands.

```
lsmod | grep uinput  # must yield an output starting with 'uinput'
ls /dev/uinput       # must yield the output '/dev/uinput'
```

## *gpiosvr* package registration

Currently, the *gpisvr* package does not ship with an installation script. As soon as it is cloned or downloaded from GitHub, the python files are ready to use.

However, for setting up `gpio_server.py` and/or `key_monitor.py` as system-wide daemons, it is recommended to add the package to Python's global paths. The following is the recommended approach with a `.pth` file. If you use a virtual environment of Python in lieu of system-wide one, please adjust the paths accordingly.

```
echo GPIOSVR_PARENT_PATH | sudo tee -a "/usr/local/lib/python$( python3 --version | grep -oP '\d\.\d{1,2}' )/dist-packages/xoocoon.pth"
# Check if GPIOSVR_PARENT_PATH is included in the global paths.
python3 -m site
```

... where `GPIOSVR_PARENT_PATH` is the path to the directory containing the *gpiosvr* package.

## CLI tools registration

As *gpiosvr* contains command line tools under `bin/`, you might want to add them to your shell's path, e.g. in `~/.profile`.

The following is one possible way to extend an existing `PATH` entry in `~/.profile`.

```
perl -i -pe 's~^PATH=("?)(.+?)("?)$~PATH=\1\2:GPIOSVR_PATH/bin\3~g' ~/.profile
```

... where `GPIOSVR_PATH` is the path to the directory of the *gpiosvr* package.

## systemd service setup

In the repo under `templates/`, you will find systemd templates for installing `gpio_server.py` and `key_monitor.py` as system-wide systemd services. If you do not want to use systemd as an init system, you can still use the `*.service` files as a reference for command lines.

The files `pi_server.service` and `pico_server.service` serve as examples for services attached to the native GPIOs of a Raspberry Pi, and a Raspberry Pico, respectively. Please adjust the contents of the files according to your environment and deploy them to `/etc/systemd/system`. Then have systemd recognize the new service(s) by executing `sudo systemctl daemon-reload`.

In the case of `pi_server.service`, have systemd start up the service automatically by executing `sudo systemctl enable pi_server.service`.

By contrast, it is **not** recommended to have `pico_server.service` start up automatically. In the system boot process, it is relatively indeterminable when a Raspberry Pico device will really become available. Use an udev rule instead. A sample udev rule is included under `templates/59-pico.rules`. It needs to be adjusted and deployed to `/etc/udev/rules.d/`. It will then start up the service as soon as the specified Pico device becomes available.

`key_monitor.service` serves as an example for monitoring the key events of a `/dev/input/*` device, be it generated by an instance of `gpio_server.py` or not. Note that in the example, `After` and `Requires` dependencies are configured for `pi_server.service`. In the Pico case, change this to `pico_server.service`.

## Raspberry Pico setup

To make a Raspberry Pico hardware work with *gpiosvr*, the microcode contained in `picod/picod.uf2` must be deployed to it. For that, hold down the "BOOT/SEL" button on the Pico, while connected it to a host machine via USB 2.0. In a file browser of your choice, copy the file to the `RPI-RP2` volume. After the automatic reboot, the Pico will be ready to talk to an accordingly configured `gpio_server.py` instance.

The source code for the microcode was obtained from the [picod project](https://abyz.me.uk/picod/index.html).

# Project state

The code for the functionality described above already exists and has matured over years. Hopefully, I will find the time to curate and document the code up to a state eligible for its sharing as OSS.

Please feel free to contact me if the functionality might be useful to you. I might share portions of the code beforehand on a bilateral basis.
