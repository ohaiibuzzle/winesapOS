# Mac Linux Gaming Stick

Linux gaming, on a stick, designed for Mac enthusiasts. This is an opinonated take on creating a portable USB flash drive with Linux installed to enable gaming on a Mac (or any laptop) via Steam and Proton/Wine.

## Why?

mac OS limitations:

- No 32-bit support. The latest version is now 64-bit only. As of August 2020, there are [less than 70 full PC games](https://www.macgamerhq.com/opinion/32-bit-mac-games/) (i.e., not apps) on mac OS that are available as 64-bit.
- As of August 2020, [77% of Steam games run on Linux](https://www.protondb.com/).
- Steam Play's Proton is only [supported on Linux](https://github.com/ValveSoftware/Proton/wiki/Requirements) ([not mac OS](https://github.com/ValveSoftware/Proton/issues/1344)).
- Old and incomplete implementation of OpenGL.
- No native Vulkan support.
    - MoltenVK is [incomplete due to missing functionality in Apple's Metal API](https://github.com/KhronosGroup/MoltenVK/issues/203).
- Linux has better gaming support because it supports 32-bit applications, DirectX (via Wine with WineD3D, DXVK, and/or Vkd3d), OpenGL, and Vulkan.

## Goals

Goals:

- Portability. The flash drive should be bootable on both BIOS and UEFI systems.
- Gaming support out-of-the-box.
- Minimze writes to the flash drive to improve its longevity.
- Full backups via BtrFS and Snapper.
- Automatic operating system updates are disabled. Updates should always be intentional and planned.
- Battery optimizations.
- As much reproducible automation as possible via Ansible.
    - Any manual steps will be documented in this README file.

Not planned to support:

- Built-in sound.
- Built-in WiFi and/or Bluetooth.

It is easier and more reliable to buy additional hardware and use a USB-C hub than to rely on hacky Linux drivers for Mac. Workarounds do exist for [sound](https://github.com/davidjo/snd_hda_macbookpro) and [WiFi](https://gist.github.com/roadrunner2/1289542a748d9a104e7baec6a92f9cd7#gistcomment-3080934).

## Target Hardware

Mac:

- 2016-2017 Macbook Pro.

This guide should work for older models of Macs as well. Compatibility may vary with the latest Mac hardware.

Suggested hardware to buy:

- USB-C hub with USB-A ports and a 3.5mm audio port.
- Fast USB flash drive.
- WiFi USB adapter.
- Bluetooth USB adapter.
- USB speakers.

## Planning

- Test with Ubuntu 20.04 and build automation using Ansible.
    - Install Linux onto a USB flash drive.
    - Optimize the file systems to decrease writes which will increse the longevity of the flash drive.
    - Automatic BtrFS backups.
    - Setup and configure the system for gaming.
    - Optimize Linux for maximum battery usage on a laptop.
    - Boot the flash drive on a Mac.
- Switch to elementary OS 6 (ETA: October 2020).
- Switch to Linux kernel 5.9 (ETA: October 4th 2020).

## Setup

### Linux Installation

It is recommended to use a virtual machine with USB passthrough to setup the USB flash drive. This will avoid ruining the bootloader and/or storage devices on the actual computer.

virt-manager:

```
File > New Virtual Machine > Local install media (ISO image or CDROM) > Forward > Choose ISO or CDROM install media > Browse... > ubuntu-20.04.1-desktop-amd64.iso > Forward > Forward (keep default CPU and RAM settings) > uncheck "Enable storage for this virtual machine" > Forward > check "Customize configuration before installation" > Finish > Add Hardware > USB Host Device > (select the device, in my case it was "004:004 Silicion Motion, Inc. - Taiwan (formerly Feiya Technology Corp.) Flash Drive") > Finish > Boot Options > (check the "USB" option to allow it to be bootable to test the installation when it is done) > Apply > Begin Installation
```

The elementary OS and Ubuntu installers are extremely limited when it comes to custom partitions. It is not possible to specify a BIOS or GPT partition table, customize BtrFS subvolumes, or set partition flags. Instead, use the `parted` command to format the flash drive. DO AT YOUR OWN RISK. DO NOT USE THE WRONG DEVICE.

```
$ lsblk
$ sudo dd if=/dev/zero of=/dev/<DEVICE> bs=1M count=5
$ sudo parted /dev/<DEVICE>
# GPT is required for UEFI boot.
(parted) mklabel gpt
# An empty partition is required for BIOS boot backwards compatibility.
(parted) mkpart primary 2048s 2M
# EFI partition.
(parted) mkpart fat32 primary 2M 500M
(parted) set 2 boot on
(parted) set 2 esp on
# 8GB swap.
(parted) mkpart primary linux-swap 500M 8500M
(parted) set 3 swap on
# Root partition using the rest of the space.
(parted) mkpart primary btrfs 8500M 100%
(parted) quit
```

#### Ubuntu 20.04

Start the installer:

```
Install Ubuntu > (select the desired language) > Continue (select the desired Keyboard layout) > Continue > (check "Normal Installation", "Download updates while installing Ubuntu", and "Install third-party software for graphics and Wi-Fi hardware and additional media formats") > Continue > (select "Something else" for the partition Installation type) > Continue
```

Configure the partitions:

```
/dev/<DEVICE>1 > Change... > Reserved BIOS boot area > OK
/dev/<DEVICE>2 > Change... > EFI System Partition > OK
/dev/<DEVICE>3 > Change... > swap area > OK
/dev/<DEVICE>4 > Change... > Use as: btrfs journaling file system, check "Format the partition:", Mount pount: / > OK
```

Finish the installation: `Install Now`

### Legacy BIOS Boot

Macs [made after 2014](https://twocanoes.com/boot-camp-boot-process/) do not support legacy BIOS boot. For older computers, it can be installed by rebooting and running the command below. Use the same USB flash drive device. This will enable both legacy BIOS and UEFI boot.

```
$ sudo grub-install --target=i386-pc /dev/<DEVICE>
```

### Optimize the File Systems

Minimize writes to the disk by using the included `tmpfs` Ansible role. For system stability, it is recommended to not set the swappiness level to 0.

```
$ cat inventory_stick.ini
linux-stick ansible_host=<VM_IP_ADDRESS> ansible_user=ekultails
$ cat playbook_tmpfs.yaml
---
- hosts: linux-stick
  roles:
    - name: tmpfs
      vars:
        tmpfs_vm_swappiness: 10
$ ansible-playbook -i inventory_stick.ini playbook_tmpfs.yaml --become --ask-become-pass
```

Also configure the root and home file systems to use new mount options that will lower the amount of writes and evenly spread the wear on the flash drive: `noatime,nodiratime,ssd_spread` (ssd_spread is for BtrFS only).

```
$ sudo vim /etc/fstab
UUID=<UUID>    /        btrfs    defaults,subvol=@,noatime,nodiratime,ssd_spread        0    1
UUID=<UUID>    /home    btrfs    defaults,subvol=@home,noatime,nodiratime,ssd_spread    0    2
```

## License

GPLv3