---
date: 2017-01-13
title: SDN开发实战(1)－透明HTTP代理[Openflow+floodlight]
categories: SDN
tags: [java]
---


此教程继续为上一个[SDN开发实战(1)－透明HTTP代理[Openflow+floodlight]](http://blog.csdn.net/ppp8300885/article/details/54136834)做相关配置和实验结果说明

## 3.2 Mininet配置和代理服务器脚本

### 3.2.1 代理服务器脚本

代理主机prox中需要运行一段程序来转发接收到的package，因此编写proxy.c文件如下，prox.c经过编译后运行在prox主机中，实现对收到的package在同样的端口转发出去的功能

``` c
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <pcap/pcap.h>
#include <net/ethernet.h>
#include <netinet/in.h>
#include <netinet/ip.h>
#include <arpa/inet.h>
#include <sys/types.h>
#include <sys/socket.h>

#define FILTER "icmp or tcp"
pcap_t *handle;
void got_packet(u_char *args, const struct pcap_pkthdr*, const u_char *pkt);

int main(int argc, char *argv[])
{
	char *dev ;                        /* The device to sniff on */
	char errbuf[PCAP_ERRBUF_SIZE];     /* Error string */
	struct bpf_program fp;             /* The compiled filter */
	bpf_u_int32 mask;                  /* Our netmask */
	bpf_u_int32 net;                   /* Our IP */

	if (argc != 2) {
		printf("usage: proxy <dev>\n");
		return EXIT_FAILURE;
	} else {
		dev = argv[1];
	}

	/* Find the properties for the device */
	if (pcap_lookupnet(dev, &net, &mask, errbuf) == -1) {
		printf("warning: %s: could not get network: %s\n", dev, errbuf);
		net  = 0;
		mask = 0;
	}

	/* Open the session in promiscuous mode */
	handle = pcap_open_live(dev, BUFSIZ, 1, 1000, errbuf);
	if (handle == NULL) {
		printf("error: %s: could not open: %s\n", dev, errbuf);
		return EXIT_FAILURE;
	}

	if (pcap_compile(handle, &fp, FILTER, 0, mask) == -1) {
		printf("error: could not compile filter '%s': %s\n", FILTER, pcap_geterr(handle));
		return EXIT_FAILURE;
	}

	if (pcap_setfilter(handle, &fp) == -1) {
		printf("error: could not set filter '%s': %s\n", FILTER, pcap_geterr(handle));
		return EXIT_FAILURE;
	}

	/* 获取package,返回给got_packet处理 */
	int r = pcap_loop(handle, -1, got_packet, NULL);
	printf("pcal_loop() quit with: %d\n", r);

	pcap_close(handle);
	return EXIT_SUCCESS;
}

void got_packet(
	u_char                     *args,
	const struct pcap_pkthdr   *header,
	const u_char               *packet)
{
	const struct ether_header  *ethernet;
	const struct ip            *ip;
	char src_ip_str[16];
	char dst_ip_str[16];

	ethernet = (struct ether_header*) packet;
	if (ethernet->ether_type != ntohs(ETHERTYPE_IP)) {
		printf("ignoring non-ip packet (0x%02X) of length %d\n",
			ntohs(ethernet->ether_type), header->len);
		fflush(stdout);
		return;
	}

	ip = (struct ip*) (ethernet+1);
	strcpy(src_ip_str, inet_ntoa(ip->ip_src));
	strcpy(dst_ip_str, inet_ntoa(ip->ip_dst));

	if (ip->ip_p == IPPROTO_ICMP)
		printf("%15s --> %15s  [ICMP]\n", src_ip_str, dst_ip_str);
	else if (ip->ip_p == IPPROTO_TCP)
		printf("%15s --> %15s  [TCP]\n", src_ip_str, dst_ip_str);
	else
		printf("%15s --> %15s  [%d]\n", src_ip_str, dst_ip_str, ip->ip_p);
	fflush(stdout);
	
	/* 将收到的package发送回去 */
	if (pcap_inject(handle, packet, header->len) == -1) {
		printf("error: unable to proxy packet: %s\n", pcap_geterr(handle));
		fflush(stdout);
	}
}
```

 
### 3.2.2 Mininet配置

启动Mininet网络和运行prox.c需要在终端中输入大量代码，因此我们直接把这些代码写入Python脚本来自动运行，下面的run.py能够为我们做下面几个工作：

1. 尝试用套接字连接Floodlight控制器 (localhost,port=6653), 等待直到连接成功
2. 启动Mininet并创建制定拓扑结构的虚拟网络，包括Open vswitch 和主机
3. 编译proxy.c并在host主机中运行编译出的proxy

``` python
#!/usr/bin/env python2.7

from __future__   import print_function
from argparse     import ArgumentParser
from subprocess   import Popen, STDOUT, PIPE
from socket       import socket, AF_INET, SOCK_STREAM
from time         import sleep
from sys          import stdout
from threading    import Thread;
from mininet.net  import Mininet
from mininet.topo import Topo
from mininet.node import RemoteController
from mininet.node import OVSKernelSwitch

MAGIC_MAC = "00:11:00:11:00:11"
MAGIC_IP  = "10.111.111.111"

＃ Mininet拓扑结构
class MyTopo(Topo):

	def __init__(self):
		"""Create custom topo."""

		Topo.__init__(self)

		switch1 = self.addSwitch('s1')
		switch2 = self.addSwitch('s2')
		switch3 = self.addSwitch('s3')

		h1 = self.addHost('h1')
		h2 = self.addHost('h2')
		h3 = self.addHost('h3')
		prox = self.addHost('prox')
		
		link1 = self.addLink(h1, switch1)
		link2 = self.addLink(h2, switch1)
		link4 = self.addLink(h3, switch3)
		link0 = self.addLink(prox, switch2)		

		link2 = self.addLink(switch1, switch2)
		link3 = self.addLink(switch2, switch3)
		
＃ 运行prox脚本
class Prox(Thread):

	def __init__(self, node, log=None):

		Thread.__init__(self)
		self.node = node
		self.log  = log

	def run(self):
		if self.log != None:
			self.log = open(self.log, 'w')
		self.proc = self.node.popen(
			["./proxy", "prox-eth0"],
			stdout=self.log, stderr=self.log
		)
		print("proxy is running")
		self.proc.wait()

# 尝试连接控制器
def wait_on_controller():

	s = socket(AF_INET, SOCK_STREAM)
	addr = ("localhost", 6653)

	try:
		s.connect(addr)
		s.close()
		return
	except:
		pass

	print("Waiting on controller", end=""); stdout.flush()

	while True:
		sleep(0.1)
		try:
			s.connect(addr)
			s.close()
			print("")
			return
		except:
			print(".", end=""); stdout.flush()
			continue

# 编译proxy.c文件
def build_prox(psrc):
	gcc_proc = Popen(stdout=PIPE, stderr=STDOUT,
			args=("gcc", psrc, "-o", "proxy", "-l", "pcap")
	)

	r = gcc_proc.wait()
	if r != 0:
		out, _ = gcc_proc.communicate()
		print(out)
		exit(1)

if __name__ == "__main__":

	build_prox("proxy.c")
	wait_on_controller()

	mn = Mininet(
		topo=MyTopo(),
		autoSetMacs=True,
		autoStaticArp=True,
		controller=RemoteController('c0',port=6653),
		switch=OVSKernelSwitch
	)

	mn.start()

	sleep(0.5)
	
	# 每个host向调试主机发送ping包，纪录主机的Mac地址
	for src in mn.hosts:
		# setARP能绕过ARP协议
		src.setARP(ip=MAGIC_IP, mac=MAGIC_MAC)
		src.cmd("ping", "-c1", "-W1", MAGIC_IP)

	px = Prox(mn.getNodeByName("prox"), "proxy.log")
	px.start()
	mn.interact()
```

----------

## 4. 运行和实验结果

### 4.1. 执行代码

1.在eclipse中运行上个教程中修改的[floodlight项目](https://github.com/PENGZhaoqing/TransHttpProxyDemo/tree/master/floodlight-master)，floodlight控制器会在0.0.0.0:6653监听Openflow的switch，关于如何运行floodlight控制器， 请点击[这里](https://floodlight.atlassian.net/wiki/display/floodlightcontroller/Installation+Guide#suk=)，如果运行正常，控制台会打印以下内容：

```
2017-01-07 10:58:21.707 INFO  [n.f.c.m.FloodlightModuleLoader] Loading modules from src/main/resources/floodlightdefault.properties
2017-01-07 10:58:21.883 WARN  [n.f.r.RestApiServer] HTTPS disabled; HTTPS will not be used to connect to the REST API.
2017-01-07 10:58:21.883 WARN  [n.f.r.RestApiServer] HTTP enabled; Allowing unsecure access to REST API on port 8080.
2017-01-07 10:58:21.883 WARN  [n.f.r.RestApiServer] CORS access control allow ALL origins: true
2017-01-07 10:58:22.57 WARN  [n.f.c.i.OFSwitchManager] SSL disabled. Using unsecure connections between Floodlight and switches.
2017-01-07 10:58:22.57 INFO  [n.f.c.i.OFSwitchManager] Clear switch flow tables on initial handshake as master: TRUE
2017-01-07 10:58:22.57 INFO  [n.f.c.i.OFSwitchManager] Clear switch flow tables on each transition to master: TRUE
2017-01-07 10:58:22.63 INFO  [n.f.c.i.OFSwitchManager] Setting 0x1 as the default max tables to receive table-miss flow
2017-01-07 10:58:22.124 INFO  [n.f.c.i.OFSwitchManager] OpenFlow version OF_15 will be advertised to switches. Supported fallback versions [OF_10, OF_11, OF_12, OF_13, OF_14, OF_15]
2017-01-07 10:58:22.125 INFO  [n.f.c.i.OFSwitchManager] Listening for OpenFlow switches on [0.0.0.0]:6653
...
```

2.执行run.py脚本 ，通过以下操作来检查网络是否运行正常：

 - nodes
 - net
 - h1 ping h3
 - h2 ping h3
 - h1 ping h2
 - h1 ping prox

##＃ 4.2. 实验结果

从下面的输出可以得出实验结果与我们预期是相符合的，h1和h2是直接路由模式，而他们与h3的连接是代理模式，可以从proxy.log看出（由proxy产生），但是网络延迟很长>1000ms，这种延迟应该是proxy转发不及时或者其他问题；而且，对于h1和h2来说，它们完全不知道代理主机prox的存在，因为它们与prox之间为丢包模式，因此无法ping通prox代理主机

```
peng@peng-virtual-machine:~/Downloads/TransHttpProxy$ sudo ./run.py
proxy is running
mininet> nodes
available nodes are: 
c0 h1 h2 h3 prox s1 s2 s3
mininet> net
h1 h1-eth0:s1-eth1
h2 h2-eth0:s1-eth2
h3 h3-eth0:s3-eth1
prox prox-eth0:s2-eth1
s1 lo:  s1-eth1:h1-eth0 s1-eth2:h2-eth0 s1-eth3:s2-eth2
s2 lo:  s2-eth1:prox-eth0 s2-eth2:s1-eth3 s2-eth3:s3-eth2
s3 lo:  s3-eth1:h3-eth0 s3-eth2:s2-eth3
c0
mininet> h1 ping h3
PING 10.0.0.3 (10.0.0.3) 56(84) bytes of data.
64 bytes from 10.0.0.3: icmp_seq=2 ttl=64 time=1634 ms
64 bytes from 10.0.0.3: icmp_seq=3 ttl=64 time=1629 ms
64 bytes from 10.0.0.3: icmp_seq=4 ttl=64 time=1629 ms
64 bytes from 10.0.0.3: icmp_seq=5 ttl=64 time=1629 ms
^C
--- 10.0.0.3 ping statistics ---
7 packets transmitted, 4 received, 42% packet loss, time 6016ms
rtt min/avg/max/mdev = 1629.310/1630.597/1634.350/2.166 ms, pipe 2
mininet> h2 ping h3
PING 10.0.0.3 (10.0.0.3) 56(84) bytes of data.
64 bytes from 10.0.0.3: icmp_seq=2 ttl=64 time=1996 ms
64 bytes from 10.0.0.3: icmp_seq=3 ttl=64 time=1996 ms
64 bytes from 10.0.0.3: icmp_seq=4 ttl=64 time=1993 ms
64 bytes from 10.0.0.3: icmp_seq=5 ttl=64 time=1993 ms
64 bytes from 10.0.0.3: icmp_seq=6 ttl=64 time=1993 ms
^C
--- 10.0.0.3 ping statistics ---
8 packets transmitted, 5 received, 37% packet loss, time 7014ms
rtt min/avg/max/mdev = 1993.094/1994.576/1996.525/2.127 ms, pipe 2
mininet> h1 ping h2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.195 ms
64 bytes from 10.0.0.2: icmp_seq=3 ttl=64 time=0.074 ms
64 bytes from 10.0.0.2: icmp_seq=4 ttl=64 time=0.052 ms
64 bytes from 10.0.0.2: icmp_seq=5 ttl=64 time=0.081 ms
^C
--- 10.0.0.2 ping statistics ---
5 packets transmitted, 4 received, 20% packet loss, time 4007ms
rtt min/avg/max/mdev = 0.052/0.100/0.195/0.056 ms
mininet> h1 ping prox
PING 10.0.0.4 (10.0.0.4) 56(84) bytes of data.
^C
--- 10.0.0.4 ping statistics ---
9 packets transmitted, 0 received, 100% packet loss, time 8057ms

mininet> exit
```

在代理服务器prox中运行的proxy代码会产生一个proxy.log文件能够查看他转发的package，里面记录了转发的目的Mac地址和源Mac地址以及package的类型:

```
10.0.0.1 -->        10.0.0.3  [ICMP]
10.0.0.3 -->        10.0.0.1  [ICMP]
10.0.0.1 -->        10.0.0.3  [ICMP]
10.0.0.3 -->        10.0.0.1  [ICMP]
10.0.0.1 -->        10.0.0.3  [ICMP]
10.0.0.3 -->        10.0.0.1  [ICMP]
10.0.0.1 -->        10.0.0.3  [ICMP]
10.0.0.3 -->        10.0.0.1  [ICMP]
10.0.0.1 -->        10.0.0.3  [ICMP]
10.0.0.3 -->        10.0.0.1  [ICMP]
...
```

最后我们可以查看switch中的flows table，在Mininet还运行的过程中，通过`ovs-ofctl dump-flows` 指令查看 ：

```
peng@peng-virtual-machine:~$ sudo ovs-ofctl dump-flows s1
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=9.825s, table=0, n_packets=8, n_bytes=784, idle_timeout=20, idle_age=1, priority=1,ip,in_port=1,dl_src=00:00:00:00:00:01,dl_dst=00:00:00:00:00:03 actions=output:3
 cookie=0x0, duration=9.825s, table=0, n_packets=7, n_bytes=686, idle_timeout=20, idle_age=0, priority=1,ip,in_port=3,dl_src=00:00:00:00:00:03,dl_dst=00:00:00:00:00:01 actions=output:1
 cookie=0x0, duration=209.230s, table=0, n_packets=57, n_bytes=6431, idle_age=4, priority=0 actions=CONTROLLER:65535
peng@peng-virtual-machine:~$ sudo ovs-ofctl dump-flows s2
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=11.232s, table=0, n_packets=8, n_bytes=784, idle_timeout=20, idle_age=3, priority=1,ip,in_port=2,dl_src=00:00:00:00:00:01,dl_dst=00:00:00:00:00:03 actions=output:1
 cookie=0x0, duration=11.228s, table=0, n_packets=8, n_bytes=784, idle_timeout=20, idle_age=2, priority=1,ip,in_port=1,dl_src=00:00:00:00:00:01,dl_dst=00:00:00:00:00:03 actions=output:3
 cookie=0x0, duration=11.228s, table=0, n_packets=8, n_bytes=784, idle_timeout=20, idle_age=2, priority=1,ip,in_port=3,dl_src=00:00:00:00:00:03,dl_dst=00:00:00:00:00:01 actions=output:1
 cookie=0x0, duration=11.228s, table=0, n_packets=8, n_bytes=784, idle_timeout=20, idle_age=1, priority=1,ip,in_port=1,dl_src=00:00:00:00:00:03,dl_dst=00:00:00:00:00:01 actions=output:2
 cookie=0x0, duration=210.640s, table=0, n_packets=87, n_bytes=10534, idle_age=6, priority=0 actions=CONTROLLER:65535
peng@peng-virtual-machine:~$ sudo ovs-ofctl dump-flows s3
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=12.783s, table=0, n_packets=8, n_bytes=784, idle_timeout=20, idle_age=3, priority=1,ip,in_port=2,dl_src=00:00:00:00:00:01,dl_dst=00:00:00:00:00:03 actions=output:1
 cookie=0x0, duration=12.783s, table=0, n_packets=8, n_bytes=784, idle_timeout=20, idle_age=3, priority=1,ip,in_port=1,dl_src=00:00:00:00:00:03,dl_dst=00:00:00:00:00:01 actions=output:2
 cookie=0x0, duration=212.208s, table=0, n_packets=48, n_bytes=5649, idle_age=7, priority=0 actions=CONTROLLER:65535
```

以交换机s1的第一条rule为例子解释：

```
cookie=0x0, duration=9.825s, table=0, n_packets=8, n_bytes=784, idle_timeout=20, idle_age=1, priority=1,ip,in_port=1,dl_src=00:00:00:00:00:01,dl_dst=00:00:00:00:00:03 actions=output:3
```

一个数据包如果能够从s1的1端口(s1-eth1)进入，并有着源Mac地址00:00:00:00:00:01和目的Mac地址00:00:00:00:00:03，就会从s1的3端口转发出去。注意，此包的priority＝1，若有其他的匹配的rule的优先级高于这个，就会执行优先级高的rule。


# 5. 总结

所有的代码位于[TransHttpProxDemo](https://github.com/PENGZhaoqing/TransHttpProxyDemo)，总结一下几个要注意的部分：

 - v1.3的Floodlight在创建FlowMod的时候一定要设置priority>0，因为switch向Floodlight发送未匹配的package的时候priority=0，若有其他priority=0的rule存在时，这些rule即使能够匹配package，也都不会被触发，而是作为未匹配的package交付给控制器，设置priority>0能够避免switch执行默认的交付而顺利执行其他的rule
 - v1.3Floodlight控制器的运行在6653端口，而mininet默认的为6633端口，因此需要指定mininet与控制器连接的端口为6653
 - run.py文件执行的内容其实都是终端的命令行
 - 为了可以在一开始就能知道所有主机的mac地址以及连接的switch，用了一个小trick：让所有的主机ping调试主机Magic的IP，然后在控制器中保存起来。这样能够帮助Floodlight寻找两个主机之间的路径
 - 实验结果没有考虑连接的时延和带宽，代理模式和直连模式互相能够ping通就可以，应该有其他改善的地方
