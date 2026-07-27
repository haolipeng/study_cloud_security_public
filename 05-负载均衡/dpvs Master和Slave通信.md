# 零、个人思考

master和slave核心之间通信  模块设计，最好是这块代码扣出来进行测试。

如果我设计这块，我会如何设计？

1、消息传递的方式：单播 VS 组播

组播仅限于master发送给多个slave使用

2、消息来自哪里？消息发往哪里？这里区分master和slave核心

3、出于高性能考虑，每个逻辑核心有独立的消息队列ring

4、消息发送：同步 VS 异步

消息为异步类型时，需要一个单独的消息队列进行缓冲。

5、消息发送时超时（触发条件和判断）

6、消息发送时丢包（触发条件和判断）

7、为什么会加上序列号这个属性，序列号只有master会调用

8、消息的优先级

9、死锁的处理

想不明白，什么情况下，会出现死锁呢？以及如何处理这种情况呢？

10、消息发送完成，（触发条件是什么）

11、消息的内存如何销毁

12、是否考虑了numa架构



本博客解决以下几个问题：

1.单播消息和多播消息的注册（初始化）和使用

2.注册单播消息和多播消息的区别？

3.何时注册单播消息？何时注册多播消息？

# 一、核心结构体

## 1、1 消息结构体 dpvs_msg_type

在阅读dpvs源代码的过程中，发现很多模块调用msg_type_mc_register函数或msg_type_register函数来注册dpvs_msg_type结构体，结构体定义如下：

```c++
struct dpvs_msg_type {
    msgid_t type;	//消息类型
    uint8_t prio;	//消息的优先级,参考枚举类型msg_priority
    lcoreid_t cid;          //回调函数注册在哪个cpu逻辑核心上
    msg_mode_t mode;        //消息模式，分为单播和多播
    UNICAST_MSG_CB unicast_msg_cb;     //单播的回调函数
    MULTICAST_MSG_CB multicast_msg_cb; //组播的回调函数
    rte_atomic32_t refcnt;	//引用计数
    struct list_head list;	//链表的头指针
};
```

枚举类型msg_priority定义如下：

```
//消息的优先级
typedef enum msg_priority {
    MSG_PRIO_IGN = 0, /* 内部使用 */
    MSG_PRIO_HIGH,    /* 关键例程，比如master数据传输 */
    MSG_PRIO_NORM,    /* 通常用于 SET 操作 */
    MSG_PRIO_LOW      /* 通常用于 GET 操作 */
} msg_priority_t;
```



上述回调函数的函数原型如下所示：

```
/* All msg callbacks are called on the lcore which it registers */
typedef int (*UNICAST_MSG_CB)(struct dpvs_msg *);
typedef int (*MULTICAST_MSG_CB)(struct dpvs_multicast_queue *);
```



## 1、2 消息注册类型

消息存储在何处？

在调用msg_type_mc_register函数或msg_type_register函数后，里面有一句代码：

```
list_add_tail(&mt->list, &mt_array[msg_type->cid][hashkey]);
```

将消息添加到二维数组mt_array中（或者称为哈希表？），我们来看下mt_array的定义：

```c
/* per-lcore msg-type array */
typedef struct list_head msg_type_array_t[DPVS_MT_LEN];

msg_type_array_t mt_array[DPVS_MAX_LCORE];
```

合并到一起

struct list_head mt_array[DPVS_MAX_LCORE] [DPVS_MT_LEN];

第一层是lcore逻辑核心，第二层是消息类型。



对外提供msg_type_get(type,cid)来判断消息类型是否已注册，未注册则报错。

## 1、3 消息队列（存放实际消息）

```
/* per-lcore msg queue */
struct rte_ring *msg_ring[DPVS_MAX_LCORE];//dpdk的无锁队列
```

每个核心有独立的消息队列（高性能，无锁设计）

msg_send函数会向msg_ring中入队；

msg_master_process和msg_slave_process函数会将数据从msg_ring队列中出队。



## 1、4  消息的状态

