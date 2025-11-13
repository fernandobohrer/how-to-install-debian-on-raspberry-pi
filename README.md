# How to install Debian on Raspberry Pi

## 🚀 Motivation

I wanted to run vanilla [Debian][01] - not [Raspberry Pi OS][02] - on [Raspberry Pi 4][03] single-board computers. The reason for that is related my requirement of having the operating system installed with LUKS based full disk encryption enabled by default.

This document describes hardware and installation details as well as the process I followed.

## 🔧 Hardware details

I'm not using any SD cards on my _RPi_ computers. A 2.5" SSD disk is connected to the _RPi_ via a USB 3.0 port. The operating system boots from this single SSD disk which is also used as installation media during the operating system installation process.

### 🔌 Raspberry Pi bootloader configuration

In order to be able to boot from the USB port, the _RPi_ bootloader needs to be configured to prioritize USB boot. The process of configuring the bootloader is very well documented at [Use Raspberry Pi Imager to update the bootloader][04]. In step 8, where it says _Select SD card and then Write_, select _USB_.

After configuring the _RPi_ to boot from USB, it is time to prepare the installation media.

### 💾 Installation media

The same disk that will be targeted as the destination of the new operating system is going to be used as installation media. To prepare the disk for the Debian installer initialization, follow the steps below:

1. Connect the disk to a Linux machine and create the first partition:

   > [!CAUTION]
   > In my case, the external disk that will later be connected to the _RPi_ is recognized as `/dev/sda` when connected to my computer. Remember to confirm the device before proceeding! Executing the commands below against the wrong device will cause data loss!

   > [!TIP]
   > To confirm the device, you can run `lsblk` before and after connecting the disk to your computer. The use of the `lsblk` command allows for an easy way of identifying the new device.

   > [!WARNING]
   > For the rest of this document, I will continue to use `/dev/sda`. Remember to adjust the device path accordingly to your environment!

   ```bash
   fdisk /dev/sda << EOF
   g
   n
   1
   2048
   +1G
   t
   1
   w
   EOF
   ```

1. Format the partition:

   ```bash
   mkfs -t vfat /dev/sda1
   ```

1. Download the Raspberry Pi 4 UEFI Firmware Image:

   Get the latest available release of the [Raspberry Pi 4 UEFI Firmware Image][05]. At the time this document is being created, the latest available release is `v1.42`:

   ```bash
   axel -a -n 20 https://github.com/pftf/RPi4/releases/download/v1.42/RPi4_UEFI_Firmware_v1.42.zip
   ```

1. Download the Debian ISO:

   Get the latest available [Debian ARM64 netinst ISO image][06]. At the time this document is being created, the latest available release is `13.1.0`:

   ```bash
   axel -a -n 20 https://cdimage.debian.org/debian-cd/current/arm64/iso-cd/debian-13.1.0-arm64-netinst.iso
   ```

1. Mount the partition and transfer the files to the disk:

   ```bash
   mount /dev/sda1 /mnt

   7z x ./debian-13.1.0-arm64-netinst.iso -o/mnt

   unzip ./RPi4_UEFI_Firmware_v1.42.zip -d /mnt

   sync

   umount /mnt
   ```

Now that the downloaded content was transferred to the disk, the installation process can be triggered. To do so, connect the disk, a keyboard and a monitor to the _RPi_ and power it on.

### 🚧 Remove the 3G of RAM limitation

Since you are about to run a 64 bits operating system on your _RPi_, there is no reason to keep the 3G of RAM software limitation enabled.

To disable it, during the _RPi_ initialization, press `ESC` to access the configuration screen and select `Setup` - `Device Manager` - `Raspberry Pi Configuration` - `Advanced configuration` - `Limit RAM to 3G`. Set the option to `Disabled`. This will expose all hardware memory to the operating system. Save the changes and restart the _RPi_.

At this stage, after a few seconds, the Debian installer should show up on the screen.

## 🏗️ Installation details

The Debian installation is straight forward however, the configuration required to properly make use of full disk encryption can lead to some confusion. The Debian project releases a new stable version [approximately every 2 years][07]. Whenever a new stable version is released, I like to install it from scratch.

Considering the requirement of making use of full disk encryption, the interval between Debian stable releases and the number of installations I usually perform, I automated the installation by using a preseed file that can be found [here][08]. More information about Debian preseed files can be found [here][09].

### 🌱 The preseed file

The preseed file is self explanatory however it is important to mention some details:

When using the preseed file available [here][08], download it to your local machine, search for `changeme` and replace all entries configuring the username, the user full name, user password, root password and full disk encryption password of your target Debian installation. Also search for `late_command string` and adjust the `github.com` URL to match the latest available release of the [Raspberry Pi 4 UEFI Firmware Image][05].

With the preseed file ready, it is time to make it available to the Debian installer. To do so, using a terminal, navigate to the folder where the preseed file is and start a temporary webserver using the following python command: `python3 -m http.server 8000`.

If, for any reason, it were to be easier to expose the preseed file from a docker container, the following process could be used:

```bash
docker container run -it -v ./preseed.cfg:/tmp/preseed.cfg -p 8000:8000 debian
apt-get update ; apt-get -y install python3 ; cd /tmp ; python3 -m http.server 8000
```

### 🐧 Installation process

Back to the Debian installer running on the _RPi_, select `Advanced Options` - `Automated Install`. Before informing the local address of the preseed file, you might get prompted to select a network interface. If you have a network cable connected to your _RPi_, select the `enabcm6e4ei0` interface.

Finally, when asked, inform the local address of your preseed file. It should be similar to `http://<your-computer-local-ip-address>:8000/preseed.cfg`.

From this point on, the Debian installation process is completely automated. By the end of the installation process, the _RPi_ is going to be powered off. Just power it on again to make use of your freshly installed vanilla Debian system running on a Raspberry Pi.

[01]: https://www.debian.org/
[02]: https://www.raspberrypi.com/software/operating-systems/
[03]: https://www.raspberrypi.com/products/raspberry-pi-4-model-b/
[04]: https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#imager
[05]: https://github.com/pftf/RPi4/releases/
[06]: https://cdimage.debian.org/debian-cd/current/arm64/iso-cd/
[07]: https://www.debian.org/releases/
[08]: ./preseed.cfg
[09]: https://wiki.debian.org/DebianInstaller/Preseed
