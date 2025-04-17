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
    LCORE_ROLE_IDLE,
    LCORE_ROLE_MASTER,
    LCORE_ROLE_FWD_WORKER,
    LCORE_ROLE_ISOLRX_WORKER,
    LCORE_ROLE_KNI_WORKER,
    LCORE_ROLE_MAX
} dpvs_lcore_role_t;
```



## 1、2 任务类型

任务类型，有以下：

```
typedef enum dpvs_lcore_job_type {
    LCORE_JOB_INIT,		//初始化任务
    LCORE_JOB_LOOP,		//循环任务
    LCORE_JOB_SLOW,		//慢速任务
    LCORE_JOB_TYPE_MAX	//标志位
} dpvs_lcore_job_t;
```



# 二、核心API

初始化任务的属性：dpvs_lcore_job_init

注册任务到任务队列：dpvs_lcore_job_register

从任务队列中销毁任务：dpvs_lcore_job_unregister



看看有多少地方调用dpvs_lcore_job_register

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
    dpvs_lcore_role_t role = g_lcore_role[cid];
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



lcore_process_packets遍历mbuf数组后调用

netif_deliver_mbuf

netif_rcv_mbuf

​	收到arp应答，将arp应答报文复制后，

rte_pktmbuf_adj是从mbuf的起始位置移除掉len个字节。

rte_pktmbuf_prepend是：预留len字节的长度，



dpvs源代码在这点上，和linux网络内核源代码剖析很像。

ipv4_rcv

# 四、细节剖析

sockopt_init这个函数需要好好看一下。对于我们来说，有很多好玩的技术。

对于我们来说，消息通信是如何做的？？？？

master和slave之间的消息通信？

我很关心这个事情。