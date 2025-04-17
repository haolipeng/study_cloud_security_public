dpdk的编程模式有两种，run to completion和pipeline，前者一般适合业务逻辑比较简单的流程。



# 一、核心结构体

```
/* Note: lockless, lcore_job can only be register on initialization stage and
 *       unregistered on cleanup stage.
 */
static struct list_head dpvs_lcore_jobs[LCORE_ROLE_MAX][LCORE_JOB_TYPE_MAX];
```

dpvs_lcore_jobs是二维数组，任务角色和任务类型作为行和列。



## 1、1 角色类型

任务Role角色，有以下：

```
typedef enum dpvs_lcore_role_type {
    LCORE_ROLE_IDLE,//空闲状态：表示线程未分配任何角色，未被分配任何角色
    LCORE_ROLE_MASTER,//主控线程：负责管理整个应用程序的运行，负责配置更新、状态监控、控制面处理等
    LCORE_ROLE_FWD_WORKER,//转发工作线程：负责处理数据包的转发，负责数据包接收、路由查找、转发等
    LCORE_ROLE_ISOLRX_WORKER,//隔离接收工作线程：负责接收隔离数据包，专门负责数据包的接收，与处理逻辑分离，提升接收效率
    LCORE_ROLE_KNI_WORKER,//KNI工作线程：负责处理KNI相关的任务，处理与linux内核的交互
    LCORE_ROLE_MAX//角色类型的最大值，用于边界检查
} dpvs_lcore_role_t;
```

在配置文件中要如何配置上述几种角色呢？

```
worker_defs {
	# 配置为master核心
    <init> worker cpu0 {
        type    master
        cpu_id  0
    }
	# 配置为转发工作核心
    <init> worker cpu1 {
        type    slave
        cpu_id  1
        # 可以配置端口和队列
        port    dpdk0 {
            rx_queue_ids     0
            tx_queue_ids     0
            ! isol_rx_cpu_ids  9
            ! isol_rxq_ring_sz 1048576
        }
        port    dpdk1 {
            rx_queue_ids     0
            tx_queue_ids     0
            ! isol_rx_cpu_ids  9
            ! isol_rxq_ring_sz 1048576
        }
    }

    <init> worker cpu2 {
        type    slave
        cpu_id  2
        port    dpdk0 {
            rx_queue_ids     1
            tx_queue_ids     1
            ! isol_rx_cpu_ids  10
            ! isol_rxq_ring_sz 1048576
        }
        port    dpdk1 {
            rx_queue_ids     1
            tx_queue_ids     1
            ! isol_rx_cpu_ids  10
            ! isol_rxq_ring_sz 1048576
        }
    }
    !<init> worker   cpu17 {
    !    type        kni # 配置为KNI工作核心
    !    cpu_id      17
    !    port        dpdk0 {
    !        rx_queue_ids     8
    !        tx_queue_ids     8
    !    }
    !    port        dpdk1 {
    !        rx_queue_ids     8
    !        tx_queue_ids     8
    !    }
    !}
}
```



## 1、2 任务类型

任务类型，有以下：

```
typedef enum dpvs_lcore_job_type {
    LCORE_JOB_INIT,		//初始化任务
    LCORE_JOB_LOOP,//循环任务：需要每个循环都执行的任务
    LCORE_JOB_SLOW,//慢速任务：需要定期执行但不需要每个循环都频繁执行的任务
    LCORE_JOB_TYPE_MAX//任务类型的最大值，用于边界检查
} dpvs_lcore_job_t;
```

任务是分为两种的一种是每次循环都需要走的任务，比如抓包、解析、合并、



## 1、3 代码剖析

根据源代码来继续梳理下相关的逻辑

在netif.c的config_lcores函数中对于角色的赋值如下：

```
lcore_conf[id].id = worker_min->cpu_id;
        if (!strncmp(worker_min->type, "slave", sizeof("slave")))
            lcore_conf[id].type = LCORE_ROLE_FWD_WORKER;
        else if (!strncmp(worker_min->type, "kni", sizeof("kni")))
            lcore_conf[id].type = LCORE_ROLE_KNI_WORKER;
        else
            lcore_conf[id].type = LCORE_ROLE_IDLE;
```

主要是赋值LCORE_ROLE_FWD_WORKER和LCORE_ROLE_KNI_WORKER这两个角色，



# 二、核心API

初始化任务的属性：dpvs_lcore_job_init

注册任务到任务队列：dpvs_lcore_job_register

从任务队列中删除任务：dpvs_lcore_job_unregister



看看有多少组件调用dpvs_lcore_job_register进行注册

