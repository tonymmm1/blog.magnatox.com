---
title: "ZFS Encryption"
date: 2020-09-18T10:39:43-05:00
draft: false
tags: [
	"ZFS",
]
---

With the release of OpenZFS 0.8.0 native encryption on ZFS was available. However with the lack of proper references I thought it would be prudent to provide some documentation. This documentation assumes that OpenZFS 0.8.X or greater is installed. This tutorial will not cover full zpool encryption as it is currently not recommended and its not strictly necessary as datasets are easier to manage. 


## Verifying zpool 

First zpool must be of the correct version.

```
zpool version
```

Output:
```
zfs-0.8.4-1
zfs-kmod-0.8.4-1
```

```
zpool status
```

Output:
```
  pool: pool
 state: ONLINE
  scan: none requested
config:

	NAME                 STATE     READ WRITE CKSUM
	pool                 ONLINE       0     0     0
	  mirror-0           ONLINE       0     0     0
	    /var/temp/disk1  ONLINE       0     0     0
	    /var/temp/disk2  ONLINE       0     0     0
```

If it prompts you to upgrade make sure to check the installed ZFS version then proceed to upgrade making sure to understand that rollback between versions is not possible. For example a 0.8.x pool will only be read-only on older ZFS implmentations like 0.7.x. 

```
zpool upgrade pool
```

Afterwards verify the status of the pool and it should be ready to enable encryption. 

## Enabling Encryption:

To enable encryption run this command. 

```
zpool set feature@encryption=enabled pool
```

Verify that encryption is enabled on zpool.

```
zpool get feature@encryption pool
```

## Dataset Encryption:

### Password prompt method:

Password encryption using prompt and default encryption AES-GCM 256bit. It will then prompt for passphrase to be used as the decryption key.

```
zfs create -o encryption=on -o keyformat=passphrase pool/test
```

To mount the ZFS dataset you must tell zfs to load-key.

```
zfs load-key pool/test
zfs mount pool/test
```

To unmount the ZFS dataset you must tell zfs to unload-key.

```
zfs umount pool/test
zfs unload-key pool/test
```

To verifiy that dataset is encrypted you can use zfs to show you its properties.

```
zfs get encryption pool/test
```

### Keyfile method:

Passwords can also be stored as a keyfile and mounted automatically on system boot. 

First the keyfile must be created.

```
openssl rand -hex 32 > keyfile
```

Next create the dataset while specifying which keyformat and its corresponding key location. 

```
zfs create -o encryption=on -o keyformat=hex -o keylocation=file:///keyfile pool/test
```

The process of loading the key and mounting is explained above in the Password prompt method.

### Raw method:

This method allows for metadata or even disks to be used as decryption keys as long as they are 32 bytes or less. Refer to this [link](https://github.com/openzfs/zfs/issues/6556) for more info on @tcaputi implementation. 

First the block device must be created.

```
truncate -s 32 keyfile
```

Next create the dataset while specifying which keyformat and its corresponding key location.

```
zfs create -o encryption=on -o keyformat=raw -o keylocation=file:///keyfile pool/test
```

The process of loading the key and mounting is explained above in the Password prompt method.

## References: 

- [Arch ZFS wiki](https://wiki.archlinux.org/index.php/ZFS)
- [Blog with ZFS encryption explanation](https://blog.heckel.io/2017/01/08/zfs-encryption-openzfs-zfs-on-linux/)
- [Tom Caputi - ZFS Native Encryption video](https://youtu.be/frnLiXclAMo)
- [ZFS 0.8.4 man page](https://zfsonlinux.org/manpages/0.8.4/man8/zfs.8.html)
