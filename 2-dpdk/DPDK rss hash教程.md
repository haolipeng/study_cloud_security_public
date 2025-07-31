# 一、RSS简介和背景

RSS是**网卡**提供的**分流机制**。用来将报表分流到不同的收包队列，以提高收包性能。 

 RSS（receive side scaling）是由微软提出的一种**负载分流**方法，通过计算网络数据报文中的 网络层 &传输层二/三/四元组HASH值，取HASH值的最低有效位(LSB)用于索引**间接寻址表**RETA(Redirection Table)，间接寻址表RETA中的保存索引值用于分配数据报文到不同的CPU接收处理。

**未开启rss负载分流：**

- 所有报文只从一个硬件队列来收包



**开启rss进行负载分流：**

- 解析报文中的ip地址或者tcp/udp端口信息
- 经过hash哈希函数计算出一个rss hash值，填充到struct rte_mbuf的hash.rss字段中
- rss hash的低7位会映射到RSS output index
- 无法解析的报文，rss hash和RSS output index都设置为0



# 二、DPDK RSS技术的使用

## 2、1 结构体简介

RSS配置的数据结构

```
struct rte_eth_rss_conf{
	uint8_t* rss_key;
	uint8_t rss_key_len;
	uint64_t rss_hf;
	enum rte_eth_hash_function 	algorithm;
};
```

rss_key 字段：rss_key 数组。如果 为 NULL，40字节的hash key，留给网卡设置 rss_key。

rss_key_len：rss_key数组的字节数。

rss_hf：表明要应用 RSS 哈希的数据包类型或数据包的特定部分。

algorithm: 哈希算法，有以下几种hash函数类型



**支持的头部字段**

最多支持 16 种不同的 RSS 配置。这意味着可以配置 16 种不同的标头字段组合，其中包括端口（物理端口和虚拟端口）上的一种 RSS 配置。默认情况下，所有端口均配置外层的 5 元组哈希模式。以下标头字段的内容用于：

- 源地址
- 目的地址
- UDP，TCP或SCTP的源端口
- UDP，TCP或SCTP的目的端口
- ipv4/ipv6的协议号/下一个头部

下表显示了支持的头部字段。

