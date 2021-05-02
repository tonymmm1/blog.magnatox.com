---
title: "Hyperconverged server with Linux, ZFS, KVM, Docker, and guest firewall"
date: 2020-05-17T20:25:25-05:00
description: "Hyperconverged server with Linux, ZFS, KVM, Docker, and guest firewall"
draft: false
tags: [
	"Docker",
	"Firehol",
	"Linux",
	"KVM",
	"ZFS",
]
---
(Originally posted 2020-03-04_08:06)

Setting up a hyperconverged server without relying on external networking is convenient and very space efficient. As long as virtualization layer does not get breached and the server remains fully operational. It allows for virtual guests to share host resources and benefit from having lower latency and overhead to access network mounts, virtual disks, and or guest to guest networking through cpu. 

---
## Requirements:
* Linux host(Debian 10 for this tutorial)
* ZFS on Linux(ZFS 0.8.3 for this tutorial)
* KVM/QEMU with Libvirt
* Docker
* *(IOMMU for nic passthrough)*

---
1. Make sure that Linux is fully up to date and all of the components are installed. 
    * ZFS on Linux 
    * KVM/QEMU with Libvirt
    * Docker
2. Grant user permissions to access Docker and Libvirt with ZFS being relegated to sudo and root level permission.
```
usermod -aG docker user
usermod -aG kvm user
usermod -aG libvirt user
logout 
```
3. Make sure host has internet connectivity while getting required isos and or updates.
4. Create an isolated virtual lan for your guests to use.
![screenshot_20-03-03_20-34-03-01](https://blog.magnatox.com/content/images/2020/03/screenshot_20-03-03_20-34-03-01.png)
5. Proceed to creating a virtual machine guest to use as a firewall for the host and for virtual guests. There are many options for configuring a firewall guest. Using IPfire, pfSense, and or any Linux distro with Firehol. In this example I will use Debian 10 with Firehol which is a little more intimidating for the uninitiated but still doable. 
![screenshot_20-03-03_20-49-33-01](https://blog.magnatox.com/content/images/2020/03/screenshot_20-03-03_20-49-33-01.png)
6. Add second network adapter for isolated virtual lan.
![screenshot_20-03-03_23-55-55-01](https://blog.magnatox.com/content/images/2020/03/screenshot_20-03-03_23-55-55-01.png)
7. Edit isolated virtual lan and set ip for host with an ip address range that the virtual guests will be using.
![screenshot_20-03-04_00-28-04-01](https://blog.magnatox.com/content/images/2020/03/screenshot_20-03-04_00-28-04-01.png)
8. Configure static ips on network interfaces.
```
sudo vim /etc/network/interfaces
```
```
auto en1s0 #WAN interface
iface enp1s0 inet static
    address #WAN IP
    netmask 255.255.255.0
    gateway #WAN gateway
    dns-nameservers 127.0.0.1

auto en7s0 #LAN interface
iface en7s0 inet static
    address 192.168.20.1
    netmask 255.255.255.0
    dns-nameservers 127.0.0.1
```
9. Configure firehol with basic rules.
```
sudo apt-get install firehol -y
sudo vim /etc/default/firehol
```
```
START_FIREHOL=YES
```
```
sudo systemctl enable firehol
sudo systemctl start firehol
sudo vim /etc/firehol/firehol.conf
```
```
version 6

interface4 enp1s0 enp1s0 #WAN interface
	client all accept #Allow outbound
	server ssh accept #Allow inbound ssh 

interface4 enp7s0 enp7s0 #LAN interface
	client all accept #Allow outbound 
	server all accept #Allow inbound

router4 outbound inface enp7s0 outface enp1s0
	route all accept #Allow outbound LAN

router4 inbound inface enp1s0 outface enp7s0
```
10. Configure unbound with DNSSEC.
```
sudo apt-get install unbound unbound-anchor unbound-host -y
```
```
cat /dev/null > /etc/unbound/unbound.conf
sudo vim /etc/unbound/unbound.conf
```
```
server:
	num-threads: 2
	# Common Server Options
	chroot: ""
	directory: "/etc/unbound"
	username: "nobody"
	port: 53
	do-ip4: yes
	do-ip6: no
	do-udp: yes
	do-tcp: yes
	so-reuseport: yes
	do-not-query-localhost: yes
	private-address: 10.0.0.0/8
        private-address: 100.64.0.0/10
	private-address: 127.0.0.0/8
        private-address: 169.254.0.0/16
        private-address: 172.16.0.0/12
        private-address: 192.168.0.0/16
        private-address: fc00::/7
        private-address: fe80::/10
        private-address: ::ffff:0:0/96	
	# System Tuning
	#include: "/etc/unbound/tuning.conf"
	# Logging Options
	verbosity: 1
	use-syslog: yes
	log-time-ascii: yes
	log-queries: no
	logfile: /var/log/unbound
	# Unbound Statistics
	statistics-interval: 86400
	statistics-cumulative: yes
	extended-statistics: yes

	# Prefetching
	prefetch: yes
	prefetch-key: yes

	# Randomise any cached responses
	rrset-roundrobin: yes

	# Privacy Options
	hide-identity: yes
	hide-version: yes
	qname-minimisation: yes
	minimal-responses: yes

	# DNSSEC
	val-permissive-mode: no
	val-clean-additional: yes
	val-log-level: 2
	trust-anchor-file: trusted-key.key	

	# Hardening Options
	harden-glue: yes
	harden-short-bufsize: no
	harden-large-queries: yes
	harden-dnssec-stripped: yes
	harden-below-nxdomain: yes
	harden-referral-path: yes
	harden-algo-downgrade: no
	use-caps-for-id: yes
	aggressive-nsec: yes

	# Harden against DNS cache poisoning
	unwanted-reply-threshold: 1000000

	# Listen on all interfaces
	interface-automatic: yes
	interface: 0.0.0.0

	# Allow access from everywhere
	access-control: 0.0.0.0/0 allow
	access-control: 10.10.0.0/24 allow

	# Bootstrap root servers
	#root-hints: "/etc/unbound/root.hints"

	# Include DHCP leases
	#include: "/etc/unbound/dhcp-leases.conf"

	# Include any forward zones
	#include: "/etc/unbound/forward.conf"

	# Include safe search settings
	#include: "/etc/unbound/safe-search.conf"

remote-control:
	control-enable: yes
	control-use-cert: no
	control-interface: 127.0.0.1
        server-key-file: "/etc/unbound/unbound_server.key"
        server-cert-file: "/etc/unbound/unbound_server.pem"
        control-key-file: "/etc/unbound/unbound_control.key"
        control-cert-file: "/etc/unbound/unbound_control.pem"
```
```
unbound-control-setup
unbound-anchor -a /etc/unbound/trusted-key.key
```
```
sudo vim /etc/resolv.conf
```
```
nameserver 127.0.0.1
```
```
sudo systemctl enable unbound
sudo systemctl start unbound
```
```
unbound-host -C /etc/unbound/unbound.conf -v sigok.verteiltesysteme.net
```
```
sigfail.verteiltesysteme.net has address 134.91.78.139 (BOGUS (security failure))
```
11. Configure dhcpd for lan clients.
```
sudo apt-get install isc-dhcp-server -y
cat /dev/null > /etc/dhcp/dhcpd.conf
sudo vim /etc/dhpcd.conf
```
```
subnet 192.168.20.0 netmask 255.255.255.0 {
	range 192.168.20.10 192.168.20.254;
	option routers 192.168.20.1
	option subnet-mask 255.255.255.0;
    option broadcast-address 192.168.20.1;
	option domain-name "example.com";
	option domain-name-servers 192.168.20.1;
	default-lease-time 3600;
	max-lease-time 7200;
}
```
```
sudo vim /etc/default/isc-dhcp-server
```
```
INTERFACESv4="enp7s0"#LAN interface
```
```
sudo systemctl enable isc-dhcp-server
sudo systemctl start isc-dhcp-server
```
12. Add default route for host to be able to reach internet.
```
sudo route add default gw 192.168.20.1
ping 1.1.1.1
```
```
sudo vim /usr/local/bin/route.sh
sudo chmod +x /usr/local/bin/route.sh
```
```
#!/bin/bash
route add default gw 192.168.20.1
```
```
sudo vim /etc/systemd/system/route.service
```
```
[Unit]
Description=Default gw on boot
After=libvirtd.service

[Service]
ExecStart=/bin/true
ExecStop=/usr/local/bin/route.sh

[Install]
WantedBy=multi-user.target
```
```
sudo systemctl enable route.service
sudo systemctl start route.service
```
13. Delay docker to ensure ZFS mounts.
```
sudo systemctl edit docker
```
```
After=zfs-mount.service
Requires=zfs-mount.service
Wants=zfs-mount.service
BindsTo=zfs-mount.service
```

## References
1. [Debian ZFS](https://github.com/openzfs/zfs/wiki/Debian)
2. [Debian Firehol](https://firehol.org/installing/debian/)
3. [Firehol](https://firehol.org/guides/firehol-welcome/)
4. [DHCPD](https://www.howtoforge.com/tutorial/install-and-configure-isc-dhcp-server-in-debian-9/)
5. [Unbound](https://wiki.archlinux.org/index.php/unbound)
