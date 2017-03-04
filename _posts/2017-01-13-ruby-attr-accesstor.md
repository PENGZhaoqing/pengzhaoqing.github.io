---
date: 2017-01-13 
title: Vmware CentOS虚拟机网络初始设置
categories: 学习笔记
tags: [linux]
---


# 问题描述

在使用Vmware虚拟64位CentOS 7的过程中，每次新创建一个虚拟机后（网络设置为NAT模式），会发现网络不可用，使用ping后会出现unknown host错误或者是network is unreachable等等，

```
[peng@localhost Desktop]$ ping baidu.com
ping: unknown host baidu.com
```

# 解决方案

1. 首先使用ifconfig查看网卡信息，发现第一个网络适配器eno1677736没有获得Ipv4的地址，也就是没有在网络中注册成功：

	```
	eno16777736: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
	        ether 00:0c:29:91:5c:69  txqueuelen 1000  (Ethernet)
	        RX packets 171  bytes 10260 (10.0 KiB)
	        RX errors 0  dropped 0  overruns 0  frame 0
	        TX packets 0  bytes 0 (0.0 B)
	        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
	
	lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
	        inet 127.0.0.1  netmask 255.0.0.0
	        inet6 ::1  prefixlen 128  scopeid 0x10<host>
	        loop  txqueuelen 0  (Local Loopback)
	        RX packets 304  bytes 25880 (25.2 KiB)
	        RX errors 0  dropped 0  overruns 0  frame 0
	        TX packets 304  bytes 25880 (25.2 KiB)
	        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
	
	virbr0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
	        inet 192.168.122.1  netmask 255.255.255.0  broadcast 192.168.122.255
	        ether 52:54:00:59:c4:0c  txqueuelen 0  (Ethernet)
	        RX packets 0  bytes 0 (0.0 B)
	        RX errors 0  dropped 0  overruns 0  frame 0
	        TX packets 0  bytes 0 (0.0 B)
	        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
	```

2. 进入etc/sysconfig/network-scripts，并找到这个网络适配器的配置文件，使用nano或者vi编辑

	```
	[root@localhost network-scripts]# cd /etc/sysconfig/network-scripts
	[root@localhost network-scripts]# ls
	ifcfg-eno16777736  ifdown-ppp       ifup-ib      ifup-Team
	ifcfg-lo           ifdown-routes    ifup-ippp    ifup-TeamPort
	ifdown             ifdown-sit       ifup-ipv6    ifup-tunnel
	ifdown-bnep        ifdown-Team      ifup-isdn    ifup-wireless
	ifdown-eth         ifdown-TeamPort  ifup-plip    init.ipv6-global
	ifdown-ib          ifdown-tunnel    ifup-plusb   network-functions
	ifdown-ippp        ifup             ifup-post    network-functions-ipv6
	ifdown-ipv6        ifup-aliases     ifup-ppp
	ifdown-isdn        ifup-bnep        ifup-routes
	ifdown-post        ifup-eth         ifup-sit
	[root@localhost network-scripts]# nano ifcfg-eno16777736
	```
3. 将`ONBOOT=false` 改为`ONBOOT=yes`：

	```
	TYPE=Ethernet
	BOOTPROTO=dhcp
	DEFROUTE=yes
	PEERDNS=yes
	PEERROUTES=yes
	IPV4_FAILURE_FATAL=no
	IPV6INIT=yes
	IPV6_AUTOCONF=yes
	IPV6_DEFROUTE=yes
	IPV6_PEERDNS=yes
	IPV6_PEERROUTES=yes
	IPV6_FAILURE_FATAL=no
	NAME=eno16777736
	UUID=94a36c28-dc0d-4999-bc65-334d30fff6fc
	DEVICE=eno16777736
	ONBOOT=yes
	```
	
4. 保存退出，最后使用service network restart重启网络

	```
	service network restart
	Restarting network (via systemctl):                        [  OK  ]
	```

# 检验

再次使用ifconfig检查网卡情况，发现eno16777736获得了ipv4的地址
```
[root@localhost network-scripts]# ifconfig
eno16777736: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.124.134  netmask 255.255.255.0  broadcast 192.168.124.255
        inet6 fe80::20c:29ff:fe91:5c69  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:91:5c:69  txqueuelen 1000  (Ethernet)
        RX packets 276800  bytes 372561384 (355.3 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 121742  bytes 7389047 (7.0 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 0  (Local Loopback)
        RX packets 476  bytes 40664 (39.7 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 476  bytes 40664 (39.7 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

virbr0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 192.168.122.1  netmask 255.255.255.0  broadcast 192.168.122.255
        ether 52:54:00:59:c4:0c  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

再次使用ping检查网络情况，发现网络正常：

```
[root@localhost network-scripts]# ping baidu.com
PING baidu.com (180.149.132.47) 56(84) bytes of data.
64 bytes from 180.149.132.47: icmp_seq=1 ttl=128 time=394 ms
64 bytes from 180.149.132.47: icmp_seq=2 ttl=128 time=390 ms
^C
--- baidu.com ping statistics ---
3 packets transmitted, 2 received, 33% packet loss, time 2000ms
rtt min/avg/max/mdev = 390.585/392.724/394.863/2.139 ms
```

