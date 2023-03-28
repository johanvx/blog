---
title: Install Arch Linux on a laptop
publish_date: 2023-01-15
tags:
  - linux
---

<!-- Explanation of why I decided to install Arch Linux, or rather, why I decided to install a GNU/Linux system on my computer for work. -->

## About the machine

Thinkpad neo 14 (Intel version)

- CPU: Intel Core i5-12500H
- RAM: 16 GB
- Storage: 512 GB SSD

## Table of contents

- [Prologue](#prologue)
- [Pre-installation](#pre-installation)
  - [Set the console keyboard layout](#set-the-console-keyboard-layout)
  - [Verify the boot mode](#verify-the-boot-mode)
  - [Connect to the Internet](#connect-to-the-internet)
  - [Update the system clock](#update-the-system-clock)
  - [Partition the disk(s)](#partition-the-disks)
    - [`/mnt/boot`](#mntboot)
    - [`[SWAP]`](#swap)
    - [`/mnt`](#mnt)
    - [Check and write](#check-and-write)
  - [Format the partitions](#format-the-partitions)
  - [Mount the file systems](#mount-the-file-systems)
- [Installation](#installation)
  - [Select the mirrors](#select-the-mirrors)
  - [Install essential packages](#install-essential-packages)
- [Configure the system](#configure-the-system)
  - [Fstab](#fstab)
  - [Chroot](#chroot)
  - [Time zone](#time-zone)
  - [Localization](#localization)
  - [Network configuration](#network-configuration)
  - [Root password](#root-password)
  - [Boot loader](#boot-loader)
- [Reboot](#reboot)
- [Epilogue](#epilogue)

## Prologue

To install Arch Linux, it is highly recommended to read
[Arch Linux's Installation Guide], then try to install Arch Linux manually or
with an installer. The official `archinstall` is nice.

Personally, I prefer to install Arch Linux step by step as this is a more
customizable way. In the following instruction, I'll start from the
[Set the console keyboard layout] section of Arch Linux's Installation Guide.

[Arch Linux's Installation Guide]: https://wiki.archlinux.org/title/Installation_guide
[Set the console keyboard layout]: https://wiki.archlinux.org/title/Installation_guide#Set_the_console_keyboard_layout

## Pre-installation

### Set the console keyboard layout

I prefer to swap the left Control key and the Caps Lock key.

```
console:

    # mkdir -p /usr/local/share/kbd/keymaps
    # cp /usr/share/kbd/keymaps/i386/qwerty/us.map.gz /usr/local/share/kbd/keymaps/custom.map.gz
    # cd /usr/local/share/kbd/keymaps
    # gunzip custom.map.gz
    # vim custom.map.gz

/usr/local/share/kbd/keymaps/custom.map:

    ...
-   keycode  29 = Control
+   keycode  29 = Caps_Lock
    ...
-   keycode  58 = Caps_Lock
+   keycode  58 = Control

console:

    # touch /etc/vconsole.conf
    # vim /etc/vconsole.conf

/etc/vconsole.conf:

    KEYMAP=/usr/local/share/kbd/keymaps/custom.map

console:

    # systemctl restart systemd-vconsole-setup.service
```

### Verify the boot mode

```
console:

    # ls /sys/firmware/efi/efivars
```

The following assumes that the system is booted in **UEFI** mode.

### Connect to the Internet

Check network devices.

```
console:

    # ip link
```

The following assumes that the system connects to the Internet with Wi-Fi device
`wlan0`.

There are many ways to search available Wi-Fis, and I personally prefer the
following way.

```
console:

    # iwctl

iwd:

    [iwd]# station wlan0 get-networks
```

Now, connect to the Internet with `wpa_supplicant`. Note that `iwctl` and
`wpa_cli` suffice in general.

```
console:

    # touch /etc/wpa_supplicant.conf
    # vim /etc/wpa_supplicant.conf

/etc/wpa_supplicant.conf:

    ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=wheel
    update_config=1

    network={
        ssid="SSID"
        key_mgmt=WPA-EAP
        identity="identity"
        password="password"
    }

console:

    # wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant.conf
    # dhcpcd --background
```

Verify the connection with `ping`.

```
console:

    # ping archlinux.org
```

### Update the system clock

Check the current time setting and set the time zone.

```
console:

    # timedatectl status
    # timedatectl set-timezone <Region>/<City>
```

### Partition the disk(s)

To be clear, I use single-system machines, so I don't care about
[_Dual boot with Windows_](https://wiki.archlinux.org/title/Dual_boot_with_Windows).

Identify block devices with `fdisk`.

```
console:

    # fdisk -l
```

The following assumes that the system is going to be installed in
`/dev/nvme0n1`.

```
console:

    # fdisk /dev/nvme0n1

fdisk:

    Command (m for help): p
    Command (m for help): g
    Command (m for help): p
```

For UEFI, 3 mount points are needed: `/mnt/boot`, `[SWAP]` and `/mnt`.

#### `/mnt/boot`

```
fdisk:

    Command (m for help): n
    Partition number (1-..., default 1):
    First sector (2048-...., default 2048):
    Last sector, ... (2048-..., default ...): +300M

    Created a new partition 1 of type 'Linux filesystem' and of size 300MiB.
    Partition #1 contains a vfat signature.

    Do you want to remove the signature? [Y]es/[N]o: y

    The signature will be removed by a write command.
```

#### `[SWAP]`

To be honest, I don't know much about swap, but I know that I have a large hard
disk. :laughing:

So, I decide to make the swap partition as **1.5 times** large as the RAM, that
is **24 GB**.

```
fdisk:

    Command (m for help): n
    Partition number (2-..., default 2):
    First sector (..., default ...):
    Last sector, ... (..., default: ...): +24G
```

#### `/mnt`

```
fdisk:

    Command (m for help): n
    Partition number (3-..., default 3):
    First sector (..., default ...):
    Last sector, ... (..., default ...):
```

#### Check and write

```
fdisk:

    Command (m for help): p
    Command (m for help): w
```

### Format the partitions

Check the partitions with `fdisk -l`. The following assumes the below layout.

**Layout (UEFI)**

| Mount point | Partition        |
| ----------- | ---------------- |
| `/mnt/boot` | `/dev/nvme0n1p1` |
| `[SWAP]`    | `/dev/nvme0n1p2` |
| `/mnt`      | `/dev/nvme0n1p3` |

```
console:

    # mkfs.fat -F 32 /dev/nvme0n1p1
    # mkswap /dev/nvme0n1p2
    # mkfs.ext4 /dev/nvme0n1p3
```

### Mount the file systems

```
console:

    # mount /dev/nvme0n1p3 /mnt
    # mount --mkdir /dev/nvme0n1p1 /mnt/boot
    # swapon /dev/nvme0n1p2
```

## Installation

### Select the mirrors

A manual modification after executing `reflector` may be necessary.

```
console:

    # reflector --save /etc/pacman.d/mirrorlist --protocol https --latest 20 --sort rate --country <Country 1>[,<Country 2>,...]
    # vim /etc/pacman.d/mirrorlist
```

### Install essential packages

Some of the following packages can be installed in the future, but I prefer to
install them now.

```
console:

    # pacstrap -K /mnt base base-devel linux linux-firmware man-db man-pages reflector vim wpa_supplicant dhcpcd fish git openssh grub efibootmgr
```

## Configure the system

### Fstab

```
console:

    # genfstab -U /mnt >> /mnt/etc/fstab
```

### Chroot

```
console:

    # arch-chroot /mnt
```

### Time zone

```
`arch-chroot`ed console:

    # ln-sf /usr/share/zoneinfo/<Region>/<City> /etc/localtime
    # hwclock --systohc
```

### Localization

Uncomment needed locales in `/etc/locale.gen`, then generate the locales. The
following assumes that `en_US.UTF-8 UTF-8` has been uncommented.

```
`arch-chroot`ed console:

    # vim /etc/locale.gen
    # locale-gen
```

Set the `LANG` variable.

```
`arch-chroot`ed console:

    # touch /etc/locale.conf
    # vim /etc/locale.conf

/etc/locale.conf:

    LANG=en_US.UTF-8
```

**Repeat [Set the console keyboard layout](#set-the-console-keyboard-layout) in
the `arch-chroot`ed system if wanted.**

### Network configuration

The following assumes that the hostname is set as `myhostname`.

```
`arch-chroot`ed console:

    # touch /etc/hostname
    # vim /etc/hostname

/etc/hostname:

    myhostname

`arch-chroot`ed console:

    # vim /etc/hosts

/etc/hosts:

    127.0.0.1   localhost
    ::1         localhost
    127.0.1.1   myhostname
```

### Root password

```
`arch-chroot`ed console:

    # passwd
```

### Boot loader

I use `grub` and `efibootmgr`, which have been already installed in the
[Install essential packages](#install-essential-packages) step.

For machines that have an Intel CPU, install `intel-ucode`; for those that have
an AMD CPU, install `amd-ucode`.

```
`arch-chroot`ed console:

    # pacman -S intel-ucode
```

_Tip: For multi-system users, `os-prober` may be needed._

Install GRUB to the disk, then generate the main configuration file.

```
`arch-chroot`ed console:

    # mkdir /boot/grub
    # grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
    # grub-mkconfig -o /boot/grub/grub.cfg
```

## Reboot

Remember to remove the installation medium (e.g. USB flash drives) before
rebooting the computer.

```
`arch-chroot`ed console:

    # exit

console:

    # umount -R /mnt
    # reboot
```

## Epilogue

Now, Arch Linux is installed on the laptop. I'll write a new blog post about
setting up this machine.