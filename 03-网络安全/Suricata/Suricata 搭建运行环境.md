**目的：为 Suricata 搭建隔离的运行测试流量环境**

Suricata 是网络入侵检测/防御系统（IDS/IPS），它需要**监听网络接口上的流量**。创建这些虚拟网络设备的目的是：**不依赖真实物理网卡，在本机构造一个可控的测试网络环境。**



## 创建veth pair接口

### 1、创建虚拟网桥接口

```
# 创建虚拟网桥
sudo ip link add name br-suricata type bridge
sudo ip link set br-suricata up
sudo ip addr add 192.168.100.1/24 dev br-suricata
```



### 2、创建虚拟以太网对（veth pair）

```
# 创建一对虚拟以太网接口
sudo ip link add veth-suri0 type veth peer name veth-suri1
sudo ip link set veth-suri0 up
sudo ip link set veth-suri1 up
sudo ip addr add 192.168.200.1/24 dev veth-suri0
sudo ip addr add 192.168.200.2/24 dev veth-suri1
```

### 3、 创建TUN/TAP接口

```
# 创建TAP接口
sudo ip tuntap add mode tap name tap-suri
sudo ip link set tap-suri up
sudo ip addr add 192.168.300.1/24 dev tap-suri
```



# 