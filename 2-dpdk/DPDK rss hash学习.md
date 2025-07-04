RSS配置的数据结构

```
struct rte_eth_rss_conf{
	uint8_t* rss_key;
	uint8_t rss_key_len;
	uint64_t rss_hf;
};
```

rss_key 字段：rss_key 数组。如果 为 NULL，40字节的hash key，留给网卡设置 rss_key。
rss_key_len：rss_key数组的字节数。
rss_hf：需要对报文的分析的元组类型。TODO：这里可以填写什么呢



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



**对称RSS技术**

当前 RSS 技术的一个问题是，根据数据包的方向，将来自同一 TCP 连接的数据包映射到不同的 NIC RX 队列。

上行流信息：

```
{src. IP=1.1.1.1, dst. IP=2.2.2.2, src. port=123, dst. port=456, prot=TCP}
```

下行流信息：

```
{src. IP=2.2.2.2, dst. IP=1.1.1.1, src. port=456, dst. port=123, prot=TCP}
```

上述上行流和下行流本身属于同一个流，如果被映射到不同的RX队列上的话，就导致需要在不同的逻辑核心上处理数据包，对于编写高性能程序来说是个灾难，因为需要加锁访问不同线程的共享变量或数据结构才行。



为了实现对称 RSS，需要选择用于计算数据包哈希值的哈希密钥hash key，以便哈希计算函数对两个方向的数据包产生相同的哈希值。

但是，更改 RSS 哈希键需要谨慎，因为它可能会导致处理核心之间的流量分配不佳。推荐使用以下Hash值：

```
# define RSS_HASH_KEY_LENGTH 40
static uint8_t hash_key[RSS_HASH_KEY_LENGTH] = { 
    0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A,
    0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A,
    0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A,
    0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A, 0x6D, 0x5A,
};
```



RSS的优势在于它可以在硬件中实现。如果底层硬件不支持RSS，或者需要在哈希计算时使用特殊字节范围，RSS可以通过软件实现。

DPDK提供了一个Toeplitz哈希库，用于在软件中计算Toeplitz哈希函数。





TODO： 

参考链接：

rss的科普文章

https://docs.napatech.com/r/Link-InlineTM-Software-Features/Receive-Side-Scaling-RSS

https://haryachyy.wordpress.com/2019/01/18/learning-dpdk-symmetric-rss/