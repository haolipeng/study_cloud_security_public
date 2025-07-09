# NetDefender：基于DPDK的高性能网络入侵检测系统

这个项目，我们要知道都有哪些功能需要实现？

就是提着一口气上来这个事。

关于无锁队列这个事，我感觉性能会好点，但是也别都寄予期望于这个上面。

对于自己怎么来说，都是个很好的学习过程。不用太妄自菲薄这个事。



在dpdk中每个线程都有自己的角色，比如有的线程是收包和解析数据，



suricata的检测引擎是占用了独立的线程的吗，我们的需求是什么？



# 1. 项目简介

## 1.1 学习目标与技术挑战

本项目旨在构建一个基于DPDK (Data Plane Development Kit) 的高性能网络入侵检测系统(NIDS)，作为个人技术学习和开源贡献的实践平台。通过此项目，计划达成以下学习目标：

1. **深入理解DPDK框架**：掌握DPDK的核心组件、内存管理、多核编程模型和高性能网络I/O技术
2. **网络安全技术实践**：探索现代网络入侵检测技术，包括协议分析、特征匹配和异常流量识别
3. **高性能系统设计**：学习大规模、高吞吐量系统架构设计方法和性能优化技术
4. **实时数据处理**：掌握高速网络环境下的数据捕获、分析和实时响应技术
5. **开源IDS代码研究**：通过深入学习Suricata和Zeek等开源项目的源代码，理解业界主流的网络入侵防御系统的实现细节，并将其优秀的思想应用到本开源项目中



项目面临的主要技术挑战包括：

1. **性能挑战**：在10Gbps及以上网络速率下实现零丢包捕获和实时分析
2. **多核并行处理**：设计高效的多核负载均衡和数据分发机制，避免核间竞争和缓存一致性问题
3. **内存使用效率**：优化内存分配和访问模式，减少锁竞争和缓存缺失
4. **检测精度与速度平衡**：在保证检测准确性的同时维持高吞吐量，解决安全分析中的深度与广度矛盾
5. **流量重组与状态跟踪**：高效实现TCP/IP流重组和应用层协议状态跟踪，应对网络攻击中的规避技术
6. **DPDK与传统IDS组件集成**：将Suricata等成熟IDS的核心检测组件移植到DPDK环境，解决接口适配和性能优化问题

## 1.2 架构设计思路

本NIDS系统采用分层模块化设计，将整体架构分为控制平面和数据平面两大部分：

**控制平面**专注于系统管理、配置和监控：

- 采用轻量级设计，不参与数据包处理的关键路径
- 提供RESTful API接口，便于集成到更大型监控系统
- 实现基于文件的规则管理和动态规则更新机制
- 收集并展示系统性能指标和安全事件统计

**数据平面**作为系统核心，执行高性能包处理：

- 直接基于DPDK编程，绕过传统操作系统网络栈
- 实现多级流水线设计，将数据包处理分解为多个专门阶段
- 采用无锁设计和批处理机制最大化吞吐量
- 使用共享无状态设计原则提高可扩展性

整体架构特点：

1. **松耦合设计**：通过明确定义的接口分离各功能模块，便于独立开发和测试
2. **NUMA感知**：针对现代多处理器服务器优化内存和计算资源分配
3. **可扩展性**：支持从单核心原型到多节点集群的平滑扩展
4. **开发友好**：提供插件机制支持自定义规则和检测逻辑，便于社区贡献
5. **兼容现有生态**：设计与Suricata兼容的接口，便于复用其检测规则和组件

## 1.3 关键技术点概述

本项目实现将聚焦以下关键技术点：

1. **高性能数据包捕获技术**
   - DPDK PMD (Poll Mode Driver) 零拷贝包处理
   - 基于RSS的多队列接收和负载均衡
   - 大页内存和NUMA优化数据结构
2. **流量分类与会话管理**
   - 基于五元组的流识别和跟踪
   - 高性能哈希表设计与冲突处理
   - 流超时管理和资源回收机制
3. **协议解析与重组**
   - IP分片重组和TCP流重组技术
   - 应用层协议识别和状态跟踪
   - 流内容规范化处理
4. **检测引擎实现**
   - 基于Suricata检测引擎架构的DPDK优化版本
   - 多模式字符串匹配算法（AC自动机、Wu-Manber等）
   - 正则表达式匹配引擎（与Hyperscan集成）
   - 异常行为检测和启发式规则处理
5. **多核并行处理架构**
   - Run-to-Completion与Pipeline处理模型结合
   - 无锁数据结构和单生产者单消费者队列
   - 工作线程动态调度与负载均衡
6. **性能优化技术**
   - SIMD指令集加速关键算法
   - 缓存感知数据结构设计
   - 批处理技术减少函数调用开销
7. **检测规则管理**
   - 完全兼容Suricata/Snort规则格式
   - 规则编译和优化技术
   - 基于内存映射的规则快速更新
8. **Suricata组件移植与优化**
   - 将Suricata的协议解析器和检测引擎移植到DPDK环境
   - 优化内存管理和线程模型以适应DPDK架构
   - 保持规则兼容性和检测功能的完整性



# 2. 总体架构设计

## 2.1 系统逻辑架构

### 2.1.1 多层架构划分

**TODO:这里缺少一张图**

本NIDS系统采用分层架构设计，从逻辑上划分为控制平面和数据平面两大部分，各自包含多个功能层次：

**控制平面**：

- **管理层**：提供配置管理、规则管理、系统监控和用户交互界面
- **通信层**：负责控制平面与数据平面之间的消息传递和命令下发

**数据平面**：

- **输入分发层**：负责高性能数据包捕获和多核心负载均衡
- **协议处理层**：执行协议解析、流重组和会话管理
- **检测引擎层**：实现规则匹配、异常检测和深度包检测
- **输出处理层**：处理告警生成、事件记录和可选的阻断响应

这种分层设计确保了系统的模块化，使各组件可以独立开发和优化，同时保持清晰的数据流路径。



### 2.1.2 模块功能定义

**输入分发层**：

- **DPDK包接收模块**：利用DPDK PMD驱动直接从网卡接收数据包
- **RSS配置模块**：配置网卡RSS哈希功能，实现初步的流分类
- **软件分发器**：基于五元组哈希进行流分类，将相同流的包分配至同一处理核心

**协议处理层**：

- **流表管理器**：维护活动网络流的状态表，支持快速查询和超时管理
- **TCP重组模块**：处理TCP分段重组，确保能检测跨多个数据包的威胁
- **协议解析器**：解析以太网、IP、TCP/UDP等协议，提取关键字段

**检测引擎层**：

- **快速过滤器**：使用轻量级规则和哈希技术进行初步筛选
- **字符串匹配引擎**：基于Aho-Corasick或Hyperscan算法的多模式匹配
- **正则表达式引擎**：处理复杂的模式匹配需求
- **协议异常检测器**：识别违反协议规范的行为

**输出处理层**：

- **告警生成器**：将检测到的威胁转换为结构化告警
- **统计收集器**：收集性能和安全相关指标
- **日志记录器**：记录系统事件和检测结果

**控制平面管理层**：

- **规则管理器**：负责规则导入、编译和分发
- **配置管理器**：处理系统配置参数
- **监控仪表盘**：展示系统状态和安全事件统计



### 2.1.3 模块间交互关系

1. **数据平面内部交互**：
   - 采用高效的内存共享机制，最小化模块间数据传递开销
   - 使用无锁环形缓冲区(rte_ring)在处理阶段之间传递数据包或事件
   - 实现批处理模式(Burst Mode)，一次处理多个数据包以提高吞吐量
   - 使用指针传递而非数据复制，减少内存带宽消耗
2. **控制平面与数据平面交互**：
   - 采用异步消息传递机制(如ZeroMQ或自定义共享内存通道)
   - 控制命令采用轻量级JSON格式或二进制格式
   - 数据平面定期向控制平面报告性能指标和检测统计
   - 规则更新采用原子操作确保无中断更新
3. **部署拓扑与交互**：
   - 支持单节点部署模式，控制平面与数据平面共享物理资源
   - 控制平面进程使用非DPDK核心运行，数据平面进程独占DPDK核心



## 2.2 DPDK应用框架设计

### 2.2.1 EAL初始化与资源分配

**EAL初始化流程**：

```
int main(int argc, char **argv) {
    // 初始化DPDK EAL
    int ret = rte_eal_init(argc, argv);
    if (ret < 0) {
        rte_exit(EXIT_FAILURE, "Error initializing EAL: %s\n", 
                 rte_strerror(rte_errno));
    }
    
    // 更新处理后的argc/argv
    argc -= ret;
    argv += ret;
    
    // 应用程序特定的初始化...
}
```

**关键资源分配策略**：

- **大页内存**：预分配至少4GB大页内存(-m 4096参数)
- **内存通道**：根据硬件平台配置内存通道(-n 4参数)
- **保留核心**：控制平面保留1-2个核心，其余分配给数据平面
- **代码块**：

```
// 设置内存池配置
#define MBUF_CACHE_SIZE 512
#define NUM_MBUFS 65536

// 为每个活动端口创建一个内存池
for (port_id = 0; port_id < nb_ports; port_id++) {
    char pool_name[RTE_MEMPOOL_NAMESIZE];
    snprintf(pool_name, RTE_MEMPOOL_NAMESIZE, "mbuf_pool_%u", port_id);
    
    // 根据NUMA节点放置内存池
    unsigned int socket_id = rte_eth_dev_socket_id(port_id);
    
    mbuf_pools[port_id] = rte_pktmbuf_pool_create(
        pool_name, 
        NUM_MBUFS,
        MBUF_CACHE_SIZE, 
        0, 
        RTE_MBUF_DEFAULT_BUF_SIZE, 
        socket_id);
        
    if (mbuf_pools[port_id] == NULL)
        rte_exit(EXIT_FAILURE, "Cannot create mbuf pool for port %u\n", port_id);
}
```

### 2.3.2 线程模型与CPU亲和性设计

**线程模型选择**：采用Run-to-Completion模型与Pipeline模型结合的方式

**Run-to-Completion模型**：

- 每个工作线程完成从数据包接收到检测的全部处理
- 适用于单一或简单检测规则的场景
- 最小化核心间数据传输开销

**Pipeline模型**：

- 将处理分为专门阶段：接收->协议分析->检测->输出
- 每个阶段由专门的线程组负责
- 通过rte_ring实现阶段间高效数据传递
- 适用于复杂规则集和深度检测场景

**CPU亲和性配置**：

```
// 数据包接收线程配置示例
static int launch_rx_core(void *arg) {
    unsigned int lcore_id = rte_lcore_id();
    unsigned int core_num = (uintptr_t)arg;
    unsigned int port_id = core_num / num_rx_cores_per_port;
    unsigned int queue_id = core_num % num_rx_cores_per_port;
    
    printf("RX core %u handling port %u queue %u\n", lcore_id, port_id, queue_id);
    
    // 将线程固定到特定NUMA节点上的核心
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(lcore_id, &cpuset);
    
    if (pthread_setaffinity_np(pthread_self(), sizeof(cpuset), &cpuset) < 0) {
        fprintf(stderr, "Error setting CPU affinity\n");
        return -1;
    }
    
    // 实际的接收循环
    rx_loop(port_id, queue_id);
    return 0;
}
```

**线程分配策略**：

- 控制平面：使用系统默认调度策略，不参与DPDK核心分配
- 数据平面接收线程：每个物理端口至少1个专用线程
- 数据平面处理线程：根据规则复杂度和流量特征分配2-8个处理核心
- 确保接收和处理线程位于相同NUMA节点，减少跨节点访问

### 2.3.3 NUMA感知内存分配策略

**NUMA拓扑感知**：

```
// 启动时检测NUMA配置
void check_numa_config(void) {
    printf("NUMA support: %s\n", rte_socket_numa_on() ? "enabled" : "disabled");
    
    int num_nodes = rte_socket_count();
    printf("Number of NUMA nodes: %d\n", num_nodes);
    
    // 打印每个NUMA节点的核心和端口映射
    for (int i = 0; i < num_nodes; i++) {
        printf("NUMA Node %d cores: ", i);
        
        unsigned int lcore_id;
        RTE_LCORE_FOREACH(lcore_id) {
            if (rte_lcore_to_socket_id(lcore_id) == i)
                printf("%u ", lcore_id);
        }
        
        printf("\nNUMA Node %d ports: ", i);
        for (uint16_t port_id = 0; port_id < rte_eth_dev_count_avail(); port_id++) {
            if (rte_eth_dev_socket_id(port_id) == i)
                printf("%u ", port_id);
        }
        printf("\n");
    }
}
```

**内存分配策略**：

- 为每个NUMA节点创建独立的内存池，避免跨节点访问
- 将网络设备与相同NUMA节点的核心绑定
- 针对每个NUMA节点创建独立的流表和规则缓存



**内存池配置示例**：

```
// 为每个NUMA节点创建一个内存池
struct rte_mempool *create_numa_aware_mempools(void) {
    int num_nodes = rte_socket_count();
    struct rte_mempool **pools = calloc(num_nodes, sizeof(struct rte_mempool *));
    
    for (int node_id = 0; node_id < num_nodes; node_id++) {
        char pool_name[RTE_MEMPOOL_NAMESIZE];
        snprintf(pool_name, RTE_MEMPOOL_NAMESIZE, "mbuf_pool_node%d", node_id);
        
        pools[node_id] = rte_pktmbuf_pool_create(
            pool_name,
            NUM_MBUFS_PER_NODE,
            MBUF_CACHE_SIZE,
            0,
            RTE_MBUF_DEFAULT_BUF_SIZE,
            node_id);
            
        if (pools[node_id] == NULL) {
            fprintf(stderr, "Failed to create mempool for NUMA node %d\n", node_id);
            // 清理已创建的内存池
            for (int i = 0; i < node_id; i++)
                rte_mempool_free(pools[i]);
            free(pools);
            return NULL;
        }
    }
    
    return pools;
}

// 根据NUMA节点获取适当的内存池
static inline struct rte_mempool *get_mempool_for_core(void) {
    unsigned int socket_id = rte_socket_id();
    return mbuf_pools[socket_id];
}
```

**共享数据结构处理**：

- 对于需要在核心间共享的数据结构（如流表），使用以下策略：
  1. 主副本放置在访问最频繁的NUMA节点
  2. 考虑使用分片设计，每个NUMA节点管理自己的分片
  3. 对于读多写少的数据（如规则集），在每个NUMA节点保存只读副本
  4. 利用rte_hash的多读者单写者模式减少锁竞争

通过这些NUMA感知策略，系统可以充分利用现代多插槽服务器架构，显著减少跨NUMA节点访问，提高缓存利用率和整体性能。



# 3. 数据平面设计

## 3.1 数据包捕获与接收路径

### 3.1.1 网卡配置与优化

网卡配置是DPDK应用性能的关键基础，对于NIDS系统尤为重要。根据我多年DPDK优化经验，以下配置对于确保高性能捕获至关重要：

**驱动配置与参数优化：**

```
struct rte_eth_conf port_conf = {
    .rxmode = {
        .mq_mode = ETH_MQ_RX_RSS,
        .max_rx_pkt_len = RTE_ETHER_MAX_LEN,
        .split_hdr_size = 0,
        .offloads = DEV_RX_OFFLOAD_CHECKSUM | DEV_RX_OFFLOAD_JUMBO_FRAME
    },
    .rx_adv_conf = {
        .rss_conf = {
            .rss_key = NULL,
            .rss_hf = ETH_RSS_IP | ETH_RSS_TCP | ETH_RSS_UDP
        }
    },
    .txmode = {
        .mq_mode = ETH_MQ_TX_NONE,
        .offloads = DEV_TX_OFFLOAD_IPV4_CKSUM | DEV_TX_OFFLOAD_TCP_CKSUM
    }
};
```

**网卡硬件优化设置：**

1. **描述符环配置**：

```
// 优化的RX/TX描述符数量配置
#define RX_DESC_DEFAULT 2048
#define TX_DESC_DEFAULT 1024

// 端口配置核心函数示例
static int port_init(uint16_t port_id, struct rte_mempool *mbuf_pool)
{
    ret = rte_eth_dev_configure(port_id, rx_rings, tx_rings, &port_conf);
    if (ret != 0)
        return ret;

    for (q = 0; q < rx_rings; q++) {
        ret = rte_eth_rx_queue_setup(port_id, q, RX_DESC_DEFAULT,
                rte_eth_dev_socket_id(port_id), NULL, mbuf_pool);
        if (ret < 0)
            return ret;
    }

    for (q = 0; q < tx_rings; q++) {
        ret = rte_eth_tx_queue_setup(port_id, q, TX_DESC_DEFAULT,
                rte_eth_dev_socket_id(port_id), NULL);
        if (ret < 0)
            return ret;
    }
}
```

**网卡收包优化参数**：

- **预取距离优化**：`RTE_PMD_PARAM_RX_REARM_THRESH=64`
- **批量接收参数**：`RTE_PMD_PARAM_RX_FREE_THRESH=64`
- **Jumbo帧支持**：对于需要分析完整应用层内容的NIDS，配置最大9K MTU

**网卡中断抑制**：

```
// 禁用网卡中断，纯轮询模式
struct rte_eth_dev_info dev_info;
struct rte_intr_conf intr_conf;
rte_eth_dev_info_get(port_id, &dev_info);
intr_conf.lsc = 0;  // 禁用链路状态中断
intr_conf.rxq = 0;  // 禁用RX队列中断
```

### 3.1.2 RSS与多队列设计

RSS (Receive Side Scaling) 是现代多核系统下网络流量分发的关键技术。在NIDS系统中，正确配置RSS能够显著提高处理效率：

**RSS哈希配置**：

```
// 配置RSS哈希键和函数
uint8_t rss_key[40]; // 自定义或使用默认键
struct rte_eth_rss_conf rss_conf = {
    .rss_key = rss_key,
    .rss_key_len = sizeof(rss_key),
    .rss_hf = ETH_RSS_IP | ETH_RSS_TCP | ETH_RSS_UDP
};

// 在特殊情况下可调整RSS哈希算法
rte_eth_dev_rss_hash_conf_get(port_id, &rss_conf);
// 修改哈希配置...
rte_eth_dev_rss_hash_update(port_id, &rss_conf);
```

**多队列设计策略**：

1. **队列数量优化**：
   - 收包队列数 = min(网卡最大RSS队列, 可用处理核心数)
   - 发包队列数 = min(网卡最大发送队列, 处理核心数/2)
2. **NUMA映射优化**：

```
// 队列到NUMA节点映射
for (int i = 0; i < nb_queues; i++) {
    int socket_id = get_port_socket(port_id);
    int lcore_id = get_lcore_for_queue(i, socket_id);
    queue_to_core_mapping[i] = lcore_id;
}
```



**RSS高级配置**：

```
// 用于IDS的高级RSS配置示例
static int setup_rss_advanced(uint16_t port_id)
{
    // 创建流表规则以细化RSS行为
    struct rte_flow_attr attr = { .ingress = 1 };
    struct rte_flow_item pattern[MAX_PATTERN_NUM];
    struct rte_flow_action action[MAX_ACTION_NUM];
    struct rte_flow *flow = NULL;
    
    // 配置基于五元组的RSS
    struct rte_flow_action_rss rss_conf = {
        .types = ETH_RSS_IP | ETH_RSS_TCP | ETH_RSS_UDP,
        .key_len = rss_key_len,
        .key = rss_key,
        .queue_num = nb_rx_queues,
        .queue = rx_queue_list
    };
    
    // 设置action为RSS分发
    action[0].type = RTE_FLOW_ACTION_TYPE_RSS;
    action[0].conf = &rss_conf;
    action[1].type = RTE_FLOW_ACTION_TYPE_END;
    
    // 设置匹配所有TCP流量的pattern
    configure_ipv4_tcp_pattern(pattern);
    
    // 创建流表规则
    flow = create_flow(port_id, &attr, pattern, action);
    
    return flow ? 0 : -1;
}
```

### 3.1.3 零丢包设计策略

NIDS系统必须确保零丢包，否则可能导致安全隐患。以下是实现零丢包的关键技术设计：

**预分配资源池**：

```
// 大尺寸mbuf池确保足够的接收缓冲区
#define MBUF_POOL_SIZE 256*1024  // 大容量mbuf池
#define MBUF_CACHE_SIZE 512

struct rte_mempool *create_oversized_mbuf_pool(uint16_t port_id)
{
    char pool_name[32];
    snprintf(pool_name, sizeof(pool_name), "mbuf_pool_p%d", port_id);
    
    int socket_id = rte_eth_dev_socket_id(port_id);
    return rte_pktmbuf_pool_create(pool_name, MBUF_POOL_SIZE,
            MBUF_CACHE_SIZE, 0, RTE_MBUF_DEFAULT_BUF_SIZE, socket_id);
}
```



**批量接收机制**：

```
// 高效率的批量包处理
#define BURST_SIZE 64

static int packet_receive_loop(uint16_t port_id, uint16_t queue_id)
{
    struct rte_mbuf *pkts_burst[BURST_SIZE];
    uint16_t nb_rx;
    
    while (!force_quit) {
        // 批量接收
        nb_rx = rte_eth_rx_burst(port_id, queue_id, pkts_burst, BURST_SIZE);
        
        if (nb_rx == 0)
            continue;
            
        // 快速记录丢包统计
        if (nb_rx == BURST_SIZE) {
            rx_full_burst_count++;
            // 可能需要扩大burst大小或增加处理线程
        }
        
        // 处理接收到的数据包...
        process_packet_batch(pkts_burst, nb_rx);
        
        // 释放已处理的mbuf
        for (int i = 0; i < nb_rx; i++)
            rte_pktmbuf_free(pkts_burst[i]);
    }
    
    return 0;
}
```