```
//同步 or 异步 消息
#define DPVS_MSG_F_ASYNC            1  
/* msg has been sent from sender */
#define DPVS_MSG_F_STATE_SEND       2
/* for multicast msg only, msg arrived at Master and enqueued, waiting for all other Slaves reply */
#define DPVS_MSG_F_STATE_QUEUE      4
/* msg has dequeued from ring */
#define DPVS_MSG_F_STATE_RECV       8
/* msg finished, all Slaves replied if multicast msg */
#define DPVS_MSG_F_STATE_FIN        16
/* msg drop, callback not called, for reason such as unregister, timeout ... */
#define DPVS_MSG_F_STATE_DROP       32
/* msg callback failed */
#define DPVS_MSG_F_CALLBACK_FAIL    64
/* msg timeout */
#define DPVS_MSG_F_TIMEOUT          128
```

以上几种消息的状态是很重要的。起码能分清楚，

什么时候会msg timeout？

什么时候会msg callback failed？

什么时候会DPVS_MSG_F_STATE_QUEUE？

消息的同步和异步是如何实现的？

带着这些疑问，我们接着往下阅读源代码。



## 1、5 多播场景下的消息队列

```
/* Master->Slave multicast msg queue */
struct dpvs_multicast_queue {
    msgid_t type;           /* msg type */
    uint32_t seq;           /* msg sequence number */
    //uint16_t ttl;           /* time to live */
    uint64_t mask;          /* bit-wise core mask */
    struct list_head mq;    /* recieved msg queue 接收消息队列*/
    struct dpvs_msg *org_msg; /* original msg from 'multicast_msg_send', sender should never visit me */
    struct list_head list;
};
```

dpvs_multicast_queue存储在什么数据结构中？

```
/* multicast msg hlist, to collect reply msg from slaves (used on master lcore only) */
#define DPVS_MC_HLIST_BITS 8
#define DPVS_MC_HLIST_LEN (1 << DPVS_MC_HLIST_BITS)
#define DPVS_MC_HLIST_MASK (DPVS_MC_HLIST_LEN - 1)
struct list_head mc_wait_hlist[DPVS_MC_HLIST_LEN];
```

多播消息链表，用于收集来自slave的响应消息。



dpvs_multicast_queue的创建

```
int multicast_msg_send(struct dpvs_msg *msg, uint32_t flags, struct dpvs_multicast_queue **reply)
{
    struct dpvs_msg *new_msg;
    struct dpvs_multicast_queue *mcq;

    mcq = dpvs_mempool_get(msg_pool, sizeof(struct dpvs_multicast_queue));
    if (unlikely(!mcq)) {
        //从msg_pool内存池中申请
        return EDPVS_NOMEM;
    }

    mcq->type = msg->type;
    mcq->seq = msg->seq;
    mcq->mask = slave_lcore_mask;
    mcq->org_msg = msg; /* save original msg */
    INIT_LIST_HEAD(&mcq->mq);

    /* hash mcq so that reply msg can be collected in msg_master_process */
    mc_queue_hash(mcq);
}
```





# 二、单播消息和多播消息的注册和使用

以netif.c的函数为例

```
static inline int lcore_stats_msg_init(void)
{
    int ii, err;
    //初始化msg消息
    struct dpvs_msg_type lcore_stats_msg_type = {
        .type = MSG_TYPE_NETIF_LCORE_STATS,
        .mode = DPVS_MSG_UNICAST,
        .unicast_msg_cb = lcore_stats_msg_cb,
        .multicast_msg_cb = NULL,
    };

    for (ii = 0; ii < DPVS_MAX_LCORE; ii++) {
        if ((ii == g_master_lcore_id) || (g_slave_lcore_mask & (1L << ii))) {
            lcore_stats_msg_type.cid = ii;
            err = msg_type_register(&lcore_stats_msg_type);
        }
    }

    return EDPVS_OK;
}
```

使用方法：

1、定义dpvs_msg_type结构体，并赋值初始化

2、调用msg_type_register注册即可

是不是很简单？

# 三、单播消息和多播消息注册函数的源代码分析

其涉及到的主要函数有以下4个：

```
/* register|unregister msg-type on lcore 'msg_type->cid'  */
int msg_type_register(const struct dpvs_msg_type *msg_type);
int msg_type_unregister(const struct dpvs_msg_type *msg_type);

/* register|unregister multicast msg-type on each configured lcore */
int msg_type_mc_register(const struct dpvs_msg_type *msg_type);
int msg_type_mc_unregister(const struct dpvs_msg_type *msg_type);
```

##  3.1 单播注册函数

