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
* [Tutorial](#tutorial)
    * [Part 1: Install and Configure](#part1)
    * [Part 2: Creating the VM](#part2)
    * [Part 3: Configuring Passthrough](#part3)
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
 - OS: Arch Linux x86_64
 - CPU: AMD Ryzen 7 2700X
 - Motherboard: Gigabyte X470 Aorus Gaming 7
 - GPU: NVidia GeForce GTX 1080

<h3 name="disclaimer">
  Disclaimer
</h3>

You are solely responsible for your own hardware and system. I accept no liability for you choosing to follow this guide and any problems that arise as a result. You can report an issue on GitHub if you're having problems and I will try to get back to it, but I have no guarantees I will be able to assist. You choose to follow this tutorial at your own risk.

<h2 name="tutorial">
    Tutorial
</h2>

<h3 name="part1">
  Part 1: Install and Configure
</h3>

**Check to ensure virtualization is enabled**
```
$ LC_ALL=C lscpu | grep Virtualization
```
If "AMD-V" or "VT-x" is displayed, virtualization is enabled. If not, you'll need to enable it in your BIOS.

**Check to ensure IOMMU is enabled**
```
$ sudo dmesg | grep IOMMU
```
If it isn't enabled, you'll need to enable this in your BIOS.

**Check to ensure your kernel supports KVM**
```
$ zgrep CONFIG_KVM /proc/config.gz
```
Look for "CONFIG_KVM_AMD" or "CONFIG_KVM_INTEL" in the list.

**Install KVM packages and graphical virt-manager**
```
$ sudo pacman -S qemu virt-manager ovmf vde2 ebtables bridge-utils dnsmasq openbsd-netcat
```
**Enable KVM services**
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
**Passing IOMMU and VFIO to kernel**
You'll need to pass both the IOMMU and the VFIO paramaters to the kernel.<br>
If you use GRUB, edit the file at ```/etc/default/grub``` and add the following to the ```GRYB_CMDLINE_LINUX_DEFAULT``` line:
```
iommu=1 amd_iommu=on rd.driver.pre=vfio-pci
```
For Intel, use "intel_iommu" instead of "amd_iommu."<br>
Note that this is a read-only file, so you'll need editing permissions.<b>
It's a good idea to reboot your system before continuing.

**Check IOMMU group mapping**
Create the "check-iommu.sh" file and use the following code:
```
#!/bin/bash
shopt -s nullglob
for g in /sys/kernel/iommu_groups/*; do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```
Run the script and search for the group that includes your GPU. There are probably multiple devices in that group, so be sure to note them all down since you'll need to pass the entire group to your VM.

<h3 name="part2">
  Part 2: Creating the VM
</h3>

**Create virtual machine** <br>
Open ```virt-manager``` and create a new VM (top left corner).<br>
Choose the type of install, probably local install media.<br>
Make sure you've download your desired OS and browse for the file on your system.
Virt Manager should automatically detect the OS, but if not, you can manually select it.<br>
Choose how much memory and CPU cores you'd like to use, then how much storage you'd like to allocate to the guest.<br>
Name your guest and select "Customize configuration before install"

**Customize settings** <br>
Under "Overview," choose "UEFI...OVMF_CODE.fd" as Firmware.
Under "Boot Options," choose "Enable boot menu" (optional).
Choose "Add Hardware," choose "USB Host Device" and select any USB devices you'd like, such as your keyboard and mouse.<br>

**Virtio drivers for Windows guests (optional)** <br>
If you're creating a Windows VM, you can use virtio drivers.<br>
Download the drivers [here](https://github.com/virtio-win/virtio-win-pkg-scripts/blob/master/README.md).<br>

Select "Add Hardware" in Virt Manager, choose "Storage," set device type to "CDROM" and locate your virtio drivers.<br>
Then, select the "NIC" menu and choose "virtio" as device model.<br>

**Install OS** <br>
We will do more customization to this VM later once we set up passthrough, but let's first intall the OS by choosing "Begin Installation."<br>
After installing your desired OS, check to make sure everything is working properly. The OS will feel sluggish for now and will not be full resolution, but networking should be active. Shut down the VM.

<h3 name="part3">
  Part 3: Configuring Passthrough 
</h3>

**Pass the GPU IOMMU group**
Go to the virtual machine details in Virt Manager, choose "Add Hardware," select "PCI Host Device," and choose all of the devices in the IOMMU group with your GPU.

**Nvidia Pascal Patch (optional)**
If you use Nvidia Pascal (GTX 10 series), a patch is necessary for this to work. You can download nvflash from the AUR and then type the following commands to download your current GPU BIOS:
```
$ sudo rmmod nvidia_drm nvidia_modeset nvidia
$ sudo nvflash --save backup.rom
```
Clone the [Nvidia Pascal VFIO Patcher](https://github.com/Matoking/NVIDIA-vBIOS-VFIO-Patcher) tool.<br>
If you named your saved rom "backup.rom", type the following command:
```
$ python nvidia_vbios_vfio_patcher.py -i backup.rom -o patched.rom
```
Open Virt Manager, choose "Edit," "Preferences," then "Enable XML editing."<br>
Open the VM details page and click on the XML tab. Add the following text to the file in the hostdev section.
```
<rom file='path/to/your/bios/patched.rom'/>
```
**Pass physical disk (optional)**<br>
If you're using a physical drive instead of emulated storage, you'll need to change the content of the disk section to the following and change the source to the path of your drive.
```
<disk type='block' device='disk'>
      <driver name='qemu' type='raw' />
      <source dev='/dev/sdX'/>
      <target dev='vdb' bus='virtio'/>
      <address type='drive' controller='0' bus='0' target='0' unit='0'/>
 </disk>
```
**Set up hook manager**
To transfer the VFIO drivers from the host to the guest and vice versa automatically, you first need to create the directory at ```/etc/libvirt/hooks```. Then, run the following commands from [The Passthrough Post](https://passthroughpo.st/simple-per-vm-libvirt-hooks-with-the-vfio-tools-hook-helper/):
```
$ sudo wget 'https://raw.githubusercontent.com/PassthroughPOST/VFIO-Tools/master/libvirt_hooks/qemu' \
     -O /etc/libvirt/hooks/qemu
$ sudo chmod +x /etc/libvirt/hooks/qemu
$ sudo systemctl restart libvirtd.service
```
You'll now need to create subdirectories within the ```/etc/libvirt/hooks``` directory. Create the directories just as shown below, replacing "vm_name" with the name you created for your virtual machine.
```
/etc/libvirt/hooks/
|-- qemu
`-- qemu.d
  `-- vm_name
    |-- prepare
    | `-- begin
    `-- release
      `-- end
```
**Adding scripts**<br>
Create a configuration file inside /etc/libvirt/hooks with the name "kvm.conf". Create the following content, using the bus addresses from "check-iommu.sh" preceeded by pci_0000 and using underscores instead of colons. Mine is shown as an example below, but yours will likely be different.
```
VIRSH_GPU_VIDEO=pci_0000_09_00_0
VIRSH_GPU_AUDIO=pci_0000_09_00_1
```






