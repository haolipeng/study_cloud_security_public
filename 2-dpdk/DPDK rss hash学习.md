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

`RTE_ETH_HASH_FUNCTION_SYMMETRIC_TOEPLITZ`是DPDK提供的一种特殊哈希函数，专门用于确保IP会话的哈希一致性。这对于维护双向流量的会话亲和性至关重要，确保属于同一会话的数据包（即使方向相反）会被分配到同一CPU核心处理。



## 对称性Toeplitz哈希的工作原理

标准Toeplitz哈希计算使用源IP、目标IP、源端口和目标端口进行哈希计算。而对称性Toeplitz哈希则对这些元素进行特殊处理，使得:

```
Hash(SrcIP, DstIP, SrcPort, DstPort) = Hash(DstIP, SrcIP, DstPort, SrcPort)
```

这确保了无论是请求包还是响应包，都会被引导到相同的接收队列，从而被相同的CPU核心处理。



## 配置对称性Toeplitz哈希的完整示例

以下是一个完整的示例，展示如何在DPDK应用程序中配置对称性Toeplitz哈希：

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>
#include <inttypes.h>
#include <sys/types.h>
#include <sys/queue.h>
#include <setjmp.h>
#include <stdarg.h>
#include <errno.h>
#include <getopt.h>
#include <signal.h>
#include <stdbool.h>

#include <rte_common.h>
#include <rte_log.h>
#include <rte_malloc.h>
#include <rte_memory.h>
#include <rte_memcpy.h>
#include <rte_eal.h>
#include <rte_launch.h>
#include <rte_atomic.h>
#include <rte_cycles.h>
#include <rte_prefetch.h>
#include <rte_lcore.h>
#include <rte_per_lcore.h>
#include <rte_branch_prediction.h>
#include <rte_interrupts.h>
#include <rte_random.h>
#include <rte_debug.h>
#include <rte_ether.h>
#include <rte_ethdev.h>
#include <rte_mempool.h>
#include <rte_mbuf.h>
#include <rte_ip.h>
#include <rte_tcp.h>
#include <rte_udp.h>

#define RX_RING_SIZE 1024
#define TX_RING_SIZE 1024

#define NUM_MBUFS 8191
#define MBUF_CACHE_SIZE 250
#define BURST_SIZE 32

// 全局变量
static volatile bool force_quit;
static uint16_t port_id = 0;
static struct rte_mempool *mbuf_pool = NULL;

// 设置RSS配置的函数
static int
setup_symmetric_hash(uint16_t port_id) {
    struct rte_eth_rss_conf rss_conf;
    struct rte_eth_dev_info dev_info;
    int ret;
    
    // 获取设备信息
    ret = rte_eth_dev_info_get(port_id, &dev_info);
    if (ret != 0) {
        printf("Error getting device info: %s\n", strerror(-ret));
        return ret;
    }
    
    // 检查设备是否支持对称哈希
    if (!(dev_info.hash_key_size > 0 && 
          (dev_info.flow_type_rss_offloads & RTE_ETH_RSS_IP))) {
        printf("Device does not support RSS or required RSS types\n");
        return -ENOTSUP;
    }
    
    // 准备RSS配置
    uint8_t hash_key[dev_info.hash_key_size];
    
    // 生成随机哈希密钥
    for (uint32_t i = 0; i < dev_info.hash_key_size; i++)
        hash_key[i] = (uint8_t)rte_rand();
    
    // 设置RSS配置
    memset(&rss_conf, 0, sizeof(rss_conf));
    rss_conf.rss_key = hash_key;
    rss_conf.rss_key_len = dev_info.hash_key_size;
    rss_conf.rss_hf = RTE_ETH_RSS_IP | RTE_ETH_RSS_TCP | RTE_ETH_RSS_UDP;
    
    // 设置RSS哈希函数为对称Toeplitz
    struct rte_eth_conf port_conf = {0};
    port_conf.rxmode.mq_mode = RTE_ETH_MQ_RX_RSS;
    port_conf.rx_adv_conf.rss_conf = rss_conf;
    
    // 这是关键设置 - 使用对称性Toeplitz哈希
    port_conf.rx_adv_conf.rss_conf.algorithm = RTE_ETH_HASH_FUNCTION_SYMMETRIC_TOEPLITZ;
    
    // 配置端口
    ret = rte_eth_dev_configure(port_id, dev_info.max_rx_queues, 
                               dev_info.max_tx_queues, &port_conf);
    if (ret < 0) {
        printf("Failed to configure device: %s\n", strerror(-ret));
        return ret;
    }
    
    printf("Symmetric Toeplitz hash configured successfully\n");
    return 0;
}

