<h1>
  Single GPU Passthrough Tutorial [KVM | Arch Linux]
</h1>
<h4>
  Create a KVM virtual machine and enable single GPU passthrough on an Arch-based distro.
</h4>

<h2>
    Table of Contents
</h2>

* [Introduction](#introduction)
    * [Hardware](#hardware)
    * [Disclaimer](#disclaimer)
* [KVM Install](#kvm)
    * [Part 1: Install](#part1-kvm)
    * [Part 2: Configure](#part2-kvm)
    * [Part 3: Creating the VM](#part3-kvm)
* [Credits & Resources](#credits)
* [Footnotes](#footnotes)

<h2 name="introduction">
    Introduction
</h2>

<b>Difficulty: Intermediate</b>
<br />
Solid foundations of the Linux kernel, virtualization, and hardware is expected.

This guide shows how to install and configure KVM and a GUI in any Arch-based Linux distro, then shows how to enable single GPU passthrough to gain full graphics performance on your guest OS. Note that you will be unable to graphically access your host machine any time the guest is running.

<h3 name="hardware">
  Hardware
</h3>
 
**Requirements:**
 - OS: Arch Linux, Manjaro, or another Arch-based distro
 - CPU: AMD or Intel processor that supports virtualization
 - GPU: Single dedicated or integrated card. If you have both dedicated and integrated graphics or multiple dedicated cards, I recommend setting up passthrough with multiple GPUs to allow you to use both host and guest simultaneously.

**My System:**
 - OS: Arch Linux 5.11.1
 - CPU: AMD Ryzen 7 2700X
 - Motherboard: Gigabyte X470 Aorus Gaming 7
 - GPU: NVidia GeForce GTX 1080

<h3 name="disclaimer">
  Disclaimer
</h3>

You are solely responsible for your own hardware and system. I accept no liability for you choosing to follow this guide and any problems that arise as a result. You can report an issue on GitHub if you're having problems and I will try to get back to it, but I have no guarantees I will be able to assist. You choose to follow this tutorial at your own risk.

<h2 name="kvm">
    KVM Install
</h2>

<h3 name="part1-kvm">
  Part 1: Install
</h3>

**Check to ensure virtualization is enabled.**
```
$ LC_ALL=C lscpu | grep Virtualization
```
If "AMD-V" or "VT-x" is displayed, virtualization is enabled. If not, you'll need to enable it in your BIOS.

**Check to ensure IOMMU is enabled.**
```
$ sudo dmesg | grep IOMMU
```
If it isn't enabled, you'll need to enable this in your BIOS.

**Check to ensure your kernel supports KVM.**
```
$ zgrep CONFIG_KVM /proc/config.gz
```
Look for "CONFIG_KVM_AMD" or "CONFIG_KVM_INTEL" in the list.

**Install KVM packages and graphical virt-manager.**
```
$ sudo pacman -S qemu virt-manager vde2 ebtables bridge-utils dnsmasq openbsd-netcat
```
**Enable KVM services.**
```
$ sudo systemctl enable libvirtd.service
$ sudo systemctl start libvirtd.service
```
**Change user permissions to allow libvirt access**
```
$ sudo usermod -a -G libvirt $(whoami)
$ newgrp libvirt
$ sudo systemctl restart libvirtd.service
```
**Enable nested virtualization (optional)**
<br>
You only need to do this step if you intend to create VMs in your guest machine.
Replace "amd" with "intel" if needed.
```
$ sudo modprobe -r kvm_amd
$ sudo modprobe kvm_amd nested=1
```
Then, edit the file at ```/etc/modprobe.d/kvm.conf``` and add the following line:
```
options kvm_amd nested=1
```
Verify that nested virtualization is enabled.
```
$ cat /sys.module/kvm_amd/parameters/nested
```
