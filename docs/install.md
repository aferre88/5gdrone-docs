# Setting up the SorusBoxScan Device

!!! note
    This page will guide you through the installation of the software required to run the device. [Details on the physical mounting of the device](https://www.waveshare.com/w/upload/d/d8/SIM8202G-M2-5G-HAT-Assembly-en.jpg) can be found on the manufacturer's website.

    You might also find the [official documentation for the SIM8200EA-M2 5G HAT](https://www.waveshare.com/wiki/SIM8200EA-M2_5G_HAT) helpful.

## Prerequisites

1. The device must be physically assembled.
2. A SIM card must be installed. SIM cards with PIN codes are currently not supported, though support may be added in the future. The SIM must provide network access.
3. The device must have Ubuntu installed. You can follow the [official Ubuntu documentation](https://ubuntu.com/download/raspberry-pi) to complete this step.
4. You must have root console access to the device, either physically or via SSH.

## Connecting to the Network

First, install the required packages:

```console
sudo apt update
sudo apt install libqmi-utils udhcpc
```

Then, establish the connection. You may want to create a shell script to connect to the network on startup:

``` bash title="startnet.sh"
#!/bin/sh

/usr/bin/qmi-network /dev/cdc-wdm0 start
/usr/bin/ip link set wwan0 up
/usr/sbin/udhcpc -q -f -n -i wwan0
```

If you need to set the APN, do so in `/etc/qmi-network.conf`. For example, for Movistar in Spain:

``` console title="/etc/qmi-network.conf"
APN=telefonica.es
```

## Installing the control software

??? info "What is the control software"
    The [control software](https://github.com/aferre88/5gdrone) is a Python application that connects to the HAT via serial interface and to the MQTT server via the Raspberry Pi's network connection (whether through the HAT modem, Ethernet port, or Wi-Fi antenna).

    Upon establishing a connection with the HAT, the software issues two AT commands:

    - `AT+CGPSINFO=1` to start receiving geopositioning data.
    - `AT+CPSI=1` to start receiving network quality and connectivity data.

    The HAT will then send geopositioning and network quality data every second. For each valid network data message, the control software sends a geo-tagged message to the MQTT server.

    === "Messages received from the HAT"
        ```
        DEBUG:hat:< +CGPSINFO: XXX.XXXXXX,N,XXX.XXXXXX,W,220725,170219.0,654.9,0.0,0.0
        DEBUG:hat:< +CPSI: LTE,Online,214-07,0x6FB9,73795850,13,EUTRAN-BAND3,1301,5,5,-192,-773,-374,3
        ```

    === "Message sent to the MQTT server"
        ```
        DEBUG:mqtt:signal/EUTRAN-BAND3> XXX.XXXXXX,N,XXX.XXXXXX,W,220725,170219.0,654.9,0.0,0.0,LTE,Online,214-07,0x6FB9,73795850,13,EUTRAN-BAND3,1301,5,5,-192,-773,-374,3
        ```

    If no geopositioning data is available, the message is discarded.

    In some cases, the HAT may report signal quality for multiple bands. When this occurs, the software will send multiple messages to the MQTT server, each containing the same position and timestamp but for different bands.

The control software is distributed as a [Snap package](https://snapcraft.io/). You can download the latest Snap package from the [Releases section on GitHub](https://github.com/aferre88/5gdrone/releases).

??? tip "Building a custom Snap package"
    If you need to run a modified version of the software, install Snapcraft and build your own package:

    ``` console title="Building a custom snap package"
    sudo apt update
    sudo apt install snapcraft git
    git clone https://github.com/aferre88/5gdrone.git
    cd 5gdrone
    # Make the necessary changes
    snapcraft
    ```

    A `.snap` file will be created in the directory. This is the package you'll need to install instead of the downloaded one.

``` sh
wget https://github.com/aferre88/5gdrone/releases/download/2024.10.1/5gdrone_2024.10.1_arm64.snap
sudo snap install ./5gdrone_2024.10.1_arm64.snap --dangerous # (1)
```

1. The "dangerous" flag may be required as the package is unsigned.

### Pointing to the correct MQTT server

Configure the MQTT server with either one line per variable or a one-liner:

=== "One variable per line"

    ``` console
    sudo snap set 5gdrone mqtt.host=mqtt.uc3m.es
    sudo snap set 5gdrone mqtt.port=8883
    sudo snap set 5gdrone mqtt.user=username
    sudo snap set 5gdrone mqtt.pass=password
    ```

=== "One liner"

    ``` console
    sudo snap set 5gdrone mqtt.host=mqtt.uc3m.es mqtt.port=8883 mqtt.user=username mqtt.pass=password
    ```

Then restart the service:

``` console
sudo snap restart 5gdrone
```

### Troubleshooting
#### Setting the log level to debug

``` console
sudo snap set 5gdrone loglevel=DEBUG
sudo snap restart 5gdrone
```

#### Follow the logs
``` console
sudo journalctl -u snap.5gdrone.5gdrone.service -f
```