| 函数调用者             | 任务标识           | 角色                     | 类型           | 函数功能                   |
| ---------------------- | ------------------ | ------------------------ | -------------- | -------------------------- |
| cfgfile_init           | cfgfile_reload     | LCORE_ROLE_MASTER        | LCORE_JOB_LOOP | 尝试加载配置文件           |
| msg_lcore_job_register | msg_master_job     | LCORE_ROLE_MASTER        | LCORE_JOB_LOOP | master上消息处理           |
| msg_lcore_job_register | msg_slave_job      | LCORE_ROLE_FWD_WORKER    | LCORE_JOB_LOOP | slave上消息处理            |
| sockopt_init           | sockopt_job        | LCORE_ROLE_MASTER        | LCORE_JOB_LOOP | dpvs的控制面指令发送和接收 |
| icmp_redirect_init     | icmp_redirect_proc | LCORE_ROLE_FWD_WORKER    | LCORE_JOB_LOOP | icmp报文重定向             |
| iftraf_init            | iftraf_ring_proc   | LCORE_ROLE_MASTER        | LCORE_JOB_LOOP |                            |
| ipv4_frag_init         | ipv4_frag          | LCORE_ROLE_FWD_WORKER    | LCORE_JOB_SLOW | ip分片重组                 |
|                        |                    |                          |                |                            |
| arp_init               | neigh_sync         | LCORE_ROLE_FWD_WORKER    | LCORE_JOB_SLOW | 邻居信息同步               |
| arp_init               | neigh_sync         | LCORE_ROLE_MASTER        | LCORE_JOB_LOOP | 邻居信息同步               |
|                        |                    |                          |                |                            |
| netif_lcore_init       | recv_fwd           | LCORE_ROLE_FWD_WORKER    | LCORE_JOB_LOOP | 数据包接收和转发           |
| netif_lcore_init       | xmit               | LCORE_ROLE_FWD_WORKER    | LCORE_JOB_LOOP | 数据包发送                 |
| netif_lcore_init       | timer_manage       | LCORE_ROLE_FWD_WORKER    | LCORE_JOB_LOOP | 定时器管理                 |
| netif_lcore_init       | isol_pkt_rcv       | LCORE_ROLE_ISOLRX_WORKER | LCORE_JOB_LOOP | TODO                       |
| netif_lcore_init       | timer_manage       | LCORE_ROLE_MASTER        | LCORE_JOB_LOOP | 定时器管理                 |
| netif_lcore_init       | kni_master_proc    | LCORE_ROLE_MASTER        | LCORE_JOB_LOOP | 传送kni流量                |
| netif_lcore_init       | kni_loop           | LCORE_ROLE_KNI_WORKER    | LCORE_JOB_LOOP | 传达kni流量                |
|                        |                    |                          |                |                            |
| qsch_init              | qsch_sched         | LCORE_ROLE_FWD_WORKER    | LCORE_JOB_LOOP | 流量管理                   |

因为我们最关心的还是数据面的，收取数据包和发送数据包，所以会针对性的。



# 三、任务调度

```
static int dpvs_job_loop(void *arg)
{
    struct dpvs_lcore_job *job;
    lcoreid_t cid = rte_lcore_id();
    dpvs_lcore_role_t role = g_lcore_role[cid];//如何判断逻辑核心对应的是什么角色呢？
    this_poll_tick = 0;

    /* skip irrelative job loops */
	// 跳过无关的任务循环
    if (role == LCORE_ROLE_MAX)
        return EDPVS_INVAL;
    if (role == LCORE_ROLE_IDLE) //LCORE_ROLE_IDLE类型的任务是用于干什么的
        return EDPVS_IDLE;

    /* do init job */
    list_for_each_entry(job, &dpvs_lcore_jobs[role][LCORE_JOB_INIT], list) {
        do_lcore_job(job);
    }

    while (1) {
        ++this_poll_tick;
		//更新每个逻辑核心的循环次数
        netif_update_worker_loop_cnt();

        /* do normal job */
        list_for_each_entry(job, &dpvs_lcore_jobs[role][LCORE_JOB_LOOP], list) {
            do_lcore_job(job);
        }

        /* do slow job */
        list_for_each_entry(job, &dpvs_lcore_jobs[role][LCORE_JOB_SLOW], list) {
            if (this_poll_tick % job->skip_loops == 0) {
                do_lcore_job(job);
            }
        }
    }

    return EDPVS_OK;
}
```

如何判断逻辑核心对应的是什么角色呢？





lcore_process_packets遍历mbuf数组后调用

netif_deliver_mbuf

netif_rcv_mbuf

​	收到arp应答，将arp应答报文复制后，

rte_pktmbuf_adj是从mbuf的起始位置移除掉len个字节。

rte_pktmbuf_prepend是：预留len字节的长度，



dpvs源代码在这点上，和linux网络内核源代码剖析很像。

数据包是否有需要复制的场景？



# 四、细节剖析

如何使用大页内存，会增加排查问题的风险，所以暂时不使用它，后面添加也不是很麻烦。

这种工作我之前改造过。



sockopt_init这个函数需要好好看一下。对于我们来说，有很多好玩的技术。

对于我们来说，消息通信是如何做的？？？？

master和slave之间的消息通信？

我很关心这个事情。



lcore_process_redirect_ring我要如何