// 初始化端口
static int
port_init(uint16_t port, struct rte_mempool *mbuf_pool, uint16_t nb_rxq, uint16_t nb_txq) {
    struct rte_eth_conf port_conf = {0};
    const uint16_t rx_rings = nb_rxq;
    const uint16_t tx_rings = nb_txq;
    uint16_t nb_rxd = RX_RING_SIZE;
    uint16_t nb_txd = TX_RING_SIZE;
    int retval;
    uint16_t q;
    struct rte_eth_dev_info dev_info;
    struct rte_eth_txconf txconf;
    struct rte_eth_rxconf rxconf;
    
    if (!rte_eth_dev_is_valid_port(port))
        return -1;
    
    retval = rte_eth_dev_info_get(port, &dev_info);
    if (retval != 0) {
        printf("Error during getting device info: %s\n", strerror(-retval));
        return retval;
    }
    
    // 设置RSS配置
    port_conf.rxmode.mq_mode = RTE_ETH_MQ_RX_RSS;
    port_conf.rx_adv_conf.rss_conf.rss_key = NULL;
    port_conf.rx_adv_conf.rss_conf.rss_hf = 
        RTE_ETH_RSS_IP | RTE_ETH_RSS_TCP | RTE_ETH_RSS_UDP;
    
    // 设置对称性Toeplitz哈希算法
    port_conf.rx_adv_conf.rss_conf.algorithm = RTE_ETH_HASH_FUNCTION_SYMMETRIC_TOEPLITZ;
    
    // 配置设备
    retval = rte_eth_dev_configure(port, rx_rings, tx_rings, &port_conf);
    if (retval != 0)
        return retval;
    
    retval = rte_eth_dev_adjust_nb_rx_tx_desc(port, &nb_rxd, &nb_txd);
    if (retval != 0)
        return retval;
    
    // 初始化RX队列
    rxconf = dev_info.default_rxconf;
    rxconf.offloads = port_conf.rxmode.offloads;
    
    for (q = 0; q < rx_rings; q++) {
        retval = rte_eth_rx_queue_setup(port, q, nb_rxd,
                rte_eth_dev_socket_id(port), &rxconf, mbuf_pool);
        if (retval < 0)
            return retval;
    }
    
    // 初始化TX队列
    txconf = dev_info.default_txconf;
    txconf.offloads = port_conf.txmode.offloads;
    
    for (q = 0; q < tx_rings; q++) {
        retval = rte_eth_tx_queue_setup(port, q, nb_txd,
                rte_eth_dev_socket_id(port), &txconf);
        if (retval < 0)
            return retval;
    }
    
    // 启动设备
    retval = rte_eth_dev_start(port);
    if (retval < 0)
        return retval;
    
    printf("Port %u MAC: ", port);
    struct rte_ether_addr addr;
    retval = rte_eth_macaddr_get(port, &addr);
    if (retval != 0)
        return retval;
    
    printf("%02X:%02X:%02X:%02X:%02X:%02X\n",
        addr.addr_bytes[0], addr.addr_bytes[1],
        addr.addr_bytes[2], addr.addr_bytes[3],
        addr.addr_bytes[4], addr.addr_bytes[5]);
    
    return 0;
}

// 打印数据包五元组信息和接收队列
static void
print_flow_info(struct rte_mbuf *m, uint16_t rx_queue_id) {
    struct rte_ipv4_hdr *ip_hdr;
    struct rte_tcp_hdr *tcp_hdr;
    struct rte_udp_hdr *udp_hdr;
    
    // 提取以太网头部
    struct rte_ether_hdr *eth_hdr = rte_pktmbuf_mtod(m, struct rte_ether_hdr *);
    
    // 检查是否是IPv4数据包
    if (eth_hdr->ether_type != rte_cpu_to_be_16(RTE_ETHER_TYPE_IPV4))
        return;
    
    ip_hdr = (struct rte_ipv4_hdr *)(eth_hdr + 1);
    
    // 打印源IP和目标IP
    char src_ip[INET_ADDRSTRLEN];
    char dst_ip[INET_ADDRSTRLEN];
    
    inet_ntop(AF_INET, &(ip_hdr->src_addr), src_ip, INET_ADDRSTRLEN);
    inet_ntop(AF_INET, &(ip_hdr->dst_addr), dst_ip, INET_ADDRSTRLEN);
    
    // 根据协议类型提取端口信息
    if (ip_hdr->next_proto_id == IPPROTO_TCP) {
        tcp_hdr = (struct rte_tcp_hdr *)((unsigned char *)ip_hdr + 
                (ip_hdr->version_ihl & 0x0f) * 4);
        
        printf("TCP: %s:%d -> %s:%d | Queue: %d\n",
               src_ip, rte_be_to_cpu_16(tcp_hdr->src_port),
               dst_ip, rte_be_to_cpu_16(tcp_hdr->dst_port),
               rx_queue_id);
    }
    else if (ip_hdr->next_proto_id == IPPROTO_UDP) {
        udp_hdr = (struct rte_udp_hdr *)((unsigned char *)ip_hdr + 
                (ip_hdr->version_ihl & 0x0f) * 4);
        
        printf("UDP: %s:%d -> %s:%d | Queue: %d\n",
               src_ip, rte_be_to_cpu_16(udp_hdr->src_port),
               dst_ip, rte_be_to_cpu_16(udp_hdr->dst_port),
               rx_queue_id);
    }
    else {
        printf("IP: %s -> %s | Proto: %d | Queue: %d\n",
               src_ip, dst_ip, ip_hdr->next_proto_id,
               rx_queue_id);
    }
}