```
int msg_type_register(const struct dpvs_msg_type *msg_type)
{
    int hashkey;
    struct dpvs_msg_type *mt;

	//通过type值计算哈希值
    hashkey = mt_hashkey(msg_type->type);
	
	//通过msg type和cid判断消息是否已注册，未注册则报错
    mt = msg_type_get(msg_type->type, /*msg_type->mode, */msg_type->cid);
    ......

	//申请内存空间
    mt = rte_zmalloc("msg_type", sizeof(struct dpvs_msg_type), RTE_CACHE_LINE_SIZE);
	......

	//将msg_type拷贝到内存中，并将refcnt引用计数置为0
    memcpy(mt, msg_type, sizeof(struct dpvs_msg_type));
    rte_atomic32_set(&mt->refcnt, 0);

	//使用写锁，将mt元素添加到mt_array二维数组（哈希表）的尾部
    rte_rwlock_write_lock(&mt_lock[msg_type->cid][hashkey]);
    list_add_tail(&mt->list, &mt_array[msg_type->cid][hashkey]);
    rte_rwlock_write_unlock(&mt_lock[msg_type->cid][hashkey]);

    return EDPVS_OK;
}
```

在核心结构体章节已介绍，mt_array是个二维数组：

第一个维度是lcore id

第二个维度是通过消息类型计算出来的hashkey值

## 3.2 多播注册函数

```
int msg_type_mc_register(const struct dpvs_msg_type *msg_type)
{
    lcoreid_t cid;
    struct dpvs_msg_type mt;
    int ret = EDPVS_OK;


    memset(&mt, 0, sizeof(mt));
    mt.type = msg_type->type;
    mt.mode = DPVS_MSG_MULTICAST;//消息模式为多播

	//遍历当前所有lcore id
    for (cid = 0; cid < DPVS_MAX_LCORE; cid++) {
		//master核心上注册multicast_msg_cb多播回调
        if (cid == master_lcore) {
            mt.cid = cid;
            mt.unicast_msg_cb = NULL;
            if (msg_type->multicast_msg_cb)
                mt.multicast_msg_cb = msg_type->multicast_msg_cb;//注册函数
            else
                mt.multicast_msg_cb = default_mc_msg_cb;//调用默认的多播回调
        } 
		else if (slave_lcore_mask & (1L << cid)) {
			//slave核心上注册unicast_msg_cb单播回调函数
            mt.cid = cid;
            mt.unicast_msg_cb = msg_type->unicast_msg_cb;//注册函数
            mt.multicast_msg_cb = NULL;
        } else
            continue;

		//！！！！同样也是调用msg_type_register函数
        ret = msg_type_register(&mt);
    }

    return EDPVS_OK;
}
```

多播的注册函数，本质上就是遍历所有的cpu lcore 逻辑核心，

master核心上注册多播回调函数，

slave 核心注册单播回调函数，

不管是master核心还是slave核心，最终都会调用msg_type_register函数。

# 四、单播和组播回调函数的调用

上文提及到，单播和组播的回调函数注册微msg_type->unicast_msg_cb和msg_type->multicast_msg_cb，下面我们看下是在什么地方进行调用的？

**unicast_msg_cb调用位置：**

- msg_master_process
- msg_slave_process



**multicast_msg_cb调用位置：**

- msg_send
- multicast_msg_send
- master_lcore_loop_func

所以重点看这几个函数即可

## 4、1 msg_master_process函数调用

主要就是在main函数的控制面线程中被调用

