---
title: "CentOS7 With ZFS Root"
date: 2020-05-17T20:15:25-05:00
draft: false
---
Original date of publishing: 2019-09-13
 
## Overview:
This is an install guide for how to setup CentOS with ZFS root. I prefer to use CentOS as it allows for easy rebuilding of the initramfs and supports dkms for easier kernel upgrading. There will be written detailed explanations of each step and an summary of all commands for easier copy-pasting.

## Contents:
1. Instructions
2. List of commands
3. References 

## Instructions:
 
1. First there needs to be either two drives and or the use of multiple partitions to install. Testing this setup with KVM is a good idea before launching it in production.
 
2. Install CentOS normally using either a BIOS or UEFI. Look at https://rootvideochannel.blogspot.com/2017/04/installing-centos-on-zfs-root.html for installing onto one partition. Otherwise I usually just create a 5GB partion for / and format it as ext4(adding a 500MB partion for efi boot). Then I proceed with install.

3. After CentOS is installed, make sure to run a "$yum update -y" command to ensure that packages are up to date.

4. Decide on whether to install ZFS 0.7.13 and or 0.8.1-1(must compile as EL7_6 repo stops at 0.7.x). Follow instructions from https://github.com/zfsonlinux/zfs/wiki/RHEL-and-CentOS. Then after installing repo keep the default settings for dkms or change to kmod for manual kernel management. Run "$yum install epel-release -y; yum install zfs zfs-dracut zfs-dkms -y" Run "$modprobe zfs;dkms status" to make sure that the kernel modules is loaded and dkms is configured. 

5. Install additional software for editing configs and for transfering files. Run "$yum install rsync vim -y" and change vim for editor of choice.

6. The creation of the zpool requires some further deliberation. The main concerns include the partitioning of the disk and or disks that are going to be used as well as the type of boot(UEFI or BIOS). My recommended schemes for BIOS include creating a swap partion according to amount of ram that is in the system you are running. UEFI will require an additional partition for /boot/efi of around 500MB. 

7. Before running command decide if you want to set copies = 2 and essentially have 2 replicas of every file on the pool. Next decide on the ashift size the recommend is 12 for most 4K sector disks but for ssd such as Samsung 850 EVO and Intel Optane ashift of 13 for 8K sectors might be preffered. The rest of the settings can be searched up on OpenZFS forums and or the main project wiki.  

8. To create the zpool run "$zpool create -d -o feature@async_destroy=enabled -o feature@empty_bpobj=enabled -o feature@lz4_compress=enabled -o ashift=12 -O compression=lz4 -O copies=2 -O acltype=posixacl -O xattr=sa -O utf8only=on -O atime=off -O relatime=on rpool /dev/disk/by-id/(insert you disk-part here)". 

9. Check zpool by running "$zpool status". There should be a new ZFS pool called rpool. 

10. Create a ZFS dataset for storing the root filesystem. Run "zfs create rpool/ROOT/centos", or run "zfs create rpool/ROOT" depending on how you want to setup file hierarchy. Creating datasets for other directories such as /home is possible now or later on. The procedure would be the same as shown above.

11. Next it is time to copy the current CentOS install to the ZFS pool. Run "$mkdir /mnt/tmp; mount --bind / /mnt/tmp"; rsync -avXP /mnt/tmp/. /rpool/ROOT/centos/.; umount /mnt/tmp".

12. Open editor and remove / from fstab and add /boot/efi partition if necessary. Run "$vim /rpool/ROOT/centos/etc/fstab".

13. Create necessary symlinks for drives to be detected by GRUB. Run "$cd /dev/; ln -s /dev/disk/by-id/* . -i".

14. All partitions need to be mounted in new zpool. Run "$for dir in proc sys dev;do mount --bind /$dir /rpool/ROOT/centos/$dir;done".

15. Chroot into the zpool, run "$chroot /rpool/ROOT/centos".

16. Generate GRUB config, run "$grub2-mkconfig -o /boot/grub2/grub.cfg" or for UEFI run "$yum install grub2-efi* -y; grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg".

17. Remove zpool cache to allow for new import, run "$rm /etc/zfs/zpool.cache".

18. Regenerate initramfs for booting, run "$dracut -f -v /boot/initramfs-$(uname -r).img $(uname -r)", and repeat this process for any other kernels if necessary. \

