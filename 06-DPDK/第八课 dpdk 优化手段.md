**优先处理的优化点：**

1、接收数据包后，将数据包的结构从mbuf转为packet；

2、packet内存池技术

3、状态机编写

4、会话表的结束机制

5、会话表的老化机制



**后续处理的优化点：**

https://doc.dpdk.org/api/rte__per__lcore_8h.html

业务处理流程都是per-core的，拥有各自的per-core定时器来用于周期性执行定时任务。



tcp状态机的编码和状态转换。