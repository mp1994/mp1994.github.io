---
title: Tuning KVM for Best Performance 
author: mp1994
date: 2022-04-16
category: 2boot
tags: 2boot, dual boot, kvm, linux, vm
layout: post
published: true
---

## CPU Optimization

In the previous posts, we have set up a Kernel Virtual Machine (KVM) with `libvirt` and `qemu`. We have seen how type-I hypervisors allow almost bare-metal performance. With this post, I am going to talk about some tweaking and tuning we can do to further optimize KVM performance.

[Performance tuning](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Performance_tuning
) is very well described in the Arch Linux wiki. Here we are going to see a few optimization steps to improve the performance of our VM.

### CPU pinning

The first and easiest step is to enable CPU pinning. By default, KVM handles guests as normal threads representing virtual processors. These threads are managed by the [Linux scheduler](https://docs.kernel.org/scheduler/index.html) like any other thread, being assigned to any available CPU cores based on [niceness](https://man7.org/linux/man-pages/man2/nice.2.html) and priority queues. As a result, the local CPU cache benefits (L1/L2/L3) are lost each time the host scheduler reschedules the virtual CPU thread on a different physical CPU. This can noticeably harm performance on the guest. **CPU pinning** aims to resolve this by limiting which physical CPUs the virtual CPUs are allowed to run on. The ideal setup is a 1:1 mapping such that the virtual CPU cores match physical CPU cores.

`lscpu -e` shows the CPU topology: hyper-threading splits physical CPU cores (`CORE`) into virtual CPUs (`CPU`).

```
CPU NODE SOCKET CORE L1d:L1i:L2:L3 ONLINE MAXMHZ    MINMHZ
0   0    0      0    0:0:0:0       yes    4900.0000 400.0000
1   0    0      1    1:1:1:0       yes    4900.0000 400.0000
2   0    0      2    2:2:2:0       yes    4900.0000 400.0000
3   0    0      3    3:3:3:0       yes    4900.0000 400.0000
4   0    0      0    0:0:0:0       yes    4900.0000 400.0000
5   0    0      1    1:1:1:0       yes    4900.0000 400.0000
6   0    0      2    2:2:2:0       yes    4900.0000 400.0000
7   0    0      3    3:3:3:0       yes    4900.0000 400.0000
```

In my case (Intel i7-10510U), for example, the physical core 0 is split into virtual CPUs 0 and 4. As mentioned above, to optimize performance, we should passthrough virtual CPUs that correspond to physical CPU cores, sharing the lowest level CPU cache (L1 and L2). Core 0 should remain assigned to the host [1]. To optimize performance, I have assigned vCPUs 2-7 to my Windows guest adding the following to the VM's XML file. 
``` xml
  <vcpu placement='static'>6</vcpu>
  <cputune>
    <vcpupin vcpu='0' cpuset='1'/>
    <vcpupin vcpu='1' cpuset='5'/>
    <vcpupin vcpu='2' cpuset='2'/>
    <vcpupin vcpu='3' cpuset='6'/>
    <vcpupin vcpu='4' cpuset='3'/>
    <vcpupin vcpu='5' cpuset='7'/>
  </cputune>
```

In the configuration, `vcpu` is the virtual CPU ID for the guest, while `cpuset` should correspond to `CPU` as reported by `lscpu -e`. I have left only `CORE 0` to the Linux host, as I don't plan to run heavy tasks on Linux while using the Windows guest.

### Frequency Scaling

Modern CPUs do not run with a fixed clock frequency. They have a base clock, a minimum and a maximum (or turbo) frequency. Frequency modulation is a clever solution for the trade off between power saving and performance. These days, things are getting more and more complicated (partially because of nonsense marketing terminology). Let's define the following: the *minimum frequency* is the lowest the CPU settles to at idle. This is 800 MHz on my Intel i7-10510U. Then, there is the `nominal frequency`, 1800 MHz in my case: let's say this is the minimum frequency to which the CPU should be when *not* at idle. Depending on thermal headroom and load, the CPU frequency can increase in steps: the first step is the *regular turbo* frequency (2300 MHz), up to the *absolute maximum turbo* frequency (4900 MHz). The latter is the physical limit of the CPU and may be reached only for brief bursts of power. The CPU will be mostly running at the *regular turbo* frequency under sustained loads. This behavior is controlled by the **CPU scaling governor**.

--- comments
CPU max clock 2400 MHz on Linux with `powersave` governor (nominal: 1800 MHz, "regular" turbo 2300 MHz, max turbo 4900 MHz)
CPU freq ramps up to >4000 MHz **on idle** with `performance`, and then drops down to 3000 MHz under 100% load (scikit-learn model training) > WTF ??
Geekbench5 on linux: 733 2875
              Win10: 483 1851 (66% single core)
---

https://unix.stackexchange.com/questions/64297/host-cpu-does-not-scale-frequency-when-kvm-guest-needs-it
https://forums.unraid.net/topic/44961-fps-drops-stuttering-and-other-things-that-make-me-sad/#comment-443617
https://www.intel.com/content/www/us/en/developer/articles/guide/kvm-tuning-guide-on-xeon-based-systems.html
https://www.kernel.org/doc/html/latest/admin-guide/pm/intel_pstate.html

<!-- https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Setting_up_IOMMU
Enable IOMMU ?

Enable IOMMU with GRUB: https://devopstales.github.io/linux/proxmox-pci-passthrough/
Set GRUB_CMDLINE_LINUX_DEFAULT="quiet splash intel_iommu=on iommu=pt" in /etc/default/grub
then `sudo update-grub && reboot`

# USB passthrough
works out-of-the-box with virt-manager / virt-viewer

udev rules:
SUBSYSTEM=="usb", ATTRS{idVendor}=="0c45", ATTRS{idProduct}=="6723", MODE="0666", TAG+="uaccess"
reboot

test libusb script (link repo github) -->

[1]: https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Performance_tuning

{% include date.html %}