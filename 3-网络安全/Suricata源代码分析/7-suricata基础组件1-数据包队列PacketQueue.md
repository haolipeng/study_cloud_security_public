# 一、基本概念

## 1、1 为什么需要PacketQueue？

Suricata中使用队列来缓存数据包，包括缓存线程模块内部新产生数据包的线程内队列，以及线程之间用来传递数据包的线程间队列。

线程间通信的要求。



## 1、2 数据结构分析

用于表示数据包队列的结构体为PacketQueue，其定义如下（省略了调试相关字段）

```
typedef struct PacketQueue_ {
    Packet *top;
    Packet *bot;
    uint32_t len;
    SCMutex mutex_q;
    SCCondT cond_q;
} PacketQueue;
```

为什么需要mutex_q互斥锁和conf_q条件变量呢？他们是起到什么作用呢？这块涉及的东西比较基础。



## 1、3 常用操作API

PacketQueue组件有哪些API可使用呢？

### 队列分配

PacketQueueAlloc函数



### 入队操作 (Enqueue)

PacketEnqueue() 和 PacketEnqueueNoLock() 函数



### 出队操作 (Dequeue)

PacketDequeue() 和 PacketDequeueNoLock() 函数



### 队列释放

PacketQueueFree



# 二、Packet生成路径

Hander主要是处理什么逻辑？





# 五、源代码分析

申请数据的size大小要设置为多大呢？







