# 一、流管理概述

流管理包括以下几个部分：

- 流初始化
- 流新建、查找、更新
- 流老化
- 流的回收
- 空闲流数量的动态维护

以一个完整的tcp会话为例（包括三次握手和四次挥手），先初始化会话相关的数据结构

1、遇到syn数据包，则新建流（通过五元组计算hash值）并插入到流表中；

2、同一个会话的后续数据包到来时，通过五元组计算hash值，然后查找此会话是否存在；

3、如会话存在，则更新tcp会话状态（状态机）以及各种数据指标；

4、当遇到fin或rst报文时，代表此会话正常结束，此时从会话表中删除flow结构体以及释放对应的资源；

5、当长时间未收到fin或rst报文时，酌情考虑用老化机制来删除会话；

6、空闲流数量的动态维护工作，减少流量大幅变化导致的内存频繁申请和释放问题；



状态机的部分，我会单独开篇文章来表述下suricata是如何实现状态机的。



流管理使用独立的线程，包括老化线程和回收线程，可以启动多个线程，但默认启动一个线程。

 老化线程：main-》SuricataMain-》RunModeDispatch-》FlowManagerThreadSpawn

 回收线程：main-》SuricataMain-》RunModeDispatch-》FlowRecyclerThreadSpawn





# 二、流管理相关配置项

suricata流管理需要使用内存。流越多，所需的内存就越多。为了加强对内存的控制，有几个选项可关注下：

- 设置使用的最大字节数的memcap选项
- 设置哈希表的大小hash_size
- 设置预分配的流的数量级
- 设置紧急模式下空闲流的占比



suricata关于流表的配置，有几个选项是需要格外注意的。

## 1、memcap选项

默认 memcap 为 32M

```
#define FLOW_DEFAULT_MEMCAP      (32 * 1024 * 1024) /* 32 MB */
SC_ATOMIC_SET(flow_config.memcap, FLOW_DEFAULT_MEMCAP);
```

通过`FLOW_CHECK_MEMCAP`来检查内存分配的字节数是否超过了 memcap。

```
#define FLOW_CHECK_MEMCAP(size) \
    ((((uint64_t)SC_ATOMIC_GET(flow_memuse) + (uint64_t)(size)) <= SC_ATOMIC_GET(flow_config.memcap)))
```

flow_memuse当前已使用内存字节数 + size 是否超过flow_config.memcap



## 2、hash_size选项

```
flow_config.hash_size   = FLOW_DEFAULT_HASHSIZE;//预分配的FlowBucket的个数
```

哈希表预分配的FlowBucket的个数



## 3、prealloc选项

设置内存中预分配流的数量。

```
#define FLOW_DEFAULT_PREALLOC    10000
flow_config.prealloc    = FLOW_DEFAULT_PREALLOC;
```



## 4、emergency_recovery 选项

emergency_recovery选项使得流引擎进入紧急模式。在此模式下，引擎将在较短的时间内完成老化动作。

- 紧急恢复。紧急恢复设置为 30。这是预分配流的百分比，在此百分比之后，流引擎将恢复正常（当 10000 个流中的 30％完成时）。





# 三、流表初始化

## 3、1 flow_hash 全局流表初始化

全局流表的初始化在FlowInitConfig函数中实现。

```c
/** \brief initialize the configuration
 *  \warning Not thread safe */
void FlowInitConfig(char quiet)
{
    ......
    //省略读取配置信息的代码
    ......

    //计算flow哈希表的待分配的bucket数量
    uint64_t hash_size = flow_config.hash_size * sizeof(FlowBucket);
    //分配指定数量的FlowBucket，CLS是cache line size，默认值为64
    flow_hash = SCMallocAligned(flow_config.hash_size * sizeof(FlowBucket), CLS);
    if (unlikely(flow_hash == NULL)) {
        FatalError(SC_ERR_FATAL,
                   "Fatal error encountered in FlowInitConfig. Exiting...");
    }
    memset(flow_hash, 0, flow_config.hash_size * sizeof(FlowBucket));

    uint32_t i = 0;
    //初始化flow hash数组，新建流的flow存放到这个数组的每个bucket链表里
    for (i = 0; i < flow_config.hash_size; i++) {
        FBLOCK_INIT(&flow_hash[i]);
        SC_ATOMIC_INIT(flow_hash[i].next_ts);
    }
    
	//初始化流的全局内存池
    FlowSparePoolInit();

    //初始化tcp、udp、icmp等各协议的超时时间
    FlowInitFlowProto();
    return;
}
```

