# 一、流老化配置概述

suricata单独创建一个线程来执行流老化任务，允许创建多个流老化线程，为每个线程分配老化任务，即每个老化线程只负责老化分配给自己的flow的bucket。

**如何配置多个流老化线程呢？**

```
flow:
  memcap: 128mb
  hash-size: 65536
  prealloc: 2000
  emergency-recovery: 30
  managers: 1 # 默认为一个flow管理线程，可更改
  recyclers: 1 # 默认为一个flow回收线程，可更改
```

对应的配置解析代码如下：

```
void FlowManagerThreadSpawn(void)
{
    intmax_t setting = 1;
    (void)ConfGetInt("flow.managers", &setting);//解析配置文件中flow.managers字段
    flowmgr_number = (uint32_t)setting;

    for (uint32_t u = 0; u < flowmgr_number; u++) {
    	//构造线程名称
        char name[TM_THREAD_NAME_MAX];
        snprintf(name, sizeof(name), "%s#%02u", thread_name_flow_mgr, u+1);

		//创建流管理线程
        ThreadVars *tv_flowmgr = TmThreadCreateMgmtThreadByName(name,"FlowManager", 0);
        if (TmThreadSpawn(tv_flowmgr) != TM_ECODE_OK) {
            FatalError(SC_ERR_FATAL, "flow manager thread spawn failed");
        }
    }
    return;
}
```

流老化线程的核心函数是FlowManager函数，下面我们来好好分析下这个函数。



# 二、FlowManager流管理流程梳理

## 2、1 FlowManager函数核心流程梳理

流老化线程入口函数是FlowManager模块的FlowManager函数，精简后的代码如下所示：

其正常模式下的代码如图所示

```
static TmEcode FlowManager(ThreadVars *th_v, void *thread_data)
{
    //流管理线程数据
    FlowManagerThreadData *ftd = thread_data;
    struct timeval ts;
    uint32_t established_cnt = 0, new_cnt = 0, closing_cnt = 0;
    bool emerg = false; //紧急模式标志
    bool prev_emerg = false; //上一次紧急模式标志
    uint32_t other_last_sec = 0; /**< last sec stamp when defrag etc ran */
    uint32_t flow_last_sec = 0;
    memset(&ts, 0, sizeof(ts));

    //所有协议的超时时间的最小值
    const uint32_t min_timeout = FlowTimeoutsMin();
    const uint32_t pass_in_sec = min_timeout ? min_timeout * 8 : 60;

    /* don't start our activities until time is setup */
    while (!TimeModeIsReady()) {
        if (suricata_ctl_flags != 0)
            return TM_ECODE_OK;
    }
    const bool time_is_live = TimeModeIsLive();

    SCLogDebug("FM %s/%d starting. min_timeout %us. Full hash pass in %us", th_v->name,
            ftd->instance, min_timeout, pass_in_sec);

    struct timeval startts;
    memset(&startts, 0, sizeof(startts));
    gettimeofday(&startts, NULL);

    uint32_t hash_pass_iter = 0;//记录下一次从那个bucket开始检查flow超时
    uint32_t emerg_over_cnt = 0;
    uint64_t next_run_ms = 0;

    while (1)
    {
        //检查线程暂停标志
		
        /* 获取当前时间，并转换为秒和毫秒格式*/
       
        if (ts_ms >= next_run_ms) {
            if (ftd->instance == 0) {
                const uint32_t sq_len = FlowSpareGetPoolSize();
                const uint32_t spare_perc = sq_len * 100 / flow_config.prealloc;
                /* see if we still have enough spare flows */
                if (spare_perc < 90 || spare_perc > 110) {
                    FlowSparePoolUpdate(sq_len);//超过预分配比例后，会进行增减操作
                }
            }
            const uint32_t secs_passed = rt - flow_last_sec;

            FlowTimeoutCounters counters = { 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0 };

            /* 非紧急模式：分段扫描 */
            const uint32_t chunks = MIN(secs_passed, pass_in_sec);
            for (uint32_t i = 0; i < chunks; i++) {
                FlowTimeoutHashInChunks(&ftd->timeout, &ts, ftd->min, ftd->max,
                        &counters, hash_pass_iter, pass_in_sec);
                hash_pass_iter++;
                if (hash_pass_iter == pass_in_sec) {
                    hash_pass_iter = 0;
                }
            }
            
            flow_last_sec = rt;

            uint32_t len = FlowSpareGetPoolSize();

            //正常模式下，下次轮询时间间隔为667ms，约1.5次/秒
            next_run_ms = ts_ms + 667;
        }//end with if (ts_ms >= next_run_ms)
        
        if (flow_last_sec == 0) {
			//第一次走这里
            flow_last_sec = rt;
        }

        if (ftd->instance == 0 &&
                (other_last_sec == 0 || other_last_sec < (uint32_t)ts.tv_sec)) {
            DefragTimeoutHash(&ts);//分片超时处理
            HostTimeoutHash(&ts);//主机超时处理
            IPPairTimeoutHash(&ts);//ip对超时处理
            other_last_sec = (uint32_t)ts.tv_sec;
        }

        //start 正常模式：使用条件变量等待
        struct timeval cond_tv;
        gettimeofday(&cond_tv, NULL);
        struct timeval add_tv;
        add_tv.tv_sec = 0;
        add_tv.tv_usec = 667 * 1000;
        timeradd(&cond_tv, &add_tv, &cond_tv);

        struct timespec cond_time = FROM_TIMEVAL(cond_tv);
        SCCtrlMutexLock(&flow_manager_ctrl_mutex);
        while (1) {
            int rc = SCCtrlCondTimedwait(
                    &flow_manager_ctrl_cond, &flow_manager_ctrl_mutex, &cond_time);
            if (rc == ETIMEDOUT || rc < 0)
                break;
            if (SC_ATOMIC_GET(flow_flags) & FLOW_EMERGENCY) {
                break; //紧急模式时，立即唤醒
            }
        }
        SCCtrlMutexUnlock(&flow_manager_ctrl_mutex);
        //end 正常模式：使用条件变量等待

        SCLogDebug("woke up... %s", SC_ATOMIC_GET(flow_flags) & FLOW_EMERGENCY ? "emergency":"");

        StatsSyncCountersIfSignalled(th_v);
    }

    return TM_ECODE_OK;
}
```



