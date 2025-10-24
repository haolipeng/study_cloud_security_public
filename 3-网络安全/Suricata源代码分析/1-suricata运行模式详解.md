suricata源代码分析系列是基于6.0.10版本的代码

suricata源代码分析系列是基于6.0.10版本的代码

suricata源代码分析系列是基于6.0.10版本的代码



源代码的架构层次

| 源文件名                | 模块所属     | 作用                                     |
| ----------------------- | ------------ | ---------------------------------------- |
| src/runmodes.c          | 基础架构     | 运行模式核心实现                         |
| src/runmodes.h          | 基础架构     | 运行模式定义                             |
| src/suricata.c          | 基础架构     | 主程序入口                               |
| src/tm-threads.c        | 基础架构     | 线程模块管理                             |
| src/tm-threads.h        | 基础架构     | 线程模块的定义                           |
| src/threadvars.h        | 基础架构     | 线程变量定义                             |
|                         |              |                                          |
| src/runmode-af-packet.c | 运行模式     | AF_PACKET 模式实现                       |
| src/source-af-packet.c  | 运行模式     | AF_PACKET 数据源                         |
| src/runmode-af-packet.h | 运行模式     | AF_PACKET 模式头文件(pcap,pcap-file同理) |
|                         |              |                                          |
| src/tm-queues.c         | 线程间通信   | 线程队列管理                             |
| src/tm-queues.h         | 线程间通信   | 线程队列接口                             |
| src/tmqh-*.c            | 线程间通信   | 队列处理器实现                           |
|                         |              |                                          |
| src/decode.c            | 解码模块     | 数据包解码核心                           |
| src/decode-*.c          | 解码模块     | 各种协议解码器                           |
|                         |              |                                          |
| src/detect.c            | 检测引擎模块 | 检测引擎核心                             |
| src/detect-engine.c     | 检测引擎模块 | 检测引擎实现                             |
| src/detect-*.c          | 检测引擎模块 | 各种检测模块                             |
|                         |              |                                          |
| src/flow.c              | 流管理       | 流相关文件                               |
| src/flow-manager.c      | 流管理       | 流管理器                                 |
| src/flow-*.c            | 流管理       | 流相关功能                               |
|                         |              |                                          |



**从个人角度来思考：**

如果让我来设计，从场景分析上来看，一个线程上可能运行多个不同的功能（比如数据包捕获、解析、统计信息等），这些功能之间是彼此有联系的。

那么把这些功能封装下，统称为Thread Module线程模块，每个模块中有多个串行执行的功能子模块，我觉得这个设计挺好的。



线程类型分类：

- 接收线程 (Receive Threads): 负责数据包捕获

- 解码线程 (Decode Threads): 负责数据包解码

- 检测线程 (Detect Threads): 负责规则匹配和检测

- 流管理线程 (Flow Manager): 负责流状态管理

- 输出线程 (Output Threads): 负责日志输出

- 管理线程 (Management Threads): 负责系统管理

**其中流管理线程和管理线程之间的区别是什么？**线程命名规范：

```
const char *thread_name_autofp = "RX";//autofp工作模式线程
const char *thread_name_single = "W";//单线程工作模式线程
const char *thread_name_workers = "W";//多线程worker工作模式线程
const char *thread_name_verdict = "TX";//裁决线程
const char *thread_name_flow_mgr = "FM";//流管理线程
const char *thread_name_flow_rec = "FR";//流回收线程
const char *thread_name_flow_bypass = "FB";//流绕过线程
```



具体运行模式

#### AF_PACKET 模式

```
src/runmode-af-packet.c    # AF_PACKET 模式实现
src/source-af-packet.c     # AF_PACKET 数据源
src/runmode-af-packet.h    # AF_PACKET 模式头文件
```

#### PCAP 模式深入研究

```
src/runmode-pcap.c         # PCAP 模式实现
src/source-pcap.c          # PCAP 数据源
src/runmode-pcap.h         # PCAP 模式头文件
```







# 一、初识suricata运行模式

首先来看一下suricata有几种运行模式



执行suricata --list-runmodes命令行，可以查看支持多少种运行模式。

![image-20250902111757062](./picture/image-20250902111757062.png)

![image-20250902111822705](./picture/image-20250902111822705.png)

上述只是把suricata --list-runmodes命令的执行结果的一部分进行了截图。

其中第一列是RunMode Type运行模式类型，值可以为PCAP_DEV,PCAP_FILE,AF_PACKET_DEV等。

第二列是Custom Mode自定义模式，可选值为



Suricata有多种运行模式，这些模式与抓包驱动和IDS、IPS选择相关联。抓包驱动包括：pcap，pcap file，dpdk等。

Suricata在启动时只能选择某种运行模式。如-i表示pcap，-r表示pcap file。

## 1、1 single模式

单工作线程完成所有工作。第一个模块完成抓包，其他模块依次处理，没有后续队列。

![single模式线程如图](picture/single.png)



## 1、2 workers模式（高性能，难度适中）

每个工作线程与single模式单线程工作流程一样，互不影响。

- 线程数量：网卡接口数量 * 每个网卡接口可并发的抓包线程数。
- 线程模块：内部与single模式一致。相当于一个线程内跑多个业务流程。

![img](./picture/workers.png)

其中FlowWorker为流管理模块，RespondReject为响应模块。



我感觉自己对workers模式掌握的并不熟练。

## 1、3 autofp模式（复杂，高性能）

其中autofp模式是最高效的模型。

两种数据包处理线程，分别是收包线程和检测线程。收包线程和检测线程间通过PacketQueue传递数据包进行处理，每个检测线程对应一个队列，多个检测线程时为数据包选择队列采用一致性哈希算法，以确保同一个流的数据包按顺序传递给同一个检测线程。



![image-20250901231520103](./picture/image-20250901231520103.png)

后续我们单独拿一个章节出来，好好讲下autofp模式。



从上面的流程图可以看出autofp属于最高效的模式，也是最复杂的模式，worker适中。



**packet的流水线：**

1、Receive模块抓取数据包后，封装成packet结构体，传递给Decode模块；

2、Decode模块根据数据包的类型解析对应上层协议（IP，TCP，UDP，HTTP协议等）

3、将处理后的数据报继续扔给FlowWorker流管理模块；



**线程模块：**

线程模块是对packet处理任务的抽象。线程模块分为以下几种：

| 模块名            | 作用                                                         |
| ----------------- | ------------------------------------------------------------ |
| Receive模块       | 收集网络数据包，封装成Packet对象后将其传递给Decode线程模块   |
| Decode模块        | Packet按协议4层模型(数据链路层、网络层、传输层、应用层)进行解码，获取协议和负荷信息，解码完成后将Packet传递给FlowWorker线程模块。该模块主要是进行packet解码，不处理应用层。应用层由专门的应用层解码模块处理 |
| FlowWorker模块    | 对packets进行分配flow，Tcp会话管理，TCP重组，应用层数据解析处理，Detect规则检测 |
| RespondReject模块 | 根据detect检测后的结果，对于reject的包需要向双端发送reset包  |



采用pcap file + single模式下，第一个线程中有三个线程模块，分为是ReceivePcapFile、DecodePcapFile、FlowWorker，这块很好理解。

**线程、槽、模块之间关系**

![img](picture/SouthEast.jpeg)

每个线程都包含一个slot的链表，每个slot节点都悬挂着不同的模块，程序执行时会遍历slot链表，按照加入slot链表的顺序执行模块的函数。



从上图中，我们能推测出组成**运行模式**的基本元素：

suricata由线程、线程模块、队列组成。

- 数据包在线程间传递通过队列实现，线程由多个线程模块组成，每个线程模块实现一种功能。
- 一个数据包可以由多个线程处理，数据包将通过队列传递到下一个线程
- 一个线程可以有一个或多个线程模块

这块介绍的东西太过于抽象了，新手一上来是很不好理解的。



线程模块TmModule

线程Slot，TMSlot

AF_PACKET抓包流程是如上所示：

TmThreadsSlotPktAcqLoop-》ReceiveAFPLoop-》AFPReadFromRing-》TmThreadsSlotProcessPkt-》TmThreadsSlotVarRun-》SlotFunc（FlowWorker）



核心数据结构

```
typedef struct RunMode_ {
    /* the runmode type */
    int runmode;//抓包模式的index值
    const char *name;//运行模式的字符串名
    const char *description;//描述信息
    int (*RunModeFunc)(void);//运行模式函数,在RunModeDispatch函数中被调用
} RunMode;

typedef struct RunModes_ {
    int no_of_runmodes;//运行模式的数量
    RunMode *runmodes;//存储不同运行模型下自定义模式的信息
} RunModes;

//划重点
static RunModes runmodes[RUNMODE_USER_MAX];//二维数组，存储运行模式及自定义模式
```



