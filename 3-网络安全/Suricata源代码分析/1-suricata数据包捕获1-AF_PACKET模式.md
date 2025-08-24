写这个文章前，先提出几个小问题给小伙伴们思考

AF_PACKET是如何和poll函数进行结合的？这样做有什么好处呢？



AFPPeersListAdd为什么要调用这个函数呢？意义在哪里呢？

将已抓包的多个接口信息都存储到全局的链表中。



# 一、抓包模块注册

AF_PACKET抓包模块的注册代码如下所示：

```
void TmModuleReceiveAFPRegister (void)
{
    tmm_modules[TMM_RECEIVEAFP].name = "ReceiveAFP";
    tmm_modules[TMM_RECEIVEAFP].ThreadInit = ReceiveAFPThreadInit;//收包线程初始化
    tmm_modules[TMM_RECEIVEAFP].Func = NULL;//注意：此处Func回调函数赋值为NULL
    tmm_modules[TMM_RECEIVEAFP].PktAcqLoop = ReceiveAFPLoop;//实际的收包函数
    tmm_modules[TMM_RECEIVEAFP].PktAcqBreakLoop = NULL;
    tmm_modules[TMM_RECEIVEAFP].ThreadExitPrintStats = ReceiveAFPThreadExitStats;
    tmm_modules[TMM_RECEIVEAFP].ThreadDeinit = ReceiveAFPThreadDeinit;
    tmm_modules[TMM_RECEIVEAFP].cap_flags = SC_CAP_NET_RAW;
    tmm_modules[TMM_RECEIVEAFP].flags = TM_FLAG_RECEIVE_TM;
}
```

上文讲过tmm_modules是全局的模块列表，收包模块也需要注册到里面。

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



## 1、1 收包线程初始化函数

ThreadInit回调函数被赋值为ReceiveAFPThreadInit。

```
TmEcode ReceiveAFPThreadInit(ThreadVars *tv, const void *initdata, void **data)
{
    AFPIfaceConfig *afpconfig = (AFPIfaceConfig *)initdata;

	//1.申请AFPThreadVars类型的结构体指针ptv
    AFPThreadVars *ptv = SCMalloc(sizeof(AFPThreadVars));
    memset(ptv, 0, sizeof(AFPThreadVars));

    strlcpy(ptv->iface, afpconfig->iface, AFP_IFACE_NAME_LENGTH);
    ptv->iface[AFP_IFACE_NAME_LENGTH - 1]= '\0';

    ptv->livedev = LiveGetDevice(ptv->iface);

	//2.设置ptv的buffer_size,ring_size,block_size,block_timeout等属性
    ptv->buffer_size = afpconfig->buffer_size;
    ptv->ring_size = afpconfig->ring_size;
    ptv->block_size = afpconfig->block_size;
    ptv->block_timeout = afpconfig->block_timeout;

	...省略赋值代码...
	//3.调用AFPPeersListAdd，将其添加到全局变量AFPPeersList peerslist;中,方便程序退出时进行释放peers
    if (AFPPeersListAdd(ptv) == TM_ECODE_FAILED) {
        SCFree(ptv);
        afpconfig->DerefFunc(afpconfig);
        SCReturnInt(TM_ECODE_FAILED);
    }
	
	//4.设置data和datalen
#define T_DATA_SIZE 70000
    ptv->data = SCMalloc(T_DATA_SIZE);
    ptv->datalen = T_DATA_SIZE;
#undef T_DATA_SIZE

    *data = (void *)ptv;

    afpconfig->DerefFunc(afpconfig);

    SCReturnInt(TM_ECODE_OK);
}
```

- 1.申请AFPThreadVars类型的结构体指针ptv
- 2.设置ptv的buffer_size,ring_size,block_size,block_timeout等属性
- 3.调用AFPPeersListAdd，将其添加到全局变量AFPPeersList peerslist;中,方便程序退出时进行释放peers
- 4.设置data和datalen



## 1、2 PktAcqLoop实际收包函数

