---
title: "KVM with ZFS"
date: 2020-05-17T20:23:22-05:00
description: "Guide to setting up KVM and ZFS"
draft: false
tags: [
	"KVM",
	"Linux",
	"ZFS",
]
---
(Originally posted 2020-03-03_22:44)

Using ZFS on Linux as a backing store for KVM is very beneficial as it allows using copy on write with ZFS snapshots for vm rollbacks ensuring perfect in time backups. However it is possible to adapt ZFS to perform differently for certain KVM virtual guest workloads such as databases and small random operations. 

* ## ZFS Recordsizes
1. 128k stock recordsize is not the greatest for running virtual machines as the random and sequential speeds are not optimal.
```
zfs create pool/dataset
zfs get recordsize pool/dataset
```
2. 64k recordsize is well optimized for the default qcow2 sector size meaning that the recordsize will match qcow2 layers. This should be used as default because sequential performance is decent and random operations should not be hindered too much
```
zfs create -o recordsize=64k pool/dataset
zfs get recordsize pool/dataset
```
3. 512,4k,8k,16k are recordsizes which match up with ashifts of 9,12,13,14 meaning that the recordsize will match the smallest amount that can be written to the disk which improves random write performance the most but becomes heavily fragmented reducing sequential performance.
```
zfs create -o recordsize=512,4k,8k,16k pool/dataset
zfs get recordsize pool/dataset
```
ashift of 12(4k sector) is recommended for modern hdd
ashift of 13+(8k sector) is recommended for modern ssd such as Samsung and Intel
Below is a recommended zpool with appropriate features enabled for Linux and ashift 12
```
zpool create -d -o feature@async_destroy=enabled -o feature@empty_bpobj=enabled -o feature@lz4_compress=enabled -o ashift=12 -O compression=lz4 -O acltype=posixacl -O xattr=sa -O utf8only=on -O atime=off -O relatime=on (pool name) /dev/disk/by-id/(your disk serial)
zpool get ashift
```
4. Adding a ZIL with a low latency ssd such as Intel Optane is recommended to help with synchronous writes and for sudden powerloss.
5. Keeping sync=standard is recommended as it provides the best balance of performance to reliability with sync=always only being recommended for critical datasets and or with a ZIL. Do not use sync off as it only uses asynchronous ram for transactions and thus is fast and not safe from any powerloss. 
6. Use a utility such as Sanoid for creating automatic snapshots.
```
zfs list -t snap -r pool/dataset
```

---
## References:
1. [ZVOL vs QCOW2 with KVM](https://jrs-s.net/2018/03/13/zvol-vs-qcow2-with-kvm/)
2. [Sanoid](https://github.com/jimsalterjrs/sanoid/)
3. [ZFS-auto-snapshot](https://github.com/zfsonlinux/zfs-auto-snapshot)
4. [Pyznap](https://github.com/yboetz/pyznap)