```
static int msg_master_process(int step)
{
    int n = 0;
    struct dpvs_msg *msg;
    struct dpvs_msg_type *msg_type;
    struct dpvs_multicast_queue *mcq;

    /* dequeue msg from ring on the master lcore and process it */
    while (((step <= 0) || ((step > 0) && (++n <= step))) &&
            (0 == rte_ring_dequeue(msg_ring[master_lcore], (void **)&msg))) {
        //从master lcore对应的无锁队列上取消息,存储到msg中
        add_msg_flags(msg, DPVS_MSG_F_STATE_RECV);//master已接收到消息
        msg_type = msg_type_get(msg->type, master_lcore);
        if (!msg_type) {//消息未注册，跳过，继续循环
            continue;
        }
        //master可能收到单播或组播消息
        if (DPVS_MSG_UNICAST == msg_type->mode) { /* 单播消息 */
            if (likely(msg_type->unicast_msg_cb != NULL)) {
                if (msg_type->unicast_msg_cb(msg) < 0) {//回调函数调用
                	//设置标志：回调函数调用失败
                    add_msg_flags(msg, DPVS_MSG_F_CALLBACK_FAIL);
                }
            }
            add_msg_flags(msg, DPVS_MSG_F_STATE_FIN);
            msg_destroy(&msg);
        } else { /* 组播消息 */
            mcq = mc_queue_get(msg->type, msg->seq);//为什么mcq会获取不到呢
            if (!mcq) {
                /* probably previous msg timeout */
                add_msg_flags(msg, DPVS_MSG_F_STATE_DROP);
                msg_destroy(&msg);
                msg_type_put(msg_type);//引用技术+1
                continue;
            }
            assert(msg_type->multicast_msg_cb != NULL);
            if (mcq->mask & (1UL << msg->cid)) { //是正在等待的消息
                list_add_tail(&msg->mq_node, &mcq->mq);
                /* set QUEUE flag for slave's reply msg */
                add_msg_flags(msg, DPVS_MSG_F_STATE_QUEUE);
                mcq->mask &= ~(1UL << msg->cid);//从mask掩码中去除cid的标记
                
                if (test_msg_flags(msg, DPVS_MSG_F_CALLBACK_FAIL)) 
                	/* callback on slave failed */
                    add_msg_flags(mcq->org_msg, DPVS_MSG_F_CALLBACK_FAIL);

                if (unlikely(0 == mcq->mask)) { /* okay, all slave reply msg arrived */
                    if (msg_type->multicast_msg_cb(mcq) < 0) {//回调函数调用
                    	/* callback on master failed */
                        add_msg_flags(mcq->org_msg, DPVS_MSG_F_CALLBACK_FAIL);
                    }
                    add_msg_flags(mcq->org_msg, DPVS_MSG_F_STATE_FIN);
                    msg_destroy(&mcq->org_msg);
                }
                msg_type_put(msg_type);
                continue;
            }
            /* probably previous msg timeout and new msg of this type sent */
            assert(msg->mode == DPVS_MSG_UNICAST);
            add_msg_flags(msg, DPVS_MSG_F_STATE_DROP);
            msg_destroy(&msg); /* sorry, you are late */
        }//end 组播消息
        msg_type_put(msg_type);
    }//end while
    return EDPVS_OK;
}
```



## 4.2 msg_slave_process中单播消息的处理

```
/* only unicast msg can be recieved on slave lcore */
int msg_slave_process(void)
{
    struct dpvs_msg *msg, *xmsg;
    struct dpvs_msg_type *msg_type;
    lcoreid_t cid;
    int ret = EDPVS_OK;

	//获取cpu核心id
    cid = rte_lcore_id();

    /* dequeue msg from ring on the lcore until drain */
    while (0 == rte_ring_dequeue(msg_ring[cid], (void **)&msg)) {
        add_msg_flags(msg, DPVS_MSG_F_STATE_RECV);
        if (unlikely(DPVS_MSG_MULTICAST == msg->mode)) {
        	//slave 核心上接收到multicast组播消息是错误的
            RTE_LOG(ERR, MSGMGR, "%s: multicast msg recieved on slave lcore!\n", __func__);
            continue;
        }

        msg_type = msg_type_get(msg->type, cid);
        if (!msg_type) {
            //消息未注册
            continue;
        }

		//调用unicast_msg_cb函数，函数会对msg实参进行赋值操作
        if (msg_type->unicast_msg_cb) {
            if (msg_type->unicast_msg_cb(msg) < 0) {//回调调用失败
                add_msg_flags(msg, DPVS_MSG_F_CALLBACK_FAIL);
            }
        }
        /* send response msg to Master for multicast msg */
        if (DPVS_MSG_MULTICAST == msg_type->mode) {
			//发送响应给master lcore
            xmsg = msg_make(msg->type, msg->seq, DPVS_MSG_UNICAST, cid, msg->reply.len,
                    msg->reply.data);
            if (unlikely(!xmsg)) {
                ret = EDPVS_NOMEM;
                add_msg_flags(msg, DPVS_MSG_F_STATE_DROP);
                goto cont;
            }
            add_msg_flags(xmsg, DPVS_MSG_F_CALLBACK_FAIL & get_msg_flags(msg));
			//以异步方式将消息发送到master lcore上
            if (msg_send(xmsg, master_lcore, DPVS_MSG_F_ASYNC, NULL)) {
                add_msg_flags(msg, DPVS_MSG_F_STATE_DROP);
                msg_destroy(&xmsg);
                goto cont;
            }
            msg_destroy(&xmsg);
        }

		//设置完成标志
        add_msg_flags(msg, DPVS_MSG_F_STATE_FIN);
cont:
        msg_destroy(&msg);
        msg_type_put(msg_type);
    }

    return ret;
}
```