## 3.2 数据包解析与预处理

### 3.2.1 协议解析框架

协议解析是NIDS系统的基础组件，其性能直接影响整个系统的吞吐量：

**分层协议解析器架构**：

```
// 定义协议解析接口
typedef int (*protocol_parser_fn)(const uint8_t *data, uint32_t data_len, void *context);

struct protocol_parser {
    uint16_t protocol_id;
    protocol_parser_fn parse;
    void *context;
    struct protocol_parser *next_parsers[MAX_NEXT_PROTOCOLS];
    uint8_t num_next_parsers;
};

// 协议解析器管理全局表
struct protocol_parser parsers[MAX_PROTOCOL_ID] = {0};

// 注册以太网解析器示例
static int eth_parser(const uint8_t *data, uint32_t data_len, void *context)
{
    struct rte_ether_hdr *eth_hdr = (struct rte_ether_hdr *)data;
    uint16_t ether_type = rte_be_to_cpu_16(eth_hdr->ether_type);
    
    // 跳过以太网头部，继续解析
    uint32_t offset = sizeof(struct rte_ether_hdr);
    
    // 查找下一层协议解析器
    for (int i = 0; i < parsers[PROTO_ETHERNET].num_next_parsers; i++) {
        struct protocol_parser *next = parsers[PROTO_ETHERNET].next_parsers[i];
        if (next->protocol_id == ether_type_to_proto_id(ether_type)) {
            return next->parse(data + offset, data_len - offset, next->context);
        }
    }
    
    return PARSER_UNKNOWN_NEXT;
}
```

**高效数据包解析优化**：

1. **批量解析**：

```
// 批量解析多个数据包以提高缓存效率
static void batch_parse_packets(struct rte_mbuf **pkts, uint16_t nb_pkts,
                              struct packet_metadata *meta)
{
    // 首先执行数据包类型分组
    group_packets_by_type(pkts, nb_pkts);
    
    // 然后对每组使用专用解析器
    for (int i = 0; i < NB_PKT_TYPES; i++) {
        if (pkt_group_count[i] > 0) {
            specialized_parser[i](pkt_groups[i], pkt_group_count[i], 
                                 meta + group_offset[i]);
        }
    }
}
```

**预解析与直接访问**：

```
// 预计算关键协议字段偏移，避免重复解析
struct packet_meta {
    uint32_t l3_offset;  // IP头部偏移
    uint32_t l4_offset;  // TCP/UDP头部偏移
    uint16_t eth_type;   // 以太网类型
    uint8_t ip_proto;    // IP协议类型
    // 其他元数据...
};

// 优化的IP头获取
static inline struct rte_ipv4_hdr *get_ipv4_header(struct rte_mbuf *pkt, 
                                                 struct packet_meta *meta)
{
    // 使用预计算偏移直接访问
    if (meta->l3_offset > 0 && meta->eth_type == RTE_ETHER_TYPE_IPV4) {
        return rte_pktmbuf_mtod_offset(pkt, struct rte_ipv4_hdr *, 
                                      meta->l3_offset);
    }
    return NULL;
}
```

**协议状态机优化**：

```
// 为常见协议使用查表法实现状态转换
#define MAX_HTTP_STATES 32
typedef int (*http_state_handler)(const uint8_t *data, uint32_t len, 
                                struct http_session *sess);
                                
static http_state_handler http_state_table[MAX_HTTP_STATES][MAX_HTTP_EVENTS];

// 初始化状态转换表
static void init_http_state_machine(void)
{
    http_state_table[HTTP_START][HTTP_EVENT_REQUEST] = handle_http_request;
    http_state_table[HTTP_START][HTTP_EVENT_RESPONSE] = handle_http_response;
    // 更多状态转换...
}
```



**SIMD加速解析**：

```
// 使用AVX2加速多数据包头部字段提取
#ifdef __AVX2__
static void extract_ipv4_5tuples_avx2(struct rte_mbuf **pkts, 
                                    uint16_t nb_pkts,
                                    struct ipv4_5tuple *tuples)
{
    // 使用SIMD指令同时处理多个包
    for (int i = 0; i < nb_pkts - 7; i += 8) {
        __m256i ip_src = _mm256_set_epi32(/* 8个包的源IP */);
        // 更多SIMD处理...
    }
    
    // 处理剩余包
    for (int i = (nb_pkts & ~7); i < nb_pkts; i++) {
        extract_ipv4_5tuple_scalar(pkts[i], &tuples[i]);
    }
}
#endif
```

### 3.2.2 五元组提取与流跟踪

五元组是网络流量分类的基础，也是NIDS系统流量关联的核心：

**五元组提取与规范化**：

```
// 五元组结构定义
struct flow_5tuple {
    uint32_t src_ip;
    uint32_t dst_ip;
    uint16_t src_port;
    uint16_t dst_port;
    uint8_t protocol;
} __attribute__((__packed__));

// 规范化五元组(考虑方向)
static void normalize_5tuple(struct flow_5tuple *tuple)
{
    // 确保方向一致性(较小IP地址作为源)
    if ((tuple->src_ip > tuple->dst_ip) ||
        (tuple->src_ip == tuple->dst_ip && tuple->src_port > tuple->dst_port)) {
        // 交换源目信息
        uint32_t tmp_ip = tuple->src_ip;
        tuple->src_ip = tuple->dst_ip;
        tuple->dst_ip = tmp_ip;
        
        uint16_t tmp_port = tuple->src_port;
        tuple->src_port = tuple->dst_port;
        tuple->dst_port = tmp_port;
    }
}

// 从数据包提取五元组
static int extract_5tuple(struct rte_mbuf *pkt, struct flow_5tuple *tuple)
{
    struct rte_ether_hdr *eth_hdr = rte_pktmbuf_mtod(pkt, struct rte_ether_hdr *);
    uint16_t ether_type = rte_be_to_cpu_16(eth_hdr->ether_type);
    
    if (ether_type == RTE_ETHER_TYPE_IPV4) {
        struct rte_ipv4_hdr *ip_hdr = (struct rte_ipv4_hdr *)(eth_hdr + 1);
        
        tuple->src_ip = rte_be_to_cpu_32(ip_hdr->src_addr);
        tuple->dst_ip = rte_be_to_cpu_32(ip_hdr->dst_addr);
        tuple->protocol = ip_hdr->next_proto_id;
        
        if (tuple->protocol == IPPROTO_TCP) {
            struct rte_tcp_hdr *tcp_hdr = 
                (struct rte_tcp_hdr *)((uint8_t *)ip_hdr + (ip_hdr->version_ihl & 0xf) * 4);
            tuple->src_port = rte_be_to_cpu_16(tcp_hdr->src_port);
            tuple->dst_port = rte_be_to_cpu_16(tcp_hdr->dst_port);
        } else if (tuple->protocol == IPPROTO_UDP) {
            struct rte_udp_hdr *udp_hdr = 
                (struct rte_udp_hdr *)((uint8_t *)ip_hdr + (ip_hdr->version_ihl & 0xf) * 4);
            tuple->src_port = rte_be_to_cpu_16(udp_hdr->src_port);
            tuple->dst_port = rte_be_to_cpu_16(udp_hdr->dst_port);
        } else {
            // 非TCP/UDP协议
            tuple->src_port = 0;
            tuple->dst_port = 0;
        }
        
        return 0;
    }
    
    // 不支持的协议
    return -1;
}
```

**流会话管理**：

```
// 流会话结构
struct flow_record {
    struct flow_5tuple key;       // 五元组键
    uint64_t first_seen;          // 首次看到时间
    uint64_t last_seen;           // 最近看到时间
    uint64_t pkt_count[2];        // 双向包计数
    uint64_t byte_count[2];       // 双向字节计数
    uint8_t tcp_flags_seen[2];    // 看到的TCP标志
    uint16_t flow_state;          // 流状态
    void *app_data;               // 应用层数据指针
};

// 使用rte_hash实现流表
struct flow_table {
    struct rte_hash *hash;        // DPDK哈希表
    uint32_t max_flows;           // 最大流数量
    struct flow_record *flows;    // 流记录数组
    rte_spinlock_t locks[FLOW_LOCKS_COUNT]; // 细粒度锁数组
};

// 流表初始化
struct flow_table *flow_table_create(uint32_t max_flows, int socket_id)
{
    struct flow_table *table = rte_zmalloc_socket("flow_table",
            sizeof(struct flow_table), RTE_CACHE_LINE_SIZE, socket_id);
    
    // 配置哈希表参数
    struct rte_hash_parameters hash_params = {
        .name = "flow_hash_table",
        .entries = max_flows,
        .key_len = sizeof(struct flow_5tuple),
        .hash_func = rte_jhash,
        .hash_func_init_val = 0,
        .socket_id = socket_id
    };
    
    table->hash = rte_hash_create(&hash_params);
    table->max_flows = max_flows;
    table->flows = rte_zmalloc_socket("flow_records", 
                    sizeof(struct flow_record) * max_flows, 
                    RTE_CACHE_LINE_SIZE, socket_id);
                    
    // 初始化锁
    for (int i = 0; i < FLOW_LOCKS_COUNT; i++)
        rte_spinlock_init(&table->locks[i]);
        
    return table;
}

// 获取或创建流记录
struct flow_record *get_or_create_flow(struct flow_table *table, 
                                     struct flow_5tuple *key, int direction)
{
    int ret;
    int32_t hash_idx;
    struct flow_record *flow = NULL;
    
    // 规范化五元组
    normalize_5tuple(key);
    
    // 计算锁索引
    uint32_t lock_idx = rte_jhash(key, sizeof(*key), 0) % FLOW_LOCKS_COUNT;
    
    // 加锁查找/添加
    rte_spinlock_lock(&table->locks[lock_idx]);
    
    // 查找现有流
    hash_idx = rte_hash_lookup(table->hash, key);
    
    if (hash_idx < 0) {
        // 创建新流
        hash_idx = rte_hash_add_key(table->hash, key);
        if (hash_idx >= 0) {
            flow = &table->flows[hash_idx];
            memset(flow, 0, sizeof(*flow));
            flow->key = *key;
            flow->first_seen = flow->last_seen = rte_rdtsc();
            flow->flow_state = FLOW_STATE_NEW;
        }
    } else {
        // 更新现有流
        flow = &table->flows[hash_idx];
        flow->last_seen = rte_rdtsc();
    }
    
    // 更新统计
    if (flow) {
        flow->pkt_count[direction]++;
        // 其他统计更新...
    }
    
    rte_spinlock_unlock(&table->locks[lock_idx]);
    
    return flow;
}
```

**流超时管理**：

```
// 定义超时值
#define TCP_FIN_TIMEOUT_SEC  120
#define TCP_ACTIVE_TIMEOUT_SEC 3600
#define UDP_TIMEOUT_SEC 300

// 清理过期流
static void expire_flows(struct flow_table *table)
{
    uint64_t now = rte_rdtsc();
    uint64_t hz = rte_get_timer_hz();
    
    for (uint32_t i = 0; i < table->max_flows; i++) {
        struct flow_record *flow = &table->flows[i];
        
        // 检查是否有效流
        if (flow->first_seen == 0)
            continue;
            
        uint32_t lock_idx = rte_jhash(&flow->key, sizeof(flow->key), 0) % FLOW_LOCKS_COUNT;
        
        bool should_expire = false;
        uint64_t idle_time = (now - flow->last_seen) / hz;
        
        // 根据协议和状态确定超时策略
        if (flow->key.protocol == IPPROTO_TCP) {
            if ((flow->tcp_flags_seen[0] | flow->tcp_flags_seen[1]) & RTE_TCP_FIN_FLAG) {
                // FIN已见到，使用更短的超时
                should_expire = (idle_time > TCP_FIN_TIMEOUT_SEC);
            } else {
                // 活跃TCP连接
                should_expire = (idle_time > TCP_ACTIVE_TIMEOUT_SEC);
            }
        } else if (flow->key.protocol == IPPROTO_UDP) {
            should_expire = (idle_time > UDP_TIMEOUT_SEC);
        } else {
            // 其他协议使用较短超时
            should_expire = (idle_time > DEFAULT_TIMEOUT_SEC);
        }
        
        if (should_expire) {
            rte_spinlock_lock(&table->locks[lock_idx]);
            // 处理流过期逻辑
            if (flow->app_data) {
                free_app_data(flow->app_data);
                flow->app_data = NULL;
            }
            // 从哈希表移除
            rte_hash_del_key(table->hash, &flow->key);
            // 清除流记录
            memset(flow, 0, sizeof(*flow));
            rte_spinlock_unlock(&table->locks[lock_idx]);
            
            expired_flow_counter++;
        }
    }
}
```



### 3.2.3 数据包重组与规范化

处理分片IP和TCP段是NIDS系统的关键能力，否则攻击者可以通过分段规避检测：

**IP分片重组**：

```
// IP分片处理上下文
struct ip_frag_ctx {
    rte_spinlock_t lock;
    struct rte_hash *frag_table;  // 分片表
    struct rte_mempool *frag_mbuf_pool; // 分片mbuf池
    struct rte_ip_frag_tbl *frag_tbl;  // DPDK分片表
    struct rte_ip_frag_death_row death_row; // 过期分片
};

// 初始化IP分片处理
struct ip_frag_ctx *init_ip_frag_ctx(int socket_id)
{
    struct ip_frag_ctx *ctx = rte_zmalloc_socket("ip_frag_ctx",
            sizeof(*ctx), RTE_CACHE_LINE_SIZE, socket_id);
    
    // 初始化锁
    rte_spinlock_init(&ctx->lock);
    
    // 创建分片表
    ctx->frag_tbl = rte_ip_frag_table_create(
            FRAG_TBL_BUCKET_ENTRIES,  // 桶条目数
            FRAG_TBL_TOTAL_ENTRIES,   // 总条目数
            FRAG_TBL_TTL,             // 生存时间(秒)
            socket_id);
            
    // 初始化过期行
    memset(&ctx->death_row, 0, sizeof(ctx->death_row));
    
    return ctx;
}

// 处理IP分片
struct rte_mbuf *handle_ip_frags(struct ip_frag_ctx *ctx, 
                                struct rte_mbuf *pkt,
                                uint64_t tms)
{
    struct rte_ipv4_hdr *ip_hdr = rte_pktmbuf_mtod_offset(pkt, 
            struct rte_ipv4_hdr *, sizeof(struct rte_ether_hdr));
    
    // 处理分片
    rte_spinlock_lock(&ctx->lock);
    struct rte_mbuf *reassembled = rte_ipv4_frag_reassemble_packet(
            ctx->frag_tbl, &ctx->death_row, pkt, tms, ip_hdr);
    
    // 回收过期分片
    rte_ip_frag_free_death_row(&ctx->death_row, PREFETCH_OFFSET);
    rte_spinlock_unlock(&ctx->lock);
    
    return reassembled;
}
```

**TCP会话重组**：

```
// TCP会话结构
struct tcp_session {
    struct flow_5tuple flow_id;
    uint32_t cli_next_seq;  // 客户端下一序列号
    uint32_t srv_next_seq;  // 服务器下一序列号
    struct rte_reorder_buffer *cli_buffer;  // 客户端重排序缓冲区
    struct rte_reorder_buffer *srv_buffer;  // 服务器重排序缓冲区
    uint8_t *cli_stream;    // 客户端流数据
    uint8_t *srv_stream;    // 服务器流数据
    uint32_t cli_stream_size;
    uint32_t srv_stream_size;
    uint32_t cli_bytes_stored;
    uint32_t srv_bytes_stored;
    uint8_t state;          // TCP状态
};

// 初始化TCP会话
struct tcp_session *tcp_session_create(struct flow_5tuple *flow_id, 
                                     struct rte_tcp_hdr *tcp, int direction)
{
    struct tcp_session *sess = rte_zmalloc("tcp_session", sizeof(*sess), 0);
    
    sess->flow_id = *flow_id;
    
    // 分配重排序缓冲区
    sess->cli_buffer = rte_reorder_create("cli_reorder", 
            rte_socket_id(), TCP_REORDER_BUFFER_SIZE);
    sess->srv_buffer = rte_reorder_create("srv_reorder", 
            rte_socket_id(), TCP_REORDER_BUFFER_SIZE);
    
    // 分配流缓冲区
    sess->cli_stream_size = TCP_STREAM_BUFFER_SIZE;
    sess->srv_stream_size = TCP_STREAM_BUFFER_SIZE;
    sess->cli_stream = rte_zmalloc("cli_stream", sess->cli_stream_size, 0);
    sess->srv_stream = rte_zmalloc("srv_stream", sess->srv_stream_size, 0);
    
    // 设置初始序列号
    if (direction == 0) {
        sess->cli_next_seq = rte_be_to_cpu_32(tcp->sent_seq) + 1; // SYN占一个序列号
    } else {
        sess->srv_next_seq = rte_be_to_cpu_32(tcp->sent_seq) + 1;
    }
    
    sess->state = TCP_STATE_SYN;
    
    return sess;
}

// 处理TCP段并重组流
int tcp_process_segment(struct tcp_session *sess, struct rte_mbuf *pkt, 
                       struct rte_tcp_hdr *tcp, int direction, 
                       uint8_t *payload, uint32_t payload_len)
{
    uint32_t seq = rte_be_to_cpu_32(tcp->sent_seq);
    uint32_t *next_seq = (direction == 0) ? &sess->cli_next_seq : &sess->srv_next_seq;
    uint8_t *stream = (direction == 0) ? sess->cli_stream : sess->srv_stream;
    uint32_t *stored = (direction == 0) ? &sess->cli_bytes_stored : &sess->srv_bytes_stored;
    uint32_t max_size = (direction == 0) ? sess->cli_stream_size : sess->srv_stream_size;
    
    // 序列号检查
    if (payload_len > 0) {
        if (seq == *next_seq) {
            // 按序数据
            if (*stored + payload_len <= max_size) {
                memcpy(stream + *stored, payload, payload_len);
                *stored += payload_len;
                *next_seq += payload_len;
                
                // 处理可能存在的已排序数据
                process_reordered_segments(sess, direction);
                
                return 0;
            }
        } else if (seq > *next_seq) {
            // 乱序数据，放入重排序缓冲区
            struct segment_data *seg = rte_malloc("segment", 
                    sizeof(*seg) + payload_len, 0);
            seg->seq = seq;
            seg->len = payload_len;
            memcpy(seg->data, payload, payload_len);
            
            struct rte_reorder_buffer *buffer = 
                (direction == 0) ? sess->cli_buffer : sess->srv_buffer;
            
            if (rte_reorder_insert(buffer, (struct rte_mbuf *)seg) == 0) {
                // 插入成功
                return 1;
            } else {
                // 缓冲区满或其他错误
                rte_free(seg);
                return -1;
            }
        }
        // seq < next_seq 是重传或部分重叠的数据
    }
    
    // 更新TCP状态
    update_tcp_state(sess, tcp, direction);
    
    return 0;
}
```

**规范化策略实现**：

```
// 数据包规范化处理
int normalize_packet(struct rte_mbuf *pkt)
{
    struct rte_ether_hdr *eth_hdr = rte_pktmbuf_mtod(pkt, struct rte_ether_hdr *);
    
    // 仅处理IPv4
    if (rte_be_to_cpu_16(eth_hdr->ether_type) != RTE_ETHER_TYPE_IPV4)
        return 0;
        
    struct rte_ipv4_hdr *ip_hdr = (struct rte_ipv4_hdr *)(eth_hdr + 1);
    uint8_t ip_hdr_len = (ip_hdr->version_ihl & 0xf) * 4;
    
    // IP规范化(TTL标准化)
    if (ip_hdr->time_to_live < MIN_TTL) {
        ip_hdr->time_to_live = MIN_TTL;
        // 重计算IP校验和
        ip_hdr->hdr_checksum = 0;
        ip_hdr->hdr_checksum = rte_ipv4_cksum(ip_hdr);
    }
    
    // TCP规范化
    if (ip_hdr->next_proto_id == IPPROTO_TCP) {
        struct rte_tcp_hdr *tcp_hdr = 
            (struct rte_tcp_hdr *)((uint8_t *)ip_hdr + ip_hdr_len);
        
        uint8_t tcp_hdr_len = ((tcp_hdr->data_off & 0xf0) >> 4) * 4;
        
        // 移除危险的TCP选项
        if (tcp_hdr_len > 20) {
            normalize_tcp_options(tcp_hdr);
            
            // 重计算TCP校验和
            tcp_hdr->cksum = 0;
            tcp_hdr->cksum = rte_ipv4_udptcp_cksum(ip_hdr, tcp_hdr);
        }
    }
    
    return 0;
}
```

## 3.3 流量分发与负载均衡

### 3.3.1 流表设计与哈希算法

高效的流表设计对于NIDS的性能至关重要，尤其是在高流量环境下：

**流表数据结构设计**：