// 单个逻辑核心的主循环
static int
lcore_main(void *arg) {
    uint16_t port = (uintptr_t)arg;
    uint16_t lcore_id = rte_lcore_id();
    uint16_t queue_id = lcore_id; // 假设lcore_id映射到相应的队列
    
    printf("Core %u handling port %u queue %u\n", lcore_id, port, queue_id);
    
    while (!force_quit) {
        struct rte_mbuf *bufs[BURST_SIZE];
        uint16_t nb_rx = rte_eth_rx_burst(port, queue_id, bufs, BURST_SIZE);
        
        if (unlikely(nb_rx == 0))
            continue;
        
        for (uint16_t i = 0; i < nb_rx; i++) {
            // 打印流信息，观察对称哈希的效果
            print_flow_info(bufs[i], queue_id);
            rte_pktmbuf_free(bufs[i]);
        }
    }
    
    return 0;
}

// 信号处理
static void
signal_handler(int signum) {
    if (signum == SIGINT || signum == SIGTERM) {
        printf("\n\nSignal %d received, preparing to exit...\n", signum);
        force_quit = true;
    }
}

int
main(int argc, char **argv) {
    int ret;
    uint16_t nb_rxq, nb_txq;
    
    // 初始化EAL
    ret = rte_eal_init(argc, argv);
    if (ret < 0)
        rte_exit(EXIT_FAILURE, "Error with EAL initialization\n");
    
    argc -= ret;
    argv += ret;
    
    // 设置信号处理
    force_quit = false;
    signal(SIGINT, signal_handler);
    signal(SIGTERM, signal_handler);
    
    // 创建内存池
    mbuf_pool = rte_pktmbuf_pool_create("MBUF_POOL", NUM_MBUFS,
                                       MBUF_CACHE_SIZE, 0,
                                       RTE_MBUF_DEFAULT_BUF_SIZE,
                                       rte_socket_id());
    if (mbuf_pool == NULL)
        rte_exit(EXIT_FAILURE, "Cannot create mbuf pool\n");
    
    // 获取可用的逻辑核心数量
    nb_rxq = rte_lcore_count() - 1; // 减去主核心
    nb_txq = rte_lcore_count() - 1;
    
    // 确保至少有一个接收和发送队列
    if (nb_rxq < 1)
        nb_rxq = 1;
    if (nb_txq < 1)
        nb_txq = 1;
    
    // 初始化端口
    if (port_init(port_id, mbuf_pool, nb_rxq, nb_txq) != 0)
        rte_exit(EXIT_FAILURE, "Cannot init port %"PRIu16 "\n", port_id);
    
    // 在每个逻辑核心上启动数据包处理
    unsigned lcore_id;
    unsigned core_num = 0;
    
    RTE_LCORE_FOREACH_SLAVE(lcore_id) {
        if (core_num < nb_rxq) {
            rte_eal_remote_launch(lcore_main, (void*)(uintptr_t)port_id, lcore_id);
            core_num++;
        }
    }
    
    // 等待所有线程完成
    rte_eal_mp_wait_lcore();
    
    // 关闭端口并清理
    rte_eth_dev_stop(port_id);
    rte_eth_dev_close(port_id);
    rte_eal_cleanup();
    
    printf("Bye...\n");
    return 0;
}
```



TODO： 

参考链接：

rss的科普文章

https://docs.napatech.com/r/Link-InlineTM-Software-Features/Receive-Side-Scaling-RSS

https://haryachyy.wordpress.com/2019/01/18/learning-dpdk-symmetric-rss/