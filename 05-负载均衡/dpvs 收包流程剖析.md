dpvs的收包是分为两种情况的

```
static void lcore_job_recv_fwd(void *arg)
{
    int i, j;
    portid_t pid;
    lcoreid_t cid;
    struct netif_queue_conf *qconf;

    cid = rte_lcore_id();
    assert(LCORE_ID_ANY != cid);

    for (i = 0; i < lcore_conf[lcore2index[cid]].nports; i++) {
        pid = lcore_conf[lcore2index[cid]].pqs[i].id;//这句代码太绕了，啥意思啊？
        assert(pid <= bond_pid_end);

        for (j = 0; j < lcore_conf[lcore2index[cid]].pqs[i].nrxq; j++) {
            qconf = &lcore_conf[lcore2index[cid]].pqs[i].rxqs[j];

            lcore_process_arp_ring(cid);
            lcore_process_redirect_ring(cid);
            qconf->len = netif_rx_burst(pid, qconf);

            lcore_stats_burst(&lcore_stats[cid], qconf->len);

            lcore_process_packets(qconf->mbufs, cid, qconf->len, 0);
        }
    }
}
```



lcore_conf是个什么类型？
lcore2index是将逻辑核心id转换为什么？
pqs是什么？rxqs是什么意思？



## 1、lcore_conf变量的定义

```
// netif.c中的定义
static struct netif_lcore_conf lcore_conf[DPVS_MAX_LCORE + 1];
```

netif_lcore_conf的定义

```
// 每个逻辑核心的配置结构
struct netif_lcore_conf {
    lcoreid_t id;          // 逻辑核心ID
    int nports;            // 该核心处理的端口数量
    struct port_queue_conf pqs[NETIF_MAX_PORTS];  // 端口队列配置数组
};
```

pqs是什么呢？

```
struct port_queue_conf {
    portid_t id;          // 端口ID
    int nrxq;            // 接收队列数量
    int ntxq;            // 发送队列数量
    struct netif_queue_conf rxqs[NETIF_MAX_QUEUES];  // 接收队列配置数组
    struct netif_queue_conf txqs[NETIF_MAX_QUEUES];  // 发送队列配置数组
};
```

每个端口的队列配置，包括接收队列rxqs和发送队txqs列的配置。

NETIF_MAX_QUEUES宏定义的值什么呢？



rxqs是什么呢？

```
struct netif_queue_conf {
    queueid_t id;        // 队列ID
    uint16_t len;        // 当前队列长度
    struct rte_mbuf *mbufs[NETIF_MAX_PKT_BURST];  // 数据包缓冲区
    struct rx_partner *isol_rxq;  // 隔离接收队列指针
};
```



## 2、lcore2index 数组

```
static int lcore2index[DPVS_MAX_LCORE];  // 逻辑核心ID到配置索引的映射
```

- 作用：将逻辑核心ID(cid)转换为配置数组的索引

- 例如：如果核心ID是8，但它可能是第3个工作核心，那么lcore2index[8]可能等于3



## 3、解析这句代码

```
pid = lcore_conf[lcore2index[cid]].pqs[i].id;
```

1、先查找逻辑核心cid对应的配置索引

2、然后在lcore_conf数组中找到该核心的位置

3、在该核心配置中，访问第 i 个端口的队列配置：pqs[i]

4、最后访问该端口的第 j 个接收队列：rxqs[j]



图示说明

```
lcore_conf[index] --> 核心配置
    |
    +-- pqs[i] --> 端口队列配置
         |
         +-- rxqs[j] --> 接收队列配置
         |    |
         |    +-- id, len, mbufs[], isol_rxq
         |
         +-- txqs[k] --> 发送队列配置
              |
              +-- id, len, mbufs[]
```



这种多层次的设计的好处：

1. 管理多核心系统中的网络资源
2. 支持灵活的队列分配
3. 实现高效的数据包处理
4. 支持隔离接收等高级特性