---
title: "Setting Up Linux PXE Boot Server"
date: 2021-12-09T12:20:49-06:00
draft: false
tags: [
	"Linux",
    "Networking",
    "PXE"
]
---

![Dnsmasq](/images/Dnsmasq_icon.svg)

Having a Linux PXE Boot server is a very useful tool for either booting systems using a common image or even for installing the same image onto local media. This post will describe how to setup a 
DHCP and PXE server using [Dnsmasq](https://thekelleys.org.uk/dnsmasq/doc.html) and using [Debian](https://www.debian.org/) as the distribution. PXE loads the kernel and initramfs upon system startup if network interface supports network boot. NFS is used for the root file system afterwards and thus the entire boot process is done through the network.

## Linux environment: 

1. First it is important to have a Linux vm or host that can serve as the PXE/DHCP server connected to a network. This network must only have one dhcp server broadcast. Usually configuring a separate vlan for PXE traffic is recommended.

2. Assumptions made in this post:

    - Debian host
    - sudo access
    - host interface: eth0
    - host ip: 192.168.1.2/24
    - host network: 192.168.1.0/24
    - pxe root path: /pxeroot
    - tftp path: /srv/tftp

## Setup Debian:

### 1. Install Debian to host.

Install latest version of Debian onto a vm and make sure to have at least 5-10GB of free disk space for the PXE boot mount. 

### 2. Install Dnsmasq.

```shell
sudo apt-get update -y 
sudo apt-get install dnsmasq nfs-common -y
```

### 3. Install debootstrap.

Deboostrap allows debian to install a working image into a directory for chroot.
First install debootstrap.

```shell
sudo apt-get install debootstrap -y
```

### 4. Install Debian to directory using debootstrap.

This process will take a bit of time depending on speed of local system.

```shell
sudo mkdir /pxeroot
sudo debootstrap bullseye /pxeroot
```

### 5. Configure networking for PXE filesystem

Setting dhcp for default interface is recommended.

```shell
sudo cat << EOF >> /pxeroot/etc/network/interfaces
auto lo
iface lo inet loopback
auto eth0
iface eth0 inet dhcp
EOF
```

### 6. Configure hostname for PXE server

```shell
sudo echo pxeboot > /pxeroot/etc/hostname
sudo cat << EOF >> /pxeroot/etc/hosts
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
EOF
```

### 7. Configure fstab for PXE 

The PXE boot Linux process needs to mount /dev/ram0 as root. Other mountpoints can be added at this point if needed.

```shell
sudo cat << EOF >> /pxeroot/etc/fstab
/dev/ram0	/	ext4	defaults	0	0
proc		/proc	proc	defaults	0	1
tmpfs		/tmp	tmpfs	defaults	0	1
EOF
```

### 8. Chroot into /pxeroot to make any changes

Chroot allows the directory to be mounted as if it was the current root filesystem and thus allows for changes to be made that only affect PXE boot clients.
Do not forget to unmount after exiting chroot.

```shell
sudo mount -t proc none /pxeroot/proc
sudo mount --rbind /sys /pxeroot/sys
sudo mount --rbind /dev /pxeroot/dev
sudo chroot /pxeroot
```

### 9. Inside chroot install Linux image package.

Install Linux image package and other packages if needed.

```shell
apt-get install linux-image-amd64
exit
```

### 10. Create NFS share for pxe root filesystem

Edit the file accordingly and reload exports. Check that the filesytem is exported correctly.

```shell
sudo cat << EOF >> /etc/exports
/pxeroot 192.168.1.0/24(fsid=1,ro,sync,no_root_squash,no_subtree_check)
EOF
sudo exportfs -arv
sudo showmount -e
```

### 11. Create TFTP path and add PXE boot files

First create a directory for tftp and then download pxelinux.0 which is used for bootstrapping.

```shell
sudo mkdir /srv/tftp
cd /srv/tftp
sudo wget https://deb.debian.org/debian/dists/bullseye/main/installer-amd64/current/images/netboot/pxelinux.0
```

Next create another directory to hold the pxelinux config. Edit the config file linux line for mount and ro/rw on boot.

```shell
sudo mkdir /pxelinux.cfg
sudo vim /pxelinux.cfg/default
```

```
DEFAULT linux

LABEL linux
    kernel vmlinuz
    append vga=normal initrd=initrd.gz root=/dev/nfs nfsroot=192.168.1.2:/pxeroot ro --

PROMPT 0
TIIMEOUT 0
```

Copy the initramfs and vmlinuz files from /pxeroot and adjust the kernel version accordingly.

```shell
sudo cp /pxeroot/boot/vmlinuz-5.10.0-9-amd64 vmlinuz
sudo cp /pxeroot/boot/initrd.img-5.10.0-9-amd64 initrd.gz
```

Set directory and its contents to dnsmasq user with correct permissions.

```shell
sudo chown -R dnsmasq /srv/tftp
sudo chmod -R 755 /srv/tftp
```

### 12. Configure dnsmasq to serve dhcp and PXE.

Make sure to configure ``/etc/dnsmasq.conf`` with correct variables as this file contains the assumptions made above.

```shell
vim /etc/dnsmasq.conf
```

```
#Interface information 
#--use ip addr to see the name of the interface on your system
interface=eth0
bind-interfaces
#domain= #uncomment and add if necessary

#--------------------------
#DHCP Settings
#--------------------------
#-- Set dhcp scope
#network range, subnet, duration of dhcp lease
dhcp-range=192.168.1.0,192.168.1.254,255.255.255.0,1h

#-- Set gateway option
dhcp-option=3, 192.168.1.1

#-- Set DNS server option
dhcp-option=6, 192.168.1.1

#-- dns Forwarder info
#server=8.8.8.8

#logging
log-queries
log-dhcp

#----------------------#
# Specify TFTP Options #
#----------------------#

#--location of the pxeboot file
dhcp-boot=pxelinux.0,pxeserver,192.168.1.2

dhcp-no-override

#--enable tftp service
enable-tftp

#-- Root folder for tftp
tftp-root=/srv/tftp

tftp-secure

tftp-no-blocksize

#--Detect architecture and send the correct bootloader file
dhcp-match=set:efi-x86_64,option:client-arch,7 
dhcp-boot=tag:efi-x86_64,ipxe.efi
```

### 13. Start and enable dnsmasq and NFS on boot.

If any issues are reported with ``systemctl status`` make sure to check the config file for any issues.

```shell
sudo systemctl enable --now nfs-server
sudo systemctl enable --now dnsmasq.service
sudo systemctl status dnsmasq.service
```

### 14. Additional information:

- The PXE server uses NFS for the root filesystem and thus allows changes to be pushed to already booted machines.
- Changes to kernel and modules requires initramfs to be recreated with appropriate kernel version. This file needs to then be copied to the tftp path with the correct permissions set.

```shell
chroot /pxeroot
update-initramfs -u -k $(uname -r)
```

## References

- <https://www.howtoforge.com/pxe_booting_debian>
- <https://wiki.debian.org/Debootstrap>
- <https://autostatic.com/setting-up-a-pxe-server-with-dnsmasq/>