```
// 哈希函数选择
#define DEFAULT_HASH_FUNC rte_jhash
#define HASH_INIT_VAL 0xdeadbeef

// 优化的流表桶设计
struct flow_bucket {
    struct flow_entry entries[FLOW_BUCKET_SIZE];
    rte_atomic32_t used;  // 使用计数
    rte_spinlock_t lock;  // 细粒度锁
} __rte_cache_aligned;

// 流表结构
struct flow_table {
    uint32_t size;  // 桶数量
    uint32_t mask;  // 哈希掩码(size-1)
    struct flow_bucket *buckets;  // 桶数组
    rte_atomic32_t flow_count;    // 流计数
};

// 创建流表
struct flow_table *flow_table_create(uint32_t size, int socket_id)
{
    // 大小调整为2的幂
    uint32_t adjusted_size = rte_align32pow2(size);
    
    struct flow_table *table = rte_zmalloc_socket("flow_table",
                sizeof(*table), RTE_CACHE_LINE_SIZE, socket_id);
    
    table->size = adjusted_size;
    table->mask = adjusted_size - 1;
    table->buckets = rte_zmalloc_socket("flow_buckets",
                adjusted_size * sizeof(struct flow_bucket),
                RTE_CACHE_LINE_SIZE, socket_id);
    
    // 初始化每个桶的锁
    for (uint32_t i = 0; i < adjusted_size; i++) {
        rte_spinlock_init(&table->buckets[i].lock);
    }
    
    return table;
}
```

**缓存优化的哈希算法实现**：

```
// 优化的五元组哈希函数
static inline uint32_t flow_hash(const struct flow_5tuple *flow)
{
    // 折叠哈希算法，减少64位计算
    const uint8_t *key = (const uint8_t *)flow;
    uint32_t hash = HASH_INIT_VAL;
    
    #ifdef __SSE4_2__
    // 使用SSE4.2的CRC32指令加速哈希计算
    hash = _mm_crc32_u32(hash, *(const uint32_t *)&key[0]);  // src_ip
    hash = _mm_crc32_u32(hash, *(const uint32_t *)&key[4]);  // dst_ip
    hash = _mm_crc32_u16(hash, *(const uint16_t *)&key[8]);  // src_port
    hash = _mm_crc32_u16(hash, *(const uint16_t *)&key[10]); // dst_port
    hash = _mm_crc32_u8(hash, key[12]);                      // protocol
    #else
    // 回退到标准哈希函数
    hash = DEFAULT_HASH_FUNC(key, sizeof(struct flow_5tuple), HASH_INIT_VAL);
    #endif
    
    return hash;
}

// 流表查找/插入
struct flow_entry *flow_table_find_or_add(struct flow_table *table, 
                                        const struct flow_5tuple *flow,
                                        int create)
{
    uint32_t hash = flow_hash(flow);
    uint32_t idx = hash & table->mask;
    struct flow_bucket *bucket = &table->buckets[idx];
    
    // 加锁前先检查(乐观)
    for (int i = 0; i < FLOW_BUCKET_SIZE; i++) {
        if (memcmp(&bucket->entries[i].key, flow, sizeof(*flow)) == 0) {
            // 可能找到，加锁确认
            rte_spinlock_lock(&bucket->lock);
            if (bucket->entries[i].active) {
                struct flow_entry *entry = &bucket->entries[i];
                entry->last_seen = rte_rdtsc();
                rte_spinlock_unlock(&bucket->lock);
                return entry;
            }
            rte_spinlock_unlock(&bucket->lock);
            break;
        }
    }
    
    if (!create)
        return NULL;
    
    // 加锁创建
    rte_spinlock_lock(&bucket->lock);
    
    // 再次检查(避免竞争条件)
    for (int i = 0; i < FLOW_BUCKET_SIZE; i++) {
        if (memcmp(&bucket->entries[i].key, flow, sizeof(*flow)) == 0 && 
            bucket->entries[i].active) {
            struct flow_entry *entry = &bucket->entries[i];
            entry->last_seen = rte_rdtsc();
            rte_spinlock_unlock(&bucket->lock);
            return entry;
        }
    }
    
    // 查找空槽位
    for (int i = 0; i < FLOW_BUCKET_SIZE; i++) {
        if (!bucket->entries[i].active) {
            struct flow_entry *entry = &bucket->entries[i];
            
            // 初始化新流条目
            memcpy(&entry->key, flow, sizeof(*flow));
            entry->active = 1;
            entry->first_seen = entry->last_seen = rte_rdtsc();
            rte_atomic32_inc(&bucket->used);
            rte_atomic32_inc(&table->flow_count);
            
            rte_spinlock_unlock(&bucket->lock);
            return entry;
        }
    }
    
    // 桶已满，使用环形替换或其他策略
    int victim = hash % FLOW_BUCKET_SIZE;  // 简单替换策略
    struct flow_entry *entry = &bucket->entries[victim];
    
    // 释放现有流资源
    if (entry->app_data && entry->cleanup)
        entry->cleanup(entry->app_data);
        
    // 初始化新流条目
    memcpy(&entry->key, flow, sizeof(*flow));
    entry->app_data = NULL;
    entry->cleanup = NULL;
    entry->first_seen = entry->last_seen = rte_rdtsc();
    
    rte_spinlock_unlock(&bucket->lock);
    return entry;
}
```

**高级流表优化技术**：

```
// 使用预取提高缓存效率
static inline struct flow_entry *flow_table_lookup_with_prefetch(
        struct flow_table *table, const struct flow_5tuple *flow)
{
    uint32_t hash = flow_hash(flow);
    uint32_t idx = hash & table->mask;
    
    // 预取桶数据到缓存
    rte_prefetch0(&table->buckets[idx]);
    
    // 等待预取完成后再访问
    struct flow_bucket *bucket = &table->buckets[idx];
    
    // 查找流程
    for (int i = 0; i < FLOW_BUCKET_SIZE; i++) {
        // 预取下一个条目
        if (i < FLOW_BUCKET_SIZE - 1)
            rte_prefetch0(&bucket->entries[i+1]);
            
        if (memcmp(&bucket->entries[i].key, flow, sizeof(*flow)) == 0 && 
            bucket->entries[i].active) {
            return &bucket->entries[i];
        }
    }
    
    return NULL;
}

// 批量流表查找
int flow_table_lookup_bulk(struct flow_table *table, 
                          const struct flow_5tuple **flows,
                          struct flow_entry **entries,
                          uint16_t n)
{
    uint16_t matches = 0;
    
    // 批量预取第一轮
    for (int i = 0; i < RTE_MIN(n, PREFETCH_OFFSET); i++) {
        uint32_t hash = flow_hash(flows[i]);
        uint32_t idx = hash & table->mask;
        rte_prefetch0(&table->buckets[idx]);
    }
    
    // 主处理循环
    for (uint16_t i = 0; i < n; i++) {
        // 预取后续的桶
        if (i + PREFETCH_OFFSET < n) {
            uint32_t hash = flow_hash(flows[i + PREFETCH_OFFSET]);
            uint32_t idx = hash & table->mask;
            rte_prefetch0(&table->buckets[idx]);
        }
        
        // 查找当前流
        uint32_t hash = flow_hash(flows[i]);
        uint32_t idx = hash & table->mask;
        struct flow_bucket *bucket = &table->buckets[idx];
        
        for (int j = 0; j < FLOW_BUCKET_SIZE; j++) {
            if (memcmp(&bucket->entries[j].key, flows[i], sizeof(struct flow_5tuple)) == 0 &&
                bucket->entries[j].active) {
                entries[matches++] = &bucket->entries[j];
                break;
            }
        }
    }
    
    return matches;
}
```



### 3.3.2 多核分发策略

在多核NIDS系统中，有效的流量分发策略对于性能至关重要：

**基于RSS的硬件分发**：

```
// 设置RSS高级配置
static int setup_rss_distribution(uint16_t port_id, uint16_t nb_queues)
{
    // 自定义RSS键，优化分散性
    static uint8_t rss_key[40] = {
        0x6d, 0x5a, 0x56, 0xda, 0x25, 0x5b, 0x0e, 0xc2,
        0x41, 0x67, 0x25, 0x3d, 0x43, 0xa3, 0x8f, 0xb0,
        0xd0, 0xca, 0x2b, 0xcb, 0xae, 0x7b, 0x30, 0xb4,
        0x77, 0xcb, 0x2d, 0xa3, 0x80, 0x30, 0xf2, 0x0c,
        0x6a, 0x42, 0xb7, 0x3b, 0xbe, 0xac, 0x01, 0xfa
    };
    
    // 设置RSS哈希配置
    struct rte_eth_rss_conf rss_conf = {
        .rss_key = rss_key,
        .rss_key_len = sizeof(rss_key),
        .rss_hf = ETH_RSS_IP | ETH_RSS_TCP | ETH_RSS_UDP
    };
    
    // 设置RSS RETA (Redirection Table)
    struct rte_eth_rss_reta_entry64 reta_conf[RSS_RETA_SIZE_MAX/64];
    memset(reta_conf, 0, sizeof(reta_conf));
    
    // 配置每个RETA条目
    for (uint16_t i = 0; i < RSS_RETA_SIZE_MAX; i++) {
        uint16_t reta_id = i / 64;
        uint16_t reta_pos = i % 64;
        
        // 设置队列映射的掩码位
        reta_conf[reta_id].mask |= (1ULL << reta_pos);
        
        // 设置每个条目的队列映射，使用智能分配
        uint16_t queue = i % nb_queues;
        
        // 特殊处理：相同流分配到相同核心
        reta_conf[reta_id].reta[reta_pos] = queue;
    }
    
    // 应用RSS RETA配置
    return rte_eth_dev_rss_reta_update(port_id, reta_conf, RSS_RETA_SIZE_MAX);
}
```

**软件流量分发器实现**：

```
// 软件分发上下文
struct sw_distributor {
    struct rte_ring **worker_rings;  // 每个工作线程的接收环
    uint16_t nb_workers;             // 工作线程数量
    rte_atomic64_t *worker_stats;    // 每个工作线程的包统计
    struct rte_hash *flow_to_worker; // 流到工作线程的映射表
};

// 初始化软件分发器
struct sw_distributor *sw_distributor_create(uint16_t nb_workers, 
                                           uint32_t max_flows, 
                                           uint32_t ring_size)
{
    struct sw_distributor *dist = rte_zmalloc("sw_distributor", 
            sizeof(*dist), RTE_CACHE_LINE_SIZE);
    
    dist->nb_workers = nb_workers;
    
    // 创建工作线程接收环
    dist->worker_rings = rte_zmalloc("worker_rings", 
            sizeof(struct rte_ring *) * nb_workers, RTE_CACHE_LINE_SIZE);
    
    for (uint16_t i = 0; i < nb_workers; i++) {
        char ring_name[32];
        snprintf(ring_name, sizeof(ring_name), "worker_ring_%u", i);
        
        dist->worker_rings[i] = rte_ring_create(ring_name, ring_size, 
                rte_socket_id(), RING_F_SC_DEQ); // 单消费者模式
    }
    
    // 创建统计计数器
    dist->worker_stats = rte_zmalloc("worker_stats", 
            sizeof(rte_atomic64_t) * nb_workers, RTE_CACHE_LINE_SIZE);
    
    // 创建流到工作线程的映射表
    struct rte_hash_parameters hash_params = {
        .name = "flow_worker_map",
        .entries = max_flows,
        .key_len = sizeof(struct flow_5tuple),
        .hash_func = rte_jhash,
        .hash_func_init_val = 0
    };
    
    dist->flow_to_worker = rte_hash_create(&hash_params);
    
    return dist;
}

// 分发数据包
int sw_distribute_packets(struct sw_distributor *dist, 
                        struct rte_mbuf **pkts, 
                        uint16_t nb_pkts)
{
    // 预分类数组，每个工作线程一个数组
    struct rte_mbuf *worker_pkts[dist->nb_workers][MAX_PKT_BURST];
    uint16_t worker_counts[dist->nb_workers];
    memset(worker_counts, 0, sizeof(worker_counts));
    
    // 按五元组分类数据包
    for (uint16_t i = 0; i < nb_pkts; i++) {
        struct flow_5tuple flow;
        int worker_id;
        
        // 提取五元组
        if (extract_5tuple(pkts[i], &flow) < 0) {
            // 不支持的协议，轮询分配
            worker_id = i % dist->nb_workers;
        } else {
            // 查找流到工作线程的映射
            int ret = rte_hash_lookup(dist->flow_to_worker, &flow);
            
            if (ret < 0) {
                // 新流，分配工作线程
                // 策略：选择负载最小的工作线程
                worker_id = select_least_loaded_worker(dist);
                
                // 更新映射
                rte_hash_add_key_data(dist->flow_to_worker, &flow, 
                        (void *)(uintptr_t)worker_id);
            } else {
                worker_id = (uintptr_t)ret;
            }
        }
        
        // 添加到对应工作线程的包数组
        worker_pkts[worker_id][worker_counts[worker_id]++] = pkts[i];
    }
    
    // 分发到各工作线程
    for (uint16_t i = 0; i < dist->nb_workers; i++) {
        if (worker_counts[i] > 0) {
            uint16_t enqueued = rte_ring_enqueue_burst(dist->worker_rings[i],
                    (void **)worker_pkts[i], worker_counts[i], NULL);
                    
            // 更新统计
            rte_atomic64_add(&dist->worker_stats[i], enqueued);
            
            // 处理未能入队的数据包
            if (unlikely(enqueued < worker_counts[i])) {
                for (uint16_t j = enqueued; j < worker_counts[i]; j++) {
                    rte_pktmbuf_free(worker_pkts[i][j]);
                }
            }
        }
    }
    
    return 0;
}
```

**工作线程设计**：

```
// 工作线程上下文
struct worker_context {
    uint16_t worker_id;
    struct rte_ring *rx_ring;
    struct ids_context *ids_ctx;
};

// 工作线程主循环
int worker_thread_main(void *arg)
{
    struct worker_context *ctx = (struct worker_context *)arg;
    struct rte_mbuf *pkts[MAX_PKT_BURST];
    
    printf("Worker %u started on core %u\n", ctx->worker_id, rte_lcore_id());
    
    // 设置CPU亲和性
    set_core_affinity(rte_lcore_id());
    
    while (!force_quit) {
        // 从接收环批量取包
        uint16_t nb_rx = rte_ring_dequeue_burst(ctx->rx_ring,
                (void **)pkts, MAX_PKT_BURST, NULL);
                
        if (nb_rx == 0) {
            // 无包可处理，短暂睡眠避免空转
            rte_pause();
            continue;
        }
        
        // 处理数据包批次
        process_packet_batch(ctx->ids_ctx, pkts, nb_rx);
        
        // 释放处理完的mbuf
        for (uint16_t i = 0; i < nb_rx; i++) {
            rte_pktmbuf_free(pkts[i]);
        }
    }
    
    return 0;
}
```

**工作队列优化**：

```
// 优先级工作队列
struct priority_worker_queue {
    struct rte_ring *high_priority_ring;  // 高优先级数据包
    struct rte_ring *normal_priority_ring; // 普通优先级数据包
    struct rte_ring *low_priority_ring;    // 低优先级数据包
    uint32_t high_ratio;  // 高优先级处理比例
    uint32_t normal_ratio; // 普通优先级处理比例
    uint32_t low_ratio;    // 低优先级处理比例
};

// 创建优先级队列
struct priority_worker_queue *create_priority_queue(uint32_t ring_size, 
                                                  int socket_id)
{
    struct priority_worker_queue *queue = rte_zmalloc_socket("prio_queue",
            sizeof(*queue), RTE_CACHE_LINE_SIZE, socket_id);
            
    // 创建三个不同优先级的环形缓冲区
    queue->high_priority_ring = rte_ring_create("high_prio_ring", 
            ring_size, socket_id, RING_F_SP_ENQ | RING_F_SC_DEQ);
            
    queue->normal_priority_ring = rte_ring_create("normal_prio_ring", 
            ring_size, socket_id, RING_F_SP_ENQ | RING_F_SC_DEQ);
            
    queue->low_priority_ring = rte_ring_create("low_prio_ring", 
            ring_size, socket_id, RING_F_SP_ENQ | RING_F_SC_DEQ);
            
    // 设置默认处理比例 (6:3:1)
    queue->high_ratio = 6;
    queue->normal_ratio = 3;
    queue->low_ratio = 1;
    
    return queue;
}

// 基于数据包特征分配优先级
static int get_packet_priority(struct rte_mbuf *pkt)
{
    struct flow_5tuple tuple;
    
    if (extract_5tuple(pkt, &tuple) < 0) {
        return PRIO_LOW;  // 未知协议，低优先级
    }
    
    // 检查是否为新连接
    if (tuple.protocol == IPPROTO_TCP) {
        struct rte_tcp_hdr *tcp = get_tcp_header(pkt);
        if (tcp && (tcp->tcp_flags & RTE_TCP_SYN_FLAG)) {
            return PRIO_HIGH;  // 新连接尝试，高优先级
        }
    }
    
    // 检查重要服务端口
    if (is_critical_service_port(tuple.dst_port)) {
        return PRIO_HIGH;
    }
    
    // 其他普通流量
    return PRIO_NORMAL;
}

// 工作线程带优先级的处理循环
int priority_worker_thread_main(void *arg)
{
    struct worker_context *ctx = (struct worker_context *)arg;
    struct priority_worker_queue *queue = ctx->priority_queue;
    struct rte_mbuf *pkts[MAX_PKT_BURST];
    uint16_t nb_rx;
    
    uint32_t cycle_counter = 0;
    
    while (!force_quit) {
        // 根据优先级比例选择队列
        struct rte_ring *ring_to_dequeue;
        
        uint32_t modulo = cycle_counter % (queue->high_ratio + 
                                         queue->normal_ratio + 
                                         queue->low_ratio);
                                         
        if (modulo < queue->high_ratio) {
            ring_to_dequeue = queue->high_priority_ring;
        } else if (modulo < queue->high_ratio + queue->normal_ratio) {
            ring_to_dequeue = queue->normal_priority_ring;
        } else {
            ring_to_dequeue = queue->low_priority_ring;
        }
        
        // 从选定队列取包
        nb_rx = rte_ring_dequeue_burst(ring_to_dequeue,
                (void **)pkts, MAX_PKT_BURST, NULL);
                
        if (nb_rx == 0) {
            // 如果选定队列为空，尝试其他队列
            if (ring_to_dequeue == queue->high_priority_ring) {
                ring_to_dequeue = queue->normal_priority_ring;
            } else if (ring_to_dequeue == queue->normal_priority_ring) {
                ring_to_dequeue = queue->low_priority_ring;
            } else {
                ring_to_dequeue = queue->high_priority_ring;
            }
            
            nb_rx = rte_ring_dequeue_burst(ring_to_dequeue,
                    (void **)pkts, MAX_PKT_BURST, NULL);
        }
        
        if (nb_rx > 0) {
            // 处理获取的包
            process_packet_batch(ctx->ids_ctx, pkts, nb_rx);
            
            // 释放mbuf
            for (uint16_t i = 0; i < nb_rx; i++) {
                rte_pktmbuf_free(pkts[i]);
            }
        } else {
            // 无包可处理
            rte_pause();
        }
        
        cycle_counter++;
    }
    
    return 0;
}
```



# 4. 检测引擎设计

## 4.1 规则管理子系统

### 4.1.1 规则格式定义

DPDK-NIDS支持业界标准的规则格式，主要兼容Suricata/Snort规则语法，同时引入DPDK特定的优化扩展。

**规则基本结构**：

```
action protocol src_ip src_port direction dst_ip dst_port (options)
```

**示例规则**：

```
alert tcp $EXTERNAL_NET any -> $HOME_NET 22 (msg:"SSH连接尝试"; flow:to_server,established; content:"SSH-"; classtype:network-scan; sid:1000001; rev:1;)
```

**核心规则组件**：

1. **规则头部**：
   - 操作类型（alert、drop、pass等）
   - 协议类型（TCP、UDP、ICMP等）
   - 源/目的IP地址和端口
   - 流向指示符
2. **规则选项**：
   - 内容匹配（content、pcre）
   - 流控制（flow、flowbits）
   - 协议特定选项（http_*、tls_*等）
   - 元数据（msg、sid、rev等）
3. **DPDK优化扩展**：
   - `dpdk_queue: N;` - 指定规则优先在哪个接收队列处理
   - `dpdk_core: N;` - 指定规则优先在哪个核心处理
   - `dpdk_prio: high|medium|low;` - 指定规则优先级
   - `dpdk_batch: true|false;` - 指定是否适用于批处理模式

**规则内部表示**：

```
// 规则头部结构
struct rule_header {
    uint8_t action;           // 规则动作类型
    uint8_t protocol;         // 协议类型
    addr_t src_addr;          // 源地址（可能是IP列表或CIDR）
    addr_t dst_addr;          // 目的地址
    port_t src_port;          // 源端口（可能是范围）
    port_t dst_port;          // 目的端口
    uint8_t direction;        // 方向指示符
};

// 规则选项结构
struct rule_option {
    uint16_t type;            // 选项类型
    uint16_t length;          // 选项数据长度
    void *data;               // 选项数据指针
    struct rule_option *next; // 下一个选项
};

// 完整规则表示
struct rule {
    uint32_t id;              // 规则唯一ID (SID)
    struct rule_header header;
    struct rule_option *options;
    uint32_t flags;           // 各种规则标志
    uint8_t rev;              // 规则版本号
    uint8_t priority;         // 规则优先级
    char *msg;                // 规则消息
    void *compiled_pattern;   // 编译后的模式
    uint32_t ref_count;       // 引用计数
};
```



