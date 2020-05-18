---
title: "Setting Up VFIO for Gpu Passthrough on Linux"
date: 2020-05-17T21:56:10-05:00
draft: true
description: "Setting Up VFIO for Gpu Passthrough on Linux"
tags: [
	"KVM",
	"Linux",
	"VFIO",
]
---
(Original Post 2020-05-18_02:56)

## Overview:
This is a setup guide for running vms in Linux with gpus for uses such as having multiple simultaneous desktop users and or for gpu accelerated workloads. Virtual guests can be of numerous different operating systems including Windows. 

## Contents:
1. Instructions:
2. Optimizations:
3. Known Bugs:
4. References: 

## Instructions:

1. Make sure hardware is compatible for VFIO passthrough such as motherboard and cpu from Intel or AMD that have the required support for VT-d and or AMD-Vi. 
2. Choose a Linux distro that has KVM support such as Debian, Ubuntu, CentOS, and or Fedora.
3. Enable iommu in the Linux kernel by writing corresponding string into the GRUB_CMDLINE_LINUX="" of /etc/default/grub for corresponding cpu
    a. AMD: amd_iommu=on
    b. Intel: intel_iommu=on
4. Rebuild grub 
```
grub2-mkconfig -o /boot/grub2/grub.cfg"
```
5. Reboot
6. Check for IOMMU
```
dmesg | grep -i -e DMAR -e IOMMU
```
Output should contain something similar if successful
```
[    1.301355] DMAR: IOMMU enabled
```
7. Next make sure you have decent IOMMU Group separation for your motherboard this can vary between manufacturers and even bios versions. 
8. 

## References:
1. https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#Enabling_IOMMU
2. 

