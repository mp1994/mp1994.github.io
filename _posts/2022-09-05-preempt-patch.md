---
title: Real-time PREEMPT patch for the Linux Kernel 
author: mp1994
date: 2022-09-05
category: linux
tags: linux, rt, preempt
layout: post
published: true
---

This guide is to install the PREEMPT RT patch for the Linux kernel.

### 1- Install required tools
{% include codeHeader.html %}
``` bash
sudo apt install build-essential bc curl \
wget ca-certificates gnupg2 libssl-dev \
lsb-release libelf-dev bison flex dwarves \
zstd libncurses-dev dwarves
```

### 2- Prepare the build environment
We are going to create a folder in the `$HOME` folder, where we will apply the patch and compile the kernel.
{% include codeHeader.html %}
``` bash
cd
mkdir kernel && cd kernel
```

### 3- Download the kernel and the corresponding patch
This guide was written after patching kernel `v5.4.19` with the PREEMPT RT patch version `rt-11`. We can download the kernel 
and the patch using `wget`, or browsing https://mirrors.edge.kernel.org/pub/linux/kernel/ and https://mirrors.edge.kernel.org/pub/linux/kernel/projects/rt/, respectively.
{% include codeHeader.html %}
``` bash
wget https://mirrors.edge.kernel.org/pub/linux/kernel/v5.x/linux-5.4.19.tar.xz
wget https://mirrors.edge.kernel.org/pub/linux/kernel/projects/rt/5.4/older/patch-5.4.19-rt11.patch.xz
```

### 4- Extract the archives and apply the patch
{% include codeHeader.html %}
``` bash
xz -d *.xz
tar xf linux-*.tar
cd linux-*/
patch -p1 < ../patch-*.patch
```

### 5- Configure the kernel
First, we are going to copy our current configuration from the boot partition
{% include codeHeader.html %}
``` bash
cp -v /boot/config-$(uname -r) .config
```
In the command above, $(uname -r) expands to the current kernel version (e.g., `5.4.0-124-generic` in my case) and copies it to the local file `.config`.
Now, we need to configure the kernel. Specifically, we need to set the preemption model to enable the full real-time capability.

{% include codeHeader.html %}
``` bash
make olddefconfig
make menuconfig
```

The latest command will open a TUI (terminal user interface): navigate with the arrow keys and select: 

<ul>
<li><b>General Setup > Preemption Model > Fully Preemptible Kernel (Real-Time)</b></li>
<li><b>Cryptographic API > Certificates for signature checking</b> (at the very bottom of the list) <b>> Provide system-wide ring of trusted keys > Additional X.509 keys for default system keyring > "" </b>(i.e., remove "debian/canonical-certs.pem" from the text input field)</li>
</ul>

Save to `.config` and exit.

### 6- Build the kernel as with `dpkg`
{% include codeHeader.html %}
``` bash
make -j$(nproc) deb-pkg
sudo dpkg -i ../linux-headers-*.deb ../linux-image-*.deb
```

### 7- Reboot and verify
The install process should have created the initial ramdisk file and the kernel file, and added the new boot entry to GRUB. We can verify with `ls -l /boot | grep rt`.

{% include codeHeader.html %}
``` bash
$ ls -l /boot/ | grep rt
config-5.4.19-rt11preempt
initrd.img-5.4.19-rt11preempt
System.map-5.4.19-rt11preempt
vmlinuz-5.4.19-rt11preempt
```

We can now reboot and select the RT kernel from GRUB. To verify all is set correctly, we can check the output of `uname -a` and verify that `cat /sys/kernel/realtime` returns `1`.

#### Note 1: UEFI and Secure Boot
--- to do

#### Note 2: RT permissions
It may be handy to allow the user to set real-time permissions to executable without needing root permission (i.e., without `sudo`). We can 
achieve this by adding the user to the `realtime` group.
{% include codeHeader.html %}
``` bash
sudo addgroup realtime
sudo usermod -a -G realtime $(whoami)
```

{% include date.html %}