| DPDK RSS 标志                  | 描述                                                         |
| :----------------------------- | :----------------------------------------------------------- |
| RTE_ETH_RSS_IPV4               | IPv4 源/目的 地址                                            |
| RTE_ETH_RSS_FRAG_IPV4          | IPv4 源/目标地址和分片组标识                                 |
| RTE_ETH_RSS_NONFRAG_IPV4_TCP   | IPv4 源/目标地址和 IP 协议（针对 TCP）                       |
| RTE_ETH_RSS_NONFRAG_IPV4_UDP   | IPv4 源/目标地址和 IP 协议（针对 UDP）                       |
| RTE_ETH_RSS_NONFRAG_IPV4_SCTP  | IPv4 源/目标地址和 IP 协议（针对 SCTP）                      |
| RTE_ETH_RSS_NONFRAG_IPV4_OTHER | IPv4 源/目标地址和 IP 协议（针对非 TCP/UDP/SCTP 协议）       |
| RTE_ETH_RSS_IPV6               | IPv4 或 IPv6 源/目标地址                                     |
| RTE_ETH_RSS_FRAG_IPV6          | IPv4 或 IPv6 源/目标地址和分片组标识                         |
| RTE_ETH_RSS_NONFRAG_IPV6_TCP   | IPv4 或 IPv6 源/目标地址和 IP 协议（针对 TCP）               |
| RTE_ETH_RSS_NONFRAG_IPV6_UDP   | IPv4 或 IPv6 源/目标地址和 IP 协议（针对 UDP）               |
| RTE_ETH_RSS_NONFRAG_IPV6_SCTP  | IPv4 或 IPv6 源/目标地址和 IP 协议（针对 SCTP）              |
| RTE_ETH_RSS_NONFRAG_IPV6_OTHER | IPv4 或 IPv6 源/目标地址和 IP 协议（针对非 TCP/UDP/SCTP 协议） |
| RTE_ETH_RSS_IPV6_EX            | 与 RTE_ETH_RSS_IPV6 相同                                     |
| RTE_ETH_RSS_IPV6_TCP_EX        | 与 RTE_ETH_RSS_IPV6_TCP 相同                                 |
| RTE_ETH_RSS_IPV6_UDP_EX        | 与 RTE_ETH_RSS_IPV6_UDP 相同                                 |
| RTE_ETH_RSS_PORT               | 第 4 层源/目标端口和 IP 协议。与 TCP、UDP 或 SCTP 协议之一一起使用 |
| RTE_ETH_RSS_GTPU               | 从最外层 GTPU 头部提取的 GTP TEID                            |
| RTE_ETH_RSS_ETH                | 源/目标 MAC 地址                                             |
| RTE_ETH_RSS_S_VLAN             | 最外层 802.1Q 标签和 IP 标识                                 |
| RTE_ETH_RSS_C_VLAN             | 最内层 802.1Q 标签和 IP 标识                                 |
| RTE_ETH_RSS_IPV4_CHKSUM        | IPv4 校验和                                                  |
| RTE_ETH_RSS_L4_CHKSUM          | 第 4 层校验和。与 TCP、UDP 或 SCTP 协议之一一起使用          |
| RTE_ETH_RSS_L2_SRC_ONLY        | 源 MAC 地址                                                  |
| RTE_ETH_RSS_L2_DST_ONLY        | 目标 MAC 地址                                                |
| RTE_ETH_RSS_L3_SRC_ONLY        | IPv4 或 IPv6 源地址。默认情况下，预期为 IPv4                 |
| RTE_ETH_RSS_L3_DST_ONLY        | IPv4 或 IPv6 目标地址。默认情况下，预期为 IPv4               |
| RTE_ETH_RSS_L4_SRC_ONLY        | 第 4 层源端口和 IP 协议                                      |
| RTE_ETH_RSS_L4_DST_ONLY        | 第 4 层目标端口和 IP 协议                                    |
| RTE_ETH_RSS_LEVEL_OUTERMOST    | 最外层。与其他 RSS 标志一起使用                              |
| RTE_ETH_RSS_LEVEL_INNERMOST    | 最内层。与其他 RSS 标志一起使用                              |

注意事项：

- 默认情况下，RTE_ETH_RSS_LEVEL_OUTERMOST 已设置。
- 如果同时配置了 RTE_ETH_RSS_LEVEL_OUTERMOST 和 RTE_ETH_RSS_LEVEL_INNERMOST，则使用 RTE_ETH_RSS_LEVEL_INNERMOST。
- 如果在同一层上同时设置了 RTE_ETH_RSS_Lx_SRC_ONLY 和 RTE_ETH_RSS_Lx_DST_ONLY标志，则这两个标志都会被忽略。
- 如果配置了任何与 IPv6 相关的 RSS 标志，则会对 IPv4 和 IPv6 地址执行哈希计算。

RTE_ETH_RSS_L3_SRC_ONLY 和 RTE_ETH_RSS_L3_DST_ONLY 默认接受 IPv4 地址。要选择 IPv6 地址标头字段，必须同时配置与 IPv6 相关的标志，例如 RTE_ETH_RSS_IPV6。



疑问点：RTE_ETH_RSS_FRAG_IPV4和其下面几个是什么差别呢？



关于哈希算法的选择