其步骤如下所示：

**1、先计算flow hash数组中待分配的bucket数量；**

**2、分配指定数量的FlowBucket；**

**3、初始化flow hash数组，新建流的flow存放到这个数组的每个bucket的链表里；**

**4、初始化流的全局内存池（后面分析）；**

**5、初始化tcp、udp、icmp等各协议的超时时间（后面分析）；**



## 3、2 初始化全局flow内存池

**每个工作线程新建流时，从自己的flow队列中获取flow流，当线程所属的flow队列中无flow可用时，就从全局flow内存池中获取一个flow队列使用。**

### 3、2、1 FlowSparePoolInit函数

FlowSparePoolInit函数如下所示：

```
//初始化全局内存池
void FlowSparePoolInit(void)
{
    SCMutexLock(&flow_spare_pool_m);//加锁
    //预先申请内存数
    for (uint32_t cnt = 0; cnt < flow_config.prealloc; ) {
        FlowSparePool *p = FlowSpareGetPool();//简单malloc一个FlowSparePool
        FlowSparePoolUpdateBlock(p);//重要函数！！！
        cnt += p->queue.len;

        /* prepend to list */
        p->next = flow_spare_pool;//FlowSparePool链接成链表
        flow_spare_pool = p;
        flow_spare_pool_flow_cnt = cnt;//更新总的flow数量
    }
    SCMutexUnlock(&flow_spare_pool_m);//释放锁
}
```

FlowSparePoolInit这个函数强烈建议大家debug单步调试下，受益匪浅。



FlowSpareGetPool函数就是简单的malloc申请一个FlowSparePool的内存空间。

```
static FlowSparePool *FlowSpareGetPool(void)
{
    FlowSparePool *p = SCCalloc(1, sizeof(*p));
    if (p == NULL)
        return NULL;
    return p;
}
```



FlowSparePool结构体的定义如下：

```
typedef struct FlowSparePool {
    FlowQueuePrivate queue;//flow队列，由100个flow组成
    struct FlowSparePool *next;//next指针，用于将多个flow队列组成链表
} FlowSparePool;
```



### 3、2、2 FlowSparePoolUpdateBlock函数

FlowSparePoolInit函数中调用了FlowSparePoolUpdateBlock函数。

**1、for循环i的范围从p->queue.len变化到flow_spare_pool_block_size**

```
static bool FlowSparePoolUpdateBlock(FlowSparePool *p)
{
	//flow_spare_pool_block_size默认值为100
	//p->queue.len的值一直小于100，假如原先队列里有60个flow，那只需再添加40个flow
    for (uint32_t i = p->queue.len; i < flow_spare_pool_block_size; i++)
    {
        Flow *f = FlowAlloc();//申请flow结构体，分配flow
        if (f == NULL)
            return false;
		//将Flow结构体实例添加到FlowSparePool的FlowQueuePrivate中
        FlowQueuePrivateAppendFlow(&p->queue, f);
    }
    return true;
}
```

for循环i的范围从p->queue.len变化到flow_spare_pool_block_size（默认值为100），假如原先队列里有60个flow，那只需再添加40个flow。

缺少的40个flow是调用FlowQueuePrivateAppendFlow函数来添加的。



**2、先调用FlowAlloc函数申请Flow结构体**

```
Flow *FlowAlloc(void)
{
    Flow *f;
    size_t size = sizeof(Flow) + FlowStorageSize();//计算申请的内存大小

    f = SCMalloc(size);
    memset(f, 0, size);
    return f;
}
```



