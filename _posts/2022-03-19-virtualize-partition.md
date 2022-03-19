---
title: Virtualize a Physical Windows Partition from Linux with KVM
author: mp1994
date: 2022-03-19
category: 2boot
tags: 2boot, dual boot, kvm, linux, vm
layout: post
published: true
---

If you are familiar with a dual-boot configuration, you must have wondered at least once: _is there a way I can avoid rebooting every time?_ Sometimes you just need to open an app that runs on Windows only for 5 minutes. Whatever the case may be, it would be very handy to be able to **virtualize an existing partition**. 

Please mind that this is not rocket science. It also isn't something new. Yet, I remember I had to struggle a bit to get this working, and I could not find much online, so I hope this is helpful.

Long-story short: yes, we can do this. We can virtualize the Windows partition from Linux. I.e., we can create a virtual machine and use the existing Windows partition as the drive for the VM. When I started off, I was familiar only with VirtualBox. Turns out it is not that hard to make it working: you just have to create a virtual disk that points both to the Windows partition and to the UEFI partition for boot. You can check out this post (coming soon) if you prefer to use VirtualBox.

Today, we are going to set this up using KVM and QEMU. **KVM** stands for **Kernel Virtual Machine**. The main advantage of KVM is performance. In short, "<span style="color:#1d71d1">KVM converts Linux into a type-1 (bare-metal) hypervisor"</span>. Hence, KVM can take advantage of the Linux kernel's memory controller, process scheduler, I/O stack, network stack, and so on. This allows much higher performance with less over-head compared to a type-2 hypervisor (e.g., VirtualBox).

### Building the virtual disk

We are going to use `mdadm` to create and manage the virtual RAID array. First, we need to check how our disk is structured. Please mind that here we are consindering a single-disk dual boot setup. We can use `fdisk -l <disk>` to check our disk. Here is how the output looks like in my case.

``` 
$ fdisk -l /dev/nvme0n1
Disk /dev/nvme0n1: 477 GiB, 512110190592 bytes, 1000215216 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
(...)

Device             Start       End   Sectors   Size Type
/dev/nvme0n1p1      2048   1599487   1597440   780M EFI System
/dev/nvme0n1p2   1599488  12085247  10485760     5G Microsoft reserved
/dev/nvme0n1p3  12085248 821014527 808929280 385.7G Linux filesystem
/dev/nvme0n1p4 821014528 998633471 177618944  84.7G Microsoft basic data
```

We can identify the Windows partition as the one with the label `nvme0n1p4`. Now we need to create two files: one (100 MB) to hold the partition table and the EFI partition, and the second one (1 MB) to hold a backup of the partition table. Then, we can create a loop device to associate to each of these files with `losetup`:

{% include codeHeader.html %}
``` bash
dd if=/dev/zero of=$HOME/efi1 bs=1M count=100
dd if=/dev/zero of=$HOME/efi2 bs=1M count=1
LOOP1=$(sudo losetup -f)
sudo losetup ${LOOP1}  $HOME/efi1
LOOP2=$(sudo losetup -f)
sudo losetup ${LOOP2} $HOME/efi2
```

The creation of the files `ef1` and `efi2` is a one-time procedure, while the `losetup` needs to be run every time we want to boot the VM. Finally, we can merge all these with the physical partition (`/dev/nvme0n1p4`) and create the RAID array:

{% include codeHeader.html %}
``` bash
sudo mdadm --build --verbose /dev/md0 --chunk=512 --level=linear --raid-devices=3 ${LOOP1} /dev/nvme0n1p4 ${LOOP2}
```

Another one-time-only step is to create the partition table in the virtual RAID disk. We can do this with `parted`:

{% include codeHeader.html %}
``` bash
sudo parted /dev/md0
(parted) unit s
(parted) mktable gpt
(parted) mkpart primary fat32 2048 204799    # depends on size of efi1 file
(parted) mkpart primary ntfs 204800 -2049    # depends on size of efi1 and efi2 files
(parted) set 1 boot on
(parted) set 1 esp on
(parted) set 2 msftdata on
(parted) name 1 EFI
(parted) name 2 Windows
(parted) quit
```

With these commands, we have created the EFI partition `/dev/md0p1` and the Windows partition `/dev/md0p2`. We need to format the EFI partition with: 

{% include codeHeader.html %}
``` bash
sudo mkfs.msdos -F 32 -n EFI /dev/md0p1
```

The `mdadm` array is built and ready to be used. As a last step, we need to change its ownership to allow our user to perform R/W operations on it: `sudo chown $USER:$USER /dev/md0`.

### Creating a KVM with virt-manager