完整版直接看

```
static TmEcode FlowManager(ThreadVars *th_v, void *thread_data)
{
    FlowManagerThreadData *ftd = thread_data;
    struct timeval ts;
    uint32_t established_cnt = 0, new_cnt = 0, closing_cnt = 0;
    bool emerg = false;
    bool prev_emerg = false;
    uint32_t other_last_sec = 0; /**< last sec stamp when defrag etc ran */
    uint32_t flow_last_sec = 0;
    memset(&ts, 0, sizeof(ts));

    //所有协议的超时时间的最小值
    const uint32_t min_timeout = FlowTimeoutsMin();
    const uint32_t pass_in_sec = min_timeout ? min_timeout * 8 : 60;
    
    const bool time_is_live = TimeModeIsLive();

    uint32_t hash_pass_iter = 0;//记录下一次从那个bucket开始检查flow超时
    uint32_t emerg_over_cnt = 0;//空闲flow数占比大于配置的紧急模式百分比的连续多少次数
    uint64_t next_run_ms = 0;//记录下一次运行的时间

    while (1)
    {
		......省略代码(设置线程状态)......
		
        //检查紧急模式标志,在FlowGetNew设置FLOW_EMERGENCY
        if (SC_ATOMIC_GET(flow_flags) & FLOW_EMERGENCY) {
            emerg = true;
        }
        
        /* 获取当前时间 */
        memset(&ts, 0, sizeof(ts));
        TimeGet(&ts);  
        const uint64_t ts_ms = ts.tv_sec * 1000 + ts.tv_usec / 1000;
        const uint32_t rt = (uint32_t)ts.tv_sec;

        //避免重复进入紧急模式
        const bool emerge_p = (emerg && !prev_emerg);
        
        //到达下一次检查时间，则开始超时检查
        if (ts_ms >= next_run_ms) {
        	//instance为零代表是第一个老化线程，只有第一个实例负责维护空闲flow的数量
        	//将范围维护在90%-110%之间，上下浮动10%的样子
            if (ftd->instance == 0) {
            	//获取全局空闲flow池中总数
                const uint32_t sq_len = FlowSpareGetPoolSize();
                
                //计算空闲flow百分比
                const uint32_t spare_perc = sq_len * 100 / flow_config.prealloc;
                /* see if we still have enough spare flows */
                if (spare_perc < 90 || spare_perc > 110) {
                	//超过预分配比例后，会进行增减操作
                    FlowSparePoolUpdate(sq_len);
                }
            }
            const uint32_t secs_passed = rt - flow_last_sec;

            ......省略代码......

            if (emerg) {
                //紧急模式下，进行“全表扫描”
                FlowTimeoutHash(&ftd->timeout, &ts, ftd->min, ftd->max, &counters);
            } else {
                //正常模式下，进行flow bucket桶分时-分段检查
                //chunks就是根据经过的秒数，把bucket分成多个范围，每次轮询只检查数个bucket
                //当然如果经过的秒数是0，则一个bucket也不检查.
                const uint32_t chunks = MIN(secs_passed, pass_in_sec);
                for (uint32_t i = 0; i < chunks; i++) {
                    FlowTimeoutHashInChunks(&ftd->timeout, &ts, ftd->min, ftd->max,
                            &counters, hash_pass_iter, pass_in_sec);
                    hash_pass_iter++;
                    if (hash_pass_iter == pass_in_sec) {
                    	//遍历到最后一个bucket，则从第一个bucket开始检查超时，避免有遗漏的现象
                        hash_pass_iter = 0;
                    }
                }
            }
            
            //空闲flow总数占比 大于 配置的紧急模式下的百分比
            flow_last_sec = rt;
            uint32_t len = FlowSpareGetPoolSize();
            if (len * 100 / flow_config.prealloc > flow_config.emergency_recovery) {
                emerg_over_cnt++;
            } else {
            	//反之，继续停留在急救模式下
                emerg_over_cnt = 0;
            }

			//空闲flow占比连续30次大于配置的急救模式百分比(flow_config.emergency_recovery)，则解除紧急模式
            if (emerg_over_cnt >= 30) {
                SC_ATOMIC_AND(flow_flags, ~FLOW_EMERGENCY);
                FlowTimeoutsReset();

                emerg = false;//解除紧急模式
                prev_emerg = FALSE;
                emerg_over_cnt = 0;
                hash_pass_iter = 0;
            }
        }
        next_run_ms = ts_ms + 667;
        if (emerg)
        	//紧急模式下，轮序的时间间隔为250毫秒，加快老化检查
            next_run_ms = ts_ms + 250;
        }
        if (flow_last_sec == 0) {
            flow_last_sec = rt;
        }

		//如果是第一个流老化线程，需要执行额外的老化任务,如DefragTimeoutHash和IPPairTimeoutHash
        if (ftd->instance == 0 && (other_last_sec == 0 || other_last_sec < (uint32_t)ts.tv_sec)) {
            DefragTimeoutHash(&ts);
            HostTimeoutHash(&ts);
            IPPairTimeoutHash(&ts);
            other_last_sec = (uint32_t)ts.tv_sec;
        }

        if (emerg || !time_is_live) {
        	//紧急模式下250毫秒检查一次流超时
            usleep(250);
        } else {
        	//正常模式667毫秒检查一次流超时
            struct timeval cond_tv;
            gettimeofday(&cond_tv, NULL);
            struct timeval add_tv;
            add_tv.tv_sec = 0;
            add_tv.tv_usec = 667 * 1000;
            timeradd(&cond_tv, &add_tv, &cond_tv);
            ......省略代码......
        }
    }

    return TM_ECODE_OK;
}
```

