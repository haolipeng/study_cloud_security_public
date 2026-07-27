



研究之前先提出问题，有目的的分析和研究才更有效果。

**问题1：Neuvector的微隔离功能是否依赖于dp组件？dp组件在微隔离中，起到什么样的作用？**

**问题2：微隔离的原理到底是如何实现的？**

**问题3：SA和DA是什么？个人分析，SA是源地址，DA是目的地址。**

**问题4：vin和vex接口之间的关系是什么？**

**问题5：vbr-neuv和vth-neuv接口是起什么作用的？**



**问题6：tc规则中的bcmac和ucmac代表什么意思？**

答：bcmac是广播地址，ucmac是组播地址。



**问题7：是每个业务容器都对应会创建一个vin和vex接口吗？需要做实验来好好验证下，新建两个容器，会有4个vin和vex接口吗？**

业务容器中创建的的vin和vex的作用是什么呢？

**问题8：在vin、vex、vbr-neuv、vth-neuv等接口上进行抓包，然后分析。**



接下来，我们就上述几个问题，进行分析。

设置规则和没设置规则的区别在哪里？



# 一、现象

## 1、1 enforcer容器接口情况

**vin和vex接口**

39: vexbe87-eth0@if40: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 8e:15:42:f2:9d:cc brd ff:ff:ff:ff:ff:ff link-netnsid 0
40: vinbe87-eth0@if39: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 7e:36:2f:31:5b:f5 brd ff:ff:ff:ff:ff:ff link-netnsid 1

vex的数据源是从哪里来的？

疑问：

vin和vex接口分别起到什么作用？



**vbr-neuv和vth-neuv接口**

10000000: vbr-neuv@vth-neuv: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 2048 qdisc noqueue state UP group default qlen 1000
    link/ether 2a:c8:cf:da:b1:26 brd ff:ff:ff:ff:ff:ff
10000001: vth-neuv@vbr-neuv: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 2048 qdisc noqueue state UP group default 
    link/ether 66:f7:6c:62:c8:8b brd ff:ff:ff:ff:ff:ff



疑问：

vbr-neuv和vth-neuv接口分别起到什么作用？



## 1、2 业务容器接口情况

139: eth0@if40: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever



在部署时，我部署了两个业务容器，一个nginx容器，一个centos容器，在centos容器中发送curl命令，去请求nginx容器，成功则会返回nginx的页面信息。



## 1、3 vin和vex的关系

**enforcer容器接口结论：**

**5: vin362a-eth0@if106:**

**6: vex362a-eth0@if5**

**vex362a-eth0接口的对端veth索引是5，而vin362a-eth0接口的索引恰好是5，得出结论：**

**vex362a-eth0直连vin362a-eth0接口。**



当将centos和nginx的容器都设置成protect模式后，每个容器都会生成一个vin和vex的veth pair对。

37: vexb76f-eth0@if38: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether c2:b7:8b:70:b7:03 brd ff:ff:ff:ff:ff:ff link-netnsid 0
38: vinb76f-eth0@if137: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 8a:09:42:8b:19:fa brd ff:ff:ff:ff:ff:ff link-netnsid 2
39: vexbe87-eth0@if40: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 06:46:9c:cc:37:e3 brd ff:ff:ff:ff:ff:ff link-netnsid 0
40: vinbe87-eth0@if139: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 3e:04:71:72:1c:12 brd ff:ff:ff:ff:ff:ff link-netnsid 1
45: eth0@if46: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:11:00:03 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.3/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 2001:db8:abc1::242:ac11:3/64 scope global nodad 
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe11:3/64 scope link 
       valid_lft forever preferred_lft forever

10000000: vbr-neuv@vth-neuv: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 2048 qdisc noqueue state UP group default qlen 1000
    link/ether 12:80:09:ee:95:51 brd ff:ff:ff:ff:ff:ff
10000001: vth-neuv@vbr-neuv: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 2048 qdisc noqueue state UP group default 
    link/ether 5e:22:98:52:08:6c brd ff:ff:ff:ff:ff:ff



40: vinbe87-eth0@if139接口的对端容器是139，即业务容器的接口。

# 二、traffic control 规则

## 2、1 tc中mac地址分析

重新计算各种mac地址的含义

