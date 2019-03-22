# Systemd-Nspawn container

This repo contains steps to create a Centos 7 systemd-container with internal networking.

The connection is implemented with a network bridge between host and container. 

## Centos 7 and systemd-nspawn

### Make a directory to install Centos 7 container. The default folder to set-up container is /var/lib/machines/

`[root@host ~]# cd /var/lib/machines/ `

`[root@host machines]# mkdir -p centos7template/var/lib/rpm `

### Create rpmDB and download the rpm Centos release

`[root@host machines]# rpm --rebuilddb --root=/var/lib/machines/centos7template/ `

`[root@host machines]# wget http://mirror.centos.org/centos-7/7/os/x86_64/RPM-GPG-KEY-CentOS-7 `

`[root@host machines]# wget http://mirror.centos.org/centos-7/7/os/x86_64/Packages/centos-release-7-6.1810.2.el7.centos.x86_64.rpm `

`[root@host machines]# mv centos-release-7-* /tmp/ `

### Install the release rpm for Centos (without deps) and create a basic Centos 7 bootstrap

`[root@host machines]# rpm --import RPM-GPG-KEY-CentOS-7 `

`[root@host machines]# rpm -i --root=/var/lib/machines/centos7template/ --nodeps /tmp/centos-release-7-5.1804.el7.centos.x86_64.rpm ` 

`[root@host machines]# yum -y --releasever=7  --installroot=/var/lib/machines/centos7template/ groupinstall -y -q 'Minimal Install'`

`[root@host machines]# yum --releasever=7 --installroot=/var/lib/machines/centos7template/ install centos-release  `

`[root@host machines]# yum --releasever=7 --installroot=/var/lib/machines/centos7template/ install -y wget git net-tools bind-utils yum-utils iptables-services bridge-utils bash-completion kexec-tools sos psacct `

### Set password in your container.

`[root@host machines]# setenforce 0 `

`[root@host machines]# chroot /var/lib/machines/centos7template/ `

`[root@host /]# passwd `

`[root@host /]# exit `


### Create bridge in  host machine

`[root@host machines]# brctl addbr br0 `

`[root@host machines]# brctl show `
	
	> bridge name     bridge id               STP enabled     interfaces
	> br0             8000.000000000000       no

### Active interface br0 and boot container

`[root@host machines]# brctl setfd br0 0 `

`[root@host machines]# ifconfig br0 10.0.0.11 netmask 255.255.255.0 `

`[root@host machines]# systemd-nspawn -D /var/lib/machines/centos7template/ --machine centos_chroot --network-bridge=br0 -b`

### Add ip, route, enable network and disable NM

`[root@centos_chroot ~]# systemctl stop firewalld `

`[root@centos_chroot ~]# touch  /etc/sysconfig/network`

`[root@centos-chroot ~]# systemctl start network `

`[root@centos-chroot ~]# systemctl enable network `

`[root@centos-chroot ~]# systemctl stop NetworkManager`

`[root@centos-chroot ~]# systemctl disable NetworkManager`

`[root@centos_chroot ~]# ip a a 10.0.0.11/24 dev host0 `

`[root@centos_chroot ~]# ip route add default via 10.0.0.254 `

### Add DNS (change the DNSnameserver ip)

```
[root@centos_chroot ~]# cat <<EOF > /etc/resolv.conf 
nameserver 172.16.47.11 
EOF
```

### Add ifconfig network-script to host0 interface

`[root@centos_chroot ~]# touch /etc/sysconfig/network-scripts/ifcfg-host0 `

```
[root@centos_chroot ~]# cat <<EOF > /etc/sysconfig/network-scripts/ifcfg-host0
TYPE="Ethernet" 
BOOTPROTO="none" 
DEFROUTE="yes" 
IPV4_FAILURE_FATAL="no" 
DEVICE="host0" 
ONBOOT="yes" 
PEERDNS="yes" 
PEERROUTES="yes" 
ARPCHECK="no" 
IPADDR=10.0.0.11 
NETMASK=255.255.255.0 
GATEWAY=10.0.0.254 
NM_CONTROLLED="no"
EOF
```

`[root@centos-chroot ~]# systemctl restart network `


### Add ifconfig network-script to br0 interface (change gw ip)

`[root@host machines]# touch /etc/sysconfig/network-scripts/ifcfg-br0 `

```
[root@host machines]# cat <<EOF > /etc/sysconfig/network-scripts/ifcfg-br0
DEVICE=br0
BOOTPROTO=static 
IPADDR=10.0.0.254 
NETMASK=255.255.255.0 
GATEWAY=172.16.32.1 
ONBOOT=yes 
TYPE=Bridge 
NM_CONTROLLED=no
EOF

```

`[root@host machines]# systemctl restart network `

### Add ip rule for filter network-package from host to container and viceversa (change eth0 with your network-card name) 

`[root@host machines]# iptables -A FORWARD -i br0 -o eth0 -j ACCEPT `

`[root@host machines]# iptables -A FORWARD -i eth0 -o br0 -m state --state ESTABLISHED,RELATED -j ACCEPT `

`[root@host machines]# iptables -t nat -A POSTROUTING -o eth0  -j MASQUERADE `

`[root@host machines]# sysctl -w net.ipv4.conf.br0.forwarding=1 `

`[root@host machines]# sysctl -w net.ipv4.conf.eth0.forwarding=1 `

### Test connection on container

`[root@centos_chroot ~]# ping 8.8.8.8 `

	```
        PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
        64 bytes from 8.8.8.8: icmp_seq=1 ttl=119 time=8.64 ms
        64 bytes from 8.8.8.8: icmp_seq=2 ttl=119 time=8.30 ms
        --- 8.8.8.8 ping statistics ---
        2 packets transmitted, 2 received, 0% packet loss, time 1002ms
        tt min/avg/max/mdev = 8.308/8.478/8.649/0.193 ms
	```

### Test connection ssh from host to container

`[root@host machines]# ssh 10.0.0.11 `

	> The authenticity of host '10.0.0.11 (10.0.0.11)' can't be established.
	> ...
	> root@10.0.0.11's password:
	> [root@centos_chroot ~]#

### (Optional) Start-up with systemd service

edit the following file: /etc/systemd/system/systemd-nspawn@centos7templete.service

```
[Unit]
Description=Container %I
Documentation=man:systemd-nspawn(1)
PartOf=machines.target
Before=machines.target

[Service]
ExecStart=/usr/bin/systemd-nspawn --quiet --keep-unit --boot --link-journal=try-guest --network-bridge=br0 --machine=%I
KillMode=mixed
Type=notify
RestartForceExitStatus=133
SuccessExitStatus=133
Slice=machine.slice
Delegate=yes

[Install]
WantedBy=machines.target

```

`[root@host ~]# systemctl start systemd-nspawn@centos7template.service `

`[root@host ~]# systemctl enable systemd-nspawn@centos7template.service `

`[root@host ~]# machinectl `

	> MACHINE       CLASS     SERVICE
	> centos_chroot container nspawn
	>
	> 1 machines listed.