## 4.3 msg_send单播消息发送函数

```
/* "msg" must be produced by "msg_make" */
int msg_send(struct dpvs_msg *msg, lcoreid_t cid, uint32_t flags, struct dpvs_msg_reply **reply)
{
    struct dpvs_msg_type *mt;
    int res;
    uint32_t tflags;
    uint64_t start, delay;

    add_msg_flags(msg, flags);

    if (unlikely(!msg || !((cid == master_lcore) || (slave_lcore_mask & (1L << cid))))) {
        RTE_LOG(WARNING, MSGMGR, "%s: invalid args\n", __func__);
        add_msg_flags(msg, DPVS_MSG_F_STATE_DROP);
        return EDPVS_INVAL;
    }

	//从mt_array二维数组中获取dpvs_msg_type元素
    mt = msg_type_get(msg->type, cid);
    if (unlikely(!mt)) {
        RTE_LOG(WARNING, MSGMGR, "%s: msg type %d not registered\n", __func__, msg->type);
        add_msg_flags(msg, DPVS_MSG_F_STATE_DROP);
        return EDPVS_NOTEXIST;
    }
	//引用计数减少1
    msg_type_put(mt);

    /* two lcores will be using the msg now, increase its refcnt */
    rte_atomic16_inc(&msg->refcnt);
	
	//投递到master对应的队列msg_ring[master_lcore]中
    res = rte_ring_enqueue(msg_ring[cid], msg);
    if (unlikely(-EDQUOT == res)) {
        RTE_LOG(WARNING, MSGMGR, "%s: msg ring of lcore %d quota exceeded\n",
                __func__, cid);
    } else if (unlikely(-ENOBUFS == res)) {
        RTE_LOG(ERR, MSGMGR, "%s: msg ring of lcore %d is full\n", __func__, res);
        add_msg_flags(msg, DPVS_MSG_F_STATE_DROP);
        rte_atomic16_dec(&msg->refcnt); /* not enqueued, free manually */
        return EDPVS_DPDKAPIFAIL;
    } else if (res) {
        RTE_LOG(ERR, MSGMGR, "%s: unkown error %d for rte_ring_enqueue\n", __func__, res);
        add_msg_flags(msg, DPVS_MSG_F_STATE_DROP);
        rte_atomic16_dec(&msg->refcnt); /* not enqueued, free manually */
        return EDPVS_DPDKAPIFAIL;
    }

	//异步消息，投递到队列中即可返回
    if (flags & DPVS_MSG_F_ASYNC)
        return EDPVS_OK;

    /* blockable msg, wait here until done or timeout */
	/*同步发送，一直等待消息发送出去或者超时*/
    add_msg_flags(msg, DPVS_MSG_F_STATE_SEND);//已发送标志
    start = rte_get_timer_cycles();//当前cpu时钟周期
   //检测msg的标志是否是DPVS_MSG_F_STATE_FIN或DPVS_MSG_F_STATE_DROP，或者等待超时退出while循环
    while(!(test_msg_flags(msg, (DPVS_MSG_F_STATE_FIN | DPVS_MSG_F_STATE_DROP)))) {
        /* to avoid dead lock when one send a blockable msg to itself */
        if (rte_lcore_id() == master_lcore)
            msg_master_process();
        else
            msg_slave_process();
        delay = (uint64_t)msg_timeout * rte_get_timer_hz() / 1E6;
		//大于时间间隔delay，已超时，将msg标志改为DPVS_MSG_F_TIMEOUT
        if (start + delay < rte_get_timer_cycles()) {
            RTE_LOG(WARNING, MSGMGR, "%s: uc_msg(type:%d, cid:%d->%d, flags=%d) timeout"
                    "(%d us), drop...\n", __func__,
                    msg->type, msg->cid, cid, get_msg_flags(msg), msg_timeout);
            add_msg_flags(msg, DPVS_MSG_F_TIMEOUT);
            return EDPVS_MSG_DROP;
        }
    }
    if (reply)
        *reply = &msg->reply;

    tflags = get_msg_flags(msg);
    if (tflags & DPVS_MSG_F_CALLBACK_FAIL)
        return EDPVS_MSG_FAIL;
    else if (tflags & DPVS_MSG_F_STATE_FIN)
        return EDPVS_OK;
    else
        return EDPVS_MSG_DROP;
}
```

 1） 从mt_array二维数组中获取dpvs_msg_type元素

 2） 投递到master对应的队列msg_ring[master_lcore]中

