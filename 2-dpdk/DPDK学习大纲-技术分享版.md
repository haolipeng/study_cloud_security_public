学习计划安排

| 内容      | 时间       | 是否完成   |
| --------- | ---------- | ---------- |
| Hash库    | 2025-07-10 | **未完成** |
| Reorder库 | 2025-07-12 | **未完成** |
|           |            |            |



# 零、环境搭建和初体验

## 1、dpdk源代码的编译

需要安装一些依赖库，具体要安装的库在这里。

https://doc.dpdk.org/guides/linux_gsg/sys_reqs.html#compilation-of-the-dpdk



## 2、helloworld程序的解读(或手写)

需要大家熟悉下以下的api函数：

rte_eal_init

RTE_LCORE_FOREACH_WORKER

rte_eal_remote_launch

https://doc.dpdk.org/api/rte__launch_8h.html#a2bf98eda211728b3dc69aa7694758c6d

rte_eal_mp_wait_lcore（和rte_eal_remote_launch函数的关联是什么）

rte_eal_cleanup



# 一、内存管理

**1、Lcore变量**

**2、Memory Pool内存池库**

**3、Mbuf库**

**4、多进程支持**



# 二、CPU Packet Processing数据包处理

**1、Toeplitz Hash库**

https://doc.dpdk.org/guides/prog_guide/toeplitz_hash_lib.html



**2、Hash库**

https://doc.dpdk.org/guides/prog_guide/hash_lib.html



**3、IP分片和重组库**

https://doc.dpdk.org/guides/prog_guide/ip_fragment_reassembly_lib.html



**4、Packet Classification And Access Control(ACL)库**

https://doc.dpdk.org/guides/prog_guide/packet_classif_access_ctrl.html



**5、Packet 转发库**

https://doc.dpdk.org/guides/prog_guide/packet_distrib_lib.html



**6、Reorder库**

https://doc.dpdk.org/guides/prog_guide/reorder_lib.html



# 三、高级库

**1、Packet Framework Library**

https://doc.dpdk.org/guides/prog_guide/packet_framework.html



**2、Graph Library and Inbuilt Nodes**

https://doc.dpdk.org/guides/prog_guide/graph_lib.html



## 四、组件库

**1、参数解析库**

**2、命令行库**

**3、指针压缩库**

**4、定时器库**

**5、RCU库**

**6、Ring库**

**7、Stack库**

**8、日志库**

**9、Metrics库**

**10、Telemetry Library**

**11、Packet Capture Library**

**12、Packet Capture Next Generation Library**

**13、BPF库**

**14、Trace库**