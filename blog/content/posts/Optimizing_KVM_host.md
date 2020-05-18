---
title: "Optimizing KVM host"
date: 2020-05-17T20:19:09-05:00
draft: false
description: "Guide to optimizing KVM host"
tags: [
	"KVM",
	"Linux",
]
---
(Originally posted 2020-03-03_22:20)

There are many tunables in Linux and KVM that allow for performance to suit certain workloads such as optimizing throughput for databases and or reducing latency for other tasks. In this post I will present some of the optimizations that I chose to use that I noticed decreased my dpc latency in my Windows virtual guests and helped alleviate i/o bottlenecks. 

## Contents:
* ### Virtual machine creation
* ### Hugepages
* ### Isolcpus
* ### Iothreads
* ### Tuning for NUMA
* ### Multi-queue Virtio Networking
* ### References

---
* ##Virtual machine creation
1. Create a virtual guest using virt-manager 
![image1](/images/screenshot_20-03-03_16:08:03-01.png)
2. Select Q35 and UEFI
![image2](/images/screenshot_20-03-03_16:09:07-01.png)
3. Configure cpu allocation, cpu model and topology
![image3](/images/screenshot_20-03-03_16:10:20-01.png)
4. Set NIC to virtio and select virtual network or macvtap to physical nic on host
![image4](/images/screenshot_20-03-03_16:11:16-01.png)
5. Set Disk to virtio and use threads with writeback/writethrough mode
![image5](/images/screenshot_20-03-03_16:12:47-01.png)


---
* ##Hugepages
Hugepages are essentially memory that are arranged as blocks for services such as virtual machines to use that decrease some of the overhead required to access many smaller pages especially with systems with hundreds if not thousands of GBs of ram. 

There are only a couple of steps needed to setup hugepages.

1. First is a calculation of how many pages you want to reserve.
``
bc
``
```
((1024^3 * (number in GB))/2)*1.02 
```
This will take the number of GB you want and calculate the number of 2048KB pages that it will require with an additional two percent of buffer overhead.

2. Write value to /etc/sysctl.conf
``
sudo vim /etc/sysctl.conf
``
```
vm.nr_hugepages = (value from above)
```
3. Reboot
4. Check for hugepages
``
cat /proc/meminfo | grep Huge
``
5. Add config to virtual guest
``
sudo virsh edit (vm name)
``
Add before <os> tag 
```
<memoryBacking>
    <hugepages/>
</memoryBacking>
```
6. Virtual guest should now be able to run with much less latency and higher performance
---
* ##Isolcpus
This section will cover how to pin cpu cores at boot from scheduler so that virtual guests will have exclusive access to cores.
    
1. Run lscpu to see the list of cpu cores
``
lscpu
``
```
Architecture:                    x86_64
CPU op-mode(s):                  32-bit, 64-bit
Byte Order:                      Little Endian
Address sizes:                   46 bits physical, 48 bits virtual
CPU(s):                          4
On-line CPU(s) list:             0-3
Thread(s) per core:              2
```
2. Add cores to pin on boot to /etc/default/grub and append 
``
sudo vim /etc/default/grub
``
GRUB_CMDLINE_LINUX="...isolcpus=(range of cores exclude core 0 for host)"
``
3. Reboot
4. Check for isolated cpus
``
cat /sys/devices/system/cpu/isolated
``
``
top(htop)
``
5. Pin cpu cores to isolated cores
```
<vcu placement ...></vcpu>
<cputune>
    <vcpupin vcpu='0' cpuset='x'/>
    <vcpupin vcpu='1' cpuset='x'/>
    <vcpupin vcpu='2' cpuset='x'/>
    <vcpupin vcpu='3' cpuset='x'/>
<cputune>
```
6. Virtual guest should now be good to go with isolated cores
    
---
* ## IOTHREADS
Configuring iothreads can help reduce virtual guest load by offloading io to host cores. 
``
sudo virsh edit (virtual guest name)
``
```  
<vcpu placement...</vcpu>
<iothreads>1</iothreads>
<cputune>
```
```
    <iothreadpin iothread='1' cpuset='x'/>
</cputune>
```