### 4.1.2 规则编译与优化

规则编译过程将文本规则转换为高效的内部表示，进行多级优化以最大化检测性能。

**规则编译流程**：

1. **解析阶段**：
   - 词法和语法分析，提取规则组件
   - 展开变量和规则宏
   - 验证规则语法和语义正确性
2. **优化阶段**：
   - **规则分组**：将具有相似特征的规则组合，减少重复计算
   - **规则排序**：基于规则优先级、内容长度和评估成本排序
   - **快速路径识别**：标识可以快速排除的规则
   - **内容预过滤**：提取固定字符串用于快速预过滤
3. **编译阶段**：
   - 将内容模式编译为优化的匹配结构
   - 将正则表达式编译为DFA或Hyperscan格式
   - 生成高效的评估条件代码

**DPDK特定优化**：

```
// 根据CPU拓扑和NUMA信息组织规则
void optimize_rules_for_topology(struct rule_set *ruleset) {
    int num_sockets = rte_socket_count();
    
    // 为每个NUMA节点创建规则副本
    for (int i = 0; i < num_sockets; i++) {
        ruleset->numa_local_rules[i] = rte_zmalloc_socket(
            "numa_rules", sizeof(struct rule_group),
            RTE_CACHE_LINE_SIZE, i);
        
        // 复制并重排规则，优化缓存访问模式
        reorganize_rules_for_cache(ruleset->numa_local_rules[i]);
    }
    
    // 根据NUMA与网卡亲和性分配规则
    for (uint16_t port_id = 0; port_id < nb_ports; port_id++) {
        int socket_id = rte_eth_dev_socket_id(port_id);
        if (socket_id >= 0) {
            assign_port_specific_rules(ruleset, port_id, socket_id);
        }
    }
}

// 批处理友好型规则优化
void optimize_rules_for_batch_processing(struct rule_set *ruleset) {
    // 识别适合SIMD处理的规则
    mark_simd_friendly_rules(ruleset);
    
    // 重组规则以优化缓存行使用
    align_rules_for_cache_efficiency(ruleset);
    
    // 创建批处理专用规则组
    create_batch_optimized_rule_groups(ruleset);
}
```



**规则评估成本计算**：

```
// 计算规则评估成本，用于优化规则排序
uint32_t calculate_rule_cost(struct rule *rule) {
    uint32_t cost = BASE_RULE_COST;
    struct rule_option *opt = rule->options;
    
    while (opt) {
        switch (opt->type) {
            case OPTION_CONTENT:
                // 内容匹配成本与模式长度相关
                cost += calculate_content_cost((content_option_t *)opt->data);
                break;
            case OPTION_PCRE:
                // 正则表达式成本通常较高
                cost += PCRE_BASE_COST;
                break;
            case OPTION_FLOWBITS:
                // 流状态检查成本
                cost += FLOWBIT_BASE_COST;
                break;
            // 其他选项类型...
        }
        opt = opt->next;
    }
    
    return cost;
}
```

### 4.1.3 规则集动态更新机制

NIDS系统需要支持在不中断检测流程的情况下更新规则。

**规则版本控制**：

```
struct rule_set_version {
    uint64_t version_id;      // 版本唯一标识符
    uint64_t timestamp;       // 创建时间戳
    uint32_t num_rules;       // 规则数量
    struct rule **rules;      // 规则数组
    atomic_t ref_count;       // 引用计数
    struct rule_index *index; // 规则索引结构
};

// 全局规则集管理
struct rule_manager {
    rte_rwlock_t lock;                // 读写锁
    struct rule_set_version *current; // 当前活动规则集
    struct rule_set_version *next;    // 下一个待激活规则集
    struct rule_set_version *prev;    // 上一个规则集(回滚用)
};
```

**原子规则切换过程**：

```
// 加载新规则集
int load_new_ruleset(const char *rulefile) {
    // 1. 解析并编译新规则
    struct rule_set_version *new_version = parse_and_compile_rules(rulefile);
    if (!new_version)
        return -1;
    
    // 2. 获取写锁并更新规则集指针
    rte_rwlock_write_lock(&rule_manager.lock);
    
    // 保存旧规则集用于回滚
    if (rule_manager.prev) {
        free_ruleset_when_unused(rule_manager.prev);
    }
    rule_manager.prev = rule_manager.current;
    
    // 设置新规则集为下一个将要使用的
    rule_manager.next = new_version;
    
    rte_rwlock_write_unlock(&rule_manager.lock);
    
    // 3. 向工作线程发送规则更新通知
    notify_workers_ruleset_update();
    
    return 0;
}

// 工作线程切换规则集(无锁设计)
void worker_switch_ruleset(void) {
    // 检查是否有新规则集
    if (rule_manager.next == NULL)
        return;
    
    // 获取读锁，确保规则管理器状态一致性
    rte_rwlock_read_lock(&rule_manager.lock);
    
    // 缓存当前使用的规则集版本指针
    struct rule_set_version *old_version = rule_manager.current;
    // 切换到新规则集
    struct rule_set_version *new_version = rule_manager.next;
    
    // 增加新规则集引用计数
    atomic_inc(&new_version->ref_count);
    
    // 更新线程局部规则集指针
    local_ruleset = new_version;
    
    // 减少旧规则集引用计数
    if (old_version) {
        atomic_dec(&old_version->ref_count);
    }
    
    rte_rwlock_read_unlock(&rule_manager.lock);
    
    // 确认切换完成
    confirm_ruleset_switch();
}
```

**增量规则更新优化**：

```
// 创建增量规则更新
struct rule_delta *create_rule_delta(struct rule_set_version *old_set, 
                                   struct rule_set_version *new_set) {
    struct rule_delta *delta = rte_zmalloc("rule_delta", 
                                         sizeof(struct rule_delta), 0);
    
    // 找出需要添加的规则
    for (uint32_t i = 0; i < new_set->num_rules; i++) {
        if (!find_rule_by_id(old_set, new_set->rules[i]->id)) {
            add_to_delta_add_list(delta, new_set->rules[i]);
        }
    }
    
    // 找出需要删除的规则
    for (uint32_t i = 0; i < old_set->num_rules; i++) {
        if (!find_rule_by_id(new_set, old_set->rules[i]->id)) {
            add_to_delta_remove_list(delta, old_set->rules[i]->id);
        }
    }
    
    // 找出需要修改的规则
    for (uint32_t i = 0; i < new_set->num_rules; i++) {
        struct rule *old_rule = find_rule_by_id(old_set, new_set->rules[i]->id);
        if (old_rule && old_rule->rev != new_set->rules[i]->rev) {
            add_to_delta_modify_list(delta, new_set->rules[i]);
        }
    }
    
    return delta;
}

// 应用增量更新
int apply_rule_delta(struct rule_set_version *ruleset, struct rule_delta *delta) {
    // 应用删除操作
    for (uint32_t i = 0; i < delta->num_removes; i++) {
        remove_rule_by_id(ruleset, delta->remove_ids[i]);
    }
    
    // 应用添加操作
    for (uint32_t i = 0; i < delta->num_adds; i++) {
        add_rule_to_set(ruleset, delta->add_rules[i]);
    }
    
    // 应用修改操作
    for (uint32_t i = 0; i < delta->num_modifies; i++) {
        update_rule_in_set(ruleset, delta->modify_rules[i]);
    }
    
    // 更新索引和优化结构
    rebuild_rule_indexes(ruleset);
    
    return 0;
}
```



## 4.2 特征匹配引擎

### 4.2.1 多模式匹配算法(AC/DFA/Hyperscan)

多模式匹配是NIDS的核心功能，需要高效地在数据包有效载荷中搜索大量模式。

**算法实现与选择**：

```
// 匹配算法类型枚举
enum pattern_match_type {
    MATCH_AC,          // Aho-Corasick算法
    MATCH_WUMANBER,    // Wu-Manber算法
    MATCH_HYPERSCAN,   // Intel Hyperscan
    MATCH_REGEX,       // 正则表达式引擎
    MATCH_DIRECT       // 直接比较(少量模式时)
};

// 多模式匹配上下文
struct pattern_matcher {
    enum pattern_match_type type;
    void *context;           // 算法特定上下文
    uint32_t num_patterns;   // 模式数量
    uint32_t min_length;     // 最短模式长度
    uint32_t max_length;     // 最长模式长度
    
    // 函数指针表
    int (*add_pattern)(struct pattern_matcher *pm, 
                     const uint8_t *pattern, uint32_t len, 
                     uint32_t id, uint32_t flags);
    int (*compile)(struct pattern_matcher *pm);
    int (*search)(struct pattern_matcher *pm, 
                const uint8_t *data, uint32_t len,
                match_callback_t callback, void *callback_ctx);
    void (*free)(struct pattern_matcher *pm);
};
```

**Aho-Corasick实现**：

```
// AC状态机节点
struct ac_node {
    uint32_t id;              // 模式ID(如果是结束节点)
    uint8_t depth;            // 节点深度
    uint16_t character;       // 字符
    struct ac_node *parent;   // 父节点
    struct ac_node *next;     // 同深度链表
    struct ac_node *child;    // 第一个子节点
    struct ac_node *sibling;  // 同父节点的下一个兄弟
    struct ac_node *failure;  // 失败指针
};

// 创建AC匹配器
struct pattern_matcher *create_ac_matcher(void) {
    struct pattern_matcher *pm = rte_zmalloc("ac_matcher", 
                                          sizeof(struct pattern_matcher), 0);
    if (!pm)
        return NULL;
    
    pm->type = MATCH_AC;
    pm->context = rte_zmalloc("ac_context", sizeof(struct ac_context), 0);
    pm->add_pattern = ac_add_pattern;
    pm->compile = ac_compile;
    pm->search = ac_search;
    pm->free = ac_free;
    
    // 初始化AC根节点
    struct ac_context *ctx = (struct ac_context *)pm->context;
    ctx->root = create_ac_node(0, 0, NULL);
    
    return pm;
}

// AC搜索实现
int ac_search(struct pattern_matcher *pm, const uint8_t *data, uint32_t len,
            match_callback_t callback, void *callback_ctx) {
    struct ac_context *ctx = (struct ac_context *)pm->context;
    struct ac_node *current = ctx->root;
    int matches = 0;
    
    for (uint32_t i = 0; i < len; i++) {
        uint8_t ch = data[i];
        
        // 在子节点中查找匹配
        while (current != ctx->root && !find_child(current, ch)) {
            current = current->failure;
        }
        
        struct ac_node *next = find_child(current, ch);
        if (next) {
            current = next;
        }
        
        // 检查是否存在匹配
        struct ac_node *check = current;
        while (check != ctx->root) {
            if (check->id > 0) {
                // 找到匹配
                struct pattern_match match = {
                    .pattern_id = check->id,
                    .offset = i - check->depth + 1,
                    .length = check->depth
                };
                
                // 调用回调处理匹配
                if (callback(callback_ctx, &match) != 0) {
                    // 回调请求停止搜索
                    return matches;
                }
                matches++;
            }
            check = check->failure;
        }
    }
    
    return matches;
}
```

**Hyperscan集成**：

```
#ifdef HAVE_HYPERSCAN
#include <hs/hs.h>

// Hyperscan上下文
struct hs_context {
    hs_database_t *db;        // 编译后的模式数据库
    hs_scratch_t *scratch;    // 搜索工作区
    uint32_t *pattern_ids;    // 模式ID映射
    uint32_t num_patterns;    // 模式数量
};

// Hyperscan匹配回调
static int hs_match_handler(unsigned int id, unsigned long long from,
                          unsigned long long to, unsigned int flags, void *ctx) {
    struct hs_callback_ctx *callback_ctx = (struct hs_callback_ctx *)ctx;
    struct pattern_match match = {
        .pattern_id = callback_ctx->pattern_ids[id],
        .offset = (uint32_t)from,
        .length = (uint32_t)(to - from)
    };
    
    // 调用用户提供的回调
    return callback_ctx->callback(callback_ctx->user_ctx, &match);
}

// 创建Hyperscan匹配器
struct pattern_matcher *create_hyperscan_matcher(void) {
    struct pattern_matcher *pm = rte_zmalloc("hs_matcher", 
                                         sizeof(struct pattern_matcher), 0);
    if (!pm)
        return NULL;
    
    pm->type = MATCH_HYPERSCAN;
    pm->context = rte_zmalloc("hs_context", sizeof(struct hs_context), 0);
    pm->add_pattern = hs_add_pattern;
    pm->compile = hs_compile;
    pm->search = hs_search;
    pm->free = hs_free;
    
    return pm;
}

// Hyperscan搜索实现
int hs_search(struct pattern_matcher *pm, const uint8_t *data, uint32_t len,
            match_callback_t callback, void *callback_ctx) {
    struct hs_context *ctx = (struct hs_context *)pm->context;
    if (!ctx->db || !ctx->scratch)
        return -1;
    
    // 准备回调上下文
    struct hs_callback_ctx hs_ctx = {
        .pattern_ids = ctx->pattern_ids,
        .callback = callback,
        .user_ctx = callback_ctx
    };
    
    // 执行Hyperscan搜索
    hs_error_t err = hs_scan(ctx->db, (const char *)data, len, 0,
                           ctx->scratch, hs_match_handler, &hs_ctx);
    
    return (err == HS_SUCCESS) ? 0 : -1;
}
#endif /* HAVE_HYPERSCAN */
```

**算法自动选择逻辑**：

```
// 根据模式特征自动选择最佳算法
enum pattern_match_type select_best_algorithm(const struct pattern_set *patterns) {
    uint32_t num_patterns = patterns->num_patterns;
    uint32_t avg_length = calculate_average_length(patterns);
    bool has_case_insensitive = has_case_insensitive_patterns(patterns);
    bool has_regex = has_regex_patterns(patterns);
    
    // 少量模式使用直接比较
    if (num_patterns < 5)
        return MATCH_DIRECT;
    
    // 有正则表达式模式首选Hyperscan
    if (has_regex && is_hyperscan_available())
        return MATCH_HYPERSCAN;
    
    // 大量短模式使用AC算法
    if (num_patterns > 100 && avg_length < 8)
        return MATCH_AC;
    
    // 中等数量的较长模式使用Wu-Manber
    if (num_patterns > 20 && avg_length >= 8)
        return MATCH_WUMANBER;
    
    // 默认使用AC算法
    return MATCH_AC;
}
```



### 4.2.2 正则表达式引擎

正则表达式提供了更灵活的模式匹配能力，但也带来了更高的性能开销。

**正则表达式管理**：

```
// 正则表达式信息
struct regex_pattern {
    uint32_t id;              // 模式ID
    char *pattern;            // 原始正则表达式
    uint32_t flags;           // 匹配标志
    void *compiled;           // 编译后的正则表达式
    uint32_t refcount;        // 引用计数
};

// 正则表达式集
struct regex_set {
    uint32_t num_patterns;    // 模式数量
    struct regex_pattern **patterns; // 模式数组
    void *engine_ctx;         // 引擎特定上下文
};
```

**基于Hyperscan的正则表达式引擎**：

```
#ifdef HAVE_HYPERSCAN
// 创建Hyperscan正则表达式集
struct regex_set *create_hyperscan_regex_set(void) {
    struct regex_set *set = rte_zmalloc("regex_set", 
                                      sizeof(struct regex_set), 0);
    if (!set)
        return NULL;
    
    // 创建Hyperscan引擎上下文
    struct hs_regex_ctx *ctx = rte_zmalloc("hs_regex_ctx", 
                                         sizeof(struct hs_regex_ctx), 0);
    set->engine_ctx = ctx;
    
    // 初始化Hyperscan环境
    hs_error_t err = hs_valid_platform();
    if (err != HS_SUCCESS) {
        fprintf(stderr, "This platform does not support Hyperscan\n");
        rte_free(ctx);
        rte_free(set);
        return NULL;
    }
    
    return set;
}

// 编译正则表达式
int compile_hyperscan_regex(struct regex_set *set) {
    struct hs_regex_ctx *ctx = (struct hs_regex_ctx *)set->engine_ctx;
    uint32_t num = set->num_patterns;
    
    // 准备模式数组
    const char **patterns = rte_malloc("hs_patterns", 
                                     sizeof(char *) * num, 0);
    unsigned int *flags = rte_malloc("hs_flags", 
                                   sizeof(unsigned int) * num, 0);
    unsigned int *ids = rte_malloc("hs_ids", 
                                 sizeof(unsigned int) * num, 0);
    
    for (uint32_t i = 0; i < num; i++) {
        patterns[i] = set->patterns[i]->pattern;
        flags[i] = convert_to_hs_flags(set->patterns[i]->flags);
        ids[i] = set->patterns[i]->id;
    }
    
    // 编译模式数据库
    hs_compile_error_t *compile_err = NULL;
    hs_error_t err = hs_compile_multi(patterns, flags, ids, num, 
                                    HS_MODE_BLOCK, NULL, 
                                    &ctx->database, &compile_err);
    
    if (err != HS_SUCCESS) {
        fprintf(stderr, "HS compile error: %s\n", compile_err->message);
        hs_free_compile_error(compile_err);
        rte_free(patterns);
        rte_free(flags);
        rte_free(ids);
        return -1;
    }
    
    // 创建搜索工作区
    err = hs_alloc_scratch(ctx->database, &ctx->scratch);
    if (err != HS_SUCCESS) {
        fprintf(stderr, "HS scratch allocation error\n");
        hs_free_database(ctx->database);
        ctx->database = NULL;
        rte_free(patterns);
        rte_free(flags);
        rte_free(ids);
        return -1;
    }
    
    rte_free(patterns);
    rte_free(flags);
    rte_free(ids);
    return 0;
}

// 正则表达式匹配
int match_hyperscan_regex(struct regex_set *set, const uint8_t *data, 
                       uint32_t len, regex_match_cb callback, void *user_ctx) {
    struct hs_regex_ctx *ctx = (struct hs_regex_ctx *)set->engine_ctx;
    if (!ctx->database || !ctx->scratch)
        return -1;
    
    // 设置回调上下文
    struct hs_match_ctx match_ctx = {
        .callback = callback,
        .user_ctx = user_ctx,
        .regex_set = set
    };
    
    // 执行扫描
    hs_error_t err = hs_scan(ctx->database, (const char *)data, len, 0,
                          ctx->scratch, hs_match_event_handler, &match_ctx);
    
    return (err == HS_SUCCESS) ? 0 : -1;
}
#endif /* HAVE_HYPERSCAN */
```



**回退正则表达式引擎(PCRE)**：

```
#ifdef HAVE_PCRE
// PCRE上下文
struct pcre_regex_ctx {
    pcre **compiled;          // 编译后的正则表达式数组
    pcre_extra **extra;       // 优化数据
    uint32_t num_patterns;    // 模式数量
};

// 创建PCRE正则表达式集
struct regex_set *create_pcre_regex_set(void) {
    struct regex_set *set = rte_zmalloc("regex_set", 
                                      sizeof(struct regex_set), 0);
    if (!set)
        return NULL;
    
    struct pcre_regex_ctx *ctx = rte_zmalloc("pcre_regex_ctx", 
                                          sizeof(struct pcre_regex_ctx), 0);
    set->engine_ctx = ctx;
    
    return set;
}

// 编译PCRE表达式
int compile_pcre_regex(struct regex_set *set) {
    struct pcre_regex_ctx *ctx = (struct pcre_regex_ctx *)set->engine_ctx;
    uint32_t num = set->num_patterns;
    
    ctx->compiled = rte_malloc("pcre_compiled", 
                            sizeof(pcre *) * num, 0);
    ctx->extra = rte_malloc("pcre_extra", 
                          sizeof(pcre_extra *) * num, 0);
    ctx->num_patterns = num;
    
    for (uint32_t i = 0; i < num; i++) {
        const char *error;
        int erroffset;
        int pcre_flags = convert_to_pcre_flags(set->patterns[i]->flags);
        
        ctx->compiled[i] = pcre_compile(set->patterns[i]->pattern, pcre_flags,
                                      &error, &erroffset, NULL);
        
        if (!ctx->compiled[i]) {
            fprintf(stderr, "PCRE compile error at %d: %s\n", erroffset, error);
            return -1;
        }
        
        ctx->extra[i] = pcre_study(ctx->compiled[i], PCRE_STUDY_JIT_COMPILE, &error);
        if (error) {
            fprintf(stderr, "PCRE study error: %s\n", error);
        }
    }
    
    return 0;
}

// 匹配PCRE表达式(单个模式匹配示例)
int match_single_pcre(struct regex_set *set, uint32_t pattern_idx,
                    const uint8_t *data, uint32_t len, 
                    regex_match_cb callback, void *user_ctx) {
    struct pcre_regex_ctx *ctx = (struct pcre_regex_ctx *)set->engine_ctx;
    if (pattern_idx >= ctx->num_patterns)
        return -1;
    
    const int OVECSIZE = 30; // 输出向量大小
    int ovector[OVECSIZE];
    
    int rc = pcre_exec(ctx->compiled[pattern_idx], ctx->extra[pattern_idx],
                     (const char *)data, len, 0, 0, ovector, OVECSIZE);
    
    if (rc >= 0) {
        // 找到匹配
        struct regex_match match = {
            .pattern_id = set->patterns[pattern_idx]->id,
            .offset = ovector[0],
            .length = ovector[1] - ovector[0]
        };
        
        return callback(user_ctx, &match);
    }
    
    return 0; // 无匹配
}
#endif /* HAVE_PCRE */
```