**流程如图所示：**

![image-20230609160317744](https://gitee.com/codergeek/picgo-image/raw/master/image/202505121428013.png)



有三个变量很重要，单独拿出来说下这个。

```
//牢牢记住这三个变量的含义
uint32_t hash_pass_iter = 0;//记录下一次从那个bucket开始检查flow超时
uint32_t emerg_over_cnt = 0;//空闲flow数占比大于配置的紧急模式百分比的连续多少次数
uint64_t next_run_ms = 0;//记录下一次运行的时间
```



**步骤1：判断是否是第一个老化线程**

当达到老化时间并且是第一个老化线程，则负责维护空闲flow的数量，将flow数量的范围维护在90%-110%之间，上下浮动10%的样子。

调用函数为FlowSparePoolUpdate。



**步骤2：正常模式 vs 紧急模式**

**正常模式：**正常模式667毫秒检查一次流超时，调用FlowTimeoutHashInChunks完成flow超时处理，FlowTimeoutHashInChunks内部也是调用FlowTimeoutHash进行超时处理。

**紧急模式：**调用FlowTimeoutHash完成超时处理；紧急模式时间间隔为250毫秒，加快老化检查；



**步骤3：统计空闲flow总数占比 大于 紧急模式下的百分比的次数**

在流老化检查步骤中，会检查空闲fow数量，如果空闲flow数量连续30次大于配置的百分比flow_config.emergency_recovery，则解除紧急模式，恢复正常轮询间隔时间为667毫秒，老化时间设置为正常模式老化时间。

```
//空闲flow总数占比 大于 紧急模式下的百分比
uint32_t len = FlowSpareGetPoolSize();
if (len * 100 / flow_config.prealloc > flow_config.emergency_recovery) {
	emerg_over_cnt++;
} else {
	emerg_over_cnt = 0;
}

//空闲flow数量连续30次大于配置的百分比(flow_config.emergency_recovery)，则解除紧急模式
if (emerg_over_cnt >= 30) {
    SC_ATOMIC_AND(flow_flags, ~FLOW_EMERGENCY);
    FlowTimeoutsReset();

    emerg = false;//解除紧急模式
    prev_emerg = FALSE;
    emerg_over_cnt = 0;
    hash_pass_iter = 0;
}
```

当空闲flow数量连续30次大于配置的百分比(flow_config.emergency_recovery)，则解除紧急模式，此时调用FlowTimeoutsReset()宏

```
#define FlowTimeoutsReset() FlowTimeoutsInit()

void FlowTimeoutsInit(void)
{
    SC_ATOMIC_SET(flow_timeouts, flow_timeouts_normal);
}
```

flow_timeouts流老化时间设置为正常模式老化时间。



**步骤4：更新下一次老化时间**

**正常模式：**正常模式667毫秒检查一次流超时；

**紧急模式：**紧急模式时间间隔为250毫秒，加快老化检查；

```
next_run_ms = ts_ms + 667;//正常模式下为667毫秒
if (emerg)
	//紧急模式下，轮序的时间间隔为250毫秒，加快老化检查
	next_run_ms = ts_ms + 250;
}
```



**步骤5：第一个老化线程执行额外老化任务**

如果是第一个流老化线程，需要执行额外的老化任务,如DefragTimeoutHash和IPPairTimeoutHash

```
if (ftd->instance == 0 && (other_last_sec == 0 || other_last_sec < (uint32_t)ts.tv_sec)) {
	DefragTimeoutHash(&ts);
	HostTimeoutHash(&ts);
    IPPairTimeoutHash(&ts);
    other_last_sec = (uint32_t)ts.tv_sec;
}
```



流老化入口函数FlowManger调用函数FlowTimeoutHash完成流老化处理，这个函数完成了大部分老化工作，下面重点来看看这个函数。

# 三、 FlowTimeoutHash流老化流程梳理

## 3、1 流程概要分析

```
static uint32_t FlowTimeoutHash(FlowManagerTimeoutThread *td, SCTime_t ts, const uint32_t hash_min,
        const uint32_t hash_max, FlowTimeoutCounters *counters)
{
    uint32_t cnt = 0;
    //紧急模式标志获取
    const int emergency = ((SC_ATOMIC_GET(flow_flags) & FLOW_EMERGENCY));
    //计算检查的bucket数量
    const uint32_t rows_checked = hash_max - hash_min;

#define BITS 64
#define TYPE uint64_t

    const uint32_t ts_secs = SCTIME_SECS(ts);
    for (uint32_t idx = hash_min; idx < hash_max; idx+=BITS) {
        TYPE check_bits = 0;
        const uint32_t check = MIN(BITS, (hash_max - idx));
        //BITS的值为64
        //循环64次，检查64个bucket是否有超时的flow，有则设置一位到check_bits
        for (uint32_t i = 0; i < check; i++) {
            FlowBucket *fb = &flow_hash[idx+i];
            check_bits |= (TYPE)(SC_ATOMIC_LOAD_EXPLICIT(fb->next_ts, SC_ATOMIC_MEMORY_ORDER_RELAXED) <= ts_secs) << (TYPE)i;
        }
        //如果64个bucket中都没有超时的流，则继续下一轮64个bucket的超时判断
        if (check_bits == 0)
            continue;

        for (uint32_t i = 0; i < check; i++) {
            FlowBucket *fb = &flow_hash[idx+i];
            //有超时的，则通过位图在64个bucket找到有超时flow的bucket，然后老化之
            if ((check_bits & ((TYPE)1 << (TYPE)i)) != 0 && SC_ATOMIC_GET(fb->next_ts) <= ts_secs) {
                FBLOCK_LOCK(fb);
                Flow *evicted = NULL;
                if (fb->evicted != NULL || fb->head != NULL) {
                	//在建立流或获取流时检测到流超时，把flow放入evicted链表中，等待FlowManager处理
                    if (fb->evicted != NULL) {
                        /* transfer out of bucket so we can do additional work outside
                         * of the bucket lock */
                        //方便在bucket lock之外做额外的操作
                        evicted = fb->evicted;
                        fb->evicted = NULL;
                    }

					//flow bucket上有超时flow,将超时的流放到了td->aside_queue队列，
					//然后再将td->aside_queue队列中的flow移入回收线程的队列
                    if (fb->head != NULL) {
                        uint32_t next_ts = 0;
                        FlowManagerHashRowTimeout(td, fb->head, ts, emergency, counters, &next_ts);

						//设置bucket的下一次超时时间
                        if (SC_ATOMIC_GET(fb->next_ts) != next_ts)
                            SC_ATOMIC_SET(fb->next_ts, next_ts);
                    }
                    
					//如果bucket的evicted链表中没有flow， 且head为空不包含flow了
					//说明bucket上所有flow都被老化了，设置时间为UINT_MAX，不需要老化了
                    if (fb->evicted == NULL && fb->head == NULL) {
                        SC_ATOMIC_SET(fb->next_ts, UINT_MAX);
                    }
                }
                
                FBLOCK_UNLOCK(fb);
                /* processed evicted list */
                if (evicted) {
                	//将evicted队列的超时flow移入td->aside_queue队列，等待后续处理
                    FlowManagerHashRowClearEvictedList(td, evicted, ts, counters);
                }
            }
        }
        if (td->aside_queue.len) {
        	//处理超时的flow,把td->aside_queue的flow移入回收线程的回收队列flow_recycle_q
            cnt += ProcessAsideQueue(td, counters);
        }
    }

    if (td->aside_queue.len) {
    	//处理超时的flow,把td->aside_queue的flow移入回收线程的回收队列flow_recycle_q
        cnt += ProcessAsideQueue(td, counters);
    }

    return cnt;
}
```

画图，进行流程总结：

**步骤1：循环64次，检查64个bucket是否有超时的flow，有则设置一位到check_bits；接下来判断check_bits的值**

**步骤2：64个bucket中都没有超时的流，则继续下一轮64个bucket的超时判断；**

**步骤3：64个bucket中有超时的流，则通过位图check_bits在64个bucket找到有超时flow的bucket，然后分不同情况进行老化；**

**步骤4：在建立流或获取流时检测到流超时，把flow放入evicted链表中;**

**步骤5：flow bucket上有超时flow,将超时的流放到了td->aside_queue队列，然后再将td->aside_queue队列中的flow移入回收线程的队列;**

**步骤6：处理超时的flow,把td->aside_queue的flow移入回收线程的回收队列flow_recycle_q**



## 3、2 fb->evicted队列不为空

其中这个流程需要细说下，因为属于比较特殊的情况。

```
if (fb->evicted != NULL || fb->head != NULL) {
	//在建立流或获取流时检测到流超时，把flow放入evicted链表中，等待FlowManager处理
	if (fb->evicted != NULL) {
		/* transfer out of bucket so we can do additional work outside
		 * of the bucket lock */
		//方便在bucket lock之外做额外的操作
		evicted = fb->evicted;
		fb->evicted = NULL;
	}
	...省略代码...
	
	/* processed evicted list */
    if (evicted) {
    	FlowManagerHashRowClearEvictedList(td, evicted, ts, counters);
    }
}
```

当fb->evicted链表不为空时，代表有需要超时的元素，先将fb->evicted存放到临时变量evicted中，然后调用FlowManagerHashRowClearEvictedList函数处理变量evicted。



FlowBucket_结构体中的evicted字段解释如下：

```
typedef struct FlowBucket_ {
    /** head of the list of evicted flows for this row. Waiting to be
     *  collected by the Flow Manager. */
    //被驱逐的流的链表的头指针
    Flow *evicted;
}
```

通过变量赋值，查到fb->evicted是在MoveToWorkQueue函数被赋值的。

在FlowGetFlowFromHash函数中，suricata在新建流或获取流的时机，会检查流是否超时，从而调用MoveToWorkQueue函数。代码如下所示：

```
Flow *FlowGetFlowFromHash(ThreadVars *tv, FlowLookupStruct *fls, Packet *p, Flow **dest)
{
    do {
        //检测流超时
        const bool timedout = (fb_nextts < (uint32_t)p->ts.tv_sec && FlowIsTimedOut(f, (uint32_t)p->ts.tv_sec, emerg));
        if (timedout) {
            //超时且引用计数为0，没有packet引用这个flow，则放到线程所属的work queue
            if (likely(f->use_cnt == 0)) {
                MoveToWorkQueue(tv, fls, fb, f, prev_f);
            }
        } else if (FlowCompare(f, p) != 0) {
			//找到了和packet匹配的流
            if (unlikely(TcpSessionPacketSsnReuse(p, f, f->protoctx) == 1)) {
                Flow *new_f = TcpReuseReplace(tv, fls, fb, f, hash, p);
                if (likely(f->use_cnt == 0)) {
                    MoveToWorkQueue(tv, fls, fb, f, prev_f); /* evict old flow */
                }
            }
        }
    } while (f != NULL);

    return NULL;
}
```



## 3、3 fb->head队列不为空

其中这个流程需要细说下，因为属于比较特殊的情况。

```
if (fb->evicted != NULL || fb->head != NULL) {
	//这个flow bucket上有超时flow需处理，调用FlowManagerHashRowTimeout函数
	if (fb->head != NULL) {
		int32_t next_ts = 0;
		FlowManagerHashRowTimeout(td, fb->head, ts, emergency, counters, &next_ts);

		if (SC_ATOMIC_GET(fb->next_ts) != next_ts)
			SC_ATOMIC_SET(fb->next_ts, next_ts);
	}
}
```

下面我们好好看下FlowManagerHashRowTimeout函数的细节。

```
static void FlowManagerHashRowTimeout(FlowManagerTimeoutThread *td,
        Flow *f, struct timeval *ts,
        int emergency, FlowTimeoutCounters *counters, int32_t *next_ts)
{
    uint32_t checked = 0;
    Flow *prev_f = NULL;

    do {
        checked++;

		//检查flow是否超时，返回0表示未超时
        if (FlowManagerFlowTimeout(f, ts, next_ts, emergency) == 0) {

            counters->flows_notimeout++;

            prev_f = f;
            f = f->next;//未超时则继续检查下个flow
            continue;
        }

        FMFlowLock(f); //FLOWLOCK_WRLOCK(f);

        Flow *next_flow = f->next;

        //flow流被当前处理的数据包所引用，则不能老化
        if (f->use_cnt > 0 || !FlowBypassedTimeout(f, ts, counters)) {
            FLOWLOCK_UNLOCK(f);
            prev_f = f;
            f = f->next;//不符合老化条件则继续检查下个flow
            continue;
        }
		
		//走到此都是超时的，设置标志为超时老化
        f->flow_end_flags |= FLOW_END_FLAG_TIMEOUT;

		//从bucket上移除flow
        RemoveFromHash(f, prev_f);

		//将超时的flow添加到aside_queue队列中
        FlowQueuePrivateAppendFlow(&td->aside_queue, f);
        /* flow is still locked in the queue */
		//注意此时的锁的状态

        f = next_flow;
    } while (f != NULL);
}
```

1、调用FlowManagerFlowTimeout检查是否超时，返回0表示未超时

2、flow流被当前处理的数据包所引用，则不能进行老化

3、把老化的flow从flow_hash的bucket上移除

4、将超时的flow添加到aside_queue队列中



## 3、4 处理aside_queue队列中的flow超时流

```
static uint32_t ProcessAsideQueue(FlowManagerTimeoutThread *td, FlowTimeoutCounters *counters)
{
    FlowQueuePrivate recycle = { NULL, NULL, 0 };
    counters->flows_aside += td->aside_queue.len;

    uint32_t cnt = 0;
    Flow *f;
	
	//aside_queue从队列中循环取出每个超时的flow
    while ((f = FlowQueuePrivateGetFromTop(&td->aside_queue)) != NULL) {
		//如果是TCP的flow流，则判断重组
		//如果需要重组，将其发回原来处理它的线程，
		//由原线程的FlowWorker中FlowWorkerProcessInjectedFlows完成重组老化回收()
        if (f->proto == IPPROTO_TCP && !(f->flags & FLOW_TIMEOUT_REASSEMBLY_DONE) &&
                !FlowIsBypassed(f) && FlowForceReassemblyNeedReassembly(f) == 1) {
            /* Send the flow to its thread */
			//需要重组，则把flow放入原线程的tv->flow_queue队列中
            FlowForceReassemblyForFlow(f);
            FLOWLOCK_UNLOCK(f);
            /* flow ownership is passed to the worker thread */

            counters->flows_aside_needs_work++;
            continue;
        }
        //这块要进行解锁啊
        FLOWLOCK_UNLOCK(f);

		//把flow添加到临时回收队列recycle
        FlowQueuePrivateAppendFlow(&recycle, f);
        if (recycle.len == 100) {
			//recycle队列元素满100后，放入全局回收队列flow_recycle_q
            FlowQueueAppendPrivate(&flow_recycle_q, &recycle);
            FlowWakeupFlowRecyclerThread();//唤醒FlowRecycler线程
        }
        cnt++;
    }
    if (recycle.len) {
		//recycle队列元素小于100时即剩下的零头flow，将recycle队列放入全局回收队列中
        FlowQueueAppendPrivate(&flow_recycle_q, &recycle);
        FlowWakeupFlowRecyclerThread();
    }
    return cnt;
}
```



流程总结：

1、把flow添加到临时回收对列recycle

2、recycle队列元素满100后，放入全局回收队列flow_recycle_q

3、唤醒FlowRecycler线程

4、recycle队列元素小于100时即剩下的零头flow，将recycle队列放入全局回收队列中（收尾工作）



## 3、5 重组流 vs 超时流

ProcessAsideQueue函数中有如下代码

```
//如果是TCP的flow流，则判断重组
//如果需要重组，将其发回原来处理它的线程，
//由原线程的FlowWorker中FlowWorkerProcessInjectedFlows完成重组老化回收()
if (f->proto == IPPROTO_TCP && !(f->flags & FLOW_TIMEOUT_REASSEMBLY_DONE) &&
		!FlowIsBypassed(f) && FlowForceReassemblyNeedReassembly(f) == 1) {
	/* Send the flow to its thread */
	//需要重组，则把flow放入原线程的tv->flow_queue队列中
	FlowForceReassemblyForFlow(f);
	FLOWLOCK_UNLOCK(f);
	/* flow ownership is passed to the worker thread */

	counters->flows_aside_needs_work++;
	continue;
}
```

FlowForceReassemblyNeedReassembly函数：判断一个flow流是否需要强制进行重组，返回值为1表示需要强制重组。

FlowForceReassemblyForFlow继续调用TmThreadsInjectFlowById函数

```
void FlowForceReassemblyForFlow(Flow *f)
{
	//thread_id代表flow来自哪个线程
    const int thread_id = (int)f->thread_id[0];
    //把flow打回原线程继续重组老化回收处理
    TmThreadsInjectFlowById(f, thread_id);
}
```

TmThreadsInjectFlowById函数如下

```
void TmThreadsInjectFlowById(Flow *f, const int id)
{
    int idx = id - 1;

    Thread *t = &thread_store.threads[idx];
    ThreadVars *tv = t->tv;
	
	//tv->flow_queue专门存放老化时需要重组的flow
    FlowEnqueue(tv->flow_queue, f);

    /* wake up listening thread(s) if necessary */
    if (tv->inq != NULL) {
        SCCondSignal(&tv->inq->pq->cond_q);//唤醒对应的线程
    } else if (tv->break_loop) {
        ThreadBreakLoop(tv);
    }
}
```

那么在什么地方处理flow_queue呢？函数调用栈如下：

```
FlowWorker
	FlowWorkerProcessInjectedFlows
		CheckWorkQueue
```

CheckWorkQueue函数的代码如下所示：

```
static void CheckWorkQueue(ThreadVars *tv, FlowWorkerThreadData *fw,
        void *detect_thread, // TODO proper type?
        FlowTimeoutCounters *counters,
        FlowQueuePrivate *fq)
{
    Flow *f;
    while ((f = FlowQueuePrivateGetFromTop(fq)) != NULL) {
        f->flow_end_flags |= FLOW_END_FLAG_TIMEOUT; //TODO emerg

        if (f->proto == IPPROTO_TCP) {
            if (!(f->flags & FLOW_TIMEOUT_REASSEMBLY_DONE) && !FlowIsBypassed(f) &&
                    FlowForceReassemblyNeedReassembly(f) == 1 && f->ffr != 0) {
                //调用FlowFinish函数
                int cnt = FlowFinish(tv, f, fw, detect_thread);
            }
        }

		//进行资源释放
        FlowClearMemory (f, f->protomap);
        //线程所属的flow队列的flow数大于200，则调用FlowSparePoolReturnFlow函数，否则调用FlowQueuePrivatePrependFlow
        if (fw->fls.spare_queue.len >= 200) { // TODO match to API? 200 = 2 * block size
            FlowSparePoolReturnFlow(f);
        } else {
            FlowQueuePrivatePrependFlow(&fw->fls.spare_queue, f);
        }
    }
}
```



FlowSparePoolReturnFlow函数的代码简化后如下所示：

```
void FlowSparePoolReturnFlow(Flow *f)
{
    /* if the top is full, get a new block */
    if (flow_spare_pool->queue.len >= flow_spare_pool_block_size) {
        FlowSparePool *p = FlowSpareGetPool();
        DEBUG_VALIDATE_BUG_ON(p == NULL);
        p->next = flow_spare_pool;
        flow_spare_pool = p;
    }
    /* add to the (possibly new) top */
    FlowQueuePrivateAppendFlow(&flow_spare_pool->queue, f);
}
```

将flow归还给FlowSparePool内存池。调用FlowQueuePrivateAppendFlow函数。



流程总结如下：

1、从线程所属的flow队列中取出一个flow进行处理

2、flow的协议是tcp，并且未完成流重组超时，并且是需要强制进行流重组的流，则调用FlowFinish函数

3、FlowClearMemory清理flow相关的内存资源。

4、线程所属的flow队列的flow数大于200，则调用FlowSparePoolReturnFlow函数，否则调用FlowQueuePrivatePrependFlow



# 四、 FlowTimeoutHashInChunks函数

```
static uint32_t FlowTimeoutHashInChunks(FlowManagerTimeoutThread *td,
        struct timeval *ts,
        const uint32_t hash_min, const uint32_t hash_max,
        FlowTimeoutCounters *counters, uint32_t iter, const uint32_t chunks)
{
    //本线程负责的bucket个数
    const uint32_t rows = hash_max - hash_min;
    //每次遍历的bucket个数，chunks就是传递进来的秒数
    const uint32_t chunk_size = rows / chunks;
 
    ---------------------------------
    //chunks是传递进来的pass_in_sec变量，
    //这两行是在这个函数外边定义的，拿进来方便理解
    //这行获取正常模式的所有协议超时时间中最小的超时秒数
    const uint32_t min_timeout = FlowTimeoutsMin();
    //这个根据超时秒做了个乘8的操作，最小秒数是30，所以pass_in_sec = 240
    //知道怎么算了，暂时没理解
    const uint32_t pass_in_sec = min_timeout ? min_timeout * 8 : 60;
    -----------------------------------------------------------------------
 
    //本次遍历bucket的起始索引，随着iter的值加1，这个索引往后移一个chunk_size
    //每次iter传递进来都是加1后的值，所以每次都是依次遍历后边的bucket，
    //因为前边的bucket在前一秒或者前数秒刚刚遍历过，本次接着往后遍历，
    //这个也是这个函数的主要目的
    const uint32_t min = iter * chunk_size + hash_min;
    //本次遍历bucket的结束索引
    uint32_t max = min + chunk_size;
    /* we start at beginning of hash at next iteration so let's check
     * hash till the end */
    if (iter + 1 == chunks) {
        //如果有余数就是剩余的不够一个chunk_size的bucket，则归入本次遍历范围
        max = hash_max;
    }
    //这个函数在上篇文章已经解释过，做具体的老化任务
    const uint32_t cnt = FlowTimeoutHash(td, ts, min, max, counters);
    return cnt; //返回老化flow个数
}
```



# 五、疑问点

ProcessAsideQueue的调用栈是什么样的？