pcap模式，其内部还分为pcap live实时抓包，pcap offline离线抓包模式。

# 二、注册运行模式，填充runmodes表

## 2、1 注册不同运行模式

```c
void RunModeRegisterRunModes(void)
{
    memset(runmodes, 0, sizeof(runmodes));

    RunModeIdsPcapRegister();//ids + pcap 模式
    RunModeFilePcapRegister();//file + pcap 模式
    RunModeIdsAFPRegister();//IDS+AFP 模式（重点关注）
    ......//略
    return;
}
```

我们最关心的是AF_PACKET和pcap模式，以注册AF_PACKET运行模式的RunModeIdsAFPRegister()函数为例进行讲解：

```c
void RunModeIdsAFPRegister(void)
{
    //single运行模式
    RunModeRegisterNewRunMode(RUNMODE_AFP_DEV, "single",
                              "Single threaded af-packet mode",
                              RunModeIdsAFPSingle);
    //workers运行模式
    RunModeRegisterNewRunMode(RUNMODE_AFP_DEV, "workers",
                              "Workers af-packet mode, each thread does all"
                              " tasks from acquisition to logging",
                              RunModeIdsAFPWorkers);
    default_mode_workers = "workers";
    //autofp运行模式
    RunModeRegisterNewRunMode(RUNMODE_AFP_DEV, "autofp",
                              "Multi socket AF_PACKET mode.  Packets from "
                              "each flow are assigned to a single detect "
                              "thread.",
                              RunModeIdsAFPAutoFp);
    return;
}

```

其中每一种运行模式都调用RunModeRegisterNewRunMode函数注册各自的自定义模式。上面注册了AF_PAKCET的single、workers、autofp三种自定义模式。

运行模式的核心数据结构：

RunMode和RunModes结构体的定义如下：

```c
typedef struct RunMode_ {
    /* the runmode type */
    int runmode;//抓包模式的index值
    const char *name;//运行模式的字符串名
    const char *description;//描述信息
    int (*RunModeFunc)(void);//运行模式函数,在RunModeDispatch函数中被调用
} RunMode;

typedef struct RunModes_ {
    int no_of_runmodes;//运行模式的数量
    RunMode *runmodes;//存储不同运行模型下自定义模式的信息
} RunModes;

//划重点
static RunModes runmodes[RUNMODE_USER_MAX];//二维数组，存储运行模式及自定义模式
```

其中全局二维数组runmodes，存储着全部运行模式及自定义模式



进入RunModeRegisterNewRunMode函数，代码如下：

```c
void RunModeRegisterNewRunMode(int runmode, const char *name,
                               const char *description,
                               int (*RunModeFunc)(void))
{
    void *ptmp;
    ptmp = SCRealloc(runmodes[runmode].runmodes,(runmodes[runmode].no_of_runmodes + 1) * sizeof(RunMode));
    runmodes[runmode].runmodes = ptmp;

    RunMode *mode = &runmodes[runmode].runmodes[runmodes[runmode].no_of_runmodes];
    runmodes[runmode].no_of_runmodes++;

    mode->runmode = runmode;
    mode->name = SCStrdup(name);
    mode->description = SCStrdup(description);
    mode->RunModeFunc = RunModeFunc;//此处为注册回调函数

    return;
}

```

根据runmode值去寻找runmodes一级索引的位置，然后对RunMode类型结构体变量并进行赋值。



当将所有运行模式都注册完毕后，其全局二维数组runmodes的情况如下所示：

![image-20230420230948738](picture/image-20230420230948738.png)



重点注意下，之前注册的回调函数RunModeFunc，它在何处被调用呢？

答案：在**RunModeDispatch**函数中被调用。



## 2、2 运行不同模式的回调函数

```
void RunModeDispatch(int runmode, const char *custom_mode,
    const char *capture_plugin_name, const char *capture_plugin_args)
{
   	...省略代码...

	//获取默认的自定义的模式
    if (custom_mode == NULL || strcmp(custom_mode, "auto") == 0) {
        switch (runmode) {
            case RUNMODE_PCAP_DEV:
                custom_mode = RunModeIdsGetDefaultMode();
                break;
            case RUNMODE_PCAP_FILE:
                custom_mode = RunModeFilePcapGetDefaultMode();
                break;
            case RUNMODE_AFP_DEV:
                custom_mode = RunModeAFPGetDefaultMode();
                break;
            default:
                FatalError(SC_ERR_FATAL, "Unknown runtime mode. Aborting");
        }
    }

	//遍历二维数组runmodes，找到第一级索引为runmode，custom_mode的RunMode类型值
    RunMode *mode = RunModeGetCustomMode(runmode, custom_mode);

	//调用模式注册的回调函数RunModeFunc
    mode->RunModeFunc();

   	...省略代码...
}
```



## 2、3 RunModeFunc pcap-file流程

RunModeFunc回调函数我们上面讲过，我们以最简单的RunModeFilePcapSingle为例进行讲解.

```
int RunModeFilePcapSingle(void)
{
	//创建ThreadVars，这块比较多，马上讲解
 	ThreadVars *tv = TmThreadCreatePacketHandler(tname,"packetpool", "packetpool",
                                                 "packetpool", "packetpool","pktacqloop");

	//数据接收模块
    TmModule *tm_module = TmModuleGetByName("ReceivePcapFile");
    TmSlotSetFuncAppend(tv, tm_module, file);

	//数据解析模块
    tm_module = TmModuleGetByName("DecodePcapFile");
    TmSlotSetFuncAppend(tv, tm_module, NULL);

	//流处理模块
    tm_module = TmModuleGetByName("FlowWorker");
    TmSlotSetFuncAppend(tv, tm_module, NULL);

	//设置线程cpu亲和性
    TmThreadSetCPU(tv, WORKER_CPU_SET);

	//创建线程
    TmThreadSpawn(tv);
    return 0;
}
```

RunModeFilePcapSingle函数主要做了下3件事：

- 创建ThreadVars
- 创建不同功能的线程模块，并调用TmSlotSetFuncAppend添加到TmSlot中
- 创建工作线程，即上述所有功能都在这个创建的工作线程中被调用



以数据接收模块举例：

```
//数据接收模块
TmModule *tm_module = TmModuleGetByName("ReceivePcapFile");
TmSlotSetFuncAppend(tv, tm_module, file);
```

操作流程为：先调用TmModuleGetByName获取模块的指针，

```
TmModule *TmModuleGetByName(const char *name)
{
    TmModule *t;
    uint16_t i;

	//遍历tmm_modules模块列表，具体可参考第二章节
    for (i = 0; i < TMM_SIZE; i++) {
        t = &tmm_modules[i];

        if (t->name == NULL)
            continue;

		//模块名称的名字相同，则直接返回
        if (strcmp(t->name, name) == 0)
            return t;
    }

    return NULL;
}
```

然后调用TmSlotSetFuncAppend函数将模块挂载到slot上。

```
void TmSlotSetFuncAppend(ThreadVars *tv, TmModule *tm, const void *data)
{
    TmSlot *slot = SCMalloc(sizeof(TmSlot));
    if (unlikely(slot == NULL))
        return;
    memset(slot, 0, sizeof(TmSlot));
    SC_ATOMIC_INITPTR(slot->slot_data);
    slot->SlotThreadInit = tm->ThreadInit;//线程模块的线程初始化
    slot->slot_initdata = data;//slot槽数据的初始化
    if (tm->Func) {
    	//当为pcap、pcap-file、af_packet模式时，tm->Func为NULL
        slot->SlotFunc = tm->Func;
    } else if (tm->PktAcqLoop) {
        slot->PktAcqLoop = tm->PktAcqLoop;
        if (tm->PktAcqBreakLoop) {
            tv->break_loop = true;
        }
    } else if (tm->Management) {
        slot->Management = tm->Management;
    }
    
    slot->SlotThreadDeinit = tm->ThreadDeinit;//线程模块的线程销毁
    slot->tm_id = TmModuleGetIDForTM(tm); //获取每个模块对应的ID值，方便debug调试查看是flowWorker、decode、receive类型的模块

    if (tv->tm_slots == NULL) {
    	//tm_slots为空，则添加在链表头结点
        tv->tm_slots = slot;
    } else {
        TmSlot *a = (TmSlot *)tv->tm_slots, *b = NULL;

        /* get the last slot */
        //找到最后一个slot槽
        for ( ; a != NULL; a = a->slot_next) {
             b = a;
        }
        //添加到尾部，为保证顺序
        /* append the new slot */
        if (b != NULL) {
            b->slot_next = slot;
        }
    }
    return;
}
```