## 4.3 流检测与状态跟踪

### 4.3.1 流会话管理

流会话管理是跟踪网络连接和维护状态的核心组件。

**流会话数据结构**：

```
// 流会话状态
enum flow_state {
    FLOW_STATE_NEW,
    FLOW_STATE_ESTABLISHED,
    FLOW_STATE_CLOSING,
    FLOW_STATE_CLOSED
};

// 流方向
enum flow_direction {
    FLOW_DIR_ORIGINAL,   // 原始方向(客户端->服务器)
    FLOW_DIR_REPLY       // 回复方向(服务器->客户端)
};

// 流会话结构
struct flow_session {
    struct flow_key key;         // 流标识键(五元组)
    uint64_t flow_id;            // 唯一流ID
    enum flow_state state;       // 流状态
    uint64_t first_seen;         // 首次看到的时间戳
    uint64_t last_seen;          // 最后看到的时间戳
    uint64_t pkts[2];            // 两个方向的包计数
    uint64_t bytes[2];           // 两个方向的字节计数
    uint8_t flags;               // 流标志
    
    // 流元数据
    uint16_t app_proto;          // 应用层协议
    void *app_data;              // 应用层数据
    void (*app_data_free)(void *); // 应用数据释放函数
    
    // 流检测状态
    uint32_t detect_flags;       // 检测标志
    struct detect_state *detect_state; // 检测状态
    
    // 分片/重组相关
    struct stream_reassembly *stream_data[2]; // 流重组数据
    
    // 锁和引用计数
    rte_spinlock_t lock;         // 流锁
    rte_atomic32_t use_cnt;      // 引用计数
};

// 流管理器结构
struct flow_manager {
    struct rte_hash *flow_hash;  // 流哈希表
    uint32_t max_flows;          // 最大流数量
    struct flow_session *flows;  // 流会话数组
    rte_atomic32_t flow_count;   // 当前流计数
    
    // 流超时值(纳秒)
    uint64_t new_timeout;        // 新流超时
    uint64_t est_timeout;        // 已建立流超时
    uint64_t fin_timeout;        // FIN后超时
    uint64_t rst_timeout;        // RST后超时
    
    // 锁和计时器
    rte_spinlock_t *locks;       // 细粒度锁数组
    uint32_t num_locks;          // 锁数量
    uint64_t last_cleanup;       // 上次清理时间
};
```

**流管理器实现**：

```
// 创建流管理器
struct flow_manager *flow_manager_create(uint32_t max_flows) {
    struct flow_manager *mgr = rte_zmalloc("flow_manager", 
                                         sizeof(*mgr), RTE_CACHE_LINE_SIZE);
    
    mgr->max_flows = max_flows;
    
    // 创建哈希表
    struct rte_hash_parameters hash_params = {
        .name = "flow_hash",
        .entries = max_flows,
        .key_len = sizeof(struct flow_key),
        .hash_func = rte_jhash,
        .hash_func_init_val = 0,
        .socket_id = rte_socket_id()
    };
    
    mgr->flow_hash = rte_hash_create(&hash_params);
    
    // 分配流会话数组
    mgr->flows = rte_zmalloc("flow_sessions", 
                         sizeof(struct flow_session) * max_flows, 
                         RTE_CACHE_LINE_SIZE);
    
    // 初始化细粒度锁(使用大致1:32的锁与流比例)
    mgr->num_locks = RTE_MAX(16, max_flows / 32);
    mgr->locks = rte_zmalloc("flow_locks", 
                          sizeof(rte_spinlock_t) * mgr->num_locks, 
                          RTE_CACHE_LINE_SIZE);
    
    for (uint32_t i = 0; i < mgr->num_locks; i++) {
        rte_spinlock_init(&mgr->locks[i]);
    }
    
    // 设置默认超时值
    mgr->new_timeout = 30 * NS_PER_SEC;       // 30秒
    mgr->est_timeout = 300 * NS_PER_SEC;      // 5分钟
    mgr->fin_timeout = 10 * NS_PER_SEC;       // 10秒
    mgr->rst_timeout = 5 * NS_PER_SEC;        // 5秒
    
    mgr->last_cleanup = rte_rdtsc();
    
    return mgr;
}

// 获取或创建流会话
struct flow_session *flow_get_or_create(struct flow_manager *mgr, 
                                      struct flow_key *key, 
                                      enum flow_direction dir,
                                      bool *is_new) {
    // 标准化流键
    normalize_flow_key(key);
    
    // 计算锁索引(基于流键哈希)
    uint32_t hash = rte_jhash(key, sizeof(*key), 0);
    uint32_t lock_idx = hash % mgr->num_locks;
    
    struct flow_session *flow = NULL;
    int ret;
    
    // 获取流锁
    rte_spinlock_lock(&mgr->locks[lock_idx]);
    
    // 查找现有流
    ret = rte_hash_lookup_data(mgr->flow_hash, key, (void **)&flow);
    
    if (ret < 0) {
        // 流不存在，创建新流
        if (rte_atomic32_read(&mgr->flow_count) >= mgr->max_flows) {
            // 流表已满，尝试清理过期流
            if (flow_cleanup_expired(mgr, 0.2) == 0) {
                // 无法清理足够的流
                rte_spinlock_unlock(&mgr->locks[lock_idx]);
                return NULL;
            }
        }
        
        // 分配新流对象
        int32_t pos = rte_hash_add_key(mgr->flow_hash, key);
        if (pos < 0) {
            rte_spinlock_unlock(&mgr->locks[lock_idx]);
            return NULL;
        }
        
        flow = &mgr->flows[pos];
        memset(flow, 0, sizeof(*flow));
        
        // 初始化新流
        rte_memcpy(&flow->key, key, sizeof(*key));
        flow->flow_id = generate_flow_id();
        flow->state = FLOW_STATE_NEW;
        flow->first_seen = flow->last_seen = rte_rdtsc();
        rte_spinlock_init(&flow->lock);
        rte_atomic32_set(&flow->use_cnt, 1);  // 初始引用计数
        
        rte_atomic32_inc(&mgr->flow_count);
        
        if (is_new) *is_new = true;
    } else {
        // 流已存在
        if (is_new) *is_new = false;
        
        // 增加引用计数
        rte_atomic32_inc(&flow->use_cnt);
    }
    
    rte_spinlock_unlock(&mgr->locks[lock_idx]);
    
    // 更新流统计
    if (flow) {
        rte_spinlock_lock(&flow->lock);
        flow->last_seen = rte_rdtsc();
        flow->pkts[dir]++;
        // 还可以更新字节计数、最后活动时间等
        rte_spinlock_unlock(&flow->lock);
    }
    
    return flow;
}

// 释放流引用
void flow_release(struct flow_manager *mgr, struct flow_session *flow) {
    if (!flow)
        return;
    
    // 减少引用计数
    if (rte_atomic32_dec_and_test(&flow->use_cnt)) {
        // 引用计数为0，完全释放流资源
        
        // 计算锁索引
        uint32_t hash = rte_jhash(&flow->key, sizeof(flow->key), 0);
        uint32_t lock_idx = hash % mgr->num_locks;
        
        rte_spinlock_lock(&mgr->locks[lock_idx]);
        
        // 从哈希表移除
        rte_hash_del_key(mgr->flow_hash, &flow->key);
        
        // 释放应用层数据
        if (flow->app_data && flow->app_data_free) {
            flow->app_data_free(flow->app_data);
        }
        
        // 释放流重组数据
        for (int i = 0; i < 2; i++) {
            if (flow->stream_data[i]) {
                stream_reassembly_free(flow->stream_data[i]);
            }
        }
        
        // 释放检测状态
        if (flow->detect_state) {
            detect_state_free(flow->detect_state);
        }
        
        // 减少流计数
        rte_atomic32_dec(&mgr->flow_count);
        
        rte_spinlock_unlock(&mgr->locks[lock_idx]);
    }
}
```



**流超时管理**：

```
// 定期清理过期流
int flow_cleanup_expired(struct flow_manager *mgr, float target_ratio) {
    uint64_t now = rte_rdtsc();
    uint64_t hz = rte_get_timer_hz();
    uint32_t cleaned = 0;
    uint32_t max_to_clean = mgr->max_flows * target_ratio;
    
    // 遍历所有锁区域
    for (uint32_t l = 0; l < mgr->num_locks && cleaned < max_to_clean; l++) {
        rte_spinlock_lock(&mgr->locks[l]);
        
        // 获取此锁保护的流列表
        struct flow_session *flows[MAX_FLOWS_PER_LOCK];
        uint32_t num_flows = 0;
        
        // 使用rte_hash迭代器获取流
        const void *keys[MAX_FLOWS_PER_LOCK];
        void *data[MAX_FLOWS_PER_LOCK];
        uint32_t iter = 0;
        
        int32_t ret = rte_hash_iterate(mgr->flow_hash, &keys[num_flows], 
                                     (void **)&data[num_flows], &iter);
        while (ret >= 0 && num_flows < MAX_FLOWS_PER_LOCK) {
            struct flow_session *flow = (struct flow_session *)data[num_flows];
            uint32_t flow_hash = rte_jhash(&flow->key, sizeof(flow->key), 0);
            
            // 只处理属于当前锁区域的流
            if (flow_hash % mgr->num_locks == l) {
                flows[num_flows++] = flow;
            }
            
            ret = rte_hash_iterate(mgr->flow_hash, &keys[num_flows], 
                                 (void **)&data[num_flows], &iter);
        }
        
        // 检查每个流的超时状态
        for (uint32_t i = 0; i < num_flows && cleaned < max_to_clean; i++) {
            struct flow_session *flow = flows[i];
            
            // 根据引用计数和流状态确定是否可以清理
            if (rte_atomic32_read(&flow->use_cnt) > 1) {
                continue;  // 流正在使用中
            }
            
            uint64_t idle_time = (now - flow->last_seen) / hz * NS_PER_SEC;
            bool expired = false;
            
            // 根据流状态确定超时
            switch (flow->state) {
                case FLOW_STATE_NEW:
                    expired = (idle_time > mgr->new_timeout);
                    break;
                case FLOW_STATE_ESTABLISHED:
                    expired = (idle_time > mgr->est_timeout);
                    break;
                case FLOW_STATE_CLOSING:
                    expired = (idle_time > mgr->fin_timeout);
                    break;
                case FLOW_STATE_CLOSED:
                    expired = (idle_time > mgr->rst_timeout);
                    break;
            }
            
            if (expired) {
                // 准备清理流
                flow_release(mgr, flow);
                cleaned++;
            }
        }
        
        rte_spinlock_unlock(&mgr->locks[l]);
    }
    
    mgr->last_cleanup = now;
    
    return cleaned;
}
```



### 4.3.2 协议状态跟踪

协议状态跟踪维护网络连接的状态机，是检测协议异常的基础。

**TCP状态跟踪**：

```
// TCP状态定义
enum tcp_state {
    TCP_STATE_NONE = 0,
    TCP_STATE_SYN_SENT,
    TCP_STATE_SYN_RECV,
    TCP_STATE_ESTABLISHED,
    TCP_STATE_FIN_WAIT1,
    TCP_STATE_FIN_WAIT2,
    TCP_STATE_CLOSING,
    TCP_STATE_CLOSE_WAIT,
    TCP_STATE_LAST_ACK,
    TCP_STATE_TIME_WAIT,
    TCP_STATE_CLOSED
};

// TCP会话跟踪结构
struct tcp_session {
    enum tcp_state state[2];  // 双向状态
    uint32_t next_seq[2];     // 下一个期望序列号
    uint32_t last_ack[2];     // 最后确认的序列号
    uint16_t window[2];       // 当前窗口大小
    uint16_t mss[2];          // 最大段大小
    uint8_t flags_seen[2];    // 看到的TCP标志
    uint32_t retrans_count[2]; // 重传计数
    uint8_t pkt_flags;        // 包处理标志
};

// 处理TCP数据包并更新状态
int tcp_state_update(struct flow_session *flow, struct rte_tcp_hdr *tcp, 
                   enum flow_direction dir, uint32_t pkt_len) {
    struct tcp_session *tcp_sess;
    bool is_new = false;
    
    // 获取或创建TCP会话数据
    if (!flow->app_data) {
        tcp_sess = rte_zmalloc("tcp_session", sizeof(*tcp_sess), 0);
        if (!tcp_sess)
            return -1;
        
        flow->app_data = tcp_sess;
        flow->app_data_free = tcp_session_free;
        is_new = true;
    } else {
        tcp_sess = (struct tcp_session *)flow->app_data;
    }
    
    // 提取TCP头信息
    uint8_t tcp_flags = tcp->tcp_flags;
    uint32_t seq = rte_be_to_cpu_32(tcp->sent_seq);
    uint32_t ack = rte_be_to_cpu_32(tcp->recv_ack);
    uint16_t win = rte_be_to_cpu_16(tcp->rx_win);
    
    // 更新标志记录
    tcp_sess->flags_seen[dir] |= tcp_flags;
    
    // 计算反向
    enum flow_direction rev_dir = (dir == FLOW_DIR_ORIGINAL) ? 
                                  FLOW_DIR_REPLY : FLOW_DIR_ORIGINAL;
    
    // 旧状态
    enum tcp_state old_state = tcp_sess->state[dir];
    
    // 使用TCP标志更新状态机
    switch (tcp_sess->state[dir]) {
        case TCP_STATE_NONE:
            if (tcp_flags & RTE_TCP_SYN_FLAG && !(tcp_flags & RTE_TCP_ACK_FLAG)) {
                // 客户端发送SYN
                tcp_sess->state[dir] = TCP_STATE_SYN_SENT;
                tcp_sess->next_seq[dir] = seq + 1;
            }
            break;
            
        case TCP_STATE_SYN_SENT:
            if ((tcp_flags & (RTE_TCP_SYN_FLAG | RTE_TCP_ACK_FLAG)) == 
                (RTE_TCP_SYN_FLAG | RTE_TCP_ACK_FLAG)) {
                // 服务器发送SYN+ACK
                tcp_sess->state[dir] = TCP_STATE_SYN_RECV;
                tcp_sess->next_seq[dir] = seq + 1;
                tcp_sess->last_ack[dir] = ack;
            }
            break;
            
        case TCP_STATE_SYN_RECV:
            if ((tcp_flags & RTE_TCP_ACK_FLAG) && !(tcp_flags & RTE_TCP_SYN_FLAG)) {
                // 客户端确认连接
                tcp_sess->state[dir] = TCP_STATE_ESTABLISHED;
                tcp_sess->last_ack[dir] = ack;
                
                // 如果双向都建立，更新流状态
                if (tcp_sess->state[rev_dir] == TCP_STATE_ESTABLISHED) {
                    flow->state = FLOW_STATE_ESTABLISHED;
                }
            }
            break;
            
        case TCP_STATE_ESTABLISHED:
            // 处理FIN或RST
            if (tcp_flags & RTE_TCP_FIN_FLAG) {
                tcp_sess->state[dir] = TCP_STATE_FIN_WAIT1;
                tcp_sess->next_seq[dir] = seq + 1;
                
                flow->state = FLOW_STATE_CLOSING;
            } else if (tcp_flags & RTE_TCP_RST_FLAG) {
                tcp_sess->state[dir] = TCP_STATE_CLOSED;
                flow->state = FLOW_STATE_CLOSED;
            } else {
                // 普通数据包，更新序列号
                if (pkt_len > (tcp->data_off >> 2)) {
                    uint32_t payload_len = pkt_len - (tcp->data_off >> 2);
                    tcp_sess->next_seq[dir] = seq + payload_len;
                }
                tcp_sess->last_ack[dir] = ack;
                tcp_sess->window[dir] = win;
            }
            break;
            
        // 实现其他状态转换...
        // FIN_WAIT1, FIN_WAIT2, CLOSING, CLOSE_WAIT, LAST_ACK, TIME_WAIT
    }
    
    // 检测状态变化
    if (old_state != tcp_sess->state[dir]) {
        // 更新流状态和用于检测的标志
    }
    
    // 检测重传
    if (seq < tcp_sess->next_seq[dir] && (pkt_len > (tcp->data_off >> 2))) {
        tcp_sess->retrans_count[dir]++;
    }
    
    return 0;
}
```



**HTTP状态跟踪**：

```
// HTTP消息状态
enum http_state {
    HTTP_STATE_NONE,
    HTTP_STATE_REQUEST_LINE,
    HTTP_STATE_REQUEST_HEADERS,
    HTTP_STATE_REQUEST_BODY,
    HTTP_STATE_RESPONSE_LINE,
    HTTP_STATE_RESPONSE_HEADERS,
    HTTP_STATE_RESPONSE_BODY,
    HTTP_STATE_COMPLETE
};

// HTTP会话结构
struct http_session {
    enum http_state state;         // 当前状态
    uint8_t request_method;        // 请求方法
    uint16_t response_status;      // 响应状态码
    uint8_t http_version_major;    // HTTP主版本号
    uint8_t http_version_minor;    // HTTP次版本号
    
    uint32_t content_length;       // 内容长度
    uint32_t body_received;        // 已接收正文长度
    
    bool chunked;                  // 是否分块传输
    uint32_t chunk_size;           // 当前块大小
    bool keep_alive;               // 是否保持连接
    
    struct http_header *headers;   // 头部列表
    uint16_t num_headers;          // 头部数量
    
    char *uri;                     // 请求URI
    char *host;                    // Host头部值
    char *user_agent;              // User-Agent头部值
    
    // 用于流量重建的缓冲区
    char *request_buffer;          // 请求缓冲区
    uint32_t request_buffer_size;  // 缓冲区大小
    uint32_t request_buffer_used;  // 已使用大小
};

// 解析HTTP请求
int http_parse_request(struct flow_session *flow, const uint8_t *data, 
                     uint32_t len, bool *complete) {
    struct http_session *http;
    
    // 获取或创建HTTP会话
    if (!flow->app_data) {
        http = rte_zmalloc("http_session", sizeof(*http), 0);
        if (!http)
            return -1;
        
        flow->app_data = http;
        flow->app_data_free = http_session_free;
        flow->app_proto = PROTO_HTTP;
    } else {
        http = (struct http_session *)flow->app_data;
    }
    
    // 根据当前状态解析HTTP数据
    switch (http->state) {
        case HTTP_STATE_NONE:
            // 解析请求行
            int ret = parse_http_request_line(data, len, http);
            if (ret > 0) {
                http->state = HTTP_STATE_REQUEST_HEADERS;
                data += ret;
                len -= ret;
            } else if (ret == 0) {
                // 需要更多数据
                *complete = false;
                return len;
            } else {
                // 解析错误
                return -1;
            }
            // 如果还有数据，继续解析
            if (len == 0) {
                *complete = false;
                return len;
            }
            // fall through
            
        case HTTP_STATE_REQUEST_HEADERS:
            // 解析请求头
            int consumed = parse_http_headers(data, len, http);
            if (consumed > 0) {
                data += consumed;
                len -= consumed;
                
                // 检查是否需要解析消息体
                if (has_body(http)) {
                    http->state = HTTP_STATE_REQUEST_BODY;
                } else {
                    http->state = HTTP_STATE_COMPLETE;
                    *complete = true;
                    return len;
                }
            } else if (consumed == 0) {
                // 需要更多数据
                *complete = false;
                return len;
            } else {
                // 解析错误
                return -1;
            }
            // 如果还有数据，继续解析
            if (len == 0) {
                *complete = false;
                return len;
            }
            // fall through
            
        case HTTP_STATE_REQUEST_BODY:
            // 解析请求体
            int body_consumed = parse_http_body(data, len, http);
            if (body_consumed >= 0) {
                data += body_consumed;
                len -= body_consumed;
                
                if (http->body_received == http->content_length || 
                    (http->chunked && is_last_chunk(http))) {
                    http->state = HTTP_STATE_COMPLETE;
                    *complete = true;
                } else {
                    *complete = false;
                }
                
                return len;
            } else {
                // 解析错误
                return -1;
            }
            
        case HTTP_STATE_COMPLETE:
            // 已完成解析
            *complete = true;
            return len;
    }
    
    return 0;
}
```



**DNS状态跟踪**：

