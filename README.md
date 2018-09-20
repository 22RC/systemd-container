# Systemd-container

This repo contains steps to create a Centos 7 systemd-container with internal networking.

The connection is implementes with a network bridge between host and container. 

## Centos 7 and systemd-nspawn

### Make a directory to install Centos 7 container. The default folder to set-up container is /var/lib/machines/

`[root@host ~]# cd /var/lib/machines/ `

`[root@host machines]# mkdir -p centos7template/var/lib/rpm `

### Create rpmDB and download the rpm Centos release

`[root@host machines]# rpm --rebuilddb --root=/var/lib/machines/centos7template/ `

`[root@host machines]# wget http://mirror.centos.org/centos-7/7.5.1804/os/x86_64/Packages/centos-release-7-5.1804.el7.centos.x86_64.rpm `

`[root@host machines]# mv centos-release-7-5.1804.el7.centos.x86_64.rpm /tmp/ `

### Install the release rpm for Centos (without deps) and create a basic Centos 7 bootstrap

`[root@host machines]# rpm -i --root=/var/lib/machines/centos7template/ --nodeps /tmp/centos-release-7-5.1804.el7.centos.x86_64.rpm ` 

`[root@host machines]# yum --releasever=7 --nogpg --installroot=/var/lib/machines/centos7template/ groups install -y -q 'Minimal Install' `

`[root@host machines]# yum --releasever=7 --nogpg --installroot=/var/lib/machines/centos7template/ install -y wget git net-tools bind-utils yum-utils iptables-services bridge-utils bash-completion kexec-tools sos psacct `

### Set password in your container.

`[root@host machines]# setenforce 0 `

`[root@host machines]# chroot /var/lib/machines/centos7template/ `

`[root@host /]# passwd `

`[root@host /]# exit `

`[root@host machines]# setenforce 1 `

### Create bridge in  host machine

`[root@host machines]# brctl addbr br0 `

`[root@host machines]# brctl show `
	
	> bridge name     bridge id               STP enabled     interfaces
	> br0             8000.000000000000       no

### Active interface br0 and boot container

`[root@host machines]# ifconfig br0 up`

`[root@host machines]# systemd-nspawn -D /var/lib/machines/centos7template/ --machine centos_chroot --network-bridge=br0 -b`

### Add ip, route and DNS to container

`[root@centos_chroot ~]# systemctl stop firewalld `

`[root@centos_chroot ~]# ip a a 10.0.0.11/24 dev host0 `

`[root@centos_chroot ~]# ip route add default via 10.0.0.254 `

`[root@centos_chroot ~]# cat <<EOF /etc/resolv.conf \
			 namserver 172.16.47.11 \
 			 EOF `
`[root@centos_chroot ~]# halt `

### Add ip rule for filter network-package from host to container and viceversa




 