---
* ## Tuning for NUMA 
1. Isolcpus as shown above but having comma separated ranges for multiple cpu nodes.
``
GRUB_CMDLINE_LINUX="...isolcpus=w-x,y-z"
``
2. Hugepages for numa systems can be tuned so that each node can reserve a different amount of ram. Create these text files listed below.
    
``
sudo vim /lib/systemd/system/hugetlb-gigantic-pages.service
``
```
[Unit]
Description=HugeTLB Gigantic Pages Reservation
DefaultDependencies=no
Before=dev-hugepages.mount
ConditionPathExists=/sys/devices/system/node
ConditionKernelCommandLine=hugepagesz=2M
[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/lib/systemd/hugetlb-reserve-pages
[Install]
WantedBy=sysinit.target
```
``
sudo vim /lib/systemd/hugetlb-reserve-pages
``
```
#!/bin/sh
nodes_path=/sys/devices/system/node/
if [ ! -d $nodes_path ]; then
echo "ERROR: $nodes_path does not exist"
exit 1
fi
reserve_pages()
{
echo $1 > $nodes_path/$2/hugepages/hugepages-2048kB/nr_hugepages
}
# This example reserves 2 2M pages on node0 and 1 1M page on node1.
# You can modify it to your needs or add more lines to reserve memory in
# other nodes. Don't forget to uncomment the lines, otherwise they won't
# be executed.
# reserve_pages 2 node0
# reserve_pages 1 node1
reserve_pages 6144 node1
```
```
chmod +x /lib/systemd/hugetlb-reserve-pages
systemctl enable hugetlb-gigantic-pages
reboot
```
Check for HUGEPAGES 
``
cat /sys/devices/system/node/node*/meminfo | fgrep Huge
``
```
Node 0 AnonHugePages:    169984 kB
Node 0 HugePages_Total:     0
Node 0 HugePages_Free:      0
Node 0 HugePages_Surp:      0
Node 1 AnonHugePages:    184320 kB
Node 1 HugePages_Total:  6144
Node 1 HugePages_Free:   6144
Node 1 HugePages_Surp:      0
```
``
numactl -H 
``
```
available: 2 nodes (0-1)
node 0 cpus: 0 1 2 3 4 5 6 7 8 9
node 0 size: 48327 MB
node 0 free: 4788 MB
node 1 cpus: 10 11 12 13 14 15 16 17 18 19
node 1 size: 64509 MB
node 1 free: 17216 MB
node distances:
node   0   1 
  0:  10  21 
  1:  21  10 
```
3. Further NUMA ram tuning and can be adjusted to number of cores, ram and numa nodes.
``
sudo virsh edit (virtual guest name)
``
```
</cputune>
    <numatune>
    <memory mode='strict' nodeset='0'/>
    <memnode cellid='0' mode='strict' nodeset='0'/>
</numatune>
```
```
<numa>
    <cell id='0' cpus='(0-x)' memory='(size)' unit='GiB'/>
</numa>
```

---
* ## Multi-queue Virtio Networking
This allows for multiple high bandwidth network streams between virtual guests for increased performance as well as being able to utilize multiple cores for network throughput

1. Append driver to virtio network device in virtual guest
``
sudo virsh edit (virtual guest name)
``
```
<interface type='network'>
    ...
    <model type='virtio'/>
    <driver name='vhost' queues='(set to number of guest cores)'/>
```


    
## References:
1. [Gaming on Linux with VFIO](https://vfiogaming.blogspot.com/2017/08/guide-how-to-enable-huge-pages-to.html)
2. [NUMA Hugepages](https://www.redhat.com/archives/vfio-users/2017-May/msg00034.html)
3. [Libvirt NUMA](https://libvirt.org/formatdomain.html#elementsNUMATuning)
4. [Libvirt IOThreads](https://libvirt.org/formatdomain.html#elementsIOThreadsAllocation)
5. [Libvirt](https://libvirt.org/formatdomain.html)