```
// DNS会话结构
struct dns_session {
    uint16_t id;              // 请求ID
    uint16_t flags;           // 标志
    uint16_t question_count;  // 问题数量
    uint16_t answer_count;    // 应答数量
    
    char *query_name;         // 查询名称
    uint16_t query_type;      // 查询类型
    uint16_t query_class;     // 查询类别
    
    struct dns_answer *answers; // 应答数组
    uint16_t num_answers;     // 应答数量
};

// 解析DNS查询
int dns_parse_query(struct flow_session *flow, const uint8_t *data, uint32_t len) {
    struct dns_session *dns;
    
    // 获取或创建DNS会话
    if (!flow->app_data) {
        dns = rte_zmalloc("dns_session", sizeof(*dns), 0);
        if (!dns)
            return -1;
        
        flow->app_data = dns;
        flow->app_data_free = dns_session_free;
        flow->app_proto = PROTO_DNS;
    } else {
        dns = (struct dns_session *)flow->app_data;
    }
    
    // 解析DNS头部
    if (len < 12) // DNS头部至少12字节
        return -1;
    
    struct dns_header {
        uint16_t id;
        uint16_t flags;
        uint16_t question_count;
        uint16_t answer_count;
        uint16_t auth_count;
        uint16_t add_count;
    } __attribute__((__packed__));
    
    const struct dns_header *hdr = (const struct dns_header *)data;
    
    dns->id = rte_be_to_cpu_16(hdr->id);
    dns->flags = rte_be_to_cpu_16(hdr->flags);
    dns->question_count = rte_be_to_cpu_16(hdr->question_count);
    dns->answer_count = rte_be_to_cpu_16(hdr->answer_count);
    
    // 跳过DNS头部
    data += 12;
    len -= 12;
    
    // 解析查询部分
    if (dns->question_count > 0) {
        int name_len = parse_dns_name(data, len, &dns->query_name);
        if (name_len < 0)
            return -1;
            
        data += name_len;
        len -= name_len;
        
        // 解析查询类型和类别
        if (len < 4)
            return -1;
            
        dns->query_type = rte_be_to_cpu_16(*(uint16_t *)data);
        dns->query_class = rte_be_to_cpu_16(*(uint16_t *)(data + 2));
        
        // 检查常见的恶意DNS查询模式
        detect_suspicious_dns_query(flow, dns);
    }
    
    return 0;
}
```



### 4.3.3 异常行为检测

基于状态跟踪识别异常网络行为和攻击模式。

**TCP异常检测**：

```
// TCP异常行为标志
enum tcp_anomaly_flags {
    TCP_ANOMALY_NONE             = 0x0000,
    TCP_ANOMALY_RETRANSMISSION   = 0x0001, // 高频重传
    TCP_ANOMALY_ZERO_WINDOW      = 0x0002, // 零窗口
    TCP_ANOMALY_RST_FLOOD        = 0x0004, // RST洪水
    TCP_ANOMALY_SYN_FIN          = 0x0008, // 同时SYN和FIN
    TCP_ANOMALY_BAD_FLAGS        = 0x0010, // 非法标志组合
    TCP_ANOMALY_WINDOW_SCALING   = 0x0020, // 异常窗口缩放
    TCP_ANOMALY_OOO_SEGMENT      = 0x0040, // 大量乱序段
    TCP_ANOMALY_CONN_TIMEOUT     = 0x0080  // 连接超时
};

// 检测TCP异常
uint32_t detect_tcp_anomalies(struct flow_session *flow, 
                           struct rte_tcp_hdr *tcp, 
                           enum flow_direction dir) {
    struct tcp_session *tcp_sess = (struct tcp_session *)flow->app_data;
    if (!tcp_sess)
        return TCP_ANOMALY_NONE;
        
    uint32_t anomalies = TCP_ANOMALY_NONE;
    uint8_t tcp_flags = tcp->tcp_flags;
    
    // 检测SYN+FIN组合(通常是扫描)
    if ((tcp_flags & RTE_TCP_SYN_FLAG) && (tcp_flags & RTE_TCP_FIN_FLAG)) {
        anomalies |= TCP_ANOMALY_SYN_FIN;
    }
    
    // 检测非法标志组合
    if ((tcp_flags & RTE_TCP_SYN_FLAG) && (tcp_flags & RTE_TCP_RST_FLAG)) {
        anomalies |= TCP_ANOMALY_BAD_FLAGS;
    }
    
    // 检测零窗口
    if (rte_be_to_cpu_16(tcp->rx_win) == 0 && flow->state == FLOW_STATE_ESTABLISHED) {
        anomalies |= TCP_ANOMALY_ZERO_WINDOW;
    }
    
    // 检测高频重传
    uint32_t seq = rte_be_to_cpu_32(tcp->sent_seq);
    if (seq < tcp_sess->next_seq[dir] && tcp_sess->retrans_count[dir] > 3) {
        anomalies |= TCP_ANOMALY_RETRANSMISSION;
    }
    
    // 检测RST洪水
    if ((tcp_flags & RTE_TCP_RST_FLAG) && 
        tcp_sess->state[dir] == TCP_STATE_ESTABLISHED) {
        struct timespec now;
        clock_gettime(CLOCK_MONOTONIC, &now);
        
        static __thread struct {
            uint32_t src_ip;
            uint32_t dst_ip;
            uint16_t src_port;
            uint16_t dst_port;
            struct timespec ts;
            uint32_t count;
        } rst_track[RST_TRACK_SIZE];
        
        // 查找或创建跟踪记录
        // [在实际代码中实现RST洪水检测逻辑]
    }
    
    return anomalies;
}
```

**HTTP异常检测**：

```
// HTTP异常行为标志
enum http_anomaly_flags {
    HTTP_ANOMALY_NONE              = 0x0000,
    HTTP_ANOMALY_LONG_HEADER       = 0x0001, // 异常长的头部
    HTTP_ANOMALY_LONG_URL          = 0x0002, // 异常长的URL
    HTTP_ANOMALY_DOUBLE_ENCODING   = 0x0004, // 双重编码
    HTTP_ANOMALY_DIRECTORY_TRAVERSAL = 0x0008, // 目录遍历
    HTTP_ANOMALY_SQL_INJECTION     = 0x0010, // SQL注入模式
    HTTP_ANOMALY_XSS_ATTEMPT       = 0x0020, // XSS尝试
    HTTP_ANOMALY_USER_AGENT        = 0x0040, // 可疑User-Agent
    HTTP_ANOMALY_HEADER_INJECTION  = 0x0080  // HTTP头注入
};

// 检测HTTP异常
uint32_t detect_http_anomalies(struct flow_session *flow) {
    struct http_session *http = (struct http_session *)flow->app_data;
    if (!http)
        return HTTP_ANOMALY_NONE;
        
    uint32_t anomalies = HTTP_ANOMALY_NONE;
    
    // 检测异常长的URL
    if (http->uri && strlen(http->uri) > 2000) {
        anomalies |= HTTP_ANOMALY_LONG_URL;
    }
    
    // 检测异常长的头部
    for (uint16_t i = 0; i < http->num_headers; i++) {
        if (http->headers[i].value_len > 4096) {
            anomalies |= HTTP_ANOMALY_LONG_HEADER;
            break;
        }
    }
    
    // 检测目录遍历尝试
    if (http->uri && (strstr(http->uri, "../") || 
                      strstr(http->uri, "..\\") ||
                      strstr(http->uri, "%2e%2e%2f"))) {
        anomalies |= HTTP_ANOMALY_DIRECTORY_TRAVERSAL;
    }
    
    // 检测SQL注入模式
    if (http->uri && (strstr(http->uri, "' OR ") || 
                      strstr(http->uri, "\" OR ") ||
                      strstr(http->uri, "UNION SELECT") ||
                      strstr(http->uri, "DROP TABLE"))) {
        anomalies |= HTTP_ANOMALY_SQL_INJECTION;
    }
    
    // 检测XSS尝试
    if (http->uri && (strstr(http->uri, "<script>") || 
                      strstr(http->uri, "javascript:") ||
                      strstr(http->uri, "onerror=") ||
                      strstr(http->uri, "onload="))) {
        anomalies |= HTTP_ANOMALY_XSS_ATTEMPT;
    }
    
    // 检测双重编码
    if (http->uri && (strstr(http->uri, "%25") || 
                      detect_double_encoding(http->uri))) {
        anomalies |= HTTP_ANOMALY_DOUBLE_ENCODING;
    }
    
    // 检测可疑User-Agent
    if (http->user_agent && (strstr(http->user_agent, "sqlmap") || 
                            strstr(http->user_agent, "nikto") ||
                            strstr(http->user_agent, "nmap") ||
                            is_suspicious_user_agent(http->user_agent))) {
        anomalies |= HTTP_ANOMALY_USER_AGENT;
    }
    
    return anomalies;
}
```



**端口扫描检测**：

```
// 端口扫描跟踪结构
struct port_scan_tracker {
    rte_rwlock_t lock;
    struct rte_hash *src_trackers;     // 源IP跟踪表
    uint64_t cleanup_time;             // 上次清理时间
};

// 源端扫描记录
struct scan_record {
    uint32_t src_ip;                   // 源IP
    uint64_t first_seen;               // 首次看到时间
    uint64_t last_seen;                // 最后看到时间
    uint32_t dst_ports[MAX_PORT_TRACK]; // 目标端口数组
    uint32_t num_ports;                 // 跟踪的端口数量
    uint32_t rejected_count;            // 被拒绝连接数
    uint32_t timeout_count;             // 超时连接数
    uint8_t flags;                      // 标志位
};

// 更新端口扫描跟踪
int port_scan_track(struct port_scan_tracker *tracker, 
                  uint32_t src_ip, uint32_t dst_ip, uint16_t dst_port,
                  uint8_t tcp_flags, uint8_t scan_type) {
    // 获取源IP的扫描记录
    struct scan_record *record = NULL;
    int ret;
    bool is_new = false;
    
    rte_rwlock_read_lock(&tracker->lock);
    ret = rte_hash_lookup_data(tracker->src_trackers, &src_ip, (void **)&record);
    rte_rwlock_read_unlock(&tracker->lock);
    
    if (ret < 0) {
        // 创建新记录
        record = rte_zmalloc("scan_record", sizeof(*record), 0);
        if (!record)
            return -1;
            
        record->src_ip = src_ip;
        record->first_seen = record->last_seen = rte_rdtsc();
        
        rte_rwlock_write_lock(&tracker->lock);
        ret = rte_hash_add_key_data(tracker->src_trackers, &src_ip, record);
        rte_rwlock_write_unlock(&tracker->lock);
        
        if (ret < 0) {
            rte_free(record);
            return -1;
        }
        
        is_new = true;
    }
    
    // 更新扫描记录
    record->last_seen = rte_rdtsc();
    
    // 跟踪目标端口
    bool port_found = false;
    for (uint32_t i = 0; i < record->num_ports; i++) {
        if ((record->dst_ports[i] & 0xFFFF) == dst_port) {
            port_found = true;
            break;
        }
    }
    
    if (!port_found && record->num_ports < MAX_PORT_TRACK) {
        record->dst_ports[record->num_ports++] = dst_port;
    }
    
    // 跟踪连接状态
    if (scan_type == SCAN_SYN && (tcp_flags & RTE_TCP_RST_FLAG)) {
        record->rejected_count++;
    } else if (scan_type == SCAN_TIMEOUT) {
        record->timeout_count++;
    }
    
    // 检测扫描行为
    if (record->num_ports >= PORT_SCAN_THRESHOLD) {
        // 检测SYN扫描
        if (record->rejected_count > record->num_ports / 2) {
            record->flags |= SCAN_TYPE_SYN;
            generate_port_scan_alert(src_ip, "SYN port scan detected", 
                                   record->num_ports, record->rejected_count);
        }
        
        // 检测超时扫描(FIN, NULL等)
        if (record->timeout_count > record->num_ports / 2) {
            record->flags |= SCAN_TYPE_STEALTH;
            generate_port_scan_alert(src_ip, "Stealth port scan detected", 
                                   record->num_ports, record->timeout_count);
        }
    }
    
    return 0;
}
```



## 4.4 启发式检测机制

### 4.4.1 异常流量特征分析

基于流量特征识别异常网络行为，补充基于规则的检测。

**流量基线分析**：

```
// 流量基线结构
struct traffic_baseline {
    // 时间粒度
    uint32_t interval_sec;     // 采样间隔(秒)
    uint32_t history_size;     // 历史记录大小
    
    // 基线数据
    struct {
        uint64_t packets;      // 数据包数
        uint64_t bytes;        // 字节数
        uint32_t flows;        // 流数量
        uint16_t *port_hist;   // 端口直方图
        uint8_t *proto_hist;   // 协议直方图
    } *history;
    
    uint32_t current_idx;      // 当前历史索引
    uint64_t last_update;      // 上次更新时间
    
    // 统计数据
    double pkt_mean;           // 平均数据包数
    double pkt_stddev;         // 数据包数标准差
    double byte_mean;          // 平均字节数
    double byte_stddev;        // 字节数标准差
    double flow_mean;          // 平均流数量
    double flow_stddev;        // 流数量标准差
};

// 创建流量基线
struct traffic_baseline *create_traffic_baseline(uint32_t interval_sec, 
                                              uint32_t history_size) {
    struct traffic_baseline *baseline = rte_zmalloc("traffic_baseline", 
                                                  sizeof(*baseline), 0);
    
    baseline->interval_sec = interval_sec;
    baseline->history_size = history_size;
    baseline->history = rte_zmalloc("baseline_history", 
                                 sizeof(*baseline->history) * history_size, 0);
    
    // 分配端口和协议直方图
    for (uint32_t i = 0; i < history_size; i++) {
        baseline->history[i].port_hist = rte_zmalloc("port_hist", 
                                               sizeof(uint16_t) * 65536, 0);
        baseline->history[i].proto_hist = rte_zmalloc("proto_hist", 
                                                sizeof(uint8_t) * 256, 0);
    }
    
    baseline->current_idx = 0;
    baseline->last_update = time(NULL);
    
    return baseline;
}

// 更新流量基线
void update_traffic_baseline(struct traffic_baseline *baseline, 
                          uint64_t packets, uint64_t bytes, uint32_t flows,
                          const uint16_t *port_hist, const uint8_t *proto_hist) {
    time_t now = time(NULL);
    
    // 检查是否需要更新
    if (now - baseline->last_update < baseline->interval_sec) {
        return;
    }
    
    // 更新当前基线槽
    baseline->history[baseline->current_idx].packets = packets;
    baseline->history[baseline->current_idx].bytes = bytes;
    baseline->history[baseline->current_idx].flows = flows;
    
    // 复制直方图
    memcpy(baseline->history[baseline->current_idx].port_hist, port_hist, 
          sizeof(uint16_t) * 65536);
    memcpy(baseline->history[baseline->current_idx].proto_hist, proto_hist, 
          sizeof(uint8_t) * 256);
    
    // 更新索引
    baseline->current_idx = (baseline->current_idx + 1) % baseline->history_size;
    baseline->last_update = now;
    
    // 重新计算统计数据
    calculate_baseline_stats(baseline);
}

// 检测流量异常
uint32_t detect_traffic_anomalies(struct traffic_baseline *baseline, 
                               uint64_t packets, uint64_t bytes, uint32_t flows,
                               const uint16_t *port_hist, const uint8_t *proto_hist) {
    uint32_t anomalies = 0;
    
    // 计算Z分数(标准化分数)
    double pkt_zscore = (packets - baseline->pkt_mean) / baseline->pkt_stddev;
    double byte_zscore = (bytes - baseline->byte_mean) / baseline->byte_stddev;
    double flow_zscore = (flows - baseline->flow_mean) / baseline->flow_stddev;
    
    // 检测流量异常
    if (fabs(pkt_zscore) > ANOMALY_THRESHOLD) {
        anomalies |= ANOMALY_PKT_RATE;
    }
    
    if (fabs(byte_zscore) > ANOMALY_THRESHOLD) {
        anomalies |= ANOMALY_BYTE_RATE;
    }
    
    if (fabs(flow_zscore) > ANOMALY_THRESHOLD) {
        anomalies |= ANOMALY_FLOW_RATE;
    }
    
    // 检测端口分布异常
    double port_dist = calculate_distribution_distance(
        baseline->history[baseline->current_idx].port_hist, port_hist, 65536);
    
    if (port_dist > PORT_DIST_THRESHOLD) {
        anomalies |= ANOMALY_PORT_DIST;
    }
    
    // 检测协议分布异常
    double proto_dist = calculate_distribution_distance(
        baseline->history[baseline->current_idx].proto_hist, proto_hist, 256);
    
    if (proto_dist > PROTO_DIST_THRESHOLD) {
        anomalies |= ANOMALY_PROTO_DIST;
    }
    
    return anomalies;
}
```

**行为分析模型**：

```
// 主机行为分析结构
struct host_behavior {
    uint32_t ip;                     // 主机IP
    uint64_t first_seen;             // 首次看到时间
    uint64_t last_seen;              // 最后看到时间
    
    // 连接计数
    uint32_t outbound_conn;          // 出站连接数
    uint32_t inbound_conn;           // 入站连接数
    
    // 端口统计
    uint16_t dst_ports[MAX_PORT_TRACK]; // 目标端口列表
    uint16_t dst_port_count;         // 目标端口数量
    uint16_t src_ports[MAX_PORT_TRACK]; // 源端口列表
    uint16_t src_port_count;         // 源端口数量
    
    // 协议统计
    uint32_t proto_bytes[256];       // 每个协议的字节数
    uint32_t proto_packets[256];     // 每个协议的包数
    
    // DNS行为
    uint16_t dns_queries;            // DNS查询数
    uint16_t unique_domains;         // 唯一域名数
    char **domains;                  // 域名列表
    
    // HTTP行为
    uint16_t http_requests;          // HTTP请求数
    uint16_t unique_urls;            // 唯一URL数
    uint16_t unique_user_agents;     // 唯一User-Agent数
    
    // 失败连接
    uint16_t failed_conn;            // 失败连接数
    uint16_t conn_reset;             // 连接重置数
    
    // 行为模型
    double feature_vector[BEHAVIOR_FEATURES]; // 特征向量
    uint8_t behavior_type;           // 行为类型
    double anomaly_score;            // 异常分数
};

// 更新主机行为
void update_host_behavior(struct host_behavior *behavior, 
                        struct flow_session *flow, 
                        enum flow_direction dir) {
    // 更新基本统计
    behavior->last_seen = flow->last_seen;
    
    if (dir == FLOW_DIR_ORIGINAL) {
        behavior->outbound_conn++;
        
        // 跟踪目标端口
        uint16_t dst_port = flow->key.dst_port;
        if (!port_in_list(behavior->dst_ports, behavior->dst_port_count, dst_port) && 
            behavior->dst_port_count < MAX_PORT_TRACK) {
            behavior->dst_ports[behavior->dst_port_count++] = dst_port;
        }
    } else {
        behavior->inbound_conn++;
        
        // 跟踪源端口
        uint16_t src_port = flow->key.src_port;
        if (!port_in_list(behavior->src_ports, behavior->src_port_count, src_port) && 
            behavior->src_port_count < MAX_PORT_TRACK) {
            behavior->src_ports[behavior->src_port_count++] = src_port;
        }
    }
    
    // 更新协议统计
    uint8_t proto = flow->key.protocol;
    behavior->proto_bytes[proto] += flow->bytes[dir];
    behavior->proto_packets[proto] += flow->pkts[dir];
    
    // 更新应用层行为
    if (flow->app_proto == PROTO_DNS) {
        update_dns_behavior(behavior, flow);
    } else if (flow->app_proto == PROTO_HTTP) {
        update_http_behavior(behavior, flow);
    }
    
    // 更新连接状态
    if (flow->state == FLOW_STATE_CLOSED && 
        (is_conn_failed(flow) || is_conn_reset(flow))) {
        behavior->failed_conn++;
        
        if (is_conn_reset(flow)) {
            behavior->conn_reset++;
        }
    }
    
    // 提取特征向量
    extract_behavior_features(behavior);
    
    // 计算异常分数
    behavior->anomaly_score = calculate_anomaly_score(behavior);
    
    // 确定行为类型
    behavior->behavior_type = classify_behavior(behavior);
}

// 检测异常行为
bool detect_behavior_anomaly(struct host_behavior *behavior) {
    // 检查异常分数阈值
    if (behavior->anomaly_score > BEHAVIOR_ANOMALY_THRESHOLD) {
        return true;
    }
    
    // 检查端口扫描行为
    if (behavior->dst_port_count > PORT_SCAN_THRESHOLD && 
        behavior->failed_conn > behavior->dst_port_count / 2) {
        return true;
    }
    
    // 检查快速DNS查询(可能是域名生成算法)
    if (behavior->dns_queries > DNS_QUERY_THRESHOLD && 
        behavior->unique_domains > behavior->dns_queries * 0.8) {
        return true;
    }
    
    // 检查连接模式(可能是C&C通信)
    if (is_suspicious_connection_pattern(behavior)) {
        return true;
    }
    
    return false;
}
```

### 4.4.2 统计分析模型

使用统计和机器学习技术识别复杂的攻击模式。

**统计特征提取**：