| 哈希算法的选择                                | 含义                                                         |
| --------------------------------------------- | ------------------------------------------------------------ |
| RTE_ETH_HASH_FUNCTION_DEFAULT                 | DEFAULT means driver decides which hash algorithm to pick.   |
| RTE_ETH_HASH_FUNCTION_TOEPLITZ                | Toeplitz                                                     |
| RTE_ETH_HASH_FUNCTION_SIMPLE_XOR              | Simple XOR                                                   |
| RTE_ETH_HASH_FUNCTION_SYMMETRIC_TOEPLITZ      | Symmetric Toeplitz: src, dst will be replaced by xor(src, dst). For the case with src/dst only, src or dst address will xor with zero pair. |
| RTE_ETH_HASH_FUNCTION_SYMMETRIC_TOEPLITZ_SORT | Symmetric Toeplitz: L3 and L4 fields are sorted prior to the hash function. If src_ip > dst_ip, swap src_ip and dst_ip. If src_port > dst_port, swap src_port and dst_port. |

如果你想使用对称哈希，就采用RTE_ETH_HASH_FUNCTION_SYMMETRIC_TOEPLITZ 和RTE_ETH_HASH_FUNCTION_SYMMETRIC_TOEPLITZ _SORT类型的标志。

这两个值是需要特定的网卡硬件来支持的，我测试了下intel的X540，是不支持RTE_ETH_HASH_FUNCTION_SYMMETRIC_TOEPLITZ 的。



## 2、2 对称RSS技术

当前 RSS 技术的一个问题是，根据数据包的方向，将来自同一 TCP 连接的数据包映射到不同的 NIC RX 队列。

上行流信息：

```
{src. IP=1.1.1.1, dst. IP=2.2.2.2, src. port=123, dst. port=456, prot=TCP}
```

下行流信息：

```
{src. IP=2.2.2.2, dst. IP=1.1.1.1, src. port=456, dst. port=123, prot=TCP}
```

上述上行流和下行流本身属于同一个流，如果被映射到不同的RX接收队列上的话，就导致需要在不同的逻辑核心上处理数据包，对于编写高性能程序来说是个灾难，因为需要加锁访问不同线程的共享变量或数据结构才行。



为了实现对称 RSS，需要选择用于计算数据包哈希值的哈希密钥hash key，以便哈希计算函数对两个方向的数据包产生相同的哈希值。

但是，更改 RSS 哈希键需要谨慎，因为它可能会导致处理核心之间的流量分配不佳。推荐使用微软公开的Hash值：

```
# define RSS_HASH_KEY_LENGTH 40
static uint8_t hash_key[RSS_HASH_KEY_LENGTH] = { 
    0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A,
    0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A,
    0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A,
    0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A,
};
```



RSS的优势在于它可以在硬件中实现。如果底层硬件不支持RSS，或者需要在哈希计算时使用特殊字节范围，RSS可以通过来软件实现。

DPDK提供了一个Toeplitz哈希库，用于在软件中计算Toeplitz哈希函数。

`RTE_ETH_HASH_FUNCTION_SYMMETRIC_TOEPLITZ`是DPDK提供的一种特殊哈希函数，专门用于确保IP会话的哈希一致性。这对于维护双向流量的会话亲和性至关重要，确保属于同一会话的数据包（即使方向相反）会被分配到同一CPU核心处理。



## 对称性Toeplitz哈希的工作原理

标准Toeplitz哈希计算使用源IP、目标IP、源端口和目标端口进行哈希计算。而对称性Toeplitz哈希则对这些元素进行特殊处理，使得:

```
Hash(SrcIP, DstIP, SrcPort, DstPort) = Hash(DstIP, SrcIP, DstPort, SrcPort)
```

这确保了无论是请求包还是响应包，都会被引导到相同的接收队列，从而被相同的CPU核心处理。



## 配置对称性Toeplitz哈希的完整示例

以下是一个完整的示例，展示如何在DPDK应用程序中配置对称性Toeplitz哈希：



Intel X710网卡是否支持对称哈希呢？

如果我想采用对称哈希，那么我应该如何在编写代码时进行配置呢？





参考链接：

rss的科普文章

https://docs.napatech.com/r/Link-InlineTM-Software-Features/Receive-Side-Scaling-RSS

https://haryachyy.wordpress.com/2019/01/18/learning-dpdk-symmetric-rss/