```
TmEcode ReceiveAFPLoop(ThreadVars *tv, void *data, void *slot)
{
    AFPThreadVars *ptv = (AFPThreadVars *)data;
    struct pollfd fds;
    int r;
    TmSlot *s = (TmSlot *)slot;
    time_t last_dump = 0;
    time_t current_time;
    int (*AFPReadFunc) (AFPThreadVars *);
    uint64_t discarded_pkts = 0;

    ptv->slot = s->slot_next;

	//1.根据条件赋值AFPReadFunc函数指针
    if (ptv->flags & AFP_RING_MODE) {
        if (ptv->flags & AFP_TPACKET_V3) {
            AFPReadFunc = AFPReadFromRingV3;
        } else {
            AFPReadFunc = AFPReadFromRing;
        }
    } else {
        AFPReadFunc = AFPRead;
    }

	//2.如果af_packet的状态为DOWN，则调用AFPCreateSocket创建一个
    if (ptv->afp_state == AFP_STATE_DOWN) {
        r = AFPCreateSocket(ptv, ptv->iface, 1);
        AFPPeersListReachedInc();
    }
   
    if (ptv->afp_state == AFP_STATE_UP) {
        SCLogDebug("Thread %s using socket %d", tv->name, ptv->socket);
        //AFPSynchronizeStart同步其他的抓包线程
        AFPSynchronizeStart(ptv, &discarded_pkts);
    }

    fds.fd = ptv->socket;
    fds.events = POLLIN;

    while (1) {
        /* Start by checking the state of our interface */
		//开始检查接口的状态，如果为Down状态，则AFPTryReopen重新打开AF_PACKET
        if (unlikely(ptv->afp_state == AFP_STATE_DOWN)) {
            do {
                usleep(AFP_RECONNECT_TIMEOUT);
                if (suricata_ctl_flags != 0) {
                    dbreak = 1;
                    break;
                }
                r = AFPTryReopen(ptv);//
                fds.fd = ptv->socket;
            } while (r < 0);
        }

        /* 确保在packet内存池中至少保留有一个packet, 避免在线速时申请数据包 ，这个函数好好看下*/
        PacketPoolWait();

		//调用poll函数，并根据事件集合来决定后续动作
        r = poll(&fds, 1, POLL_TIMEOUT);

        if (r > 0 && (fds.revents & (POLLHUP|POLLRDHUP|POLLERR|POLLNVAL))) {
            if (fds.revents & (POLLHUP | POLLRDHUP)) {//POLLHUP或POLLRDHUP情况
                AFPSwitchState(ptv, AFP_STATE_DOWN);
                continue;
            } else if (fds.revents & POLLERR) {//POLLERR情况
                char c;
                /* 调用recv函数获取errno */
                if (recv(ptv->socket, &c, sizeof c, MSG_PEEK) != -1)
                    continue; /* what, no error? */
                SCLogError(SC_ERR_AFP_READ,"Error reading data from iface '%s': (%d) %s",
                           ptv->iface, errno, strerror(errno));
                AFPSwitchState(ptv, AFP_STATE_DOWN);
                continue;
            } else if (fds.revents & POLLNVAL) {//POLLNVAL情况，无效的poll请求
                SCLogError(SC_ERR_AFP_READ, "Invalid polling request");
                AFPSwitchState(ptv, AFP_STATE_DOWN);
                continue;
            }
        } else if (r > 0) {
            r = AFPReadFunc(ptv);
            //根据AFPReadFunc函数的返回值来判断数据包读取成功或失败
            switch (r) {
                case AFP_READ_OK://数据包读取成功
                    break;
                case AFP_READ_FAILURE://数据包读取失败
                    /* AFPRead in error: best to reset the socket */
                    SCLogError(SC_ERR_AFP_READ,"AFPRead error reading data from iface '%s': (%d) %s",
                           ptv->iface, errno, strerror(errno));
                    AFPSwitchState(ptv, AFP_STATE_DOWN);
                    continue;
            }
        } else if (unlikely(r == 0)) {//poll函数超时
            /* 走超时处理流程TmThreadsCaptureHandleTimeout*/
            TmThreadsCaptureHandleTimeout(tv, NULL);

        } else if ((r < 0) && (errno != EINTR)) {//从网口读数据时发生错误，改变状态为AFP_STATE_DOWN
            SCLogError(SC_ERR_AFP_READ, "Error reading data from iface '%s': (%d) %s",ptv->iface,errno, strerror(errno));
            AFPSwitchState(ptv, AFP_STATE_DOWN);
            continue;
        }
    }

    SCReturnInt(TM_ECODE_OK);
}
```

其主要收包的核心流程为 while循环 + poll函数调用，然后处理

其总体流程如下所示：





## 1、3  收包线程销毁函数

ThreadDeinit回调函数被赋值为ReceiveAFPThreadDeinit



# 二、抓包模块调用栈

AF_PACKET抓包流程，其抓包如下所示：

```
TmThreadsSlotPktAcqLoop
	ReceiveAFPLoop
		AFPReadFromRing
			TmThreadsSlotProcessPkt
				TmThreadsSlotVarRun
					SlotFunc
```

抓包函数调用流程。



https://blog.csdn.net/superbfly/article/details/52677730



