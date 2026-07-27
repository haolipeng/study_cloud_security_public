# 一、Flow引用计数的核心结构体

```
typedef unsigned short FlowRefCount;  // 引用计数类型

typedef struct Flow_ {
    // ... 其他字段
    FlowRefCount use_cnt;  // 引用计数字段
    // ... 其他字段
} Flow;
```



# 二、Flow引用计数的核心函数

## 2、1 增加引用计数

```
static inline void FlowReference(Flow **d, Flow *f)
{
    if (likely(f != NULL)) {
        FlowIncrUsecnt(f);  // f->use_cnt++
        *d = f;             // 设置指针
    }
}

static inline void FlowIncrUsecnt(Flow *f)
{
    if (f == NULL) return;
    f->use_cnt++;  // 原子操作，增加引用计数
}
```



## 2、2 减少引用计数

```
static inline void FlowDeReference(Flow **d)
{
    if (likely(*d != NULL)) {
        FlowDecrUsecnt(*d);  // (*d)->use_cnt--
        *d = NULL;           // 清空指针
    }
}

static inline void FlowDecrUsecnt(Flow *f)
{
    if (f == NULL) return;
    f->use_cnt--;  // 原子操作，减少引用计数
}
```



# 三、引用计数的作用

防止Flow被过早的清理

```
Flow *FlowGetFlowFromHash(ThreadVars *tv, FlowLookupStruct *fls, Packet *p, Flow **dest)
{
	//省略N多代码
	if (timedout) {
        FromHashLockTO(f);
        // 只有引用计数为0时才能清理
        if (likely(f->use_cnt == 0)) {
            MoveToWorkQueue(tv, fls, fb, f, prev_f);  // 移动到工作队列
        }
        FLOWLOCK_UNLOCK(f);
	}
	//省略N多代码
}
```

f->use_cnt等于0，代表没有Packet在使用这个Flow，可以放心的清理。



FlowReference函数在什么哪些地方被调用呢？是否会导致内存泄漏呢？

在FlowGetFlowFromHash函数中，有三处调用

```
Flow *FlowGetFlowFromHash(ThreadVars *tv, FlowLookupStruct *fls, Packet *p, Flow **dest)
{
    Flow *f = NULL;

    /* get our hash bucket and lock it */
    const uint32_t hash = p->flow_hash;
	//以数据包的流哈希值为下标，获取其对应的FlowBucket链表
    FlowBucket *fb = &flow_hash[hash % flow_config.hash_size];
    FromHashLockBucket(fb);//行级锁，锁住一个bucket

    SCLogDebug("fb %p fb->head %p", fb, fb->head);

    /* 检测bucket桶中是否有Flow流 */
	//bucket为空,直接申请新流Flow
    if (fb->head == NULL) {
        f = FlowGetNew(tv, fls, p);
        FlowReference(dest, f);//增加引用计数

        FBLOCK_UNLOCK(fb);
        return f;
    }

    const bool emerg = (SC_ATOMIC_GET(flow_flags) & FLOW_EMERGENCY) != 0;
    const uint32_t fb_nextts = !emerg ? SC_ATOMIC_GET(fb->next_ts) : 0;
    /* ok, we have a flow in the bucket. Let's find out if it is our flow */
    Flow *prev_f = NULL; /* previous flow */
    f = fb->head;
    do {
        Flow *next_f = NULL;/*next flow*/
        //检测流超时
        const bool timedout = (fb_nextts < (uint32_t)p->ts.tv_sec && FlowIsTimedOut(f, (uint32_t)p->ts.tv_sec, emerg));
        if (timedout) {
            FromHashLockTO(f);//FLOWLOCK_WRLOCK(f);
            FLOWLOCK_UNLOCK(f);
        } else if (FlowCompare(f, p) != 0) {//找到了和packet匹配的流
            FromHashLockCMP(f);//内部调用FLOWLOCK_WRLOCK(f)，锁住flow
            FlowReference(dest, f);//增加引用计数
            FBLOCK_UNLOCK(fb);
			//返回哈希值匹配的流
            return f;
        }
         
		//除非我们删除'f'，否则在下面添加新流时,prev_f 需要指向当前'f'。
        prev_f = f;
        next_f = f->next;

flow_removed:
		//查询完所有flow都没有找到
        if (next_f == NULL) {
            f = FlowGetNew(tv, fls, p);
            FlowInit(f, p);
            FlowReference(dest, f); //增加引用计数
            FBLOCK_UNLOCK(fb);
            return f;
        }
        f = next_f;
    } while (f != NULL);

    /* should be unreachable */
    BUG_ON(1);
    return NULL;
}
```

三种情况下，会创建新流，并增加引用计数。

1、bucket桶为空，创建新流

2、bucket桶中没找到packet对应的流，创建新流