slot->SlotFunc = tm->Func;此行代码是很重要的。



### 2、3、1 线程模块

上述代码表示采用pcap file + single模式下，第一个线程中有三个线程模块，分为是ReceivePcapFile、DecodePcapFile、FlowWorker，这块很好理解。

**线程、槽、模块之间关系**

![img](picture/SouthEast.jpeg)

每个线程都包含一个slot的链表，每个slot节点都悬挂着不同的模块，程序执行时会遍历slot链表，按照加入slot链表的顺序执行模块的函数。





问题：何处去调用slot->SlotFunc呢？

答：在TmThreadsSlotVarRun函数被调用。

```
TmEcode TmThreadsSlotVarRun(ThreadVars *tv, Packet *p, TmSlot *slot)
{
    for (TmSlot *s = slot; s != NULL; s = s->slot_next) {
        TmEcode r = s->SlotFunc(tv, p, SC_ATOMIC_GET(s->slot_data));

        /* handle new packets */
        while (tv->decode_pq.top != NULL) {
            Packet *extra_p = PacketDequeueNoLock(&tv->decode_pq);
            if (unlikely(extra_p == NULL))
                continue;

            /* see if we need to process the packet */
            if (s->slot_next != NULL) {
                r = TmThreadsSlotVarRun(tv, extra_p, s->slot_next);
                if (unlikely(r == TM_ECODE_FAILED)) {
                    TmThreadsSlotProcessPktFail(tv, s, extra_p);
                    return TM_ECODE_FAILED;
                }
            }
            tv->tmqh_out(tv, extra_p);
        }
    }

    return TM_ECODE_OK;
}

```



### 2、3、2 创建ThreadVars

```
ThreadVars *TmThreadCreatePacketHandler(const char *name, const char *inq_name,
                                        const char *inqh_name, const char *outq_name,
                                        const char *outqh_name, const char *slots)
{
    ThreadVars *tv = NULL;
    tv = TmThreadCreate(name, inq_name, inqh_name, outq_name, outqh_name,slots, NULL, 0);
    return tv;
}
```

TmThreadCreatePacketHandler调用TmThreadCreate函数

#### 1）TmThreadCreate函数

```
ThreadVars *TmThreadCreate(const char *name, const char *inq_name, const char *inqh_name,
                           const char *outq_name, const char *outqh_name, const char *slots,
                           void * (*fn_p)(void *), int mucond)
{
	//分配ThreadVars内存空间
    tv = SCMalloc(sizeof(ThreadVars));
    memset(tv, 0, sizeof(ThreadVars));

	//赋值ThreadVars的inq、tmqh_in、inq_id等字段（重要）
	//如果inq队列名称不等于packetpool，则从tmq_list链表中重新获取
    if (inq_name != NULL && strcmp(inq_name, "packetpool") != 0) {
        tmq = TmqGetQueueByName(inq_name);
        tv->inq = tmq;
    }
    //inq队列的hander
    if (inqh_name != NULL) {
        int id = TmqhNameToID(inqh_name);
        tmqh = TmqhGetQueueHandlerByName(inqh_name);

        tv->tmqh_in = tmqh->InHandler;
        tv->inq_id = (uint8_t)id;
    }

	//赋值ThreadVars的tmqh_out、outq_id、outq、outctx等字段（重要）
    //outq队列的handler名称不为空
    if (outqh_name != NULL) {
        int id = TmqhNameToID(outqh_name);
        tmqh = TmqhGetQueueHandlerByName(outqh_name);

        tv->tmqh_out = tmqh->OutHandler;
        tv->outq_id = (uint8_t)id;

		//如果outq队列名称不等于packetpool，则从tmq_list链表中重新获取
        if (outq_name != NULL && strcmp(outq_name, "packetpool") != 0) {
            tmq = TmqGetQueueByName(outq_name);
            tv->outq = tmq;
            tv->outctx = NULL;
        }
    }

	/*很重要的函数*/
    TmThreadSetSlots(tv, slots, fn_p);

    return tv;
}
```

- tmqh_in：在初始化时绑定为inqh的InHander，用于从上一级队列中获取数据包。
- tmqh_out：在初始化时绑定为outqh的OutHander，用于将处理后的数据包送往下一级队列。



#### 2）TmqGetQueueByName函数

```
Tmq *TmqGetQueueByName(const char *name)
{
    Tmq *tmq = NULL;
    TAILQ_FOREACH(tmq, &tmq_list, next) {
        if (strcmp(tmq->name, name) == 0)
            return tmq;
    }
    return NULL;
}
```

遍历tmq_list，根据name查询对应的Tmq。tmq_list是何时注册的呢？



#### 3）TmqhGetQueueHandlerByName函数

```
Tmqh *TmqhGetQueueHandlerByName(const char *name)
{
    for (int i = 0; i < TMQH_SIZE; i++) {
        if (tmqh_table[i].name != NULL) {
            if (strcmp(name, tmqh_table[i].name) == 0)
                return &tmqh_table[i];
        }
    }

    return NULL;
}
```

根据name从tmqh_table表中查询出对应的handler，具体详情在本文第三章节《填充tmqh_table表》。



#### 4）TmThreadSetSlots函数

```
static TmEcode TmThreadSetSlots(ThreadVars *tv, const char *name, void *(*fn_p)(void *))
{
    if (strcmp(name, "varslot") == 0) {
        tv->tm_func = TmThreadsSlotVar;
    } else if (strcmp(name, "pktacqloop") == 0) {
        tv->tm_func = TmThreadsSlotPktAcqLoop;//执行此行代码
    } else if (strcmp(name, "management") == 0) {
        tv->tm_func = TmThreadsManagement;
    } else if (strcmp(name, "command") == 0) {
        tv->tm_func = TmThreadsManagement;
    }

    return TM_ECODE_OK;
}
```

在pcap file的single模式下，其函数堆栈如下所示，传递给TmThreadSetSlots函数name的实参为“pktacqloop”，所以执行 tv->tm_func = TmThreadsSlotPktAcqLoop;



综上所述，TmThreadCreate函数主要做四件事

1. 分配ThreadVars内存空间
2. 赋值ThreadVars的inq、tmqh_in、inq_id等字段
3. 赋值ThreadVars的tmqh_out、outq_id、outq、outctx等字段
4. 设置slot相关属性



### 2、3、3 创建工作线程

```
TmEcode TmThreadSpawn(ThreadVars *tv)
{
	//创建线程，把tv->tm_func作为线程函数
    pthread_attr_t attr;
    pthread_attr_init(&attr);
    int rc = pthread_create(&tv->t, &attr, tv->tm_func, (void *)tv);

    return TM_ECODE_OK;
}
```

 **问题：tv->tm_func是什么代码赋值的呢？**

在上面的TmThreadSetSlots函数中， tv->tm_func = TmThreadsSlotPktAcqLoop;



## 2、4 RunMode af-packet流程

其中mode->RunModeFunc();RunModeFunc在AF_PACKET运行模式下分别注册了如下函数：

| 运行模式        | 自定义模式 | RunModeFunc注册      |
| --------------- | ---------- | -------------------- |
| RUNMODE_AFP_DEV | single     | RunModeIdsAFPSingle  |
| RUNMODE_AFP_DEV | workers    | RunModeIdsAFPWorkers |
| RUNMODE_AFP_DEV | autofp     | RunModeIdsAFPAutoFp  |

以RunModeIdsAFPWorkers函数为例：

RunModeIdsAFPWorkers函数

​	-> RunModeSetLiveCaptureWorkers函数

​		-> RunModeSetLiveCaptureWorkersForDevice函数

