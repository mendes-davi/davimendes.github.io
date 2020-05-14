---
layout: post
title:  "Minimal Raspbian Buster Install"
date:   2020-05-13 21:00:00 +0530
permalink: "how-to-raspbian-minimal-install.html"
---

In this tutorial, i will provide instructions to get a working install of Raspbian Buster for your Raspberry Pi starting with a minimal image. This tutorial covers:
 - Writing the image to SD Card
 - Localisation Options
 - WiFi Setup
 - SSH Logins
 - User Creation
 - Desktop Environment Installation

 This tutorial assumes that you are comfortable with the terminal and are able to install applications, update the system and edit configuration files from the terminal.

## Part 1: Getting Started

You will need to download the Raspbian Buster Lite image from [raspberrypi.org](https://www.raspberrypi.org/downloads/raspbian/) or download the zip file from command line using _wget_.

```bash
wget https://downloads.raspberrypi.org/raspbian_lite_latest -O raspbian_lite.zip
```
## Part 2: Write image to SD Card

**NOTE**: use of the dd ("disk destroyer" ahahah) can overwrite any partition on your machine. If you specify the wrong device in the instructions. Please be careful.

### Locate SD card device and unmount
- Run `lslbk -p` to see which devices are connected.
- The device name will be something like `/dev/mmcblk0` or `/dev/sdX` (`X` indicates a number).
- If any partitions have been mounted, unmount them all using `umount` followed by the mount path.

### Copying the image to SD card
Using the `dd` command, specify the input file `if=` argument with the image file and specify output file `of=` argument with the **correct** device for the SD card.

```bash
dd bs=4M if=raspbian_lite.img of=/dev/sdX status=progress conv=fsync
# In the case of zip file, just pipe the file to dd
unzip -p raspbian_lite.zip | dd of=/dev/sdX bs=4M status=progress conv=fsync
```
## Part 3: Initial Changes to Raspbian Buster Lite
After booting the RPi, the naked tty prompts the login. The default user name is `pi` and the password `raspberry`.

#### **Localisation Options**
The importance of setting the Localisation Options is to type correctly the special characters present in the Brazilian Portuguese language which is my mother language.

1. Using `sudo raspi-config`, select `Localisation Options > Change Keyboard Layout` then select the proper layout, for me Generic 105-Key PC.
2. While still in Localisation Options select `Change Locale`, and change the language, for me this was `en_US.UTF-8 UTF-8`.
3. Before leaving, you should set the `Time Zone` and `Wi-fi Country`.

#### **Setup WiFi**
In order to setup WiFi, it is necessary to edit the `wpa_supplicant.conf` file.
```bash
sudo vi /etc/wpa_supplicant/wpa_supplicant.conf
```

Add the following to the end of the file and replace the fields with the correct WiFi network name and password.
```
network={
    ssid="networkname"
    psk="networkpassword"
}
```
Then reboot the system (`sudo reboot`), and if all the information was correct, a WiFi connection should now be established. You could check that using `ifconfig` looking for the `wlan0` device.

You can test the WiFi connection by pinging this blog!
```bash
ping -c 3 mendes-davi.github.io
```
Now that WiFi is working, it's time to hide the plain text password in `wpa_supplicant.conf`.

Run the following command replacing the `networkname` and `networkpassword` again with the correct WiFi name and password.
```bash
wpa_passphrase "networkname" "networkpassword"
```

The output will yield an encrypted `psk` field which can replace the plain text `psk` field in the `wpa_supplicant.conf` file. Also, remove the commented line containing the plain text password.

**VIM TIP:** This editing can be done very easily inside vim using the `read` command. Inside vi, simply:
```
:read !wpa_passphrase "networkname" "networkpassword"
```
The output of the command will be inserted into the current vim buffer. This allows you to easily replace the previous file content.

## **Apt Sources**
Apt downloads packages from one or more software repositories (sources) and installs them onto your RPi. A repository is generally a network server. The main Apt sources configuration file is at `/etc/apt/sources.list`. You can edit this files (as root) to download source packages by adding the line containing `deb-src` in the `sources.list` file.

```bash
❯ cat /etc/apt/sources.list
deb http://raspbian.raspberrypi.org/raspbian/ buster main contrib non-free rpi
# Uncomment line below then 'apt-get update' to enable 'apt-get source'
deb-src http://raspbian.raspberrypi.org/raspbian/ buster main contrib non-free rpi
```
Finally, update your system by running:
```bash
sudo apt update && sudo apt upgrade -y
```

## **Bluetooth devices**
The Raspbian Buster image comes with the `bluez` and `bluez-utils` packages, providing the `bluetoothctl` utility. Pairing a device from the shell is very simple and can be easily automated. For that matter, the Arch Linux Wiki provides a good article with instructions: [Bluetooth](https://wiki.archlinux.org/index.php/Bluetooth)

You might want to add your user to the `bluetooth` group to avoid problems. Replace `username` with you user name and use the following command:
```bash
sudo usermod -a -G bluetooth username
```
Later on this tutorial, a new user will be created to replace the default `pi` account. Additionally, The [How to Geek Blog](https://www.howtogeek.com/50787/add-a-user-to-a-group-or-second-group-on-linux/) provides a very nice introduction to `groups` in a linux environment.

## Part 4: Enable SSH Logins
Working outside the RPi via SSH is a must have. I do this so i can log into my Raspberry Pi from the Terminal of my main production machine.

Run the `sudo raspi-config` command, and select `Interfacing Options > SSH` to enable SSH.

Alternatively, use `systemctl` to start the service
```bash
sudo systemctl enable ssh
sudo systemctl start ssh
```

You should change the private host ID keys. Otherwise, they have the same default values. Run the following commando to remove the default keys and generate new keys:

```bash
sudo rm /etc/ssh/ssh_host_* && sudo dpkg-reconfigure openssh-server
```

### **TIPS**
- [Passwordless SSH access](https://github.com/raspberrypi/documentation/blob/master/remote-access/ssh/passwordless.md)
- [SSHFS (SSH Filesystem)](https://github.com/raspberrypi/documentation/blob/master/remote-access/ssh/sshfs.md): SSHFS allows you to mount a Raspberry Pi's files over an SSH session. This allows you to use your computer's tools (file manager, text editor, image processing tools, and so on) to edit files directly on the Pi.

## Part 5: Create a new user
Raspbian comes with the default `pi` user and the password this user is raspberry. You should first create a new user account and then remove the default pi account. It's also necessary to add the new user to previous `pi` user groups to access the gpio, i2c, video and so on.

First, check the groups that the `pi` user is on by running:
```bash
groups pi
# The output is something like:
# pi : pi adm dialout cdrom sudo audio video plugdev games users input netdev spi i2c gpio
```

To create a new user, run the following command replacing `username` with the desired user name of choice. The `-G` argument adds the users to the groups in a comma separated list.
```bash
sudo useradd -m username -G sudo,adm,dialout,cdrom,audio,video,plugdev,games,users,input,netdev,spi,i2c,gpio
# Create a password for the new user
sudo passwd username
```
Log out and log into the new user account. Now it's time to remove the `pi` account. You can do so by:
```bash
sudo deluser -remove-home pi
```
Also, remove the `/etc/sudoers.d/010_pi-nopasswd` file. This file allows the default `pi` account to use the sudo command without entering password. Since the `pi` account has been removed, the file is no longer necessary.

```bash
sudo rm /etc/sudoers.d/010_pi-nopasswd
```

## Part 6: Installing a Desktop Environment (LXDE)
This part focuses on installing GUI on top of Raspbian Lite. For this we need: Display Server; Desktop Environment and a Window Manager. I will not cover the installation of a Login Manager because it's very convenient for me to boot into the command line and start the graphical server.

First, install the Display Server using the following command:
```bash
sudo apt install xinit xserver-xorg
```

To install the Desktop Environment we need the `lxde-core` package and the `lxappearance` to change the look of applications. There will be a lot of dependent packages to install.
```bash
sudo apt-get install lxde-core lxappearance
```

Alternatively, if you want a more cohesive experience with all the pre-installed software of LXDE, just use:
```bash
sudo apt-get install lxde
```
For LXDE, the Openbox Window Manager is already installed in the `lxde-core` package. When you log in the tty, you cant start the X server using `xinit /usr/bin/openbox-lxde`. It's easier to create a `.xinitrc` in the home directory to start the X server automatically after the tty login.

In the `~/.xinitrc` file add the following lines to use the `openbox-lxde` as you default X session.
```bash
#!/bin/sh
exec openbox-lxde
```

Now, we automate the process of starting the X server after the tty login using the `~/.profile` file. First, copy the default `profile` file:
```bash
cp /etc/skel/.profile ~/.profile
```

Then, append the following commands:
```bash
if [[ "$(tty)" = "/dev/tty1" ]]; then
    pgrep openbox || startx
fi
```

## Part 7: Make it your own
This turned out to be a longer tutorial than what i had in mind at first. If you have followed to the end, you should have a minimal LXDE install with Raspbian. From now on, you can tweak it to your preferences and install you preferred software.

As for me, I'm going to create a backup of the SD card. Bye!