We are going to use **virt-manager** ([Virtual Machine Manager](https://virt-manager.org/)) to handle our QEMU/KVM virtual machine. The virt-manager application allows both GUI-based and XML-based configuration and uses `libvirt` for KVM machines. You can install with your packet manager, or from [source](https://virt-manager.org/download/). Please mind that depending on your distribution, you may get an older version. For example, `sudo apt install virt-manager` installs version `1.5.1` on Ubuntu 18.04 LTS, while the latest is `4.0.0` at the time of writing.

Launch virt-manager to create the virtual machine. We are going to need the Windows ISO even if we already have it installed in our partition. Make sure that "Connection" is QEMU/KVM and select "Import existing disk image", as shown in the screenshot below.

![New VM](/assets/img/NewVM_KVM_1.png)

Specify the mdadm array we created before as the existing drive path, i.e., `/dev/md0`. You can go further with the configuration. At the end, make sure to select "Customize configuration before install" before clicking on "Finish". Now we can further customize the VM. I recommend adding the Windows 10 ISO (Add Hardware > Storage), as we will need it for the first boot. You may also edit the default configuration as you like.

![Add CDROM](/assets/img/AddCD_2.png)

You can find my configuration at this [Gist](https://gist.github.com/mp1994/9c245095105dcdc73b7b7d158684a4ff#file-win10-xml). I especially recommend the following configuration blocks. For the CPU, the best setting is `host-passthrough`:

{% include codeHeader.html %}
``` xml
<cpu mode='host-passthrough' check='none'>
    <topology sockets='1' cores='4' threads='2'/>
</cpu>
```

Next, we may change the clock configuration: 

{% include codeHeader.html %}
``` xml
<clock offset='localtime'>
  <timer name='hpet' present='yes'/>
  <timer name='hypervclock' present='yes'/>
</clock>
```

You can also check [this guide](https://github.com/Fmstrat/winapps/blob/main/docs/KVM.md) if you need step-by-step instructions.

### First boot

In the MDADM drive array we created before, we have wrapped the physical Windows partition with a virtual EFI/UEFI partition. Hence, we need to have a virtual EFI firmware on our computer. We are going to use OVMF for this purpose. First, check the `os` tag of the XML configuration and make sure the VM is using OVMF. 

{% include codeHeader.html %}
``` xml
<os>
    <type arch='x86_64' machine='pc-i440fx-bionic'>hvm</type>
    <loader readonly='yes' type='pflash'>/usr/share/OVMF/OVMF_CODE.fd</loader>
    <nvram template='/usr/share/OVMF/OVMF_VARS.fd'>/var/lib/libvirt/qemu/nvram/Win10_VARS.fd</nvram>
    <bootmenu enable='yes'/>
</os>
```

The loader (i.e., the virtual EFI firmware) is the file `OVMF_CODE.fd`. The second file is for virtual RAM. You may check your paths with `find / -name "OVMF_*" 2>/dev/null`. If you can't find it, you can install it with your packet manager (e.g., `sudo apt install ovmf`). In case of issue, checkout this [guide](https://github.com/tianocore/tianocore.github.io/wiki/How-to-run-OVMF).

Make sure you have your SATA CDROM1 at the top of the Boot Options and click on "Begin Installation". The machine should boot to the Windows installation disk. We need to assign a letter to the EFI partition to make the VM boot. If you followed the [previous post](/_posts/2022-03-05-winux.md), you should already have Windows installed. We just need to assign a letter to the EFI partition with `ðiskpart`. To do this, press `Shift+F10` to open up a Windows terminal.

{% include codeHeader.html %}
``` bash
diskpart
DISKPART> list disk
DISKPART> select disk 0    # Select the disk
DISKPART> list volume      # Find EFI volume (partition) number
DISKPART> select volume 2  # Select EFI volume
DISKPART> assign letter=B  # Assign B: to EFI volume
DISKPART> exit
```

The last touch is to copy the BCD boot entry for the Windows partition to the volume we just created. From the same terminal, we can run:

{% include codeHeader.html %}
``` 
bcdboot C:\Windows /s B: /f ALL
```

We can now shutdown the machine, remove the Windows ISO and boot again. Finally -- after a _very long_ post -- we are ready to boot our physical Windows partition!

![VM](/assets/img/VM_3.png)

In case of trouble, you may debug your setup by launching directly QEMU from the terminal, by-passing `virt-manager`:

{% include codeHeader.html %}
``` bash
qemu-system-x86_64 \
    -bios /usr/share/OVMF/OVMF_CODE.fd           \  # Use OVMF
    -drive file=/dev/md0,media=disk,format=raw   \  # Boot from /dev/md0
    -cpu host -enable-kvm                        \  # Copy host configuration for CPU
    -m 2G                                           # RAM (2 GB)
```

### Wrap-up

To conclude, it is worth clearing up the ideas. We have created our KVM and booted the physical Windows partition under Linux. As mentioned, we need to create the `mdadm` array every time we reboot our physical machine. For convenience, I have set up [this bash script](https://gist.github.com/mp1994/9c245095105dcdc73b7b7d158684a4ff#file-mdadm-sh). <br/>
In case you need to mount the Windows partition (of course, when the VM is _not_ running!!), you may stop the array with `sudo mdadm --stop /dev/md0` before.

In a future post, I will talk about an AppIndicator that I have been developing to help managing the KVM from Ubuntu/Linux.

&nbsp; <!-- vertical space -->
&nbsp; <!-- vertical space -->

##### References

[Boot Your Windows Partition from Linux using KVM \| jianmin\|dev](https://jianmin.dev/2020/jul/19/boot-your-windows-partition-from-linux-using-kvm/#KVM)<br/>
[Boot physical Windows inside Qemu guest machine \| Moez Bouhlel [lejenome]](https://lejenome.tik.tn/post/boot-physical-windows-inside-qemu-guest-machine)

{% include date.html %}