19. Install GRUB, run "$grub2-install --boot-directory=/boot /dev/(change)

20. Exit chroot, umount filesystems, and reboot. Run "exit;  for dir in proc sys dev;do umount /rpool/ROOT/centos/$dir;done; reboot". There might be a error about a partition still being mounted do not worry let system reboot. On login there might be a dracut error prompt. Run "zpool import rpool -f; exit; exit". That should allow you to import zfs pool using zfs kernel module and exit the dracut emergency shell. 

21. Now the regular login screen should be presented and checking for ZFS root should be as simple as "$df -h" and "$zpool list". 

22. It is important to fix the reboot process and make sure that the rpool is imported every time. Running "$systemctl enable zfs-import.target" should fix the issue. Try adding "ZFS_MOUNT='yes'" and "ZFS_UNMOUNT=yes'". Make sure that whenever grub and or system files are changed to rerun the dracut command from above. Now after a reboot CentOS should be running on ZFS root. 

23. ZFS tips include adding a rpool/reservation partition of 20% of the total pool space to make sure that the zpool never reaches full and cause complete slow down because of lack of space for cow operations. Create snapshots for initial install in case this is a testing system so that changes can be rolled back, run "$zfs snap rpool/ROOT/centos@initial". System snapshots require a reboot to apply and to rollback system to snapshot state run "$zfs rollback rpool/ROOT/centos@initial". There is a really neat tool for managing and replicating ZFS systems by Jim Salter called sanoid and syncoid respectively on https://github.com/jimsalterjrs/sanoid. 

## List of all commands:
```
yum update -y ;yum install epel -y; yum install http://download.zfsonlinux.org/epel/zfs-release.el7_6.noarch.rpm -y; yum install zfs-dracut zfs-dkms zfs -y

modprobe zfs; dkms status

yum install rsync vim -y

zpool create -d -o feature@async_destroy=enabled -o feature@empty_bpobj=enabled -o feature@lz4_compress=enabled -o ashift=12 -O compression=lz4 -O copies=2 -O acltype=posixacl -O xattr=sa -O utf8only=on -O atime=off -O relatime=on rpool /dev/disk/by-id/

zpool status

zfs create rpool/ROOT/centos

mkdir /mnt/tmp; mount --bind / /mnt/tmp; rsync -avXP /mnt/tmp/. /rpool/ROOT/centos/.; umount /mnt/tmp

vim /rpool/ROOT/centos/etc/fstab

cd /dev/; ln -s /dev/disk/by-id/* . -i

for dir in proc sys dev;do mount --bind /$dir /rpool/ROOT/centos/$dir;done

chroot /rpool/ROOT/centos

grub2-mkconfig -o /boot/grub2/grub.cfg

rm /etc/zfs/zpool.cache

dracut -f -v /boot/initramfs-$(uname -r).img $(uname -r)

grub2-install --boot-directory=/boot /dev/

exit; for dir in proc sys dev;do umount /rpool/ROOT/centos/$dir;done; reboot

zpool import -f rpool

exit 

exit

df -h

zpool list

systemctl enable zfs-import.target

echo -e "ZFS_MOUNT='yes' \n ZFS_UNMOUNT='yes'">/etc/default/zfs.conf

dracut -f -v /boot/initramfs-$(uname -r).img $(uname -r)

reboot
```
## References: 

1. [ZFS on Linux](https://github.com/zfsonlinux/pkg-zfs/wiki/HOWTO-install-EL7-(CentOS-RHEL)-to-a-Native-ZFS-Root-Filesystem)
2. [Installing CentOS on ZFS root](https://rootvideochannel.blogspot.com/2017/04/installing-centos-on-zfs-ro)
3. [Ubuntu 18.04 Root on ZFS](https://github.com/zfsonlinux/zfs/wiki/Ubuntu-18.04-Root-on-ZFS)
4. [RHEL and CentOS](https://github.com/zfsonlinux/zfs/wiki/RHEL-and-CentOS)
5. [Sanoid](https://github.com/jimsalterjrs/sanoid)
6. [ZFS Performance Tuning](http://open-zfs.org/wiki/Performance_tuning)
7. [On ZFS Recordsize](https://jrs-s.net/2019/04/03/on-zfs-recordsize/)