```
// 统计分析配置
struct stats_config {
    uint32_t window_size;      // 分析窗口大小(秒)
    uint32_t update_interval;  // 更新间隔(秒)
    double threshold;          // 检测阈值
};

// 流量统计结构
struct traffic_stats {
    // 时间窗口
    uint64_t window_start;     // 窗口开始时间
    uint64_t last_update;      // 上次更新时间
    
    // 基本统计
    uint64_t packets;          // 数据包计数
    uint64_t bytes;            // 字节计数
    uint32_t flows;            // 流计数
    
    // 协议分布
    uint32_t proto_count[256]; // 协议计数
    
    // 端口分布
    struct port_stats {
        uint16_t port;         // 端口号
        uint32_t count;        // 计数
    } *top_src_ports;          // 源端口统计
    struct port_stats *top_dst_ports; // 目标端口统计
    uint32_t max_port_stats;   // 最大端口统计数
    
    // 流大小分布
    uint32_t flow_size_hist[HIST_BUCKETS]; // 流大小直方图
    
    // 包大小分布
    uint32_t pkt_size_hist[HIST_BUCKETS];  // 包大小直方图
    
    // 流持续时间分布
    uint32_t flow_duration_hist[HIST_BUCKETS]; // 流持续时间直方图
    
    // 临时存储用于计算标准差
    double *pkt_rate_history;  // 包率历史
    double *byte_rate_history; // 字节率历史
    uint32_t rate_history_size;   // 历史大小
    uint32_t rate_history_index;  // 当前索引
    
    // 统计值
    double pkt_rate_mean;      // 平均包率
    double pkt_rate_stddev;    // 包率标准差
    double byte_rate_mean;     // 平均字节率
    double byte_rate_stddev;   // 字节率标准差
};

// 创建流量统计结构
struct traffic_stats *create_traffic_stats(uint32_t window_size, 
                                        uint32_t max_port_stats,
                                        uint32_t rate_history_size) {
    struct traffic_stats *stats = rte_zmalloc("traffic_stats", 
                                            sizeof(*stats), 0);
    
    stats->window_start = rte_rdtsc();
    stats->last_update = stats->window_start;
    
    stats->top_src_ports = rte_zmalloc("src_ports", 
                                     sizeof(*stats->top_src_ports) * max_port_stats, 0);
    stats->top_dst_ports = rte_zmalloc("dst_ports", 
                                     sizeof(*stats->top_dst_ports) * max_port_stats, 0);
    stats->max_port_stats = max_port_stats;
    
    stats->pkt_rate_history = rte_zmalloc("pkt_rate_history", 
                                        sizeof(double) * rate_history_size, 0);
    stats->byte_rate_history = rte_zmalloc("byte_rate_history", 
                                         sizeof(double) * rate_history_size, 0);
    stats->rate_history_size = rate_history_size;
    
    return stats;
}

// 更新流量统计
void update_traffic_stats(struct traffic_stats *stats, 
                       struct flow_session *flow, 
                       uint64_t current_time) {
    // 增加基本计数
    stats->packets += flow->pkts[0] + flow->pkts[1];
    stats->bytes += flow->bytes[0] + flow->bytes[1];
    stats->flows++;
    
    // 更新协议分布
    stats->proto_count[flow->key.protocol]++;
    
    // 更新端口分布
    update_port_stats(stats->top_src_ports, stats->max_port_stats, 
                     flow->key.src_port);
    update_port_stats(stats->top_dst_ports, stats->max_port_stats, 
                     flow->key.dst_port);
    
    // 更新流大小直方图
    uint64_t flow_size = flow->bytes[0] + flow->bytes[1];
    update_histogram(stats->flow_size_hist, HIST_BUCKETS, flow_size, 
                    FLOW_SIZE_MIN, FLOW_SIZE_MAX);
    
    // 更新包大小直方图(假设已有平均包大小)
    uint64_t avg_pkt_size = flow_size / (flow->pkts[0] + flow->pkts[1]);
    update_histogram(stats->pkt_size_hist, HIST_BUCKETS, avg_pkt_size, 
                    PKT_SIZE_MIN, PKT_SIZE_MAX);
    
    // 更新流持续时间直方图
    uint64_t flow_duration = flow->last_seen - flow->first_seen;
    update_histogram(stats->flow_duration_hist, HIST_BUCKETS, flow_duration, 
                    FLOW_DURATION_MIN, FLOW_DURATION_MAX);
    
    // 检查是否需要更新窗口统计
    uint64_t window_duration_ticks = current_time - stats->window_start;
    uint64_t window_duration_sec = window_duration_ticks / rte_get_timer_hz();
    
    if (window_duration_sec >= STATS_UPDATE_INTERVAL) {
        // 计算速率
        double pkt_rate = stats->packets / (double)window_duration_sec;
        double byte_rate = stats->bytes / (double)window_duration_sec;
        
        // 更新速率历史
        stats->pkt_rate_history[stats->rate_history_index] = pkt_rate;
        stats->byte_rate_history[stats->rate_history_index] = byte_rate;
        stats->rate_history_index = (stats->rate_history_index + 1) % stats->rate_history_size;
        
        // 计算平均值和标准差
        calculate_stats_mean_stddev(stats);
        
        // 重置窗口
        stats->window_start = current_time;
        stats->packets = 0;
        stats->bytes = 0;
        stats->flows = 0;
    }
    
    stats->last_update = current_time;
}

// 计算统计值
void calculate_stats_mean_stddev(struct traffic_stats *stats) {
    // 计算包率平均值
    double pkt_rate_sum = 0;
    for (uint32_t i = 0; i < stats->rate_history_size; i++) {
        pkt_rate_sum += stats->pkt_rate_history[i];
    }
    stats->pkt_rate_mean = pkt_rate_sum / stats->rate_history_size;
    
    // 计算包率标准差
    double pkt_rate_var_sum = 0;
    for (uint32_t i = 0; i < stats->rate_history_size; i++) {
        double diff = stats->pkt_rate_history[i] - stats->pkt_rate_mean;
        pkt_rate_var_sum += diff * diff;
    }
    stats->pkt_rate_stddev = sqrt(pkt_rate_var_sum / stats->rate_history_size);
    
    // 计算字节率平均值和标准差(类似)
    // ...
}
```



**异常检测算法**：

```
// 异常检测配置
struct anomaly_detector {
    // 算法类型
    enum {
        ANOMALY_ALGO_ZSCORE,       // Z-score
        ANOMALY_ALGO_EWMA,         // 指数加权移动平均
        ANOMALY_ALGO_DBSCAN,       // DBSCAN聚类
        ANOMALY_ALGO_ISOLATION_FOREST // 隔离森林
    } algorithm;
    
    // 特征提取
    uint32_t num_features;         // 特征数量
    double **feature_history;      // 特征历史
    uint32_t history_size;         // 历史大小
    uint32_t history_index;        // 当前历史索引
    
    // 算法特定参数
    union {
        struct {
            double threshold;      // Z-score阈值
        } zscore;
        
        struct {
            double alpha;          // EWMA平滑因子
            double beta;           // 异常阈值倍数
            double *means;         // 每个特征的EWMA
            double *vars;          // 每个特征的方差EWMA
        } ewma;
        
        struct {
            double eps;            // DBSCAN邻域距离
            uint32_t min_pts;      // 最小点数
            uint32_t *clusters;    // 聚类分配
        } dbscan;
        
        struct {
            void *forest;          // 隔离森林模型
            uint32_t n_trees;      // 树数量
            uint32_t sample_size;  // 采样大小
        } iforest;
    };
    
    // 检测阈值
    double threshold;              // 异常分数阈值
};

// 初始化Z-score检测器
struct anomaly_detector *init_zscore_detector(uint32_t num_features, 
                                           uint32_t history_size,
                                           double threshold) {
    struct anomaly_detector *detector = rte_zmalloc("anomaly_detector", 
                                                  sizeof(*detector), 0);
    
    detector->algorithm = ANOMALY_ALGO_ZSCORE;
    detector->num_features = num_features;
    detector->history_size = history_size;
    detector->threshold = threshold;
    detector->zscore.threshold = threshold;
    
    // 分配特征历史
    detector->feature_history = rte_zmalloc("feature_history", 
                                         sizeof(double *) * history_size, 0);
                                         
    for (uint32_t i = 0; i < history_size; i++) {
        detector->feature_history[i] = rte_zmalloc("features", 
                                               sizeof(double) * num_features, 0);
    }
    
    return detector;
}

// 使用Z-score检测异常
double detect_anomalies_zscore(struct anomaly_detector *detector, 
                            const double *features) {
    // 存储当前特征
    memcpy(detector->feature_history[detector->history_index], features, 
          sizeof(double) * detector->num_features);
    
    // 更新索引
    detector->history_index = (detector->history_index + 1) % detector->history_size;
    
    // 计算每个特征的均值和标准差
    double *means = rte_zmalloc("means", sizeof(double) * detector->num_features, 0);
    double *stddevs = rte_zmalloc("stddevs", sizeof(double) * detector->num_features, 0);
    
    for (uint32_t f = 0; f < detector->num_features; f++) {
        // 计算均值
        double sum = 0;
        for (uint32_t i = 0; i < detector->history_size; i++) {
            sum += detector->feature_history[i][f];
        }
        means[f] = sum / detector->history_size;
        
        // 计算标准差
        double var_sum = 0;
        for (uint32_t i = 0; i < detector->history_size; i++) {
            double diff = detector->feature_history[i][f] - means[f];
            var_sum += diff * diff;
        }
        stddevs[f] = sqrt(var_sum / detector->history_size);
    }
    
    // 计算Z-scores
    double *zscores = rte_zmalloc("zscores", 
                               sizeof(double) * detector->num_features, 0);
                               
    for (uint32_t f = 0; f < detector->num_features; f++) {
        if (stddevs[f] > 0) {
            zscores[f] = fabs(features[f] - means[f]) / stddevs[f];
        } else {
            zscores[f] = 0;
        }
    }
    
    // 寻找最大Z-score
    double max_zscore = 0;
    for (uint32_t f = 0; f < detector->num_features; f++) {
        if (zscores[f] > max_zscore) {
            max_zscore = zscores[f];
        }
    }
    
    // 释放内存
    rte_free(means);
    rte_free(stddevs);
    rte_free(zscores);
    
    // 返回异常分数
    return max_zscore;
}
```

**检测模型融合**：

```
// 检测器类型
enum detector_type {
    DETECTOR_RULE,        // 基于规则
    DETECTOR_STATISTICAL, // 基于统计
    DETECTOR_BEHAVIORAL,  // 基于行为
    DETECTOR_ANOMALY      // 基于异常检测
};

// 检测结果
struct detection_result {
    uint32_t rule_id;         // 匹配规则ID(如果有)
    uint32_t score;           // 检测分数
    enum detector_type type;  // 检测器类型
    uint64_t timestamp;       // 时间戳
    char *description;        // 描述
    uint32_t confidence;      // 置信度(0-100)
};

// 检测器融合
struct detector_fusion {
    // 检测器配置
    bool enable_rule;         // 启用规则检测
    bool enable_statistical;  // 启用统计检测
    bool enable_behavioral;   // 启用行为检测
    bool enable_anomaly;      // 启用异常检测
    
    // 检测器阈值
    uint32_t rule_threshold;         // 规则阈值
    uint32_t statistical_threshold;  // 统计阈值
    uint32_t behavioral_threshold;   // 行为阈值
    uint32_t anomaly_threshold;      // 异常阈值
    
    // 融合权重
    double rule_weight;       // 规则权重
    double statistical_weight; // 统计权重
    double behavioral_weight; // 行为权重
    double anomaly_weight;    // 异常权重
    
    // 融合阈值
    uint32_t fusion_threshold; // 融合阈值
};

// 融合检测结果
bool fuse_detection_results(struct detector_fusion *fusion, 
                          struct detection_result **results, 
                          uint32_t num_results,
                          struct detection_result *final_result) {
    if (num_results == 0)
        return false;
        
    double weighted_score = 0;
    uint32_t total_confidence = 0;
    uint32_t rule_count = 0;
    uint32_t anomaly_count = 0;
    
    // 计算加权分数
    for (uint32_t i = 0; i < num_results; i++) {
        double weight = 0;
        
        switch (results[i]->type) {
            case DETECTOR_RULE:
                weight = fusion->rule_weight;
                rule_count++;
                break;
            case DETECTOR_STATISTICAL:
                weight = fusion->statistical_weight;
                break;
            case DETECTOR_BEHAVIORAL:
                weight = fusion->behavioral_weight;
                break;
            case DETECTOR_ANOMALY:
                weight = fusion->anomaly_weight;
                anomaly_count++;
                break;
        }
        
        weighted_score += results[i]->score * weight * 
                        (results[i]->confidence / 100.0);
        total_confidence += results[i]->confidence;
    }
    
    // 标准化加权分数
    if (total_confidence > 0) {
        weighted_score /= total_confidence;
    }
    
    // 设置最终结果
    final_result->score = (uint32_t)weighted_score;
    final_result->timestamp = rte_rdtsc();
    
    // 选择最高置信度的检测结果作为基础
    uint32_t max_confidence = 0;
    uint32_t max_idx = 0;
    
    for (uint32_t i = 0; i < num_results; i++) {
        if (results[i]->confidence > max_confidence) {
            max_confidence = results[i]->confidence;
            max_idx = i;
        }
    }
    
    final_result->type = results[max_idx]->type;
    final_result->description = strdup(results[max_idx]->description);
    
    // 设置置信度
    if (rule_count > 0 && anomaly_count > 0) {
        // 规则和异常检测都触发时，提高置信度
        final_result->confidence = (uint32_t)(max_confidence * 1.2);
    } else {
        final_result->confidence = max_confidence;
    }
    
    // 限制置信度最大值为100
    if (final_result->confidence > 100) {
        final_result->confidence = 100;
    }
    
    // 决定是否触发警报
    return (final_result->score >= fusion->fusion_threshold);
}
```



### 4.4.3 阈值调优机制

自动化调整检测阈值，减少误报和漏报。

**自适应阈值调整**：

```
// 阈值调优配置
struct threshold_tuner {
    // 调优参数
    uint32_t min_threshold;    // 最小阈值
    uint32_t max_threshold;    // 最大阈值
    uint32_t current_threshold; // 当前阈值
    
    // 性能指标
    uint32_t true_positives;   // 真阳性
    uint32_t false_positives;  // 假阳性
    uint32_t true_negatives;   // 真阴性
    uint32_t false_negatives;  // 假阴性
    
    // 调优目标
    double target_precision;   // 目标精确率
    double target_recall;      // 目标召回率
    double target_f1;          // 目标F1分数
    
    // 调优状态
    uint64_t last_update;      // 上次更新时间
    uint32_t update_interval;  // 更新间隔(秒)
    uint32_t stable_count;     // 稳定计数
    
    // 历史记录
    struct {
        uint32_t threshold;    // 阈值
        double precision;      // 精确率
        double recall;         // 召回率
        double f1;             // F1分数
    } *history;                // 历史记录
    uint32_t history_size;     // 历史大小
    uint32_t history_index;    // 当前历史索引
};

// 创建阈值调优器
struct threshold_tuner *create_threshold_tuner(uint32_t min_threshold, 
                                            uint32_t max_threshold,
                                            uint32_t initial_threshold,
                                            uint32_t update_interval,
                                            uint32_t history_size) {
    struct threshold_tuner *tuner = rte_zmalloc("threshold_tuner", 
                                              sizeof(*tuner), 0);
    
    tuner->min_threshold = min_threshold;
    tuner->max_threshold = max_threshold;
    tuner->current_threshold = initial_threshold;
    tuner->update_interval = update_interval;
    
    // 设置目标性能指标
    tuner->target_precision = 0.95;
    tuner->target_recall = 0.9;
    tuner->target_f1 = 0.92;
    
    // 分配历史记录
    tuner->history_size = history_size;
    tuner->history = rte_zmalloc("threshold_history", 
                              sizeof(*tuner->history) * history_size, 0);
    
    tuner->last_update = time(NULL);
    
    return tuner;
}

// 更新性能指标
void update_tuner_metrics(struct threshold_tuner *tuner, 
                        bool is_positive, bool ground_truth) {
    if (is_positive && ground_truth) {
        tuner->true_positives++;
    } else if (is_positive && !ground_truth) {
        tuner->false_positives++;
    } else if (!is_positive && ground_truth) {
        tuner->false_negatives++;
    } else {
        tuner->true_negatives++;
    }
}

// 计算当前性能指标
void calculate_tuner_metrics(struct threshold_tuner *tuner, 
                          double *precision, double *recall, double *f1) {
    // 计算精确率
    if (tuner->true_positives + tuner->false_positives > 0) {
        *precision = (double)tuner->true_positives / 
                    (tuner->true_positives + tuner->false_positives);
    } else {
        *precision = 1.0;  // 没有预测为正，精确率定义为1
    }
    
    // 计算召回率
    if (tuner->true_positives + tuner->false_negatives > 0) {
        *recall = (double)tuner->true_positives / 
                 (tuner->true_positives + tuner->false_negatives);
    } else {
        *recall = 1.0;  // 没有实际为正，召回率定义为1
    }
    
    // 计算F1分数
    if (*precision + *recall > 0) {
        *f1 = 2 * (*precision * *recall) / (*precision + *recall);
    } else {
        *f1 = 0;
    }
}

// 调整阈值
void tune_threshold(struct threshold_tuner *tuner) {
    time_t now = time(NULL);
    
    // 检查是否需要调整
    if (now - tuner->last_update < tuner->update_interval) {
        return;
    }
    
    // 计算当前性能指标
    double precision, recall, f1;
    calculate_tuner_metrics(tuner, &precision, &recall, &f1);
    
    // 记录历史
    tuner->history[tuner->history_index].threshold = tuner->current_threshold;
    tuner->history[tuner->history_index].precision = precision;
    tuner->history[tuner->history_index].recall = recall;
    tuner->history[tuner->history_index].f1 = f1;
    
    tuner->history_index = (tuner->history_index + 1) % tuner->history_size;
    
    // 检查是否需要调整阈值
    if (f1 < tuner->target_f1) {
        // 尝试提高F1分数
        if (precision < tuner->target_precision) {
            // 精确率太低，增加阈值
            tuner->current_threshold = RTE_MIN(tuner->current_threshold + 
                                        tuner->current_threshold / 10, 
                                        tuner->max_threshold);
        } else if (recall < tuner->target_recall) {
            // 召回率太低，降低阈值
            tuner->current_threshold = RTE_MAX(tuner->current_threshold - 
                                        tuner->current_threshold / 10, 
                                        tuner->min_threshold);
        }
    } else {
        // F1分数已达标，检查是否可以进一步优化
        if (precision > tuner->target_precision + 0.05 && recall < tuner->target_recall) {
            // 精确率远超目标，可以牺牲一些提高召回率
            tuner->current_threshold = RTE_MAX(tuner->current_threshold - 
                                        tuner->current_threshold / 20, 
                                        tuner->min_threshold);
        } else if (recall > tuner->target_recall + 0.05 && precision < tuner->target_precision) {
            // 召回率远超目标，可以牺牲一些提高精确率
            tuner->current_threshold = RTE_MIN(tuner->current_threshold + 
                                        tuner->current_threshold / 20, 
                                        tuner->max_threshold);
        } else {
            // 指标都达标，保持稳定
            tuner->stable_count++;
        }
    }
    
    // 重置计数器
    tuner->true_positives = 0;
    tuner->false_positives = 0;
    tuner->true_negatives = 0;
    tuner->false_negatives = 0;
    
    tuner->last_update = now;
}
```

**基于反馈的规则调优**：