**3、最后FlowQueuePrivateAppendFlow函数，将Flow结构体实例添加到全局内存池的flow queue中。**

```
void FlowQueuePrivateAppendFlow(FlowQueuePrivate *fqc, Flow *f)
{
    if (fqc->top == NULL) {
    	//队列无flow，将flow放置到队列头部，并更新个数为1
        fqc->top = fqc->bot = f;
        fqc->len = 1;
    } else {
    	//队列已有flow，则将flow放置到队列尾部，并移动队列尾部指针
        fqc->bot->next = f;
        fqc->bot = f;
        fqc->len++;
    }
    f->next = NULL;
}
```



## 3、3 初始化各种协议的超时时间

```
void FlowInitFlowProto(void)
{
    FlowTimeoutsInit();

#define SET_DEFAULTS(p, n, e, c, b, ne, ee, ce, be)     \
    flow_timeouts_normal[(p)].new_timeout = (n);     \
    flow_timeouts_normal[(p)].est_timeout = (e);     \
    flow_timeouts_normal[(p)].closed_timeout = (c);  \
    flow_timeouts_normal[(p)].bypassed_timeout = (b); \
    flow_timeouts_emerg[(p)].new_timeout = (ne);     \
    flow_timeouts_emerg[(p)].est_timeout = (ee);     \
    flow_timeouts_emerg[(p)].closed_timeout = (ce); \
    flow_timeouts_emerg[(p)].bypassed_timeout = (be); \

	//设置默认协议的超时时间
    SET_DEFAULTS(FLOW_PROTO_DEFAULT,FLOW_DEFAULT_NEW_TIMEOUT, FLOW_DEFAULT_EST_TIMEOUT,
                    0, FLOW_DEFAULT_BYPASSED_TIMEOUT,FLOW_DEFAULT_EMERG_NEW_TIMEOUT, FLOW_DEFAULT_EMERG_EST_TIMEOUT,
                    0, FLOW_DEFAULT_EMERG_BYPASSED_TIMEOUT);
    //设置TCP协议的超时时间
    SET_DEFAULTS(FLOW_PROTO_TCP,FLOW_IPPROTO_TCP_NEW_TIMEOUT, FLOW_IPPROTO_TCP_EST_TIMEOUT,
                    FLOW_IPPROTO_TCP_CLOSED_TIMEOUT, FLOW_IPPROTO_TCP_BYPASSED_TIMEOUT,
                FLOW_IPPROTO_TCP_EMERG_NEW_TIMEOUT, FLOW_IPPROTO_TCP_EMERG_EST_TIMEOUT,
                    FLOW_IPPROTO_TCP_EMERG_CLOSED_TIMEOUT, FLOW_DEFAULT_EMERG_BYPASSED_TIMEOUT);
    //设置UDP协议的超时时间
    SET_DEFAULTS(FLOW_PROTO_UDP,FLOW_IPPROTO_UDP_NEW_TIMEOUT, FLOW_IPPROTO_UDP_EST_TIMEOUT,
                    0, FLOW_IPPROTO_UDP_BYPASSED_TIMEOUT,FLOW_IPPROTO_UDP_EMERG_NEW_TIMEOUT, FLOW_IPPROTO_UDP_EMERG_EST_TIMEOUT,
                    0, FLOW_DEFAULT_EMERG_BYPASSED_TIMEOUT);
    //设置ICMP协议的超时时间
    SET_DEFAULTS(FLOW_PROTO_ICMP, FLOW_IPPROTO_ICMP_NEW_TIMEOUT, FLOW_IPPROTO_ICMP_EST_TIMEOUT,
                    0, FLOW_IPPROTO_ICMP_BYPASSED_TIMEOUT,FLOW_IPPROTO_ICMP_EMERG_NEW_TIMEOUT, 	
                    FLOW_IPPROTO_ICMP_EMERG_EST_TIMEOUT,0, FLOW_DEFAULT_EMERG_BYPASSED_TIMEOUT);
}
```



suricata相关的测试用例，是如何进行编写的？这块还需要好好看下。