![image-20220712191953437](https://gitee.com/codergeek/picgo-image/raw/master/image/202601122201285.png)

![image-20220712192012539](https://gitee.com/codergeek/picgo-image/raw/master/image/202601122201990.png)

![image-20220712192038988](https://gitee.com/codergeek/picgo-image/raw/master/image/202601122201433.png)

pair.MAC

////////////////////通过上面三张图来计算得出//////////////////////////////

mac: 02:42:ac:11:00:02
bcmac: ff:ff:ff:00:00:27
ucmac: 4e:65:75:56:00:27

这个mac（02:42:ac:11:00:02）就是业务容器中eth0接口对应的mac(即源mac)



nginx容器的mac地址为

mac: 02:42:ac:11:00:04

//////////////////////////////////////////////////



## 2、2 tc规则分析

查看tc的规则

**规则1**

// Ingress --
// Bypass multicast
// fmt.Sprintf("tc filter add dev %v pref %v parent ffff: protocol all "+
// 	"u32 match u8 1 1 at -14 "+
// 	"action mirred egress mirror dev %v", pair.exPort, tcPrefBase, pair.inPort)

// Forward IP packet, forward unicast packet with DA to the workload 带有DA的组播数据转发给工作负载

```shell
tc filter add dev vexbe87-eth0 pref 10001 parent ffff: protocol ip u32 
match u8 0 1 at -14 
match u16 0x0242 0xffff at -14 
match u32 0xac110002 0xffffffff at -12 
action pedit 
munge offset -14 u16 set 0x4e65 
munge offset -12 u32 set 0x75560027 
pipe action mirred egress mirror dev vbr-neuv
```

mirred操作允许对接收到数据包进行镜像或重定向。



对应代码为

```
cmd = fmt.Sprintf("tc filter add dev %v pref %v parent ffff: protocol ip "+
		"u32 match u8 0 1 at -14 "+
		"match u16 0x%02x%02x 0xffff at -14 
		match u32 0x%02x%02x%02x%02x 0xffffffff at -12 "+
		
		"action pedit 
		munge offset -14 u16 set 0x%02x%02x 
		munge offset -12 u32 set 0x%02x%02x%02x%02x pipe "+
		"action mirred egress mirror dev %v",
		pair.exPort, tcPrefBase+1,
		pair.MAC[0], pair.MAC[1], pair.MAC[2], pair.MAC[3], pair.MAC[4], pair.MAC[5],
		pair.UCMAC[0], pair.UCMAC[1], pair.UCMAC[2], pair.UCMAC[3], pair.UCMAC[4], pair.UCMAC[5],
		nvVbrPortName)
```

pedit是通用的数据包编辑操作。

将源mac地址（02:42:ac:11:00:02）修改为4e:65:75:56:00:27

这个4e:65:75:56:00:27是的mac地址呢？

**规则2**

// Forward the rest

```shell
tc filter add dev vexbe87-eth0 pref 10002 parent ffff: protocol all u32 match u8 0 0 action mirred egress mirror dev vinbe87-eth0
```

对应的代码为

```
cmd = fmt.Sprintf("tc filter add dev %v pref %v parent ffff: protocol all "+
		"u32 match u8 0 0 "+
		"action mirred egress mirror dev %v",
		pair.exPort, tcPrefBase+2, pair.inPort)
```



**规则3**

```
// Egress --
// Bypass multicast
// fmt.Sprintf("tc filter add dev %v pref %v parent ffff: protocol all "+
// 	"u32 match u8 1 1 at -14 "+
// 	"action mirred egress mirror dev %v", pair.inPort, tcPrefBase, pair.exPort)

// Forward IP packet, forward unicast packet with SA from the workload
```

```
tc filter add dev vinbe87-eth0 pref 10001 parent ffff: protocol ip u32 match u8 0 1 at -14 match u32 0x0242ac11 0xffffffff at -8 match u16 0x0002 0xffff at -4 action pedit munge offset -8 u32 set 0x4e657556 munge offset -4 u16 set 0x0027 pipe action mirred egress mirror dev vbr-neuv
```

对应的代码为：

```
cmd = fmt.Sprintf("tc filter add dev %v pref %v parent ffff: protocol ip "+
		"u32 match u8 0 1 at -14 "+
		"match u32 0x%02x%02x%02x%02x 0xffffffff at -8 match u16 0x%02x%02x 0xffff at -4 "+
		"action pedit munge offset -8 u32 set 0x%02x%02x%02x%02x munge offset -4 u16 set 0x%02x%02x pipe "+
		"action mirred egress mirror dev %v",
		pair.inPort, tcPrefBase+1,
		pair.MAC[0], pair.MAC[1], pair.MAC[2], pair.MAC[3], pair.MAC[4], pair.MAC[5],
		pair.UCMAC[0], pair.UCMAC[1], pair.UCMAC[2], pair.UCMAC[3], pair.UCMAC[4], pair.UCMAC[5],
		nvVbrPortName)
```



**规则4**

// Forward the rest

```
tc filter add dev vinbe87-eth0 pref 10002 parent ffff: protocol all u32 match u8 0 0 action mirred egress mirror dev vexbe87-eth0
```

对应的代码为

```
cmd = fmt.Sprintf("tc filter add dev %v pref %v parent ffff: protocol all "+
		"u32 match u8 0 0 "+
		"action mirred egress mirror dev %v",
		pair.inPort, tcPrefBase+2, pair.exPort)
```



**规则5** 

// Forward the packets from enforcer

**规则5.1**

```
tc filter add dev vbr-neuv pref 39 parent ffff: protocol all u32 match u16 0x4e65 0xffff at -14 match u32 0x75560027 0xffffffff at -12 action pedit munge offset -14 u16 set 0x0242 munge offset -12 u32 set 0xac110002 pipe action mirred egress mirror dev vinbe87-eth0
```

对应的代码为

```
cmd = fmt.Sprintf("tc filter add dev %v pref %v parent ffff: protocol all "+
		"u32 match u16 0x%02x%02x 0xffff at -14 match u32 0x%02x%02x%02x%02x 0xffffffff at -12 "+
		"action pedit munge offset -14 u16 set 0x%02x%02x munge offset -12 u32 set 0x%02x%02x%02x%02x pipe "+
		"action mirred egress mirror dev %v",
		nvVbrPortName, exInfo.pref,
		pair.UCMAC[0], pair.UCMAC[1], pair.UCMAC[2], pair.UCMAC[3], pair.UCMAC[4], pair.UCMAC[5],
		pair.MAC[0], pair.MAC[1], pair.MAC[2], pair.MAC[3], pair.MAC[4], pair.MAC[5],
		pair.inPort)
```



**规则5.2**

```
tc filter add dev vbr-neuv pref 40 parent ffff: protocol all u32 match u32 0x4e657556 0xffffffff at -8 match u16 0x0027 0xffff at -4 action pedit munge offset -8 u32 set 0x0242ac11 munge offset -4 u16 set 0x0002 pipe action mirred egress mirror dev vexbe87-eth0
```

对应的代码为

```
cmd = fmt.Sprintf("tc filter add dev %v pref %v parent ffff: protocol all "+
		"u32 match u32 0x%02x%02x%02x%02x 0xffffffff at -8 match u16 0x%02x%02x 0xffff at -4 "+
		"action pedit munge offset -8 u32 set 0x%02x%02x%02x%02x munge offset -4 u16 set 0x%02x%02x pipe "+
		"action mirred egress mirror dev %v",
		nvVbrPortName, inInfo.pref,
		pair.UCMAC[0], pair.UCMAC[1], pair.UCMAC[2], pair.UCMAC[3], pair.UCMAC[4], pair.UCMAC[5],
		pair.MAC[0], pair.MAC[1], pair.MAC[2], pair.MAC[3], pair.MAC[4], pair.MAC[5],
		pair.exPort)
```

mac地址修改这块，绝对是需要好好看看的。



# 三、不同场景分析

## 3、1 两端都未设置保护模式

enforcer容器中，有vbr-neuv和vth-neuv接口，但是vbr和vth接口上没有流量经过。

容器之间还是采用容器网络veth pair进行网络通信。



## 3、2 仅一端设置保护模式

enforcer容器中，有vbr-neuv和vth-neuv接口，一个vin接口和一个vex接口。



## 3、3 两端都设置了保护模式



# 四、源代码解读

```
type pipeInterface interface {
	Connect(jumboframe bool)
	Cleanup()
	AttachPortPair(pair *InterceptPair) (net.HardwareAddr, net.HardwareAddr)
	DetachPortPair(pair *InterceptPair)
	TapPortPair(pid int, pair *InterceptPair)
	FwdPortPair(pid int, pair *InterceptPair)
	ResetPortPair(pid int, pair *InterceptPair)
	GetPortPairRules(pair *InterceptPair) (string, string, string)
}
```

上述是实现微隔离的interface接口。



Neuvector有几种微隔离模式

FwdPortPair

TapPortPair

这二种模式分别在什么情况下使用？



InterceptContainerPorts函数

初始化并创建vin和vex接口的函数



FwdPortPair

```
func (d *tcPipeDriver) FwdPortPair(pid int, pair *InterceptPair) {
	log.WithFields(log.Fields{"inPort": pair.inPort, "exPort": pair.exPort}).Debug("")
	var cmd string
	var ok bool
	var inInfo, exInfo *tcPortInfo
	//查找vin 接口是否存在
	
	//查找vex 接口是否存在

	一、Ingress方向流量
	1、1 忽略multicast多播数据包
	1、2 将带有 DA 的单播数据包转发到工作负载
	1、3 转发剩余流量，从vex接口转发到vin接口

	二、 Egress方向流量
	2、1 忽略multicast多播数据包
	2、2 转发源自工作负载的带有 SA 的单播数据包
	2、3 转发剩余流量，从vin接口转发到vex接口

	三、 转发来自enforcer的数据包
	3、1 vbr-neuv转发数据包给vin接口
	3、2 vbr-neuv转发数据包给vex接口
}
```



GetPortPairRule:获取PortPair上的规则，也就是vin和vex上的tc规则。



```
// 1. Rename, remove IP and MAC of original port, link
// 1. Create a veth pair, local and peer
// 2. Switch IP and MAC address between link and local port
// 3. Move link and peer to service container
func pullContainerPort(
	link netlink.Link, addrs []netlink.Addr, pid, dstNs int, localPortIndex, inPortIndex int,
) (int, error) {
	var err error
	attrs := link.Attrs()
	exPortName, inPortName := getIntcpPortNames(pid, attrs.Name)

	defer func() {
		if err != nil {
			netlink.LinkSetName(link, attrs.Name)
			netlink.LinkSetHardwareAddr(link, attrs.HardwareAddr)
			netlink.LinkSetUp(link)
		}
	}()

	// Down the link
	if err = netlink.LinkSetDown(link); err != nil {
		log.WithFields(log.Fields{"error": err}).Error("Error in disabling port")
		return 0, err
	}
	// Change link name to exPortName.
	if err = netlink.LinkSetName(link, exPortName); err != nil {
		log.WithFields(log.Fields{"error": err}).Error("Error in changing name")
		return 0, err
	}
	// Get link again as name is changed.
	if link1, err := netlink.LinkByName(exPortName); err != nil {
		log.WithFields(log.Fields{"error": err}).Error("Cannot find port")
		return 0, err
	} else {
		link = link1
	}
	// Remove IP addresses
	for _, addr := range addrs {
		netlink.AddrDel(link, &addr)
	}
	// Temp. set MAC address
	tmp, _ := net.ParseMAC("00:01:02:03:04:05")
	if err = netlink.LinkSetHardwareAddr(link, tmp); err != nil {
		log.WithFields(log.Fields{"error": err}).Error("Error in changing MAC")
		return 0, err
	}

	log.WithFields(log.Fields{"inPort": inPortName}).Debug("Create internal pair")

	//创建一个veth pair，一端是原来的端口名，另一端是inPortName
	// Create a new veth pair: one end is the original port name, the other is inPortName
	veth := &linkVeth{
		LinkAttrs: netlink.LinkAttrs{
			Name:   attrs.Name,
			TxQLen: attrs.TxQLen,
			MTU:    attrs.MTU,
			Index:  localPortIndex,
		},
		PeerName:  inPortName,
		PeerIndex: inPortIndex,
	}
	//添加vethAdd
	if err = vethAdd(veth); err != nil {
		log.WithFields(log.Fields{"error": err}).Error("Error in creating veth pair")
		return 0, err
	}

	log.WithFields(log.Fields{"port": attrs.Name}).Debug("Setting up local port")

	// Get the local link of the veth pair
	var local netlink.Link
	var localMAC net.HardwareAddr
	if local, err = netlink.LinkByName(attrs.Name); err != nil {
		log.WithFields(log.Fields{"error": err}).Error("Cannot find port")
		return 0, err
	}
	if err = netlink.LinkSetDown(local); err != nil {
		log.WithFields(log.Fields{"error": err}).Error("Error in disabling port")
		return 0, err
	}

	if cfg.cnet_type == CNET_MACVLAN {
		// Duplicate the local mac, for Container network  like macvlan, mac in host need persistent, so same mac on vex and container eth0
		localMAC = attrs.HardwareAddr
	} else {
		localMAC = local.Attrs().HardwareAddr
	}

	if err = netlink.LinkSetHardwareAddr(local, attrs.HardwareAddr); err != nil {
		log.WithFields(log.Fields{"error": err}).Error("Error in setting MAC")
		return 0, err
	}
	// TODO: For some reason, there always is an extra IPv6 address that cannot be removed,
	//       the external port _sometimes_ also has an extra IPv6 address left.
	// Get all addresses of the local link
	var localAddrs []netlink.Addr
	if localAddrs, err = netlink.AddrList(local, netlink.FAMILY_ALL); err != nil {
		log.WithFields(log.Fields{"error": err}).Error("Error in getting address")
		return 0, err
	}
	for _, addr := range localAddrs {
		log.WithFields(log.Fields{"addr": addr}).Debug("Delete address")
		netlink.AddrDel(local, &addr)
	}
	for _, addr := range addrs {
		log.WithFields(log.Fields{"addr": addr}).Debug("Add address")
		netlink.AddrAdd(local, &addr)
	}
	// Set local link up
	if err = netlink.LinkSetUp(local); err != nil {
		log.WithFields(log.Fields{"error": err}).Error("Error in enabling port")
		return 0, err
	}
	// Set customer container intf seg/chksum off
	DisableOffload(attrs.Name)
	log.WithFields(log.Fields{"port": inPortName}).Debug("Setting up inPort")

	// Get the peer link
	var peer netlink.Link
	if peer, err = netlink.LinkByName(inPortName); err != nil {
		log.WithFields(log.Fields{"error": err}).Error("Cannot find port")
		return 0, err
	}
	if err = netlink.LinkSetDown(peer); err != nil {
		log.WithFields(log.Fields{"error": err}).Error("Error in disabling port")
		return 0, err
	}
	// Move the peer to the service container
	if err = netlink.LinkSetNsFd(peer, dstNs); err != nil {
		log.WithFields(log.Fields{"error": err}).Error("Error in moving namespace")
		return 0, err
	}

	log.WithFields(log.Fields{"port": exPortName}).Debug("Setting up exPort")

	// Set the original port MAC to local port MAC
	if err = netlink.LinkSetHardwareAddr(link, localMAC); err != nil {
		log.WithFields(log.Fields{"error": err}).Error("Error in changing MAC")
		return 0, err
	}
	// Move the original port to service container namespace
	if err = netlink.LinkSetNsFd(link, dstNs); err != nil {
		log.WithFields(log.Fields{"error": err}).Error("Error in moving namespace")
		return 0, err
	}

	return local.Attrs().Index, nil
}
```

上述代码能有效解决下面问题：

vbr-neuv和vex是如何建立关系的

vbr-neuv和veth-neuv的关系



猜想：

Neuvector开启protect保护模式时，会调用DPCtrlAddSrvcPort函数

Neuvector关闭protect保护模式时，会调用DPCtrlDelSrvcPort函数



对于数据流程的分析

![image-20220524143021751](https://gitee.com/codergeek/picgo-image/raw/master/image/202601122200711.png)

本质：用tc模拟出所有流量的镜像。



1、vbr-neuv和vth-neuv是veth pair，向vth-neuv发送数据，vbr-neuv能收到，即vth-neuv是流量入口

1、vex362a-eth0->vth-neuv流量

2、dp解析出流量的属性，ip，port，protocol，进行匹配。

3、带有标志的流量注入回vbr-neuv接口时，才会走tc的流量镜像规则，到vin620-eth0



这个图画的还是不够全面的。有点问题

TODO:应该把两遍的图都画以下。实体画的少了很多细节。

在非保护模式下，只有vth-neuv和vbr-neuv接口，并没有vin和vex接口。



在源容器和目标容器都不设置protect模式时，vth-neuv和vbr-neuv接口上并没有任何流量。



上图是FwdPortPair函数的流程

一、Ingress方向流量
	1、1 忽略multicast多播数据包
	1、2 将带有 DA 的单播数据包转发到工作负载
	1、3 转发剩余流量，从vex接口转发到vin接口

二、 Egress方向流量
2、1 忽略multicast多播数据包
2、2 转发源自工作负载的带有 SA 的单播数据包
2、3 转发剩余流量，从vin接口转发到vex接口

三、 转发来自enforcer的数据包
3、1 vbr-neuv转发数据包给vin接口
3、2 vbr-neuv转发数据包给vex接口



```
cmd = fmt.Sprintf("tc filter add dev %v pref %v parent ffff: protocol all "+
   "u32 match u16 0x%02x%02x 0xffff at -14 match u32 0x%02x%02x%02x%02x 0xffffffff at -12 "+
   "action pedit munge offset -14 u16 set 0x%02x%02x munge offset -12 u32 set 0x%02x%02x%02x%02x pipe "+
   "action mirred egress mirror dev %v",
   nvVbrPortName, exInfo.pref,
   pair.UCMAC[0], pair.UCMAC[1], pair.UCMAC[2], pair.UCMAC[3], pair.UCMAC[4], pair.UCMAC[5],
   pair.MAC[0], pair.MAC[1], pair.MAC[2], pair.MAC[3], pair.MAC[4], pair.MAC[5],
   pair.inPort)
```

上述tc的意思是匹配到报文后修改目的mac。



dp.DPCtrlAddMAC(nvSvcPort, pair.MAC, pair.UCMAC, pair.BCMAC, oldMAC, pMAC, nil)



# 四、proxymesh下的流量分析

proxymesh的规则如下

```shell
:INPUT ACCEPT [37:3477]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [64:5804]
:NV_INPUT_PROXYMESH - [0:0]
:NV_OUTPUT_PROXYMESH - [0:0]
-A INPUT -j NV_INPUT_PROXYMESH
-A OUTPUT -j NV_OUTPUT_PROXYMESH
-A NV_INPUT_PROXYMESH -p tcp -m tcp --sport 9000 -j NFQUEUE --queue-num 0 --queue-bypass
-A NV_INPUT_PROXYMESH -p tcp -m tcp --sport 15021 -j NFQUEUE --queue-num 0 --queue-bypass
-A NV_INPUT_PROXYMESH -j RETURN
-A NV_OUTPUT_PROXYMESH -p tcp -m tcp --dport 9000 -j NFQUEUE --queue-num 0 --queue-bypass
-A NV_OUTPUT_PROXYMESH -p tcp -m tcp --dport 15021 -j NFQUEUE --queue-num 0 --queue-bypass
-A NV_OUTPUT_PROXYMESH -j RETURN
COMMIT
# Completed on Tue May 24 17:05:13 2022
# Generated by iptables-save v1.8.4 on Tue May 24 17:05:13 2022
*nat
:PREROUTING ACCEPT [13243:794580]
:INPUT ACCEPT [13244:794640]
:OUTPUT ACCEPT [1220:109772]
:POSTROUTING ACCEPT [1220:109772]
:ISTIO_INBOUND - [0:0]
:ISTIO_IN_REDIRECT - [0:0]
:ISTIO_OUTPUT - [0:0]
:ISTIO_REDIRECT - [0:0]
-A PREROUTING -p tcp -j ISTIO_INBOUND
-A OUTPUT -p tcp -j ISTIO_OUTPUT
-A ISTIO_INBOUND -p tcp -m tcp --dport 15008 -j RETURN
-A ISTIO_INBOUND -p tcp -m tcp --dport 22 -j RETURN
-A ISTIO_INBOUND -p tcp -m tcp --dport 15090 -j RETURN
-A ISTIO_INBOUND -p tcp -m tcp --dport 15021 -j RETURN
-A ISTIO_INBOUND -p tcp -m tcp --dport 15020 -j RETURN
-A ISTIO_INBOUND -p tcp -j ISTIO_IN_REDIRECT
-A ISTIO_IN_REDIRECT -p tcp -j REDIRECT --to-ports 15006
-A ISTIO_OUTPUT -s 127.0.0.6/32 -o lo -j RETURN
-A ISTIO_OUTPUT ! -d 127.0.0.1/32 -o lo -m owner --uid-owner 1337 -j ISTIO_IN_REDIRECT
-A ISTIO_OUTPUT -o lo -m owner ! --uid-owner 1337 -j RETURN
-A ISTIO_OUTPUT -m owner --uid-owner 1337 -j RETURN
-A ISTIO_OUTPUT ! -d 127.0.0.1/32 -o lo -m owner --gid-owner 1337 -j ISTIO_IN_REDIRECT
-A ISTIO_OUTPUT -o lo -m owner ! --gid-owner 1337 -j RETURN
-A ISTIO_OUTPUT -m owner --gid-owner 1337 -j RETURN
-A ISTIO_OUTPUT -d 127.0.0.1/32 -j RETURN
-A ISTIO_OUTPUT -j ISTIO_REDIRECT
-A ISTIO_REDIRECT -p tcp -j REDIRECT --to-ports 15001
COMMIT
```

service mesh服务网格是通过iptables queue实现去进行微隔离的。

在service mesh环境下，还是需要traffic control规则 + iptables 规则来实现微隔离。



ingress规则如下

```
tc filter show dev vbr-neuv ingress
filter parent ffff: protocol all pref 5 u32 chain 0
filter parent ffff: protocol all pref 5 u32 chain 0 fh 800: ht divisor 1
filter parent ffff: protocol all pref 5 u32 chain 0 fh 800::800 order 2048 key ht 800 bkt 0 terminal flowid ??? not_in_hw
  match 00004e65/0000ffff at -16
  match 75560005/ffffffff at -12
        action order 1:  pedit action pipe keys 2
         index 3 ref 1 bind 1
         key #0  at -16: val 00000a3c mask ffff0000
         key #1  at -12: val 8cf9a266 mask 00000000

        action order 2: mirred (Egress Mirror to device vin74f9-eth0) pipe
        index 5 ref 1 bind 1

filter parent ffff: protocol all pref 27 u32 chain 0
filter parent ffff: protocol all pref 27 u32 chain 0 fh 801: ht divisor 1
filter parent ffff: protocol all pref 27 u32 chain 0 fh 801::800 order 2048 key ht 801 bkt 0 terminal flowid ??? not_in_hw
  match 4e657556/ffffffff at -8
  match 00050000/ffff0000 at -4
        action order 1:  pedit action pipe keys 2
         index 4 ref 1 bind 1
         key #0  at -8: val 0a3c8cf9 mask 00000000
         key #1  at -4: val a2660000 mask 0000ffff

        action order 2: mirred (Egress Mirror to device vex74f9-eth0) pipe
        index 6 ref 1 bind 1
```



egress规则如下：

```
tc filter show dev vbr-neuv egress
filter parent ffff: protocol all pref 5 u32 chain 0
filter parent ffff: protocol all pref 5 u32 chain 0 fh 800: ht divisor 1
filter parent ffff: protocol all pref 5 u32 chain 0 fh 800::800 order 2048 key ht 800 bkt 0 terminal flowid ??? not_in_hw
  match 00004e65/0000ffff at -16
  match 75560005/ffffffff at -12
        action order 1:  pedit action pipe keys 2
         index 3 ref 1 bind 1
         key #0  at -16: val 00000a3c mask ffff0000
         key #1  at -12: val 8cf9a266 mask 00000000

        action order 2: mirred (Egress Mirror to device vin74f9-eth0) pipe
        index 5 ref 1 bind 1

filter parent ffff: protocol all pref 27 u32 chain 0
filter parent ffff: protocol all pref 27 u32 chain 0 fh 801: ht divisor 1
filter parent ffff: protocol all pref 27 u32 chain 0 fh 801::800 order 2048 key ht 801 bkt 0 terminal flowid ??? not_in_hw
  match 4e657556/ffffffff at -8
  match 00050000/ffff0000 at -4
        action order 1:  pedit action pipe keys 2
         index 4 ref 1 bind 1
         key #0  at -8: val 0a3c8cf9 mask 00000000
         key #1  at -4: val a2660000 mask 0000ffff

        action order 2: mirred (Egress Mirror to device vex74f9-eth0) pipe
        index 6 ref 1 bind 1
```







**备注：**

Queueing Discipline (qdisc，排队规则）

我感觉核心的几个问题都没梳理清楚。

所有前端web api的处理都在rest.go文件。



在dp目录的ctrl.c文件的dp_ctrl_handler函数中



dp在学习阶段会解析数据包并记录网络规则。



切换成保护模式容器就会调用changeContainerWire方法

流量抓取先优先搞，简单的规则分析。