3） 如flags是DPVS_MSG_F_ASYNC，表明是异步消息，投递到队列中即可函数返回

4） 同步发送，分为循环等待消息发送完成或消息丢弃 and 同步发送超时。



## 4.4 multicast_msg_send多播消息发送函数

```

```

1）判断msg在mc_wait_list等待链表中是否已经存在（下文中mc_wait_list都统称为等待队列）

2） 从master向所有的slave核心，异步方式调用msg_send(new_msg, ii, DPVS_MSG_F_ASYNC, NULL);发送单播消息

3） 申请dpvs_multicast_queue结构体类型的资源，并对其进行赋值。

dpvs_multicast_queue是什么东西？什么作用？心里好奇不？

多播时，master和slave之间交互的队列

mc_msg->type = msg->type;//消息的类型
mc_msg->seq = msg->seq;//消息的序列号
mc_msg->mask = slave_lcore_mask;//消息的cpu逻辑核心掩码
mc_msg->org_msg = msg; //保存以前的消息
INIT_LIST_HEAD(&mc_msg->mq);//初始化接受msg的队列

4）加写锁，读取mc_wait_list.free_cnt判断msg队列是否还有剩余空间，如没有，则将msg丢弃

5）将申请的mc_msg添加到mc_wait_list多播等待队列的尾部

list_add_tail(&mc_msg->list, &mc_wait_list.list);
  --mc_wait_list.free_cnt;//减少mc_wait_list的可用数量

6）异步消息投递到队列中即可函数返回；

同步发送，分为循环等待消息发送完成或消息丢弃 and 同步发送超时（上面已经分析）。

7）消息发送出去后，通过get_msg_flags函数得到msg的标志，返回不同的值。



# 五、消息的引用计数及销毁消息

为什么要对消息采用引用计数？

使用时，引用计数加1

不用时，引用计数减1

减少到0，就释放

谁最后用完，谁释放。



引用计数是如何变化的？

这块比较复杂，暂时先不分析。



在消息处理过程中如果有丢包发生，则会调用msg_destroy函数来销毁消息。

```
int msg_destroy(struct dpvs_msg **pmsg)
{
    struct dpvs_msg *msg;

    if (unlikely(!pmsg || !(*pmsg)))
        return EDPVS_INVAL;
    msg = *pmsg;

	//引用计数为0？
    if (unlikely(rte_atomic16_read(&msg->refcnt) == 0)) {
        char buf[1024];
        msg_dump(msg, buf, sizeof(buf));
        RTE_LOG(ERR, MSGMGR, "%s: bad msg refcnt at destroy:\n%s", __func__, buf);
        assert(0);
    }

	//引用计数为1？
    if (!rte_atomic16_dec_and_test(&msg->refcnt)) {
        *pmsg = NULL;
        return EDPVS_OK;
    }

    /* i'm the only one hold the msg, free it now */
    if (msg->mode == DPVS_MSG_MULTICAST) {
        struct dpvs_msg *cur, *next;
        struct dpvs_multicast_queue *mcq;
        assert(rte_lcore_id() == master_lcore);
        mcq = mc_queue_get(msg->type, msg->seq);
        if (likely(mcq != NULL)) {
        	//遍历mcq->mq,里面是已接收的slave消息
            list_for_each_entry_safe(cur, next, &mcq->mq, mq_node) {
                list_del_init(&cur->mq_node);
                add_msg_flags(cur, DPVS_MSG_F_STATE_FIN); /* in case slaves reply with blockable msg */
                msg_destroy(&cur);
            }
            mc_queue_unhash(mcq);
            dpvs_mempool_put(msg_pool, mcq);
        }
    }

    if (msg->reply.data) {
        assert(msg->reply.len != 0);
        msg_reply_free(msg->reply.data);
        msg->reply.len = 0;
    }
    dpvs_mempool_put(msg_pool, msg);
    *pmsg = NULL;

    msg_debug_free();

    return EDPVS_OK;
}
```



