# 一、为什么需要TCP会话表

仅仅解析一个数据包是不够的：难以区分一条具体 TCP 连接中包含了哪些负载数据，以及解析出上层应用协议。

数据包是无状态的，需要通过五元组 (src ip, dst ip, src port, dst port, protocol) 标识一条会话

应用场景：DFI流量统计、防火墙、DPI 等，深度包检测和深度流检测。



# 二、TCP会话表基础概念

## 2、1 会话表的作用

- 将离散的数据包聚合成流 (flow/session)
- 在表项中维护会话状态和统计信息



## 2、2 会话表的键 (key) 与值 (value)

hashmap<key,value>

key: TCP 五元组 ,(src ip, dst ip, src port, dst port, protocol) 标识一条会话

value: 包含状态机 (如 ESTABLISHED 等)、统计数据 (bps/bpp)、当前会话所包含的原始数据也存储等



## 2、3 会话表的生命周期

- 会话创建 (syn握手包或第一个数据包插入表)
- 会话更新 （后续数据包到来，更新会话统计)
- 会话销毁 (FIN/RST 或超时回收)，采用会话老化机制



# 三、tcp会话表实现机制

## **3、1 定义数据结构**

- 会话 key：五元组
- 会话 value：状态、上行/下行字节数、包数、时间戳



## **3、2 在 rte_hash 查找/插入**

- 没有找到则分配新会话 (mempool) 并插入
- 找到则更新统计信息



## **3、3 维护会话状态**（优先级低）

- 使用 TCP 标志位更新会话状态机 (SYN/FIN/RST)
- 超时清理机制：定时器扫描空闲连接



## **3、4 统计方向**（优先级低）

- 上行与下行的区分：根据源或目的 IP 是否是“本机/客户侧”来判定
- 每次更新时增加对应方向的字节数 / 包数



## 3、5 具体实现

**rte_hash**

- 用于快速查找五元组对应的会话

**rte_mempool**

- 管理会
- 话表中 value 的内存池分配和释放



创建tcp会话表后，我们还需要创建一个Packet结构体，这个结构体里面有以太网字段，有网络层（如ip）的字段，有传输层（如tcp，udp）的字段



dpdk内存池简介文档：

https://doc.dpdk.org/guides-25.07/prog_guide/mempool_lib.html



其api使用文档：

https://doc.dpdk.org/api-20.11/rte__mempool_8h.html



# 4、具体代码实现

主打一个就是纯手写代码，

然后使用ai工具cursor来帮我修复下。

## 4、1 数据结构定义

```
struct flow_key {
	uint32_t ip_src; 	//src address
	uint32_t ip_dst;	//dst address
	uint16_t port_src;	//src port
	uint16_t port_dst;	//dst port
	uint8_t proto;		//protocol
}__rte_packed;

struct flow_value {
    uint64_t packets;
    uint64_t bytes;
};
```



## 4、2 创建会话表项

当会话中不存在此会话时，创建会话表项。



## 4、3 更新会话表项

当会话中已经存在此会话表项时，则更新此表项中的统计值。



# 五、可优化的技术点

1、将数据包的所有信息存储到packet结构体中，包括ethernet、ip、tcp、udp信息

2、在第一步构造数据包Packet时，如果Packet频繁的

3、flow_value这个会话表元素也需要动态生成，现在都是malloc系统调用来申请的。

4、现在的会话表缺少老化机制，只有会话表项创建机制。

5、还缺少tcp状态机机制，可以参考suricata中的tcp状态机机制



# 六、编码过程中遇到的问题

在编程的过程中可能会遇到的问题，我给大家整理好。

问题1：同一个tcp会话的上行数据包和下行数据包，其计算出来的hash值是不同的。



问题2：rte_hash + 指针存储（存储指向动态分配数据的指针



问题3：将一个完整的项目拆分成多个小任务来实现，