```
static int RunModeSetLiveCaptureWorkersForDevice(ConfigIfaceThreadsCountFunc ModThreadsCount,
                              const char *recv_mod_name,
                              const char *decode_mod_name, const char *thread_name,
                              const char *live_dev, void *aconf,
                              unsigned char single_mode)
{
    int threads_count;

    if (single_mode) {
        threads_count = 1;//单工作线程模式
    } else {
        threads_count = ModThreadsCount(aconf);
        SCLogInfo("Going to use %" PRId32 " thread(s)", threads_count);
    }

    /* create the threads */
    for (int thread = 0; thread < threads_count; thread++) {
        char tname[TM_THREAD_NAME_MAX];
        ThreadVars *tv = NULL;
        TmModule *tm_module = NULL;
        const char *visual_devname = LiveGetShortName(live_dev);
		
		//构造线程名称
        if (single_mode) {
            snprintf(tname, sizeof(tname), "%s#01-%s", thread_name, visual_devname);
        } else {
            snprintf(tname, sizeof(tname), "%s#%02d-%s", thread_name,
                     thread+1, visual_devname);
        }
        tv = TmThreadCreatePacketHandler(tname,
                "packetpool", "packetpool",
                "packetpool", "packetpool",
                "pktacqloop");

        tm_module = TmModuleGetByName(recv_mod_name);//接收模块（“ReceiveAFP”）
        TmSlotSetFuncAppend(tv, tm_module, aconf);

        tm_module = TmModuleGetByName(decode_mod_name);//解码模块（“DecodeAFP”）
        TmSlotSetFuncAppend(tv, tm_module, NULL);

        tm_module = TmModuleGetByName("FlowWorker");//FlowWorker模块
        TmSlotSetFuncAppend(tv, tm_module, NULL);

        TmThreadSetCPU(tv, WORKER_CPU_SET);//设置cpu亲和性

        TmThreadSpawn(tv);//创建线程并和ThreadVars实例关联
    }

    return 0;
}
```

recv_mod_name:接收数据模块的名称

decode_mod_name :解析数据模块的名称

因为大体流程都和pcap file模式差不多，所以就不详细说了，感兴趣的自己看看。

# 三、 注册模块，填充tmm_modules表

## 3、1 模块注册

模块注册的函数调用堆栈为

```
main
	--> PostConfLoadedSetup
		--> RegisterAllModules
```

因为我关心AF_PACKET模块，所以截取了相关的代码。

```
void RegisterAllModules()
{   
    /* managers */
    TmModuleFlowManagerRegister();//流管理模块
    TmModuleFlowRecyclerRegister();//流回收模块

    /* pcap live */
    TmModuleReceivePcapRegister();//接收模块
    TmModuleDecodePcapRegister();//解码模块
    
    /* pcap file */
    TmModuleReceivePcapFileRegister();//接收模块
    TmModuleDecodePcapFileRegister();//解码模块

    /* af-packet */
    TmModuleReceiveAFPRegister();//接收模块
    TmModuleDecodeAFPRegister();//解码模块

    /* flow worker */
    TmModuleFlowWorkerRegister();//flow worker模块
}
```

每种抓包模式都会注册receive和decode两种功能模块，其他还有公用的各种模块如FlowManager模块、FlowRecycle模块、FlowWorker模块等。



### 2、1、1 收包模块注册

```
void TmModuleReceiveAFPRegister (void)
{
    tmm_modules[TMM_RECEIVEAFP].name = "ReceiveAFP";
    tmm_modules[TMM_RECEIVEAFP].ThreadInit = ReceiveAFPThreadInit;
    tmm_modules[TMM_RECEIVEAFP].Func = NULL;
    tmm_modules[TMM_RECEIVEAFP].PktAcqLoop = ReceiveAFPLoop;
    tmm_modules[TMM_RECEIVEAFP].PktAcqBreakLoop = NULL;
    tmm_modules[TMM_RECEIVEAFP].ThreadExitPrintStats = ReceiveAFPThreadExitStats;
    tmm_modules[TMM_RECEIVEAFP].ThreadDeinit = ReceiveAFPThreadDeinit;
}
```

其中PktAcqLoop是数据包获取循环的回调函数。ReceiveAFPLoop此函数是核心。



### 2、1、2 解码模块注册

```
void TmModuleDecodeAFPRegister (void)
{
    tmm_modules[TMM_DECODEAFP].name = "DecodeAFP";
    tmm_modules[TMM_DECODEAFP].ThreadInit = DecodeAFPThreadInit;
    tmm_modules[TMM_DECODEAFP].Func = DecodeAFP;
    tmm_modules[TMM_DECODEAFP].ThreadExitPrintStats = NULL;
    tmm_modules[TMM_DECODEAFP].ThreadDeinit = DecodeAFPThreadDeinit;
    tmm_modules[TMM_DECODEAFP].RegisterTests = NULL;
    tmm_modules[TMM_DECODEAFP].cap_flags = 0;
    tmm_modules[TMM_DECODEAFP].flags = TM_FLAG_DECODE_TM;
}
```



## 3、2 模块结构体和存储结构

```
typedef struct TmModule_ {
    const char *name;

    TmEcode (*ThreadInit)(ThreadVars *, const void *, void **);
    void (*ThreadExitPrintStats)(ThreadVars *, void *);
    TmEcode (*ThreadDeinit)(ThreadVars *, void *);

    /** the packet processing function */
    TmEcode (*Func)(ThreadVars *, Packet *, void *);

    TmEcode (*PktAcqLoop)(ThreadVars *, void *, void *);

    /** terminates the capture loop in PktAcqLoop */
    TmEcode (*PktAcqBreakLoop)(ThreadVars *, void *);

    TmEcode (*Management)(ThreadVars *, void *);

    /** global Init/DeInit */
    TmEcode (*Init)(void);
    TmEcode (*DeInit)(void);
    
    uint8_t cap_flags;   /**< Flags to indicate the capability requierment of
                             the given TmModule */
    /* Other flags used by the module */
    uint8_t flags;
} TmModule;
```

- name：模块的名称
- ThreadInit：初始化线程的函数指针，用于在线程开始之前进行一些初始化操作。
- ThreadExitPrintStats：线程退出并打印统计信息的函数指针。
- ThreadDeinit：线程退出并进行一些清理操作的函数指针。
- Func：实现Suricata的主要功能的函数指针，即数据包处理函数。
- PktAcqLoop：数据包采集循环的函数指针。
- PktAcqBreakLoop：终止数据包采集循环的函数指针。
- Management：管理函数指针，用于管理Suricata的配置和状态等信息。
- Init：初始化函数指针，用于在Suricata启动时进行一些初始化操作。
- DeInit：退出函数指针，用于在Suricata关闭时进行一些清理操作。
- RegisterTests：测试函数指针，用于Suricata的测试和调试。

可以发现模块被保存到了**tmm_modules**数组中，是模块类型(即TmModule)的数组。

![img](picture/webp-16861498396021.webp)

tmm_modules数组的下标为模块ID，常见的模块ID如下：

```
typedef enum {
    TMM_FLOWWORKER,
    
    TMM_RECEIVEPCAP,
    TMM_RECEIVEPCAPFILE,
    
    TMM_DECODEPCAP,
    TMM_DECODEPCAPFILE,
    
    TMM_RECEIVEAFP,
    TMM_DECODEAFP,

    TMM_FLOWMANAGER,
    TMM_FLOWRECYCLER,

    TMM_SIZE,
} TmmId;
```



## 3、3 模块初始化

模块初始化的函数是**TmModuleRunInit**，其调用之处是在RegisterAllModules函数之后，其调用堆栈如下：

```
main
	--> PostConfLoadedSetup
		--> RegisterAllModules(前)
		--> TmModuleRunInit(后)
```



定义如下：

```
void TmModuleRunInit(void)
{
    TmModule *t;
    uint16_t i;

    for (i = 0; i < TMM_SIZE; i++) {
        t = &tmm_modules[i];

        if (t->name == NULL)
            continue;

        if (t->Init == NULL)
            continue;

        t->Init();//实际没啥模块调用Init函数
    }
}
```

该函数执行了模块全局初始化函数Init()。



## 3、4 tmm_modules模块调用

tmm_modules全局变量调用的位置如下图所示：

![image-20230422195441442](picture/image-20230422195441442.png)

以简单的RunModeFilePcapSingle函数举例：

```
int RunModeFilePcapSingle(void)
{
	...省略代码...
	
	//数据接收模块
    TmModule *tm_module = TmModuleGetByName("ReceivePcapFile");
    TmSlotSetFuncAppend(tv, tm_module, file);

	//数据解析模块
    tm_module = TmModuleGetByName("DecodePcapFile");
    TmSlotSetFuncAppend(tv, tm_module, NULL);

	//流处理模块
    tm_module = TmModuleGetByName("FlowWorker");
    TmSlotSetFuncAppend(tv, tm_module, NULL);

	...省略代码...
    return 0;
}
```

调用TmModuleGetByName函数，通过名称获取模块指针。



# 四、 注册队列处理函数，填充tmqh_table表

main函数 -> PostConfLoadedSetup函数 -> TmqhSetup函数



Tmqh结构体的定义如下所示；

```
typedef struct Tmqh_ {
    char *name;
    Packet *(*InHandler)(ThreadVars *);//入流量处理函数
    void (*OutHandler)(ThreadVars *, Packet *);//出流量处理函数
} Tmqh;

Tmqh tmqh_table[TMQH_SIZE];//提供了4种类型队列:simple,flow,packetpool
```

Tmqh_结构的全称是Thread Module Queue Handler



