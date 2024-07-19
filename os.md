# CM4 OS installation and configuration

These instructions are for the Ubuntu MATE 20.04. it is EOL, just like ROS1 :,) .

As of the time of writing (mid 2024), `$ hostnamectl` yields:
```
   Static hostname: ubuntu
         Icon name: computer
        Machine ID: ac80877c29c349bea619b6e4d97341e9
           Boot ID: 16cc82b488e94745b9169004680e8b0b
  Operating System: Ubuntu 20.04.5 LTS
            Kernel: Linux 5.4.0-1111-raspi
      Architecture: arm64
```
We are using MATE, since it can run ROS1 and interact with `ublox_driver` (HKUST; TODO: add link)
We are using the server-to-mate install method, which allows for the `ros-noetic-base` to be installed w/o dependency rabbit-hole. We are also using the 64-bit version, since this takes best advantage of the CM4 hardware (ours has 4Gb RAM).

## OS installation

Install Ubuntu 20.04 Server 64-bit.

If your CM4 has eMMC, follow these [instructions](https://www.raspberrypi.com/documentation/computers/compute-module.html#flashing-the-compute-module-emmc).
When using the Raspberry Pi Imager, select `Other general purpose OS` and then `Ubuntu Server 20.04.5 LTS`.

... Else (w/o eMMC) do the same thing on a **FAST** microSD card of your choice. [brand/speed/size recommendations](https://ubuntu-mate.org/raspberry-pi/) on MATE's docsite.

TODO: link installation with MATE specifics
after server installation `sudo apt-get install mate-desktop-full` with lightweight gui option (non-default)

Install `ros-noetic-base` as you would on any ubuntu 20.04 distro.

## OS configuration

**TODO: confirm the following works on MATE? my guess is: no**
If you want to do this using SSH from your main machine, then

~* run `raspi-config` to enable SSH (under Interfacing)~ *maybe?*
* find the current IP address using `ifconfig` **yes**

Update packages
```
sudo apt update
sudo apt upgrade
```

MATE has **serial port hardware enabled by default**, but to confirm: check 
- `/boot/firmware/config.txt` contains: `enable_uart=1`
- `/boot/firmware/cmdline.txt` contains: `console=serial0,115200`

Configure the Device Tree for the CM4 and the IO board by adding the following
at the end of `/boot/firmware/config.txt`

```
# Enable GPIO pin 18 for PPS (not always necessary, but useful for testing)
dtoverlay=pps-gpio,gpiopin=18
# realtime clock
dtoverlay=i2c-rtc,pcf85063a,i2c_csi_dsi
# fan **maybe unneccessary cuz fan already working? **

dtoverlay=i2c-fan,emc2301,i2c_csi_dsi
# Make /dev/ttyAMA0 be connected to GPIO header pins 8 and 10
# This always disables Bluetooth
dtoverlay=disable-bt
```

~Disable the system service that initialises the modem:~ doesn't exist on MATE

Set the timezone:

```
sudo dpkg-reconfigure tzdata
```


Reboot.

## Static IP address

Although it's not essential, you probably want a static IP address.

### Network Manager

Raspberry Pi OS 12 by default uses Network Manager to manage network connections.

You can see the current connections using

```
nmcli con show
```

Assuming the connection on the interface is named `Wired connection 1`, you can change it to a static IP using a command like:

```
nmcli con mod "Wired connection 1" ipv4.method manual ipv4.addresses 192.168.0.5/24 ipv4.gateway 192.168.0.1 \
   ipv4.dns 8.8.8.8 ipv4.dns-search lan
```

You can activate this by doing:

```
nmcli con down "Wired connection 1"
nmcli con up "Wired connection 1"
```

### dhcpcd

Raspberry Pi OS 11 by default uses dhcpcd to manage the network. Edit the section of `/etc/dhcpcd.conf` starting with `Example static IP configuration`.

## Verify OS setup

Check that your kernel includes the necessary have ethernet PTP hardware support

```
ethtool -T eth0
```

You should see:

```
Time stamping parameters for eth0:
Capabilities:
        hardware-transmit
        hardware-receive
        hardware-raw-clock
PTP Hardware Clock: 0
Hardware Transmit Timestamp Modes:
        off
        on
        onestep-sync
        onestep-p2p
Hardware Receive Filter Modes:
        none
        ptpv2-event
```

Note that `PTP Hardware Clock: 0` means that this interface uses `/dev/ptp0` as
its hardware clock.

Check the RTC

```
sudo hwclock --show
```

Check that the current date is correct:

```
date
```

Check support for the fan controller. Do

```
ls /sys/class/thermal
```

You should see:

```
cooling_device0  thermal_zone0
```

## Verify GPS connection

You should verify that the GPS is properly connected. There are two connections

- the serial connection
- the PPS connection

### Serial connection

Assuming you have specified `dtoverlay=disable-bt` above and you have connected the GPS
RX (white) and TX (green) pins to pins 8 and 10 respectively on the J8 HAT connector,
then the serial device will be `/dev/ttyAMA0`.

Do:

```
(stty 9600 -echo -icrnl; cat) </dev/ttyAMA0
```

Here 9600 is the speed. The most common default speed is 9600, but some receivers default to 38400 or 115200.

You should see  lines starting with `$`.
In particular look for a line starting with `$GPRMC` or `$GNRMC`. The number following that should be the current UTC time;
for example, `025713.00` means `02:57:13.00` UTC.
After another 8 commas, there will be a field that should have the current UTC date;
for example, `140923` means 14th Septemember 2023.

### PPS connection

To verify that the PPS connection is working, first configure the SYNC_OUT for input: 

```
echo 1 0 | sudo tee /sys/class/ptp/ptp0/pins/SYNC_OUT
```

Replace the `0` in `ptp0` with whatever `ethtool` said was the number.
`SYNC_OUT` here is the name of the pin to which the PPS is connected. In the `echo 1 0`, 1 means to use the pin for input and 0 means the pin should use input channel 0.


Now do:
```
echo 0 1 | sudo tee /sys/class/ptp/ptp0/extts_enable
```

This means to enable timestamping of pulses on channel 0. In the `echo 0 1`, 0 means channel 0 and 1 means to enable timestamping.


Now see if we're getting timestamps:

```
sudo cat /sys/class/ptp/ptp0/fifo
```

The `cat` command should output a line, which represents a timestamp of an input pulse and consists of 3 numbers: channel number, which is zero in this case, seconds count, nanoseconds count. Repeating the last command will give lines for successive input timestamps.

If `cat` outputs nothing, then it's not working.