# 六、异常情况处理

## 6、1 丢包

何时处理的丢包？（定位到代码级别）

在msg_send函数中，有如下代码片段：

```
res = rte_ring_enqueue(msg_ring[cid], msg);
    if (unlikely(-EDQUOT == res)) {
        RTE_LOG(WARNING, MSGMGR, "%s:msg@%p, msg ring of lcore %d quota exceeded\n",
                __func__, msg, cid);
    } else if (unlikely(-ENOBUFS == res)) {
        RTE_LOG(ERR, MSGMGR, "%s:msg@%p, msg ring of lcore %d is full\n", __func__, msg, res);
        add_msg_flags(msg, DPVS_MSG_F_STATE_DROP);
        rte_atomic16_dec(&msg->refcnt); /* not enqueued, free manually */
        return EDPVS_DPDKAPIFAIL;
    } else if (res) {
        RTE_LOG(ERR, MSGMGR, "%s:msg@%p, unkown error %d for rte_ring_enqueue\n",
                __func__, msg, res);
        add_msg_flags(msg, DPVS_MSG_F_STATE_DROP);
        rte_atomic16_dec(&msg->refcnt); /* not enqueued, free manually */
        return EDPVS_DPDKAPIFAIL;
    }
```



## 6、2 超时

何时处理的丢包？（定位到代码级别）

在multicast_msg_send函数中，有如下代码片段：

```
while(!(test_msg_flags(msg, (DPVS_MSG_F_STATE_FIN | DPVS_MSG_F_STATE_DROP)))) {
        if (start + delay < rte_get_timer_cycles()) {
            RTE_LOG(WARNING, MSGMGR, "%s:msg@%p, mcq(type:%d, cid:%d->slaves) timeout"
                    "(%d us), drop...\n", __func__, msg,
                    msg->type, msg->cid, g_msg_timeout);
            add_msg_flags(msg, DPVS_MSG_F_TIMEOUT);
            /* just in case slave send reply fail.
             * it's safe here, because msg is used on master lcore only. */
            msg_destroy(&msg);
            return EDPVS_MSG_DROP;
        }
		//TODO:这里为什么要调用呢？
        msg_master_process(slave_lcore_nb * 1.5); /* to avoid dead lock if send msg to myself */
    }
```



## 6、3 投递队列状态

```
static int msg_master_process(int step)
{
    int n = 0;
    struct dpvs_msg *msg;
    struct dpvs_msg_type *msg_type;
    struct dpvs_multicast_queue *mcq;

    /* dequeue msg from ring on the master lcore and process it */
    while (((step <= 0) || ((step > 0) && (++n <= step))) &&
            (0 == rte_ring_dequeue(msg_ring[master_lcore], (void **)&msg))) {
        add_msg_flags(msg, DPVS_MSG_F_STATE_RECV);//消息从ring中出队
        msg_type = msg_type_get(msg->type, master_lcore);
        
        if (DPVS_MSG_UNICAST == msg_type->mode) { /* 单播消息 */
            msg_type->unicast_msg_cb(msg);
            add_msg_flags(msg, DPVS_MSG_F_STATE_FIN);
            msg_destroy(&msg);
        } else { /* 组播消息 */
            mcq = mc_queue_get(msg->type, msg->seq);
           
            if (mcq->mask & (1UL << msg->cid)) { /* you are the msg i'm waiting */
                list_add_tail(&msg->mq_node, &mcq->mq);
                /* set QUEUE flag for slave's reply msg */
                add_msg_flags(msg, DPVS_MSG_F_STATE_QUEUE);
                continue;
            }
        }
        msg_type_put(msg_type);
    }
    return EDPVS_OK;
}
```



小结：

1.master 核心同时处理DPVS_MSG_UNICAST单播消息和DPVS_MSG_MULTICAST多播消息

2.slave 核心仅仅处理DPVS_MSG_UNICAST单播消息，如果单播消息的模式为DPVS_MSG_MULTICAST，则调用

msg_send(xmsg, master_lcore, DPVS_MSG_F_ASYNC, NULL)以异步方式将消息发送到master lcore上