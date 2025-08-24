一、简介

Suricata中使用队列来缓存数据包，包括缓存线程模块内部新产生数据包的线程内队列，以及线程之间用来传递数据包的线程间队列。

用于表示数据包队列的结构体为PacketQueue，其定义如下（省略了调试相关字段）

```
typedef struct PacketQueue_ {
    Packet *top;
    Packet *bot;
    uint32_t len;
#ifdef DBG_PERF
    uint32_t dbg_maxlen;
#endif /* DBG_PERF */
    SCMutex mutex_q;
    SCCondT cond_q;
} PacketQueue;
```



二、线程内队列

线程内队列与线程内的每个slot相绑定，其又可细分为pre队列和post队列，定义如下：



三、线程间队列



四、队列Handler

Hander主要是处理什么逻辑？



五、意见和建议

可考虑采用无锁队列来替换



参考链接

https://blog.csdn.net/superbfly/article/details/52677730