```
void TmqhSetup (void)
{
    memset(&tmqh_table, 0, sizeof(tmqh_table));

    TmqhSimpleRegister();
    TmqhPacketpoolRegister();
    TmqhFlowRegister();
}

//simple注册
void TmqhSimpleRegister (void)
{
    tmqh_table[TMQH_SIMPLE].name = "simple";
    tmqh_table[TMQH_SIMPLE].InHandler = TmqhInputSimple;
    tmqh_table[TMQH_SIMPLE].InShutdownHandler = TmqhInputSimpleShutdownHandler;
    tmqh_table[TMQH_SIMPLE].OutHandler = TmqhOutputSimple;
}

//pool注册
void TmqhPacketpoolRegister (void)
{
    tmqh_table[TMQH_PACKETPOOL].name = "packetpool";
    tmqh_table[TMQH_PACKETPOOL].InHandler = TmqhInputPacketpool;
    tmqh_table[TMQH_PACKETPOOL].OutHandler = TmqhOutputPacketpool;
}

//flow注册
void TmqhFlowRegister(void)
{
    tmqh_table[TMQH_FLOW].name = "flow";
    tmqh_table[TMQH_FLOW].InHandler = TmqhInputFlow;
    tmqh_table[TMQH_FLOW].OutHandlerCtxSetup = TmqhOutputFlowSetupCtx;
    tmqh_table[TMQH_FLOW].OutHandlerCtxFree = TmqhOutputFlowFreeCtx;
    tmqh_table[TMQH_FLOW].RegisterTests = TmqhFlowRegisterTests;

    tmqh_table[TMQH_FLOW].OutHandler = TmqhOutputFlowHash;
    return;
}
```

填充了队列处理程序结构体Tmqh，填充队列处理程序name分别为

- “simple”
- “packetpool”
- “flow”



# 五、注册tmq_list

```
static TAILQ_HEAD(TmqList_, Tmq_) tmq_list = TAILQ_HEAD_INITIALIZER(tmq_list);

//TAILQ_HEAD宏定义
#define TAILQ_HEAD(name, type)						\
struct name {								\
	struct type *tqh_first;	/* first element */			\
	struct type **tqh_last;	/* addr of last next element */		\
}

#define TAILQ_HEAD_INITIALIZER(head)					\
	{ NULL, &(head).tqh_first }
```

经过转换后，tmq_list变量的初始化如下所示：

```
static struct TmqList_ {								\
	struct Tmq_ *tqh_first;	/* first element */			\
	struct Tmq_ **tqh_last;	/* addr of last next element */		\
} tmq_list = { NULL, &(head).tqh_first };
```

调用堆栈：

TmThreadCreate

TmqCreateQueue

TAILQ_INSERT_HEAD



# 六、线程模型-课后题

关于线程模型的一些疑问,大家感兴趣的可以一起在评论区交流

**问题1**：suricata的报文扔给PacketQueue队列后其实后面的解码和流管理、流重组逻辑是串行的吗？

**问题2**：PacketQueue队列传递的那一个地方可以让报文跨线程？

**问题3**：PacketQueue这个对列也是被多次利用在多个地方起多个线程进行调度？

**问题4**：比如抓包和decode的线程数量我设置为2，flow的线程数量设置为4，文件还原的逻辑也改为线程模型然后给线程数量为8，http协议解析线程数量给到8，ips检查匹配的线程为8

**问题5**：这样根据各个阶段进行线程个数的配置，中间通过这个PacketQueue队列来进行报文轮转？



# 七、参考链接


https://blog.csdn.net/ljq32/article/details/122380938

https://zhuanlan.zhihu.com/p/342176316

http://www.hyuuhit.com/2018/03/22/suricata-4-0-3-thread-model/

https://promisechen.github.io/suricata/review.html#id2

https://www.cnblogs.com/zhaodejin/p/16146212.html

https://blog.csdn.net/vevenlcf/article/details/73123770

---

# 八、Runmode 运行模式深入分析（基于源代码）

本章节基于 Suricata 6.0.10 源代码深入剖析 Runmode 运行模式的架构、实现细节和执行流程。

## 8.1 Runmode 架构总览

### 8.1.1 Runmode 类型枚举

### 8.1.2 自定义模式（Custom Modes）

每种 Runmode 类型下可以选择不同的自定义模式，影响线程架构：

```mermaid
graph LR
    subgraph "Custom Modes"
        A[single<br/>单线程模式]
        B[workers<br/>多Worker模式]
        C[autofp<br/>自动Flow负载均衡]
    end

    A --> A1[单线程处理<br/>所有任务]
    B --> B1[每个Worker<br/>独立处理]
    C --> C1[RX线程+<br/>多Worker线程]

    style A fill:#fff9c4
    style B fill:#c5e1a5
    style C fill:#90caf9
```

## 8.2 Runmode 注册与初始化流程

### 8.2.1 注册流程序列图

```mermaid
sequenceDiagram
    participant Main as main()
    participant Post as PostConfLoadedSetup()
    participant Reg as RunModeRegisterRunModes()
    participant AFP as RunModeIdsAFPRegister()
    participant PCAP as RunModeIdsPcapRegister()
    participant Arr as runmodes[]数组

    Main->>Post: 调用
    Post->>Reg: 注册所有运行模式

    Reg->>AFP: 注册AF_PACKET模式
    AFP->>Arr: RunModeRegisterNewRunMode<br/>(RUNMODE_AFP_DEV, "single", ...)
    AFP->>Arr: RunModeRegisterNewRunMode<br/>(RUNMODE_AFP_DEV, "workers", ...)
    AFP->>Arr: RunModeRegisterNewRunMode<br/>(RUNMODE_AFP_DEV, "autofp", ...)

    Reg->>PCAP: 注册PCAP模式
    PCAP->>Arr: RunModeRegisterNewRunMode<br/>(RUNMODE_PCAP_DEV, "single", ...)
    PCAP->>Arr: RunModeRegisterNewRunMode<br/>(RUNMODE_PCAP_DEV, "workers", ...)
    PCAP->>Arr: RunModeRegisterNewRunMode<br/>(RUNMODE_PCAP_DEV, "autofp", ...)

    Note over Arr: runmodes[RUNMODE_AFP_DEV].runmodes[0-2]<br/>runmodes[RUNMODE_PCAP_DEV].runmodes[0-2]<br/>...
```