```
// 规则性能数据
struct rule_performance {
    uint32_t rule_id;            // 规则ID
    uint32_t matches;            // 匹配次数
    uint32_t confirmed;          // 确认的匹配
    uint32_t false_positives;    // 假阳性
    uint64_t avg_processing_time; // 平均处理时间
    uint32_t num_samples;        // 样本数量
};

// 规则调优器
struct rule_tuner {
    struct rte_hash *rule_performance_map; // 规则性能映射
    uint32_t max_rules;          // 最大规则数
    
    // 规则优先级调整
    uint32_t *high_priority_rules;  // 高优先级规则
    uint32_t *low_priority_rules;   // 低优先级规则
    uint32_t num_high_priority;     // 高优先级规则数量
    uint32_t num_low_priority;      // 低优先级规则数量
    
    // 阈值调整
    struct threshold_tuner **rule_thresholds; // 规则阈值调优器
    
    // 统计
    uint64_t last_update;        // 上次更新时间
    uint32_t update_interval;    // 更新间隔(秒)
};

// 创建规则调优器
struct rule_tuner *create_rule_tuner(uint32_t max_rules, uint32_t update_interval) {
    struct rule_tuner *tuner = rte_zmalloc("rule_tuner", sizeof(*tuner), 0);
    
    // 创建规则性能映射
    struct rte_hash_parameters hash_params = {
        .name = "rule_performance",
        .entries = max_rules,
        .key_len = sizeof(uint32_t),
        .hash_func = rte_jhash,
        .hash_func_init_val = 0,
        .socket_id = rte_socket_id()
    };
    
    tuner->rule_performance_map = rte_hash_create(&hash_params);
    
    tuner->max_rules = max_rules;
    tuner->update_interval = update_interval;
    
    // 分配规则优先级数组
    tuner->high_priority_rules = rte_zmalloc("high_prio_rules", 
                                          sizeof(uint32_t) * max_rules, 0);
    tuner->low_priority_rules = rte_zmalloc("low_prio_rules", 
                                         sizeof(uint32_t) * max_rules, 0);
    
    // 分配规则阈值调优器数组
    tuner->rule_thresholds = rte_zmalloc("rule_thresholds", 
                                      sizeof(struct threshold_tuner *) * max_rules, 0);
    
    tuner->last_update = time(NULL);
    
    return tuner;
}

// 更新规则性能数据
void update_rule_performance(struct rule_tuner *tuner, uint32_t rule_id, 
                          bool matched, bool confirmed, uint64_t processing_time) {
    struct rule_performance *perf = NULL;
    int ret = rte_hash_lookup_data(tuner->rule_performance_map, &rule_id, 
                                (void **)&perf);
    
    if (ret < 0) {
        // 创建新的性能记录
        perf = rte_zmalloc("rule_perf", sizeof(*perf), 0);
        perf->rule_id = rule_id;
        
        ret = rte_hash_add_key_data(tuner->rule_performance_map, &rule_id, perf);
        if (ret < 0) {
            rte_free(perf);
            return;
        }
    }
    
    // 更新性能数据
    if (matched) {
        perf->matches++;
        
        if (confirmed) {
            perf->confirmed++;
        } else {
            perf->false_positives++;
        }
    }
    
    // 更新处理时间(指数移动平均)
    if (perf->avg_processing_time == 0) {
        perf->avg_processing_time = processing_time;
    } else {
        perf->avg_processing_time = (perf->avg_processing_time * 9 + processing_time) / 10;
    }
    
    perf->num_samples++;
}

// 优化规则优先级
void optimize_rule_priorities(struct rule_tuner *tuner) {
    time_t now = time(NULL);
    
    // 检查是否需要优化
    if (now - tuner->last_update < tuner->update_interval) {
        return;
    }
    
    // 重置优先级数组
    tuner->num_high_priority = 0;
    tuner->num_low_priority = 0;
    
    // 遍历所有规则性能数据
    const void *keys[RTE_MAX_LCORE];
    void *data[RTE_MAX_LCORE];
    uint32_t iter = 0;
    
    int32_t ret = rte_hash_iterate(tuner->rule_performance_map, &keys[0], 
                                 &data[0], &iter);
    
    while (ret >= 0 && iter < tuner->max_rules) {
        struct rule_performance *perf = (struct rule_performance *)data[0];
        
        // 计算规则有效性分数
        double effectiveness = 0;
        if (perf->matches > 0) {
            effectiveness = (double)perf->confirmed / perf->matches;
        }
        
        // 根据有效性和处理时间判断优先级
        if (effectiveness > 0.8 && perf->matches > 10) {
            // 高有效性规则提升优先级
            tuner->high_priority_rules[tuner->num_high_priority++] = perf->rule_id;
        } else if ((effectiveness < 0.2 && perf->matches > 10) || 
                 perf->avg_processing_time > HIGH_PROCESSING_TIME_THRESHOLD) {
            // 低有效性或高成本规则降低优先级
            tuner->low_priority_rules[tuner->num_low_priority++] = perf->rule_id;
        }
        
        // 获取下一个规则
        ret = rte_hash_iterate(tuner->rule_performance_map, &keys[0], 
                             &data[0], &iter);
    }
    
    // 应用优先级更改
    apply_rule_priorities(tuner);
    
    tuner->last_update = now;
}

// 为规则选择最佳动作
enum rule_action {
    RULE_ACTION_KEEP,      // 保持规则
    RULE_ACTION_DISABLE,   // 禁用规则
    RULE_ACTION_THRESHOLD, // 调整阈值
    RULE_ACTION_REWRITE    // 重写规则
};

// 推荐规则优化动作
enum rule_action recommend_rule_action(struct rule_tuner *tuner, uint32_t rule_id) {
    struct rule_performance *perf = NULL;
    int ret = rte_hash_lookup_data(tuner->rule_performance_map, &rule_id, 
                                (void **)&perf);
    
    if (ret < 0 || perf->num_samples < 10) {
        return RULE_ACTION_KEEP;  // 数据不足，保持规则
    }
    
    // 计算有效性
    double effectiveness = 0;
    if (perf->matches > 0) {
        effectiveness = (double)perf->confirmed / perf->matches;
    }
    
    // 决策逻辑
    if (effectiveness < 0.05 && perf->matches > 100) {
        // 极低有效性且触发频繁，考虑禁用
        return RULE_ACTION_DISABLE;
    } else if (effectiveness < 0.3 && perf->matches > 50) {
        // 低有效性，考虑调整阈值
        return RULE_ACTION_THRESHOLD;
    } else if (effectiveness < 0.6 && perf->avg_processing_time > HIGH_PROCESSING_TIME_THRESHOLD) {
```



# 5、应用层协议解析模块设计

应用层协议解析是DPDK网络入侵检测系统的核心功能模块之一，负责深度解析应用层协议内容，为后续规则匹配和安全分析提供基础。本设计采用高性能、模块化和可扩展的架构。

**应用层协议解析采用以下分层架构：**

```
+-----------------------------------------------+
|             协议解析器管理器                    |
|           (Protocol Parser Manager)           |
+-----------------------------------------------+
           |            |            |
+----------v---+ +------v------+ +---v----------+
| HTTP解析器    | | DNS解析器    | | SMTP解析器    | ...
| (HTTP Parser)| | (DNS Parser)| | (SMTP Parser)|
+----------+---+ +------+------+ +---+---------+
           |            |            |
+----------v------------v------------v----------+
|              协议状态跟踪系统                   |
|        (Protocol State Tracking System)       |
+-----------------------------------------------+
           |            |            |
+----------v---+ +------v------+ +---v----------+
| 会话管理      | | 数据重组     | | 异常检测      |
| (Session)    | | (Reassembly)| | (Anomaly)    |
+---------------+---------------+---------------+
```

#### 1. 协议解析器基类设计

为确保扩展性和一致性，所有协议解析器实现同一基类接口：

```
/* 协议解析器基类结构 */
struct protocol_parser {
    /* 基本信息 */
    const char *name;         /* 协议名称 */
    uint16_t default_port;    /* 默认端口 */
    uint32_t parser_id;       /* 解析器ID */
    
    /* 生命周期管理函数 */
    int (*init)(void *config);                    /* 初始化函数 */
    void (*cleanup)(void);                        /* 清理函数 */
    
    /* 会话生命周期函数 */
    void* (*create_session)(struct flow *f);      /* 创建会话上下文 */
    void (*destroy_session)(void *session);       /* 销毁会话上下文 */
    
    /* 数据处理函数 */
    int (*parse)(void *session, const uint8_t *data, 
                 uint32_t data_len, uint8_t direction);  /* 解析数据 */
    int (*process_gap)(void *session, uint32_t gap_size);/* 处理数据缺口 */
    
    /* 状态查询函数 */
    int (*get_state)(void *session);              /* 获取会话状态 */
    int (*has_completed)(void *session);          /* 会话是否完成 */
    
    /* 元数据提取函数 */
    int (*get_metadata)(void *session, void *metadata, uint32_t *size);
    
    /* 批处理支持 (优化) */
    int (*parse_burst)(void **sessions, const struct rte_mbuf **mbufs,
                      uint16_t nb_pkts, uint8_t *directions);
};
```

#### 2. HTTP协议解析器实现

HTTP协议解析器专注于高性能解析HTTP请求和响应：2. HTTP协议解析器实现

HTTP协议解析器专注于高性能解析HTTP请求和响应：

```
/* HTTP会话状态 */
enum http_state {
    HTTP_STATE_NONE = 0,
    HTTP_STATE_REQUEST_LINE,
    HTTP_STATE_REQUEST_HEADERS,
    HTTP_STATE_REQUEST_BODY,
    HTTP_STATE_RESPONSE_LINE,
    HTTP_STATE_RESPONSE_HEADERS,
    HTTP_STATE_RESPONSE_BODY,
    HTTP_STATE_COMPLETE
};

/* HTTP会话上下文 */
struct http_session {
    /* 会话状态 */
    enum http_state state;
    
    /* 请求数据 */
    struct {
        char method[16];
        char uri[MAX_URI_LENGTH];
        char version[16];
        uint16_t status_code;
        
        /* 重要头字段 */
        char host[MAX_HEADER_VALUE];
        char user_agent[MAX_HEADER_VALUE];
        char content_type[MAX_HEADER_VALUE];
        uint32_t content_length;
        
        /* 内容处理状态 */
        uint32_t body_received;
        uint8_t chunked_encoding;
    } request, response;
    
    /* 解析状态 */
    struct {
        uint32_t line_start;
        uint8_t *partial_data;
        uint32_t partial_len;
    } parser_state;
    
    /* 异常标志 */
    uint32_t anomaly_flags;
};

/* HTTP解析实现示例 */
static int http_parse(void *session, const uint8_t *data, 
                     uint32_t data_len, uint8_t direction) {
    struct http_session *http = (struct http_session *)session;
    int result = 0;
    
    /* 处理不同状态下的数据 */
    switch (http->state) {
        case HTTP_STATE_NONE:
            if (direction == FLOW_DIR_ORIGINAL)
                result = http_parse_request_line(http, data, data_len);
            break;
            
        case HTTP_STATE_REQUEST_LINE:
            result = http_parse_request_headers(http, data, data_len);
            break;
            
        /* 其他状态处理... */
    }
    
    /* 检测异常 */
    if (result >= 0)
        http_detect_anomalies(http);
    
    return result;
}

/* HTTP批量解析优化 */
static int http_parse_burst(void **sessions, const struct rte_mbuf **mbufs,
                          uint16_t nb_pkts, uint8_t *directions) {
    /* 预取优化 */
    if (nb_pkts >= 4) {
        /* 预取未来将要处理的mbuf数据 */
        rte_prefetch0(rte_pktmbuf_mtod(mbufs[2], void *));
        rte_prefetch0(rte_pktmbuf_mtod(mbufs[3], void *));
    }
    
    /* 批量处理 */
    for (uint16_t i = 0; i < nb_pkts; i++) {
        /* 预取后续包 */
        if (i + 4 < nb_pkts) {
            rte_prefetch0(rte_pktmbuf_mtod(mbufs[i+4], void *));
        }
        
        /* 解析当前包 */
        http_parse(sessions[i], 
                  rte_pktmbuf_mtod(mbufs[i], uint8_t *),
                  rte_pktmbuf_data_len(mbufs[i]),
                  directions[i]);
    }
    
    return 0;
}
```

#### 3. DNS协议解析器实现

DNS协议解析关注查询和响应分析：

```
/* DNS会话上下文 */
struct dns_session {
    /* 基本状态 */
    uint16_t transaction_id;
    uint8_t is_response;
    
    /* 查询信息 */
    struct {
        char qname[MAX_DNS_NAME_LEN];
        uint16_t qtype;
        uint16_t qclass;
    } query;
    
    /* 响应信息 */
    struct {
        uint16_t rcode;
        uint16_t answer_count;
        uint16_t authority_count;
        uint16_t additional_count;
    } response;
    
    /* 资源记录跟踪 */
    struct dns_rr {
        char name[MAX_DNS_NAME_LEN];
        uint16_t type;
        uint16_t class;
        uint32_t ttl;
        uint16_t rdlength;
        uint8_t *rdata;
    } *answers;
    
    /* 异常标志 */
    uint32_t anomaly_flags;
};

/* DNS解析函数 */
static int dns_parse(void *session, const uint8_t *data,
                   uint32_t data_len, uint8_t direction) {
    struct dns_session *dns = (struct dns_session *)session;
    
    /* DNS头部检查 */
    if (data_len < DNS_HEADER_SIZE)
        return -1;
    
    struct dns_header *hdr = (struct dns_header *)data;
    dns->transaction_id = rte_be_to_cpu_16(hdr->id);
    dns->is_response = (hdr->flags & DNS_QR_MASK) ? 1 : 0;
    
    /* 解析查询区域 */
    uint16_t qcount = rte_be_to_cpu_16(hdr->qdcount);
    if (qcount > 0) {
        dns_parse_question(dns, data + DNS_HEADER_SIZE, 
                          data_len - DNS_HEADER_SIZE);
    }
    
    /* 如果是响应，解析应答 */
    if (dns->is_response) {
        uint16_t ancount = rte_be_to_cpu_16(hdr->ancount);
        dns->response.rcode = hdr->flags & DNS_RCODE_MASK;
        dns->response.answer_count = ancount;
        
        /* 解析应答区域... */
    }
    
    /* 检测异常 */
    dns_detect_anomalies(dns);
    
    return 0;
}
```

#### 4. 协议解析器管理器

协议解析器管理器负责注册、查找和调度协议解析器：

```
/* 协议解析器管理器 */
struct parser_manager {
    /* 已注册的解析器 */
    struct protocol_parser *parsers[MAX_PARSERS];
    uint32_t parser_count;
    
    /* 端口到解析器的映射 */
    struct protocol_parser *port_map[UINT16_MAX+1];
    
    /* 锁，用于并发注册/查询 */
    rte_spinlock_t lock;
    
    /* 统计信息 */
    uint64_t parse_calls;
    uint64_t parse_errors;
};

/* 初始化协议解析器管理器 */
int parser_manager_init(void) {
    g_parser_manager = rte_zmalloc("parser_manager", 
                                 sizeof(struct parser_manager), 
                                 RTE_CACHE_LINE_SIZE);
    if (g_parser_manager == NULL)
        return -ENOMEM;
        
    rte_spinlock_init(&g_parser_manager->lock);
    
    /* 注册内置解析器 */
    register_http_parser();
    register_dns_parser();
    register_smtp_parser();
    /* 注册其他解析器... */
    
    return 0;
}

/* 注册协议解析器 */
int register_protocol_parser(struct protocol_parser *parser) {
    if (g_parser_manager->parser_count >= MAX_PARSERS)
        return -ENOSPC;
        
    rte_spinlock_lock(&g_parser_manager->lock);
    
    /* 添加到解析器列表 */
    uint32_t id = g_parser_manager->parser_count++;
    g_parser_manager->parsers[id] = parser;
    parser->parser_id = id;
    
    /* 更新端口映射 */
    if (parser->default_port != 0) {
        g_parser_manager->port_map[parser->default_port] = parser;
    }
    
    rte_spinlock_unlock(&g_parser_manager->lock);
    return 0;
}

/* 根据端口查找解析器 */
struct protocol_parser *get_parser_by_port(uint16_t port) {
    return g_parser_manager->port_map[port];
}
```



应用识别技术

1. 架构设计

```
+-----------------------------------+
|        应用识别管理器              |
|    (App Detection Manager)        |
+-----------------------------------+
         |               |
+--------v------+ +------v---------+
| 特征匹配识别器  | | 启发式识别器    |
| (Signature)   | | (Heuristic)    |
+--------------—+ +----------------+
```

应用识别采用两级处理模型：

1. **特征匹配识别器**：基于已知协议特征进行快速匹配
2. **启发式识别器**：当特征匹配失败时使用，基于协议行为特性识别

2. 特征匹配识别器

```
/* 协议特征结构 */
struct proto_signature {
    uint16_t proto_id;              /* 协议ID */
    const char *proto_name;         /* 协议名称 */
    uint8_t *pattern;               /* 匹配模式 */
    uint16_t pattern_len;           /* 模式长度 */
    uint16_t offset;                /* 检查偏移量 */
};

/* 特征匹配识别器 */
struct signature_detector {
    struct proto_signature *signatures;   /* 特征数组 */
    uint32_t sig_count;                   /* 特征数量 */
    void *pattern_matcher;                /* 模式匹配器 */
};

/* 批量识别函数 */
int detect_proto_by_pattern(struct rte_mbuf **mbufs, uint16_t nb_pkts,
                           uint16_t *proto_ids) {
    /* 批量预取优化 */
    if (nb_pkts > 2) {
        rte_prefetch0(rte_pktmbuf_mtod(mbufs[1], void *));
        rte_prefetch0(rte_pktmbuf_mtod(mbufs[2], void *));
    }
    
    /* 批量处理 */
    for (uint16_t i = 0; i < nb_pkts; i++) {
        /* 预取后续包 */
        if (i + 3 < nb_pkts)
            rte_prefetch0(rte_pktmbuf_mtod(mbufs[i+3], void *));
            
        /* 特征匹配 */
        const uint8_t *data = rte_pktmbuf_mtod(mbufs[i], uint8_t *);
        uint16_t len = rte_pktmbuf_data_len(mbufs[i]);
        
        proto_ids[i] = match_protocol_pattern(data, len);
    }
    
    return 0;
}
```

3. 启发式识别器

```
/* 协议启发式规则 */
struct proto_heuristic {
    uint16_t proto_id;                     /* 协议ID */
    const char *proto_name;                /* 协议名称 */
    
    /* 启发式检查函数 */
    int (*check)(const uint8_t *data, uint16_t len, 
                struct flow_record *flow);
    
    /* 所需的最小数据包数 */
    uint8_t min_packets;
};

/* 启发式识别器 */
struct heuristic_detector {
    struct proto_heuristic *heuristics;    /* 启发式规则数组 */
    uint32_t rule_count;                   /* 规则数量 */
};

/* 启发式检查示例 - HTTP */
static int http_heuristic_check(const uint8_t *data, uint16_t len,
                              struct flow_record *flow) {
    /* 检查是否包含HTTP方法 */
    static const char *http_methods[] = {
        "GET ", "POST ", "HEAD ", "PUT ", "DELETE ", 
        "OPTIONS ", "CONNECT ", "TRACE "
    };
    
    for (int i = 0; i < sizeof(http_methods)/sizeof(char*); i++) {
        size_t method_len = strlen(http_methods[i]);
        if (len >= method_len && 
            memcmp(data, http_methods[i], method_len) == 0) {
            return 1;  /* 匹配HTTP */
        }
    }
    
    /* 检查是否是HTTP响应 */
    if (len > 12 && 
        memcmp(data, "HTTP/1.", 7) == 0 &&
        (data[7] == '0' || data[7] == '1') && 
        data[8] == ' ') {
        return 1;  /* 匹配HTTP */
    }
    
    return 0;  /* 不匹配 */
}
```

4. 应用识别管理器

```
/* 应用识别上下文 */
struct app_detect_ctx {
    /* 识别器实例 */
    struct signature_detector sig_detector;
    struct heuristic_detector heur_detector;
    
    /* 协议映射表 */
    struct {
        uint16_t proto_id;
        const char *proto_name;
    } proto_map[MAX_PROTOCOLS];
    
    /* 端口映射 */
    uint16_t port_to_proto[UINT16_MAX+1];
};

/* 初始化应用识别模块 */
int app_detect_init(void) {
    /* 分配内存 */
    g_app_detect = rte_zmalloc("app_detector", 
                             sizeof(struct app_detect_ctx),
                             RTE_CACHE_LINE_SIZE);
    if (!g_app_detect)
        return -ENOMEM;
    
    /* 初始化特征匹配器 */
    init_signature_detector(&g_app_detect->sig_detector);
    
    /* 初始化启发式识别器 */
    init_heuristic_detector(&g_app_detect->heur_detector);
    
    /* 初始化协议和端口映射 */
    init_protocol_maps();
    
    return 0;
}

/* 流量识别主函数 */
uint16_t detect_application(struct rte_mbuf *pkt, struct flow_record *flow) {
    uint16_t proto_id = PROTO_UNKNOWN;
    
    /* 1. 首先尝试已识别流 */
    if (flow->app_proto != PROTO_UNKNOWN)
        return flow->app_proto;
    
    /* 2. 尝试端口识别 */
    struct tcp_hdr *tcp = rte_pktmbuf_mtod_offset(pkt, struct tcp_hdr *, 
                                               flow->l4_offset);
    uint16_t src_port = rte_be_to_cpu_16(tcp->src_port);
    uint16_t dst_port = rte_be_to_cpu_16(tcp->dst_port);
    
    proto_id = g_app_detect->port_to_proto[dst_port];
    if (proto_id != PROTO_UNKNOWN)
        return proto_id;
        
    /* 3. 尝试特征匹配 */
    const uint8_t *payload = rte_pktmbuf_mtod_offset(pkt, uint8_t *, 
                                                  flow->payload_offset);
    uint16_t payload_len = pkt->pkt_len - flow->payload_offset;
    
    if (payload_len > 0) {
        proto_id = match_protocol_pattern(payload, payload_len);
        if (proto_id != PROTO_UNKNOWN)
            return proto_id;
    }
    
    /* 4. 最后尝试启发式识别 */
    if (flow->packet_count >= MIN_PACKETS_FOR_HEURISTIC && payload_len > 0) {
        for (uint32_t i = 0; i < g_app_detect->heur_detector.rule_count; i++) {
            struct proto_heuristic *rule = &g_app_detect->heur_detector.heuristics[i];
            
            if (flow->packet_count >= rule->min_packets && 
                rule->check(payload, payload_len, flow)) {
                proto_id = rule->proto_id;
                break;
            }
        }
    }
    
    /* 更新流记录 */
    flow->app_proto = proto_id;
    return proto_id;
}
```

**批量处理的接口**

```
/* 批量应用识别 */
int detect_applications_burst(struct rte_mbuf **pkts, 
                            struct flow_record **flows,
                            uint16_t nb_pkts) {
    /* 预取优化 */
    if (nb_pkts >= 3) {
        rte_prefetch0(flows[1]);
        rte_prefetch0(flows[2]);
    }
    
    /* 批量处理 */
    for (uint16_t i = 0; i < nb_pkts; i++) {
        /* 预取后续流记录 */
        if (i + 3 < nb_pkts)
            rte_prefetch0(flows[i+3]);
            
        /* 跳过已识别的流 */
        if (flows[i]->app_proto != PROTO_UNKNOWN)
            continue;
            
        /* 识别应用 */
        flows[i]->app_proto = detect_application(pkts[i], flows[i]);
    }
    
    return 0;
}
```



1、数据平面
包捕获模块：基于DPDK的高效收包

预处理模块：包重组、流表管理

检测引擎：协议解析、规则匹配

响应模块：告警生成、阻断动作



2、控制平面
规则管理

系统配置



cmake-build-debug不进行搜索

对带有dpdk字样的目录不进行搜索



## 架构设计的思考
- DPDK的pipeline模型 vs run-to-completion模型？
- 设计合理的不同核间通信机制
