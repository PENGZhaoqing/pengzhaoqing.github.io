---
date: 2017-01-13
title: SDN开发实战(1)－透明HTTP代理[Openflow+floodlight]
categories: SDN
tags: [java]
---

# 1. SDN
 
软件定义网络（Software Defined Network, SDN），是Emulex网络一种新型网络创新架构，是网络虚拟化的一种实现方式，其核心技术OpenFlow通过将网络设备控制面与数据面分离开来，从而实现了网络流量的灵活控制，使网络作为管道变得更加智能

知乎解释：https://www.zhihu.com/question/20279620

## Openflow+floodlight

Openflow是SDN实现的重要的一个技术手段，由斯坦福高性能网络实验室开发，如今已形成了Openflow论坛。在Openflow框架中（如下图），每个主机（host）连接着Openflow交换机(Openflow Switch)，交换机中的流量表（flow table）由Openflow的控制器控制，通过监视并改变每个Switch中的流量表，各个主机之间的通讯能够很灵活的被Controller控制，而Controller可通过编程实现，这样就从软件层面上直接控制了网络设备中的数据转发，从而定义整个网络

![这里写图片描述](http://img.blog.csdn.net/20170106150259322?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcHBwODMwMDg4NQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

Openflow框架中的控制器有很多开源库可以实现：

 - Java: Beacon, Floodlight
 - Python: POX, Ryu, NOX (Deprecated)
 - Ruby: Trema

此博客使用基于Java的Floodlight库开发控制器，用Mininet来模拟虚拟的主机和Openflow Switch，Mininet是轻量级的软件定义网络系统平台，同时提供了对 OpenFlow 协议的支持，下面给出几个有用的传送门：

 - [Openflow入门教程](https://github.com/mininet/openflow-tutorial/wiki/Installing-Required-Software)
 - [Mininet安装](http://mininet.org/)
 - [Floodlight安装和入门教程](https://floodlight.atlassian.net/wiki/display/floodlightcontroller/Installation+Guide#app-switcher)

# 2. 透明HTTP代理

> 代理服务器的功能就是代理用户访问网络信息，代理分正向代理、反向代理、透明代理等，透明代理就是指用户并不知道代理服务器的存在，代理服务器会修改用户发送的request fields（报文），并会传送真实IP。关于代理服务器请看[这里](http://bbs.51cto.com/thread-967852-1-1.html)

这里，我们为了学习SDN开发，做出的透明HTTP代理应用，并不是真正意义上的透明代理，因为我们并不是注重在代理服务器本身，而是研究如何通过openflow+floodlight实现控制整个网络的转发，模拟的网络功能可以实现透明HTTP代理功能

![这里写图片描述](http://img.blog.csdn.net/20170107151933297?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcHBwODMwMDg4NQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

我们将创建如上图一样的拓扑网络，具有三个虚拟交换机s1、s2、s3（使用的是Open vSwitch，而非标准的Openflow Switch），四个虚拟主机h1、h2、h3、prox，以及一个控制器c0：

 - 其中，prox为具有代理服务器的虚拟主机，10.0.0.x代表每个主机的IP地址
 - hx-eth0代表主机hx的网卡适配器，sx-ethx则代表交换机sx的第x个网卡sx-ethx
 - 控制器c0由Floodlight实现，虚拟交换机和主机由Mininet模拟，之间使用TCP通讯，端口6653

根据上面的描述，我们可以看出只有h1和h2连接着同一个交换机，prox和h3分别连接各自的交换机s2和s3，因此我们现在定义各个主机的角色和整个网络的转发策略（Policy）

 1. h1和h2代表两个用户的主机，能通过同一个交换机直接互联
 2. h3代表网站服务器，里面有h1和h2想要访问的网络资源
 3. h1和h2并不能直接访问h3，需要通过prox代理服务器转发package
 4. h1和h2并不知道代理服务器prox的存在，而且无法ping通prox
 5. 所有的连接为双向有效（bi-bridge）

----------

## 3. 代码实现[[Github](https://github.com/PENGZhaoqing/TransHttpProxyDemo)]

### 3.1 Floodlight

我们使用的是最新版的v1.3版本的Floodlight (master)， 请先参考Floodlight官方教程-[How to Write a Module](https://floodlight.atlassian.net/wiki/display/floodlightcontroller/How+to+Write+a+Module#suk=)，编写一个自定义的控制器其实也就是增加一个模块，一个继承了IOFMessageListener和IFloodlightModule接口的java类，因此需要覆写接口中所有的方法。

1.我们新建一个TransHttpProxyDemo类如下：

``` java
package net.floodlightcontroller.transHttpProxy;
 
import java.util.Collection;
import java.util.Map;
 
import org.projectfloodlight.openflow.protocol.OFMessage;
import org.projectfloodlight.openflow.protocol.OFType;
import org.projectfloodlight.openflow.types.MacAddress;
 
import net.floodlightcontroller.core.FloodlightContext;
import net.floodlightcontroller.core.IOFMessageListener;
import net.floodlightcontroller.core.IOFSwitch;
import net.floodlightcontroller.core.module.FloodlightModuleContext;
import net.floodlightcontroller.core.module.FloodlightModuleException;
import net.floodlightcontroller.core.module.IFloodlightModule;
import net.floodlightcontroller.core.module.IFloodlightService;
 
public class TransHttpProxyDemo implements IOFMessageListener, IFloodlightModule {
 
    @Override
    public String getName() {
        // TODO Auto-generated method stub
        return null;
    }
 
    @Override
    public boolean isCallbackOrderingPrereq(OFType type, String name) {
        // TODO Auto-generated method stub
        return false;
    }
 
    @Override
    public boolean isCallbackOrderingPostreq(OFType type, String name) {
        // TODO Auto-generated method stub
        return false;
    }
 
    @Override
    public Collection<Class<? extends IFloodlightService>> getModuleServices() {
        // TODO Auto-generated method stub
        return null;
    }
 
    @Override
    public Map<Class<? extends IFloodlightService>, IFloodlightService> getServiceImpls() {
        // TODO Auto-generated method stub
        return null;
    }
 
    @Override
    public Collection<Class<? extends IFloodlightService>> getModuleDependencies() {
        // TODO Auto-generated method stub
        return null;
    }
 
    @Override
    public void init(FloodlightModuleContext context)
            throws FloodlightModuleException {
        // TODO Auto-generated method stub
 
    }
 
    @Override
    public void startUp(FloodlightModuleContext context) {
        // TODO Auto-generated method stub
 
    }
 
    @Override
    public Command receive(IOFSwitch sw, OFMessage msg, FloodlightContext cntx) {
        // TODO Auto-generated method stub
        return null;
    }
}
```

2.然后我们事先定义四个主机的Mac地址以及一个调试主机Magic的Mac地址（后面会讲），然后是主机连接的模式的枚举（直接连接、通过代理、无法连接），然后是一些我们即将用到的变量，相关类的包请自行用eclipse导入：

``` java
protected static final MacAddress MAGIC = MacAddress.of("00:11:00:11:00:11");
protected static final MacAddress H1 = MacAddress.of("00:00:00:00:00:01");
protected static final MacAddress H2 = MacAddress.of("00:00:00:00:00:02");
protected static final MacAddress H3 = MacAddress.of("00:00:00:00:00:03");
protected static final MacAddress PX = MacAddress.of("00:00:00:00:00:04");

protected enum RouteMode {
	ROUTE_DIRECT, ROUTE_PROXY, ROUTE_DROP,
};

protected Logger log;
protected IRoutingService routingEngine;
protected IOFSwitchService switchEngine;
protected IFloodlightProviderService floodlightProvider;
protected Map<MacAddress, SwitchPort> mac_to_switchport;
```

3.在getModuleDependencies方法中告诉Floodlight这个类将依赖IFloodlightProviderService类:

``` java
@Override // IFloodlightModule
public Collection<Class<? extends IFloodlightService>> getModuleDependencies() {
	Collection<Class<? extends IFloodlightService>> l = new ArrayList<Class<? extends IFloodlightService>>();
	l.add(IFloodlightProviderService.class);
	return l;
}
```

4.初始化定义的变量，通过context.getServiceImpl()方法从context中获得需要的类并赋值给这些全局变量

``` java
@Override // IFloodlightModule
public void init(FloodlightModuleContext context) throws FloodlightModuleException {
	floodlightProvider = context.getServiceImpl(IFloodlightProviderService.class);
	routingEngine = context.getServiceImpl(IRoutingService.class);
	switchEngine = context.getServiceImpl(IOFSwitchService.class);
	log = LoggerFactory.getLogger("TransHttpProxyDemo");
	mac_to_switchport = new HashMap<MacAddress, SwitchPort>();
}
```

5.为了能获得Switch中实际传输的，启动阶段使floodlightProvider监听传入控制器的PACKET_IN包，当switch收到一条需转发的以太网帧(Ethernet)但是却无法匹配目前的转发表时，会将构建一种PACKET_IN类型的Openflow包交给控制器请求处理，这时，我们能通过调用floodlightProvider变量中的方法来获取switch中这个实际的以太网帧

``` java
@Override // IFloodlightModule
public void startUp(FloodlightModuleContext context) {
	floodlightProvider.addOFMessageListener(OFType.PACKET_IN, this);
}
```

6.下面我们编写receive方法来对控制器收到的每个PACKET_IN包进行处理：

``` java
@Override
public net.floodlightcontroller.core.IListener.Command receive(IOFSwitch sw, OFMessage msg,
		FloodlightContext cntx) {
	
	// 首先确认收到msg的类型为PACKET_IN
	if (msg.getType() != OFType.PACKET_IN) {
		return Command.CONTINUE;
	}

	// 取出以太帧eth并确认以太帧中的载荷为Ipv4类型的数据报
	OFPacketIn pki = (OFPacketIn) msg;
	Ethernet eth = IFloodlightProviderService.bcStore.get(cntx, IFloodlightProviderService.CONTEXT_PI_PAYLOAD);

	IPacket p = eth.getPayload();
	if (!(p instanceof IPv4)) {
		return Command.CONTINUE;
	}
	
	// 获得这个eth传入switch时的网络适配器端口，注意由于Floodlight版本不同因此获得方式有所不同
	OFPort in_port = (pki.getVersion().compareTo(OFVersion.OF_12) < 0) ? pki.getInPort()
			: pki.getMatch().get(MatchField.IN_PORT);
	
	// 获得这个eth在swith中缓存队列中的id
	OFBufferId bufid = pki.getBufferId();

	// 获得这个eth的源Mac地址和目的Mac地址
	MacAddress dl_src = eth.getSourceMACAddress();
	MacAddress dl_dst = eth.getDestinationMACAddress();
	
	// 如果目的地址匹配调试Mac地址，则发送丢包指令，并储存这个eth的源MAC地址和SwitchPort

	if (dl_dst.equals(MAGIC)) {
		SwitchPort tmp = new SwitchPort(sw.getId(), in_port);
		mac_to_switchport.put(dl_src, tmp);
		send_drop_rule(tmp, bufid, dl_src, dl_dst);
		return Command.STOP;
	}

	// 调用process_pkt方法处理
	process_pkt(sw, in_port, bufid, dl_src, dl_dst);
	return Command.STOP;
}
```

**Note: 	引入调试Mac地址的目的是，将所有尝试向调试Mac地址发送包的主机的Mac地址和与之连接的switch及端口储存在mac_to_switchport，这样能够事先掌握所有主机与和与之连接的switch信息，以便后面建立各个host之间的转发渠道**

7.对于继续处理的以太帧eth，编写process_pkt方法进一步处理

``` java
private void process_pkt(IOFSwitch sw, OFPort in_port, OFBufferId bufid, MacAddress dl_src, MacAddress dl_dst) {
RouteMode rm;
	SwitchPort sp_src, sp_dst, sp_prx;

	log.debug("packet_in: " + sw.getId() + ":" + in_port + " " + dl_src + " --> " + dl_dst);
	
	// 尝试从mac_to_switchport中取出此eth的源地址、目标地址、代理地址对应的SwitchPort
	sp_src = mac_to_switchport.get(dl_src);
	sp_dst = mac_to_switchport.get(dl_dst);
	sp_prx = mac_to_switchport.get(PX);

	if (sp_src == null) {
		log.error("unknown source port");
		return;
	} else if (sp_dst == null) {
		log.error("unknown dest port");
		return;
	} else if (sp_prx == null) {
		log.error("unknown proxy port");
		return;
	}

	// 判断源地址和目的地址之间的路由模式
	rm = getCommMode(dl_src, dl_dst);
	log.info("packet_in: routing mode: " + rm);
	
	// 丢包模式
	if (rm == RouteMode.ROUTE_DROP) {
		send_drop_rule(sp_src, bufid, dl_src, dl_dst);
		
	// 代理模式
	} else if (rm == RouteMode.ROUTE_PROXY) {
		create_route(sp_src, sp_prx, dl_src, dl_dst, OFBufferId.NO_BUFFER);
		create_route(sp_prx, sp_dst, dl_src, dl_dst, OFBufferId.NO_BUFFER);
		create_route(sp_dst, sp_prx, dl_dst, dl_src, OFBufferId.NO_BUFFER);
		create_route(sp_prx, sp_src, dl_dst, dl_src, bufid);
		
	// 直连模式
	} else { 
		create_route(sp_src, sp_dst, dl_src, dl_dst, OFBufferId.NO_BUFFER);
		create_route(sp_dst, sp_src, dl_dst, dl_src, bufid);
	}
}
```

8.通过package的源Mac地址和目的Mac地址来判断package的转发模式，这里我们手动设置四种h1、h2、h3、prox之间的路由模式

``` java
private RouteMode getCommMode(MacAddress src, MacAddress dst) {

	// H1 <--> H2 : Direct
	if ((src.equals(H1) && dst.equals(H2)) || (src.equals(H2) && dst.equals(H1))) {
		log.info("pair: H1 <--> H2 : Direct");
		return RouteMode.ROUTE_DIRECT;
	}

	// H1 <--> PX : Drop
	else if ((src.equals(H1) && dst.equals(PX)) || (src.equals(PX) && dst.equals(H1))) {
		log.info("pair: H1 <--> PX : Drop");
		return RouteMode.ROUTE_DROP;
	}

	// H1 <--> H3 : Proxy
	else if ((src.equals(H1) && dst.equals(H3)) || (src.equals(H3) && dst.equals(H1))) {
		log.info("pair: H1 <--> H3 : Proxy");
		return RouteMode.ROUTE_PROXY;
	}

	// H2 <--> PX : Drop
	else if ((src.equals(H2) && dst.equals(PX)) || (src.equals(PX) && dst.equals(H2))) {
		log.info("pair: H2 <--> PX : Drop");
		return RouteMode.ROUTE_DROP;
	}

	// H2 <--> H3 : Proxy
	else if ((src.equals(H2) && dst.equals(H3)) || (src.equals(H3) && dst.equals(H2))) {
		log.info("pair: H2 <--> H3 : Proxy");
		return RouteMode.ROUTE_PROXY;
	}

	// H3 <--> PX : Drop
	else if ((src.equals(H3) && dst.equals(PX)) || (src.equals(PX) && dst.equals(H3))) {
		log.info("pair: H3 <--> PX : Drop");
		return RouteMode.ROUTE_DROP;
	} else {
		return RouteMode.ROUTE_DROP;
	}
}
```

9.直连模式需要在源主机和目的主机之间构建一条通道，而代理模式中个的这条通道则需要经过代理主机prox，由prox来中转他们之间的数据报，但这两种模式都需要找到这条通道的路径，因此我们要编写create_route方法

``` java
private void create_route(SwitchPort sp_src, SwitchPort sp_dst, MacAddress dl_src, MacAddress dl_dst,
		OFBufferId bufid) {
		
	// 通过routingEngine解析出路径，由Floodlight实现
	Path route = routingEngine.getPath(sp_src.getNodeId(), sp_src.getPortId(), sp_dst.getNodeId(),
			sp_dst.getPortId());

	log.info("Route: " + route);
	
	// 路径的表示为SwitchPort对象的List
	List<NodePortTuple> switchPortList = route.getPath();
	
	// 用write_flow方法为路径中的每个SwitchPort创建FlowMod
	for (int indx = switchPortList.size() - 1; indx > 0; indx -= 2) {
		DatapathId dpid = switchPortList.get(indx).getNodeId();
		OFPort out_port = switchPortList.get(indx).getPortId();
		OFPort in_port = switchPortList.get(indx - 1).getPortId();
		write_flow(dpid, in_port, dl_src, dl_dst, out_port, (indx == 1) ? bufid : OFBufferId.NO_BUFFER);
	}
}
```

以及编辑丢包模式中的send_drop_rule方法：

``` java
private void send_drop_rule(SwitchPort sw1, OFBufferId bufid, MacAddress src, MacAddress dst) {
	write_flow(sw1.getNodeId(), sw1.getPortId(), src, dst, null, bufid);
}
```


10.在Openflow中，交换机switch通过自身的flow tables来处理未来到达的package，而这种rules是能够通过FlowMod对象修改（增删等），因此编辑最底层的write_flow方法

``` java
private void write_flow(DatapathId dpid, OFPort in_port, MacAddress dl_src, MacAddress dl_dst, OFPort out_port,
		OFBufferId bufid) {
	
	// 通过switchEngine获得switch对象，dpid为switch的id
	IOFSwitch sw = switchEngine.getSwitch(dpid);
	
	// 获得OF工厂
	OFFactory myFactory = sw.getOFFactory();
	
	// 构造OFActions，如果设置out_port为空则为丢包模式
	List<OFAction> actionList = new ArrayList<OFAction>();
	OFActions actions = myFactory.actions();
	if (out_port != null) {
		OFActionOutput output = actions.buildOutput().setPort(out_port).setMaxLen(0xFFffFFff).build();
		actionList.add(output);
	} else {
		log.info("droping.....");
	}
	
	// 构造Match，用来匹配package的源Mac地址和目的Mac地址以及switch端口
	Match match = myFactory.buildMatch().setExact(MatchField.ETH_SRC, dl_src).setExact(MatchField.ETH_DST, dl_dst)
			.setExact(MatchField.IN_PORT, in_port).setExact(MatchField.ETH_TYPE, EthType.IPv4).build();

	// 构造OFFlowAdd，设置rule的优先级为1
	// 若优先级为0，即使匹配的package也不会按照rule正确转发，而是再次交付控制器
	OFFlowAdd flowAdd = myFactory.buildFlowAdd().setBufferId(bufid).setMatch(match).setIdleTimeout(20)
			.setPriority(1).setActions(actionList).build();

	log.info("writing flowmod: " + flowAdd);

	sw.write(flowAdd);
}
```


## 3.2 Mininet和代理服务器配置

 详见下一个教程[SDN开发实战(2)－透明HTTP代理[Openflow+floodlight]](http://blog.csdn.net/ppp8300885/article/details/54193708)