**关键代码位置：**
- 注册入口：[src/runmodes.c:215](src/runmodes.c#L215) `RunModeRegisterRunModes()`
- AF_PACKET 注册：[src/runmode-af-packet.c:60-80](src/runmode-af-packet.c#L60-L80) `RunModeIdsAFPRegister()`
- PCAP 注册：[src/runmode-pcap.c:215-240](src/runmode-pcap.c#L215-L240) `RunModeIdsPcapRegister()`

### 8.2.2 Runmode 数据结构

```mermaid
classDiagram
    class RunMode {
        +int runmode
        +const char* name
        +const char* description
        +int (*RunModeFunc)(void)
    }

    class RunModes {
        +int cnt
        +RunMode* runmodes
    }

    class GlobalArray {
        RunModes runmodes[RUNMODE_USER_MAX]
    }

    GlobalArray --> RunModes : 包含多个
    RunModes --> RunMode : 包含数组

    note for RunMode "存储单个自定义模式信息\n例如: single, workers, autofp"
    note for RunModes "存储某种Runmode类型\n的所有自定义模式"
    note for GlobalArray "全局二维数组\n索引: [Runmode类型][自定义模式]"
```

**核心数据结构定义：** [src/runmodes.c:80-94](src/runmodes.c#L80-L94)

```c
static RunModes runmodes[RUNMODE_USER_MAX];
// runmodes[RUNMODE_AFP_DEV].runmodes[0] -> {name: "single", RunModeFunc: RunModeIdsAFPSingle}
// runmodes[RUNMODE_AFP_DEV].runmodes[1] -> {name: "workers", RunModeFunc: RunModeIdsAFPWorkers}
// runmodes[RUNMODE_AFP_DEV].runmodes[2] -> {name: "autofp", RunModeFunc: RunModeIdsAFPAutoFp}
```

### 8.2.3 Runmode 启动流程

```mermaid
sequenceDiagram
    participant Main as main()
    participant Disp as RunModeDispatch()
    participant Get as RunModeGetCustomMode()
    participant Func as mode->RunModeFunc()
    participant Setup as RunModeSet***()
    participant Thread as TmThreadCreate()
    participant Mgmt as 管理线程创建

    Main->>Disp: RunModeDispatch(runmode, custom_mode)

    alt custom_mode == "auto" or NULL
        Disp->>Disp: 获取默认模式<br/>RunModeAFPGetDefaultMode()
    end

    Disp->>Get: 查找runmode函数
    Get-->>Disp: 返回 RunMode*

    Disp->>Func: 调用 mode->RunModeFunc()

    alt AF_PACKET AutoFP模式
        Func->>Setup: RunModeIdsAFPAutoFp()
        Setup->>Setup: RunModeSetLiveCaptureAutoFp()
        Setup->>Thread: 创建RX线程
        Setup->>Thread: 创建Worker线程(多个)
    else AF_PACKET Workers模式
        Func->>Setup: RunModeIdsAFPWorkers()
        Setup->>Setup: RunModeSetLiveCaptureWorkers()
        Setup->>Thread: 创建Worker线程(多个)
    else AF_PACKET Single模式
        Func->>Setup: RunModeIdsAFPSingle()
        Setup->>Setup: RunModeSetLiveCaptureSingle()
        Setup->>Thread: 创建单个线程
    end

    Disp->>Mgmt: FlowManagerThreadSpawn()
    Disp->>Mgmt: FlowRecyclerThreadSpawn()
    Disp->>Mgmt: BypassedFlowManagerThreadSpawn()
    Disp->>Mgmt: StatsSpawnThreads()

    Note over Main,Mgmt: 所有线程创建完成，进入运行状态
```

**关键代码位置：**
- 调度入口：[src/runmodes.c:284-409](src/runmodes.c#L284-L409) `RunModeDispatch()`
- AutoFP 设置：[src/util-runmodes.c:88-247](src/util-runmodes.c#L88-L247) `RunModeSetLiveCaptureAutoFp()`
- Workers 设置：[src/util-runmodes.c:331-363](src/util-runmodes.c#L331-L363) `RunModeSetLiveCaptureWorkers()`
- Single 设置：[src/util-runmodes.c:365-396](src/util-runmodes.c#L365-L396) `RunModeSetLiveCaptureSingle()`

## 8.3 三种主要运行模式详解

### 8.3.1 Single 模式 - 单线程模式

**架构特点：**
- 单个线程完成所有任务
- 接收、解码、检测、输出全部串行执行
- 适用于低流量场景或调试

```mermaid
flowchart TD
    Start([开始]) --> Init[初始化线程 W#01]
    Init --> Recv[Receive模块<br/>从网卡/文件读取数据包]
    Recv --> Decode[Decode模块<br/>解码协议层次]
    Decode --> Flow[FlowWorker模块<br/>流管理+检测引擎]
    Flow --> Detect[规则匹配<br/>应用层解析]
    Detect --> Output[RespondReject模块<br/>响应处理]
    Output --> Check{还有数据包?}
    Check -->|是| Recv
    Check -->|否| End([结束])

    style Init fill:#fff9c4
    style Recv fill:#e1f5ff
    style Decode fill:#c5e1a5
    style Flow fill:#90caf9
    style Detect fill:#ce93d8
    style Output fill:#ffab91
```

**线程结构：**

```mermaid
graph LR
    subgraph "Thread: W#01"
        direction LR
        A[TmSlot 1<br/>ReceivePcap/AFP] --> B[TmSlot 2<br/>DecodePcap/AFP]
        B --> C[TmSlot 3<br/>FlowWorker]
        C --> D[TmSlot 4<br/>RespondReject]
    end

    PacketPool[Packet Pool] -.入.-> A
    D -.出.-> PacketPool

    style A fill:#e1f5ff
    style B fill:#c5e1a5
    style C fill:#90caf9
    style D fill:#ffab91
```

**实现代码位置：**
- PCAP: [src/runmode-pcap.c:236-260](src/runmode-pcap.c#L236-L260) `RunModeIdsPcapSingle()`
- AF_PACKET: [src/runmode-af-packet.c:170-195](src/runmode-af-packet.c#L170-L195) `RunModeIdsAFPSingle()`

### 8.3.2 Workers 模式 - 多Worker独立处理

**架构特点：**
- 每个 Worker 线程独立完成全流程
- 无需中央队列分发
- 每个 Worker 绑定到不同的 RSS 队列或网卡

```mermaid
flowchart TB
    subgraph Network["网络接口 / 文件"]
        NIC[网卡 eth0<br/>RSS Queue 0-N]
    end

    subgraph Worker1["Worker Thread #01"]
        R1[Receive] --> D1[Decode] --> F1[FlowWorker] --> O1[Respond]
    end

    subgraph Worker2["Worker Thread #02"]
        R2[Receive] --> D2[Decode] --> F2[FlowWorker] --> O2[Respond]
    end

    subgraph Worker3["Worker Thread #03"]
        R3[Receive] --> D3[Decode] --> F3[FlowWorker] --> O3[Respond]
    end

    subgraph WorkerN["Worker Thread #N"]
        RN[Receive] --> DN[Decode] --> FN[FlowWorker] --> ON[Respond]
    end

    NIC -->|RSS队列0| R1
    NIC -->|RSS队列1| R2
    NIC -->|RSS队列2| R3
    NIC -->|RSS队列N| RN

    O1 --> Pool[Packet Pool]
    O2 --> Pool
    O3 --> Pool
    ON --> Pool

    style Worker1 fill:#e3f2fd
    style Worker2 fill:#e8f5e9
    style Worker3 fill:#fff3e0
    style WorkerN fill:#fce4ec
```

**数据包分配机制：**

```mermaid
graph TD
    A[网卡硬件] -->|RSS Hash| B{硬件队列分发}
    B -->|Queue 0| W1[Worker #01]
    B -->|Queue 1| W2[Worker #02]
    B -->|Queue 2| W3[Worker #03]
    B -->|Queue N| WN[Worker #N]

    W1 --> P1[独立处理<br/>Flow A, B, C]
    W2 --> P2[独立处理<br/>Flow D, E, F]
    W3 --> P3[独立处理<br/>Flow G, H, I]
    WN --> PN[独立处理<br/>Flow X, Y, Z]

    style B fill:#ffeb3b
    style W1 fill:#81c784
    style W2 fill:#81c784
    style W3 fill:#81c784
    style WN fill:#81c784
```

**实现代码位置：**
- 通用实现: [src/util-runmodes.c:331-363](src/util-runmodes.c#L331-L363) `RunModeSetLiveCaptureWorkers()`
- AF_PACKET: [src/runmode-af-packet.c:140-165](src/runmode-af-packet.c#L140-L165) `RunModeIdsAFPWorkers()`

**线程数量计算：**
```c
// src/tm-threads.c:2294-2308
uint16_t ncpus = UtilCpuGetNumProcessorsOnline();
int thread_max = ncpus * threading_detect_ratio; // 默认 ratio = 1.0
// 最小值: 1, 最大值: 1024
```

### 8.3.3 AutoFP 模式 - 自动Flow负载均衡（推荐）

**架构特点：**
- 最高性能的模式
- 接收/解码与检测分离
- 基于 Flow Hash 的负载均衡
- 保证同一 Flow 的数据包顺序

```mermaid
flowchart TB
    subgraph RX["接收线程组 (RX Threads)"]
        direction TB
        R1[RX Thread #01<br/>ReceiveAFP]
        R2[RX Thread #02<br/>ReceiveAFP]

        R1 --> D1[DecodeAFP]
        R2 --> D2[DecodeAFP]
    end

    subgraph FlowQ["Flow Queue Handler<br/>流哈希分发器"]
        direction LR
        Hash[Flow Hash计算<br/>基于五元组]
    end

    subgraph Pickup["Pickup Queues<br/>工作队列"]
        direction LR
        Q1[pickup1]
        Q2[pickup2]
        Q3[pickup3]
        QN[pickupN]
    end

    subgraph Workers["Worker线程组 (W Threads)"]
        direction TB
        W1[Worker #01<br/>FlowWorker]
        W2[Worker #02<br/>FlowWorker]
        W3[Worker #03<br/>FlowWorker]
        WN[Worker #N<br/>FlowWorker]
    end

    NIC[网卡接口] -->|抓包| R1
    NIC -->|抓包| R2

    D1 -->|数据包| Hash
    D2 -->|数据包| Hash

    Hash -->|Hash % N = 0| Q1
    Hash -->|Hash % N = 1| Q2
    Hash -->|Hash % N = 2| Q3
    Hash -->|Hash % N = N| QN

    Q1 -->|队列| W1
    Q2 -->|队列| W2
    Q3 -->|队列| W3
    QN -->|队列| WN

    W1 --> Pool[Packet Pool]
    W2 --> Pool
    W3 --> Pool
    WN --> Pool

    style RX fill:#e1f5ff
    style FlowQ fill:#fff9c4
    style Pickup fill:#ffccbc
    style Workers fill:#c5e1a5
    style Hash fill:#ffeb3b
```

**Flow Hash 分发详细流程：**

```mermaid
sequenceDiagram
    participant RX as RX Thread
    participant Decode as Decode Module
    participant Hash as Flow Hash Handler
    participant Queue as Pickup Queue
    participant Worker as Worker Thread

    RX->>RX: 从网卡接收数据包
    RX->>Decode: 传递原始包
    Decode->>Decode: 解码协议头<br/>(Ethernet/IP/TCP/UDP)
    Decode->>Hash: 解码完成的Packet

    Hash->>Hash: 计算Flow Hash<br/>hash = Hash(src_ip, dst_ip,<br/>src_port, dst_port, proto)
    Hash->>Hash: queue_id = hash % worker_count

    alt queue_id == 0
        Hash->>Queue: 入队 pickup1
        Queue->>Worker: Worker #01 取包
    else queue_id == 1
        Hash->>Queue: 入队 pickup2
        Queue->>Worker: Worker #02 取包
    else queue_id == N
        Hash->>Queue: 入队 pickupN
        Queue->>Worker: Worker #N 取包
    end

    Worker->>Worker: FlowWorker 处理<br/>流管理+检测+输出
    Worker->>Worker: 释放到 Packet Pool
```

**AutoFP 线程 CPU 亲和性：**

```mermaid
graph TB
    subgraph CPU["CPU 核心分配"]
        C1[CPU Core 0-N]
        C2[CPU Core N+1-M]
        C3[CPU Core M+1-K]
    end

    RX[RX Threads] -.绑定.-> C1
    RX -.RECEIVE_CPU_SET.-> C1

    Workers[Worker Threads] -.绑定.-> C2
    Workers -.WORKER_CPU_SET.-> C2

    Mgmt[Management Threads<br/>FM/FR/FB] -.绑定.-> C3
    Mgmt -.MANAGEMENT_CPU_SET.-> C3

    style C1 fill:#e1f5ff
    style C2 fill:#c5e1a5
    style C3 fill:#ffccbc
```

**Flow 哈希调度器配置：**

```yaml
# suricata.yaml
autofp-scheduler: hash    # 默认，基于五元组哈希
# autofp-scheduler: ippair  # 基于IP对哈希
# autofp-scheduler: active-packets  # 已废弃，转为hash
```

**实现代码位置：**
- 通用实现: [src/util-runmodes.c:88-247](src/util-runmodes.c#L88-L247) `RunModeSetLiveCaptureAutoFp()`
- Flow Handler: [src/tmqh-flow.c:210-245](src/tmqh-flow.c#L210-L245) `TmqhOutputFlowHash()`
- Queue 创建: [src/util-runmodes.c:57-84](src/util-runmodes.c#L57-L84) `RunmodeAutoFpCreatePickupQueuesString()`

**AutoFP 优势总结：**
1. **高吞吐量**：接收和检测分离，避免瓶颈
2. **负载均衡**：Hash 算法自动分配负载
3. **Flow 一致性**：同一 Flow 的包发送到同一 Worker
4. **可扩展性**：Worker 数量可根据 CPU 核心数动态调整



## 8.5 线程管理详解

### 8.5.1 ThreadVars 生命周期

```mermaid
stateDiagram-v2
    [*] --> Created: TmThreadCreate()
    Created --> Initialized: TmSlotSetFuncAppend()<br/>添加模块到槽位
    Initialized --> CPUSet: TmThreadSetCPU()<br/>设置CPU亲和性
    CPUSet --> Spawned: TmThreadSpawn()<br/>创建pthread

    Spawned --> InitDone: 线程初始化完成<br/>THV_INIT_DONE
    InitDone --> Running: TmThreadContinueThreads()<br/>THV_USE

    Running --> Paused: THV_PAUSE信号
    Paused --> Running: 恢复运行

    Running --> Stopping: THV_KILL信号
    Stopping --> Deinit: THV_DEINIT<br/>清理资源
    Deinit --> Closed: THV_CLOSED<br/>可join
    Closed --> Dead: pthread_join()<br/>THV_DEAD
    Dead --> [*]
```

**ThreadVars 关键字段：**

```mermaid
classDiagram
    class ThreadVars {
        +pthread_t t
        +char name[16]
        +char* printable_name
        +uint8_t type
        +uint16_t cpu_affinity
        +Tmq* inq
        +Tmq* outq
        +Packet* (*tmqh_in)(ThreadVars*)
        +void (*tmqh_out)(ThreadVars*, Packet*)
        +TmSlot* tm_slots
        +void* (*tm_func)(void*)
        +uint32_t flags
    }

    class TmSlot {
        +TmSlotFunc SlotFunc
        +TmEcode (*PktAcqLoop)(...)
        +TmSlot* slot_next
        +void* slot_data
        +int tm_id
    }

    class Tmq {
        +char* name
        +PacketQueue* pq
    }

    ThreadVars --> TmSlot : tm_slots链表
    ThreadVars --> Tmq : inq/outq
    TmSlot --> TmSlot : slot_next
```

**定义位置：** [src/threadvars.h:59-100](src/threadvars.h#L59-L100)

### 8.5.2 TmSlot 处理流程

```mermaid
flowchart TD
    Start([线程启动]) --> FuncCheck{检查 tm_func}

    FuncCheck -->|pktacqloop| PktLoop[TmThreadsSlotPktAcqLoop]
    FuncCheck -->|varslot| VarSlot[TmThreadsSlotVar]
    FuncCheck -->|management| Mgmt[TmThreadsManagement]

    PktLoop --> AcqLoop[slot->PktAcqLoop执行]
    AcqLoop -->|获取数据包| ProcessPkt[TmThreadsSlotProcessPkt]

    ProcessPkt --> SlotRun[TmThreadsSlotVarRun]

    SlotRun --> Traverse{遍历 tm_slots}
    Traverse -->|有下一个Slot| Execute[执行 slot->SlotFunc]
    Execute -->|Receive| RecvFunc[ReceiveAFP/Pcap]
    Execute -->|Decode| DecodeFunc[DecodeAFP/Pcap]
    Execute -->|FlowWorker| FlowFunc[FlowWorker]
    Execute -->|Respond| RespondFunc[RespondReject]

    RecvFunc --> Traverse
    DecodeFunc --> Traverse
    FlowFunc --> Traverse
    RespondFunc --> Traverse

    Traverse -->|无下一个Slot| Output[tmqh_out输出]
    Output --> AcqLoop

    AcqLoop -->|收到退出信号| Cleanup[清理资源]
    Cleanup --> End([线程结束])

    style PktLoop fill:#e1f5ff
    style SlotRun fill:#c5e1a5
    style Execute fill:#fff9c4
    style Traverse fill:#ffccbc
```

**关键代码位置：**
- Slot 执行: [src/tm-threads.c:3500-3560](src/tm-threads.c#L3500-L3560) `TmThreadsSlotVarRun()`
- Packet 处理: [src/tm-threads.c:3430-3490](src/tm-threads.c#L3430-L3490) `TmThreadsSlotProcessPkt()`
- 抓包循环: [src/tm-threads.c:3240-3360](src/tm-threads.c#L3240-L3360) `TmThreadsSlotPktAcqLoop()`



## 8.6 队列管理系统

### 8.6.1 队列处理器类型

```mermaid
graph TB
    subgraph QueueHandlers["队列处理器 (Tmqh)"]
        A[TMQH_SIMPLE<br/>简单队列]
        B[TMQH_PACKETPOOL<br/>数据包池]
        C[TMQH_FLOW<br/>Flow分发队列]
    end

    A --> A1[InHandler: TmqhInputSimple<br/>OutHandler: TmqhOutputSimple]
    B --> B1[InHandler: TmqhInputPacketpool<br/>OutHandler: TmqhOutputPacketpool]
    C --> C1[InHandler: TmqhInputFlow<br/>OutHandler: TmqhOutputFlowHash]

    A1 --> U1[用途: 简单队列传递<br/>单生产者单消费者]
    B1 --> U2[用途: 数据包内存池<br/>复用Packet结构]
    C1 --> U3[用途: AutoFP模式<br/>Flow哈希负载均衡]

    style A fill:#e1f5ff
    style B fill:#c5e1a5
    style C fill:#fff9c4
```

**定义位置：**
- Tmqh 结构: [src/tm-queuehandlers.h:27-43](src/tm-queuehandlers.h#L27-L43)
- Simple: [src/tmqh-simple.c](src/tmqh-simple.c)
- Packetpool: [src/tmqh-packetpool.c](src/tmqh-packetpool.c)
- Flow: [src/tmqh-flow.c](src/tmqh-flow.c)

### 8.6.2 Flow Queue Handler 详解

```mermaid
flowchart TB
    subgraph FlowCtx["TmqhFlowCtx 上下文"]
        direction TB
        Size[size: Worker数量]
        Last[last: 上次使用的队列]
        Queues[queues: TmqhFlowMode数组]
    end

    subgraph FlowMode["TmqhFlowMode * N"]
        direction LR
        Q1[Queue 1: pickup1]
        Q2[Queue 2: pickup2]
        Q3[Queue 3: pickup3]
        QN[Queue N: pickupN]
    end

    subgraph HashCalc["Hash 计算"]
        direction TB
        Pkt[Packet] --> Extract[提取五元组]
        Extract --> SrcIP[src_ip]
        Extract --> DstIP[dst_ip]
        Extract --> SrcPort[src_port]
        Extract --> DstPort[dst_port]
        Extract --> Proto[protocol]

        SrcIP --> HashFunc[Hash函数]
        DstIP --> HashFunc
        SrcPort --> HashFunc
        DstPort --> HashFunc
        Proto --> HashFunc

        HashFunc --> Mod[hash % size]
        Mod --> Index[队列索引]
    end

    FlowCtx --> FlowMode
    HashCalc --> Index
    Index -->|索引0| Q1
    Index -->|索引1| Q2
    Index -->|索引2| Q3
    Index -->|索引N| QN

    style HashFunc fill:#ffeb3b
    style Mod fill:#ff9800
```

**Flow Hash 核心代码：** [src/tmqh-flow.c:210-245](src/tmqh-flow.c#L210-L245)

```c
void TmqhOutputFlowHash(ThreadVars *t, Packet *p) {
    uint32_t hash = p->flow_hash;  // 五元组哈希值
    TmqhFlowCtx *ctx = (TmqhFlowCtx *)t->outctx;
    uint32_t idx = hash % ctx->size;  // 取模得到队列索引

    PacketQueue *q = ctx->queues[idx].q;
    PacketEnqueue(q, p);  // 入队到对应的 pickup 队列
}
```

### 8.6.3 队列创建与绑定

```mermaid
sequenceDiagram
    participant A as RunModeSetLiveCaptureAutoFp
    participant B as CreatePickupQueues
    participant C as TmThreadCreate
    participant D as QueueManager

    A->>B: 1. 计算worker数量
    Note over B: 生成队列名称<br/>pickup1,pickup2,...,pickupN
    B-->>A: 2. 返回队列名字符串

    loop 为每个Worker创建线程
        A->>C: 3. 创建Worker线程
        Note over C: inq_name = pickupX
        C->>D: 4. TmqGetQueueByName(pickupX)

        alt 队列已存在
            D-->>C: 返回已有队列
        else 队列不存在
            D->>D: 5. TmqCreateQueue(pickupX)
            D->>D: 6. 添加到tmq_list链表
            D-->>C: 返回新队列
        end

        C->>C: 7. tv->inq = queue
        C->>C: 8. tv->tmqh_in = TmqhInputFlow
    end

    Note over A,D: 队列懒加载创建机制<br/>全局链表tmq_list存储所有队列
```

**代码位置：**
- 队列字符串生成: [src/util-runmodes.c:57-84](src/util-runmodes.c#L57-L84) `RunmodeAutoFpCreatePickupQueuesString()`
- 队列查找/创建: [src/tm-queues.c](src/tm-queues.c) `TmqGetQueueByName()`, `TmqCreateQueue()`

## 8.7 完整启动流程综合图

```mermaid
flowchart TD
    Start([Suricata 启动]) --> ParseConfig[解析配置文件]
    ParseConfig --> PostSetup[PostConfLoadedSetup]

    subgraph Registration["注册阶段"]
        RegModules[RegisterAllModules<br/>注册所有模块到tmm_modules]
        RegRunmodes[RunModeRegisterRunModes<br/>注册所有运行模式]
        SetupQH[TmqhSetup<br/>注册队列处理器]
    end

    PostSetup --> RegModules
    RegModules --> RegRunmodes
    RegRunmodes --> SetupQH

    SetupQH --> Dispatch[RunModeDispatch]

    subgraph DispatchPhase["调度阶段"]
        GetMode[获取Custom Mode<br/>默认或配置指定]
        LookupFunc[查找RunModeFunc<br/>从runmodes数组]
        CallFunc[调用RunModeFunc]
    end

    Dispatch --> GetMode
    GetMode --> LookupFunc
    LookupFunc --> CallFunc

    subgraph ThreadCreation["线程创建阶段"]
        direction TB
        CreateRX[创建RX线程<br/>AutoFP模式]
        CreateWorkers[创建Worker线程]
        CreateMgmt[创建管理线程<br/>FM/FR/FB/Stats]
    end

    CallFunc -->|AutoFP| CreateRX
    CallFunc -->|Workers/Single| CreateWorkers
    CreateRX --> CreateWorkers
    CreateWorkers --> CreateMgmt

    subgraph ThreadInit["线程初始化"]
        direction TB
        WaitInit[TmThreadWaitOnThreadInit<br/>等待所有线程初始化]
        SetRuntime[设置运行状态<br/>SURICATA_RUNTIME]
        Continue[TmThreadContinueThreads<br/>唤醒所有线程]
    end

    CreateMgmt --> WaitInit
    WaitInit --> SetRuntime
    SetRuntime --> Continue

    Continue --> MainLoop[SuricataMainLoop<br/>主循环]
    MainLoop --> Running([运行中...])

    style Registration fill:#e1f5ff
    style DispatchPhase fill:#c5e1a5
    style ThreadCreation fill:#fff9c4
    style ThreadInit fill:#ffccbc
```

## 8.8 性能调优建议

### 8.8.1 配置参数优化

```yaml
# suricata.yaml 优化建议

# 1. 线程配置
threading:
  set-cpu-affinity: yes
  cpu-affinity:
    - management-cpu-set:
        cpu: [ 0 ]  # 管理线程独占CPU 0
    - receive-cpu-set:
        cpu: [ 1, 2 ]  # RX线程使用CPU 1-2
    - worker-cpu-set:
        cpu: [ 3-15 ]  # Worker线程使用CPU 3-15
  detect-thread-ratio: 1.0  # Worker数 = CPU核心数 * ratio

# 2. AutoFP 配置
autofp-scheduler: hash  # 使用哈希调度（推荐）

# 3. AF_PACKET 优化
af-packet:
  - interface: eth0
    threads: auto  # 自动检测RSS队列数
    cluster-type: cluster_flow  # Flow级别分发
    defrag: yes
    use-mmap: yes
    ring-size: 20000  # 增大环形缓冲区
    block-size: 32768
```

### 8.8.2 性能对比

| 模式 | 吞吐量 | CPU利用率 | 内存占用 | 复杂度 | 适用场景 |
|------|--------|-----------|----------|--------|----------|
| **Single** | 低 (100-200 Mbps) | 单核100% | 低 | 低 | 调试、低流量 |
| **Workers** | 中-高 (1-5 Gbps) | 多核均衡 | 中 | 中 | 通用场景 |
| **AutoFP** | 高 (5-10+ Gbps) | 多核优化 | 中-高 | 高 | 高性能生产环境 |

## 8.9 调试技巧

### 8.9.1 查看当前运行模式

```bash
# 查看支持的运行模式
suricata --list-runmodes

# 启动时指定运行模式
suricata -c suricata.yaml -i eth0 --runmode=autofp
```

### 8.9.2 GDB 调试断点建议

```bash
# 关键断点位置
b RunModeDispatch              # 运行模式调度
b RunModeSetLiveCaptureAutoFp  # AutoFP设置
b TmThreadCreate               # 线程创建
b TmThreadsSlotVarRun          # Slot执行
b TmqhOutputFlowHash           # Flow哈希分发
b ReceiveAFPLoop               # AF_PACKET接收循环
b FlowWorker                   # FlowWorker处理
```



---

## 8.10 总结

Suricata 的 Runmode 系统是其高性能架构的核心，通过灵活的运行模式和线程模型，能够适应从低流量调试到高性能生产环境的各种场景：

1. **架构设计**：模块化的线程-槽位-模块结构，支持灵活组合
2. **运行模式**：Single、Workers、AutoFP 三种模式满足不同需求
3. **负载均衡**：AutoFP 模式的 Flow Hash 实现智能负载分配
5. **性能优化**：CPU 亲和性、队列管理、RSS 硬件加速

通过深入理解 Runmode 系统，可以更好地配置和优化 Suricata，充分发挥其检测能力和性能潜力。

