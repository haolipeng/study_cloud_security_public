# 五、流建立过程详细图解（基于源代码）

本章节通过多个 Mermaid 图表深入分析 Suricata 流建立的完整过程。

## 5.1 流建立概览图

### 5.1.1 流建立核心数据结构关系图

```mermaid
classDiagram
    class Packet {
        +uint32_t flow_hash
        +uint32_t flags
        +Flow* flow
        +FlowAddress src, dst
        +Port sp, dp
        +uint8_t proto
    }

    class FlowLookupStruct {
        +FlowQueuePrivate spare_queue
        +DecodeThreadVars* dtv
        +FlowQueuePrivate work_queue
        +uint32_t emerg_spare_sync_stamp
    }

    class FlowQueuePrivate {
        +Flow* top
        +Flow* bot
        +uint32_t len
    }

    class FlowBucket {
        +Flow* head
        +Flow* evicted
        +SC_ATOMIC next_ts
        +FBLOCK lock
    }

    class Flow {
        +FlowAddress src, dst
        +Port sp, dp
        +uint8_t proto
        +uint32_t flow_hash
        +uint32_t use_cnt
        +SCTime_t lastts
        +FlowBucket* fb
        +Flow* next
        +uint32_t flags
    }

    class FlowSparePool {
        +FlowQueuePrivate queue
        +FlowSparePool* next
    }

    Packet --> Flow : 关联
    FlowLookupStruct --> FlowQueuePrivate : spare_queue
    FlowLookupStruct --> FlowQueuePrivate : work_queue
    FlowQueuePrivate --> Flow : top/bot
    FlowBucket --> Flow : head链表
    FlowSparePool --> FlowQueuePrivate : 包含
    FlowSparePool --> FlowSparePool : next

    note for Packet "PKT_WANTS_FLOW: 需要流处理<br/>PKT_HAS_FLOW: 已关联流"
    note for FlowLookupStruct "每个线程拥有一个<br/>管理线程本地Flow资源"
    note for FlowBucket "哈希表的桶<br/>flow_hash数组元素"
```

### 5.1.2 流建立整体流程图

```mermaid
flowchart TB
    Start([数据包到达]) --> Decode[解码模块<br/>DecodeIPv4/TCP/UDP等]

    Decode --> SetupPacket[FlowSetupPacket<br/>计算flow_hash<br/>设置PKT_WANTS_FLOW]

    SetupPacket --> CheckFlag{检查<br/>PKT_WANTS_FLOW?}

    CheckFlag -->|否| Skip[跳过流处理]
    CheckFlag -->|是| FlowWorker[FlowWorker函数]

    FlowWorker --> FlowHandle[FlowHandlePacket]

    FlowHandle --> GetFromHash[FlowGetFlowFromHash<br/>从哈希表查找/创建Flow]

    GetFromHash --> GetBucket[根据flow_hash获取FlowBucket]

    GetBucket --> BucketEmpty{Bucket为空?}

    BucketEmpty -->|是| CreateNew1[FlowGetNew<br/>创建新Flow]
    BucketEmpty -->|否| SearchBucket[遍历Bucket链表]

    SearchBucket --> CheckTimeout{检查Flow<br/>是否超时?}

    CheckTimeout -->|超时且use_cnt=0| MoveWork[MoveToWorkQueue<br/>移到工作队列]
    CheckTimeout -->|未超时| Compare{FlowCompare<br/>匹配?}

    Compare -->|匹配成功| Found[找到Flow<br/>返回]
    Compare -->|不匹配| NextFlow{还有下一个?}

    NextFlow -->|是| SearchBucket
    NextFlow -->|否| CreateNew2[FlowGetNew<br/>创建新Flow]

    MoveWork --> NextFlow

    CreateNew1 --> InitFlow1[FlowInit<br/>初始化Flow]
    CreateNew2 --> AddBucket[添加到Bucket链表头]
    AddBucket --> InitFlow2[FlowInit<br/>初始化Flow]

    InitFlow1 --> SetFlag[设置PKT_HAS_FLOW]
    InitFlow2 --> SetFlag
    Found --> SetFlag

    SetFlag --> End([流建立完成])
    Skip --> End

    style Start fill:#e1f5ff
    style End fill:#c5e1a5
    style CreateNew1 fill:#fff9c4
    style CreateNew2 fill:#fff9c4
    style GetFromHash fill:#ffccbc
```

## 5.2 FlowGetFlowFromHash 详细分析

### 5.2.1 FlowGetFlowFromHash 完整序列图

**代码位置：** [src/flow-hash.c](../src/flow-hash.c)

```mermaid
sequenceDiagram
    autonumber
    participant Worker as FlowWorker
    participant Handle as FlowHandlePacket
    participant GetHash as FlowGetFlowFromHash
    participant Bucket as FlowBucket
    participant Flow as Flow对象
    participant GetNew as FlowGetNew
    participant Init as FlowInit

    Worker->>Handle: FlowHandlePacket(tv, fls, p)
    Note over Worker,Handle: 数据包带有PKT_WANTS_FLOW标志

    Handle->>GetHash: FlowGetFlowFromHash(tv, fls, p, &p->flow)

    rect rgb(230, 245, 255)
        Note over GetHash: 阶段1: 获取FlowBucket
        GetHash->>GetHash: hash = p->flow_hash
        GetHash->>GetHash: fb = &flow_hash[hash % hash_size]
        GetHash->>Bucket: FromHashLockBucket(fb)
        Note right of Bucket: 锁定Bucket行级锁
    end

    rect rgb(255, 243, 224)
        Note over GetHash,Bucket: 阶段2: 检查Bucket是否为空
        GetHash->>Bucket: 检查 fb->head == NULL?

        alt Bucket为空
            GetHash->>GetNew: FlowGetNew(tv, fls, p)
            GetNew-->>GetHash: 返回新Flow
            GetHash->>Bucket: fb->head = f
            GetHash->>Init: FlowInit(f, p)
            Init-->>GetHash: 初始化完成
            GetHash-->>Handle: 返回Flow (已锁定)
        else Bucket非空
            Note over GetHash,Flow: 继续遍历Bucket链表
        end
    end

    rect rgb(230, 255, 230)
        Note over GetHash,Flow: 阶段3: 遍历Bucket链表查找匹配Flow
        GetHash->>GetHash: prev_f = NULL, f = fb->head

        loop 遍历链表 (while f != NULL)
            GetHash->>Flow: 检查Flow超时?

            alt Flow超时且use_cnt=0
                GetHash->>Flow: FromHashLockTO(f)
                GetHash->>GetHash: MoveToWorkQueue(tv, fls, fb, f, prev_f)
                Note right of GetHash: 从Bucket移除<br/>加入work_queue
                GetHash->>GetHash: 更新链表指针
            else Flow未超时
                GetHash->>Flow: FlowCompare(f, p)

                alt 匹配成功
                    GetHash-->>Handle: 返回Flow (已锁定)
                else 不匹配
                    GetHash->>GetHash: prev_f = f
                    GetHash->>GetHash: f = f->next
                end
            end
        end
    end

    rect rgb(255, 240, 245)
        Note over GetHash,GetNew: 阶段4: 未找到匹配Flow,创建新Flow
        GetHash->>GetNew: FlowGetNew(tv, fls, p)
        GetNew-->>GetHash: 返回新Flow

        GetHash->>GetHash: f->next = fb->head
        GetHash->>Bucket: fb->head = f
        Note right of Bucket: 插入链表头部

        GetHash->>Init: FlowInit(f, p)
        Init-->>GetHash: 初始化完成
        GetHash-->>Handle: 返回Flow (已锁定)
    end

    Handle->>Handle: p->flags |= PKT_HAS_FLOW
    Handle-->>Worker: 流处理完成

    Note over Worker,Init: 总结:<br/>1. 根据flow_hash定位Bucket<br/>2. 遍历链表查找匹配Flow<br/>3. 处理超时Flow<br/>4. 未找到则创建新Flow
```

### 5.2.2 Bucket链表查找流程图

```mermaid
flowchart TD
    Start([FlowGetFlowFromHash开始]) --> GetBucket[获取FlowBucket<br/>fb = flow_hash[hash % hash_size]]

    GetBucket --> LockBucket[FromHashLockBucket<br/>锁定Bucket]

    LockBucket --> CheckHead{fb->head<br/>== NULL?}

    CheckHead -->|是| CreateFlow1[FlowGetNew<br/>创建新Flow]
    CreateFlow1 --> SetHead[fb->head = f]
    SetHead --> InitFlow1[FlowInit初始化]
    InitFlow1 --> Return1([返回Flow])

    CheckHead -->|否| InitLoop[prev_f = NULL<br/>f = fb->head]

    InitLoop --> LoopStart{f != NULL?}

    LoopStart -->|否| CreateFlow2[FlowGetNew<br/>创建新Flow]
    CreateFlow2 --> InsertHead[f->next = fb->head<br/>fb->head = f]
    InsertHead --> InitFlow2[FlowInit初始化]
    InitFlow2 --> Return2([返回Flow])

    LoopStart -->|是| CheckTimeout{Flow超时?}

    CheckTimeout -->|是| CheckUse{use_cnt == 0?}

    CheckUse -->|是| LockFlow[FromHashLockTO<br/>锁定Flow]
    LockFlow --> MoveWork[MoveToWorkQueue<br/>移到工作队列]
    MoveWork --> UpdateList[更新链表<br/>移除当前Flow]
    UpdateList --> NextFlow1[next_f = 当前next]

    CheckUse -->|否| NextFlow2[继续下一个]

    CheckTimeout -->|否| FlowCompare{FlowCompare<br/>匹配?}

    FlowCompare -->|匹配| Return3([返回Flow])
    FlowCompare -->|不匹配| UpdatePrev[prev_f = f]
    UpdatePrev --> NextFlow2

    NextFlow1 --> LoopStart
    NextFlow2 --> UpdateCurrent[f = f->next]
    UpdateCurrent --> LoopStart

    style Start fill:#e1f5ff
    style Return1 fill:#c5e1a5
    style Return2 fill:#c5e1a5
    style Return3 fill:#c5e1a5
    style CreateFlow1 fill:#fff9c4
    style CreateFlow2 fill:#fff9c4
    style MoveWork fill:#ffccbc
```

## 5.3 FlowGetNew 流创建详细分析

### 5.3.1 FlowGetNew 完整序列图

**代码位置：** [src/flow-hash.c](../src/flow-hash.c) `FlowGetNew()`

```mermaid
sequenceDiagram
    autonumber
    participant Caller as FlowGetFlowFromHash
    participant GetNew as FlowGetNew
    participant Check as FlowCreateCheck
    participant Queue as spare_queue
    participant Spare as FlowSpareSync
    participant Pool as 全局Spare Pool
    participant Alloc as FlowAlloc
    participant Used as FlowGetUsedFlow

    Caller->>GetNew: FlowGetNew(tv, fls, p)

    rect rgb(230, 245, 255)
        Note over GetNew: 阶段1: 紧急模式检查
        GetNew->>GetNew: emerg = flow_flags & FLOW_EMERGENCY
        GetNew->>Check: FlowCreateCheck(p, emerg)

        alt 紧急模式且非TCP SYN
            Check-->>GetNew: 返回0 (不创建)
            GetNew-->>Caller: 返回NULL
        else 可以创建
            Check-->>GetNew: 返回1
        end
    end

    rect rgb(255, 243, 224)
        Note over GetNew,Queue: 阶段2: 从线程spare_queue获取
        GetNew->>Queue: FlowQueuePrivateGetFromTop(&fls->spare_queue)

        alt spare_queue有Flow
            Queue-->>GetNew: 返回Flow
            Note right of Queue: 从队列顶部取出
        else spare_queue为空
            Queue-->>GetNew: 返回NULL
            Note over GetNew: 继续步骤3
        end
    end

    rect rgb(230, 255, 230)
        Note over GetNew,Pool: 阶段3: 从全局Spare Pool获取队列

        alt spare_queue为空
            GetNew->>Spare: FlowSpareSync(tv, fls, p, emerg)

            alt 紧急模式
                Spare->>Spare: 检查时间戳避免频繁获取
                Note right of Spare: emerg_spare_sync_stamp控制
            else 正常模式
                Spare->>Pool: FlowSpareGetFromPool()

                alt Pool第一个队列len>=100 或只有一个队列
                    Pool->>Pool: 取第一个FlowSparePool
                else Pool第一个队列不满但有第二个
                    Pool->>Pool: 取第二个FlowSparePool
                end

                Pool-->>Spare: 返回FlowQueuePrivate
                Spare->>Spare: fls->spare_queue = 返回的队列
                Spare->>Spare: FlowQueuePrivateGetFromTop
                Spare-->>GetNew: 返回Flow
            end
        end
    end

    rect rgb(255, 240, 245)
        Note over GetNew,Alloc: 阶段4: 仍无Flow,检查内存并分配

        alt 仍然没有Flow
            GetNew->>GetNew: FLOW_CHECK_MEMCAP(sizeof Flow)

            alt 内存未超限
                GetNew->>Alloc: FlowAlloc()
                Alloc->>Alloc: SC_ATOMIC_ADD(flow_memuse, size)
                Alloc->>Alloc: SCMalloc(size)
                Alloc->>Alloc: memset(f, 0, size)
                Alloc->>Alloc: FLOW_INITIALIZE(f)
                Alloc-->>GetNew: 返回新Flow
            else 内存超限(紧急情况)
                Note over GetNew,Used: 进入紧急模式<br/>复用已有Flow
                GetNew->>Used: FlowGetUsedFlow(tv, dtv, &p->ts)

                Note over Used: 从flow_hash表搜索可复用Flow
                loop 尝试FLOW_GET_NEW_TRIES次
                    Used->>Used: idx = flow_prune_idx + 5
                    Used->>Used: fb = flow_hash[idx % hash_size]
                    Used->>Used: 遍历Bucket链表

                    alt 找到use_cnt=0且不活跃的Flow
                        Used->>Used: 从Bucket移除
                        Used->>Used: FlowClearMemory(f)
                        Used->>Used: f->flow_end_flags |= FLOW_END_FLAG_FORCED
                        Used-->>GetNew: 返回复用的Flow
                    else 继续查找
                        Used->>Used: 下一个Bucket
                    end
                end
            end
        end
    end

    rect rgb(240, 248, 255)
        Note over GetNew: 阶段5: Flow准备完成
        GetNew->>GetNew: FLOWLOCK_WRLOCK(f)
        GetNew->>GetNew: FlowUpdateCounter(tv, dtv, p->proto)
        GetNew-->>Caller: 返回Flow (已锁定)
    end

    Note over Caller,Used: 总结:<br/>优先级: 线程队列 > 全局Pool > 分配新内存 > 复用旧Flow
```

### 5.3.2 FlowGetNew 决策流程图

```mermaid
flowchart TD
    Start([FlowGetNew开始]) --> CheckEmerg{紧急模式?}

    CheckEmerg -->|是| CheckCreate{FlowCreateCheck<br/>检查是否允许创建?}
    CheckCreate -->|否| ReturnNull1[返回NULL]
    CheckCreate -->|是| FromSpare

    CheckEmerg -->|否| FromSpare[从spare_queue获取]

    FromSpare --> CheckSpare{spare_queue<br/>有Flow?}

    CheckSpare -->|有| GotFlow1[获取到Flow]
    CheckSpare -->|无| SyncPool[FlowSpareSync<br/>从全局Pool同步]

    SyncPool --> CheckEmerg2{紧急模式?}

    CheckEmerg2 -->|是| CheckTime{检查时间戳<br/>避免频繁同步?}
    CheckTime -->|太快| NoFlow[无Flow可用]
    CheckTime -->|允许| GetPool[FlowSpareGetFromPool]

    CheckEmerg2 -->|否| GetPool

    GetPool --> PoolCheck{Pool队列<br/>获取成功?}

    PoolCheck -->|是| UpdateQueue[更新spare_queue]
    UpdateQueue --> GetFromQueue[从队列取Flow]
    GetFromQueue --> GotFlow2[获取到Flow]

    PoolCheck -->|否| NoFlow

    NoFlow --> CheckMemcap{内存未超限?}

    CheckMemcap -->|是| AllocNew[FlowAlloc<br/>分配新Flow]
    AllocNew --> Malloc[SCMalloc分配内存]
    Malloc --> GotFlow3[获取到Flow]

    CheckMemcap -->|否| Emergency[进入紧急模式<br/>FlowGetUsedFlow]

    Emergency --> SearchHash[从flow_hash搜索]
    SearchHash --> TryCount{尝试次数<br/><5?}

    TryCount -->|是| GetBucket[获取Bucket<br/>idx = prune_idx+5]
    GetBucket --> TraverseBucket[遍历Bucket链表]

    TraverseBucket --> CheckUse{use_cnt=0<br/>且不活跃?}

    CheckUse -->|是| RemoveFlow[从Bucket移除]
    RemoveFlow --> ClearFlow[FlowClearMemory]
    ClearFlow --> GotFlow4[获取到Flow]

    CheckUse -->|否| NextBucket[下一个Bucket]
    NextBucket --> TryCount

    TryCount -->|否| ReturnNull2[返回NULL]

    GotFlow1 --> Lock[FLOWLOCK_WRLOCK]
    GotFlow2 --> Lock
    GotFlow3 --> Lock
    GotFlow4 --> Lock

    Lock --> UpdateCounter[FlowUpdateCounter<br/>更新统计]
    UpdateCounter --> Return([返回Flow])

    ReturnNull1 --> End([返回NULL])
    ReturnNull2 --> End

    style Start fill:#e1f5ff
    style Return fill:#c5e1a5
    style End fill:#ffcdd2
    style AllocNew fill:#fff9c4
    style Emergency fill:#ffccbc
```

## 5.4 FlowSpareSync 全局Pool同步

### 5.4.1 FlowSpareGetFromPool 详细流程

**代码位置：** [src/flow-spare-pool.c](../src/flow-spare-pool.c)

```mermaid
sequenceDiagram
    autonumber
    participant Caller as FlowSpareSync
    participant Get as FlowSpareGetFromPool
    participant Lock as Mutex
    participant Pool as flow_spare_pool
    participant Queue as FlowQueuePrivate

    Caller->>Get: FlowSpareGetFromPool()

    Get->>Lock: SCMutexLock(&flow_spare_pool_m)
    Note over Lock: 全局锁保护

    alt flow_spare_pool为NULL
        Get-->>Caller: 返回空队列
    end

    rect rgb(230, 245, 255)
        Note over Get,Pool: 情况1: 第一个Pool队列满或只有一个Pool
        Get->>Pool: 检查 pool->queue.len >= 100?
        Get->>Pool: 或 pool->next == NULL?

        alt 满足条件
            Get->>Pool: p = flow_spare_pool
            Get->>Pool: flow_spare_pool = p->next
            Note right of Pool: 移除第一个Pool

            Get->>Get: flow_spare_pool_flow_cnt -= p->queue.len
            Get->>Get: ret = p->queue
            Get->>Get: SCFree(p)
            Note right of Get: 释放Pool结构<br/>但保留队列
        end
    end

    rect rgb(255, 243, 224)
        Note over Get,Pool: 情况2: 第一个Pool不满但有第二个Pool
        Get->>Pool: 检查 pool->next != NULL?

        alt 有第二个Pool
            Get->>Pool: p = flow_spare_pool->next
            Get->>Pool: flow_spare_pool->next = p->next
            Note right of Pool: 移除第二个Pool

            Get->>Get: flow_spare_pool_flow_cnt -= p->queue.len
            Get->>Get: ret = p->queue
            Get->>Get: SCFree(p)
        end
    end

    Get->>Lock: SCMutexUnlock(&flow_spare_pool_m)
    Get-->>Caller: 返回 FlowQueuePrivate

    Note over Caller,Queue: 返回的队列包含:<br/>- top: 队列头指针<br/>- bot: 队列尾指针<br/>- len: Flow数量(通常100)
```

### 5.4.2 Spare Pool 组织结构图

```mermaid
graph TB
    subgraph Global["全局Spare Pool结构"]
        Head[flow_spare_pool<br/>全局链表头]
        Counter[flow_spare_pool_flow_cnt<br/>总Flow计数]
    end

    subgraph Pool1["第一个FlowSparePool"]
        direction TB
        P1[FlowSparePool 1]
        Q1[FlowQueuePrivate<br/>len=60 不满]
        F1_1[Flow 1]
        F1_2[Flow 2]
        F1_dot[...]
        F1_60[Flow 60]

        P1 --> Q1
        Q1 --> F1_1
        F1_1 --> F1_2
        F1_2 --> F1_dot
        F1_dot --> F1_60
    end

    subgraph Pool2["第二个FlowSparePool"]
        direction TB
        P2[FlowSparePool 2]
        Q2[FlowQueuePrivate<br/>len=100 满]
        F2_1[Flow 1]
        F2_2[Flow 2]
        F2_dot[...]
        F2_100[Flow 100]

        P2 --> Q2
        Q2 --> F2_1
        F2_1 --> F2_2
        F2_2 --> F2_dot
        F2_dot --> F2_100
    end

    subgraph Pool3["第三个FlowSparePool"]
        direction TB
        P3[FlowSparePool 3]
        Q3[FlowQueuePrivate<br/>len=100 满]

        P3 --> Q3
        Q3 --> F3[100 Flows...]
    end

    Head --> Pool1
    Pool1 --> Pool2
    Pool2 --> Pool3
    Pool3 --> Null[NULL]

    Note1["获取策略:<br/>1. 如果Pool1满或只有Pool1,取Pool1<br/>2. 如果Pool1不满但有Pool2,取Pool2<br/>3. Pool2通常是满的"]

    style Global fill:#fff9c4
    style Pool1 fill:#ffebee
    style Pool2 fill:#e8f5e9
    style Pool3 fill:#e1f5ff
    style Note1 fill:#ffe0b2
```

## 5.5 Flow比较与匹配

### 5.5.1 FlowCompare 函数逻辑

**代码位置：** [src/flow-hash.c](../src/flow-hash.c)

```mermaid
flowchart TD
    Start([FlowCompare开始]) --> CheckHash{flow_hash<br/>相等?}

    CheckHash -->|否| NotMatch1[返回0<br/>不匹配]
    CheckHash -->|是| CheckVlan1{vlan_id[0]<br/>相等?}

    CheckVlan1 -->|否| NotMatch2[返回0<br/>不匹配]
    CheckVlan1 -->|是| CheckVlan2{vlan_id[1]<br/>相等?}

    CheckVlan2 -->|否| NotMatch3[返回0<br/>不匹配]
    CheckVlan2 -->|是| CheckProto{proto<br/>相等?}

    CheckProto -->|否| NotMatch4[返回0<br/>不匹配]
    CheckProto -->|是| CheckRecur{recursion_level<br/>相等?}

    CheckRecur -->|否| NotMatch5[返回0<br/>不匹配]
    CheckRecur -->|是| CheckSrcAddr{src地址<br/>相等?}

    CheckSrcAddr -->|是| CheckSrcPort{src端口<br/>相等?}
    CheckSrcPort -->|是| CheckDstAddr{dst地址<br/>相等?}
    CheckDstAddr -->|是| CheckDstPort{dst端口<br/>相等?}
    CheckDstPort -->|是| Match1[返回1<br/>匹配成功]

    CheckSrcAddr -->|否| CheckDstAddr2{dst地址<br/>==pkt.src?}
    CheckSrcPort -->|否| CheckDstAddr2
    CheckDstAddr -->|否| CheckDstAddr2
    CheckDstPort -->|否| CheckDstAddr2

    CheckDstAddr2 -->|是| CheckDstPort2{dst端口<br/>==pkt.sp?}
    CheckDstPort2 -->|是| CheckSrcAddr2{src地址<br/>==pkt.dst?}
    CheckSrcAddr2 -->|是| CheckSrcPort2{src端口<br/>==pkt.dp?}
    CheckSrcPort2 -->|是| Match2[返回1<br/>匹配成功<br/>反向]

    CheckDstAddr2 -->|否| NotMatch6[返回0<br/>不匹配]
    CheckDstPort2 -->|否| NotMatch6
    CheckSrcAddr2 -->|否| NotMatch6
    CheckSrcPort2 -->|否| NotMatch6

    style Start fill:#e1f5ff
    style Match1 fill:#c5e1a5
    style Match2 fill:#c5e1a5
    style NotMatch1 fill:#ffcdd2
    style NotMatch2 fill:#ffcdd2
    style NotMatch3 fill:#ffcdd2
    style NotMatch4 fill:#ffcdd2
    style NotMatch5 fill:#ffcdd2
    style NotMatch6 fill:#ffcdd2
```

### 5.5.2 FlowCompare 匹配条件表

| 比较项 | 条件 | 说明 |
|--------|------|------|
| **flow_hash** | 必须相等 | 五元组计算的哈希值 |
| **vlan_id[0]** | 必须相等 | VLAN ID第一层 |
| **vlan_id[1]** | 必须相等 | VLAN ID第二层 |
| **proto** | 必须相等 | IP协议号(TCP/UDP/ICMP等) |
| **recursion_level** | 必须相等 | 递归级别(隧道层数) |
| **五元组匹配** | 两种方式之一 | 正向或反向匹配 |

**五元组正向匹配：**
- Flow.src == Packet.src
- Flow.sp == Packet.sp
- Flow.dst == Packet.dst
- Flow.dp == Packet.dp

**五元组反向匹配：**
- Flow.src == Packet.dst
- Flow.sp == Packet.dp
- Flow.dst == Packet.src
- Flow.dp == Packet.sp

## 5.6 完整流建立架构图

### 5.6.1 流建立多层次架构

```mermaid
graph TB
    subgraph Layer1["第1层: 数据包接收与解码"]
        Capture[数据包捕获<br/>AF_PACKET/PCAP]
        Decode[解码模块<br/>DecodeIPv4/TCP/UDP]
        Setup[FlowSetupPacket<br/>计算flow_hash]
    end

    subgraph Layer2["第2层: 流处理入口"]
        Worker[FlowWorker]
        Handle[FlowHandlePacket]
        Check{PKT_WANTS_FLOW?}
    end

    subgraph Layer3["第3层: 哈希表查找"]
        GetHash[FlowGetFlowFromHash]
        Bucket[定位FlowBucket<br/>hash % hash_size]
        Search[遍历Bucket链表]
        Compare[FlowCompare匹配]
    end

    subgraph Layer4["第4层: Flow创建"]
        GetNew[FlowGetNew]

        subgraph Sources["Flow来源(优先级)"]
            Src1[1. spare_queue<br/>线程本地队列]
            Src2[2. Spare Pool<br/>全局内存池]
            Src3[3. FlowAlloc<br/>直接分配内存]
            Src4[4. FlowGetUsedFlow<br/>复用旧Flow]
        end
    end

    subgraph Layer5["第5层: Flow初始化"]
        Init[FlowInit]
        SetFields[设置五元组等字段]
        SetFlags[设置Flow状态标志]
        AddBucket[加入Bucket链表]
    end

    subgraph Result["结果"]
        Success[PKT_HAS_FLOW<br/>流关联成功]
        Packet[Packet->flow指针]
    end

    Capture --> Decode
    Decode --> Setup
    Setup --> Check

    Check -->|是| Worker
    Worker --> Handle
    Handle --> GetHash

    GetHash --> Bucket
    Bucket --> Search
    Search --> Compare

    Compare -->|不匹配| GetNew
    GetNew --> Sources

    Src1 --> Init
    Src2 --> Init
    Src3 --> Init
    Src4 --> Init

    Init --> SetFields
    SetFields --> SetFlags
    SetFlags --> AddBucket

    AddBucket --> Success
    Compare -->|匹配| Success
    Success --> Packet

    style Layer1 fill:#e3f2fd
    style Layer2 fill:#fff9c4
    style Layer3 fill:#c5e1a5
    style Layer4 fill:#ffccbc
    style Layer5 fill:#ffe0b2
    style Result fill:#e8f5e9
```

### 5.6.2 Flow生命周期状态图

```mermaid
stateDiagram-v2
    [*] --> 未分配: 程序启动

    未分配 --> Spare池: FlowSparePoolInit<br/>预分配10000个Flow

    Spare池 --> 线程队列: FlowSpareSync<br/>同步到线程spare_queue

    线程队列 --> 使用中: FlowGetNew获取<br/>FlowInit初始化

    使用中 --> 活跃: use_cnt > 0<br/>有Packet引用

    活跃 --> 使用中: use_cnt = 0<br/>无Packet引用

    使用中 --> 超时: 超时检查<br/>lastts过期

    超时 --> work队列: MoveToWorkQueue<br/>use_cnt=0

    work队列 --> 线程队列: 流回收处理<br/>清理资源

    使用中 --> 强制回收: 内存超限<br/>FlowGetUsedFlow

    强制回收 --> 使用中: FlowClearMemory<br/>立即复用

    线程队列 --> Spare池: 线程结束<br/>归还资源

    Spare池 --> [*]: 程序退出

    note right of Spare池
        全局flow_spare_pool
        约100个Pool
        每个Pool 100个Flow
    end note

    note right of 线程队列
        fls->spare_queue
        每个线程独立
    end note

    note right of work队列
        fls->work_queue
        待清理的Flow
    end note
```

## 5.7 关键函数调用链

### 5.7.1 完整调用链序列图

```mermaid
sequenceDiagram
    autonumber
    participant Capture as 抓包线程
    participant Decode as 解码模块
    participant FlowWorker as FlowWorker
    participant FlowHandle as FlowHandlePacket
    participant GetHash as FlowGetFlowFromHash
    participant GetNew as FlowGetNew
    participant Spare as FlowSpareSync
    participant Pool as Spare Pool
    participant Init as FlowInit

    Capture->>Decode: 数据包

    Decode->>Decode: DecodeIPv4(p)
    Decode->>Decode: DecodeTCP(p)
    Decode->>Decode: FlowSetupPacket(p)
    Note right of Decode: 计算flow_hash<br/>设置PKT_WANTS_FLOW

    Decode->>FlowWorker: TmThreadsSlotVarRun<br/>调用FlowWorker

    FlowWorker->>FlowWorker: 检查PKT_WANTS_FLOW
    FlowWorker->>FlowHandle: FlowHandlePacket(tv, fls, p)

    FlowHandle->>GetHash: FlowGetFlowFromHash(tv, fls, p, &p->flow)

    GetHash->>GetHash: 定位FlowBucket
    GetHash->>GetHash: 遍历链表查找

    alt 未找到匹配Flow
        GetHash->>GetNew: FlowGetNew(tv, fls, p)

        alt spare_queue有Flow
            GetNew->>GetNew: FlowQueuePrivateGetFromTop
        else spare_queue为空
            GetNew->>Spare: FlowSpareSync(tv, fls, p, emerg)
            Spare->>Pool: FlowSpareGetFromPool()
            Pool-->>Spare: 返回FlowQueuePrivate
            Spare->>Spare: 更新fls->spare_queue
            Spare->>Spare: FlowQueuePrivateGetFromTop
        end

        GetNew-->>GetHash: 返回Flow

        GetHash->>Init: FlowInit(f, p)
        Init->>Init: 设置五元组
        Init->>Init: 设置时间戳
        Init->>Init: 设置协议等字段
        Init-->>GetHash: 初始化完成
    else 找到匹配Flow
        GetHash->>GetHash: 直接返回
    end

    GetHash-->>FlowHandle: 返回Flow
    FlowHandle->>FlowHandle: p->flags |= PKT_HAS_FLOW
    FlowHandle-->>FlowWorker: 完成

    FlowWorker->>FlowWorker: 继续其他处理<br/>(Detect/Output等)

    Note over Capture,Init: 关键路径:<br/>Decode -> FlowWorker -> FlowHandlePacket<br/>-> FlowGetFlowFromHash -> FlowGetNew<br/>-> FlowSpareSync -> FlowInit
```

## 5.8 性能优化策略图

### 5.8.1 内存管理优化

```mermaid
graph TB
    subgraph Strategy["内存管理策略"]
        direction TB

        subgraph PreAlloc["预分配策略"]
            P1[启动时预分配10000个Flow]
            P2[组织成100个Pool<br/>每个Pool 100个Flow]
            P3[避免运行时频繁malloc]
        end

        subgraph ThreadLocal["线程本地化"]
            T1[每个线程独立spare_queue]
            T2[优先从本地队列获取]
            T3[减少全局锁竞争]
        end

        subgraph LazySync["延迟同步"]
            L1[本地队列空时才同步]
            L2[一次同步整个Pool<br/>约100个Flow]
            L3[紧急模式限制同步频率<br/>emerg_spare_sync_stamp]
        end

        subgraph Emergency["紧急模式"]
            E1[内存超限触发]
            E2[FlowGetUsedFlow复用旧Flow]
            E3[非SYN包不创建新Flow]
            E4[更短的超时时间]
        end
    end

    subgraph Benefits["性能收益"]
        B1[降低内存分配开销]
        B2[减少锁竞争]
        B3[提高缓存命中率]
        B4[支持高并发]
    end

    PreAlloc --> B1
    ThreadLocal --> B2
    ThreadLocal --> B3
    LazySync --> B2
    Emergency --> B4

    style PreAlloc fill:#e3f2fd
    style ThreadLocal fill:#fff9c4
    style LazySync fill:#c5e1a5
    style Emergency fill:#ffccbc
    style Benefits fill:#e8f5e9
```

### 5.8.2 查找优化策略

```mermaid
flowchart LR
    subgraph Optimize["查找优化"]
        direction TB

        O1[哈希表<br/>O1查找复杂度]
        O2[行级锁<br/>Bucket独立锁]
        O3[超时Flow及时清理<br/>减少链表长度]
        O4[FlowCompare快速失败<br/>逐项检查]
    end

    subgraph HashOpt["哈希优化"]
        H1[五元组排序<br/>大地址在前]
        H2[hash_rand随机种子<br/>避免哈希碰撞]
        H3[hash_size可配置<br/>默认65536]
    end

    subgraph LockOpt["锁优化"]
        LK1[Bucket行级锁<br/>细粒度锁定]
        LK2[Flow独立锁<br/>FLOWLOCK]
        LK3[读写锁支持<br/>RWLOCK可选]
    end

    Optimize --> HashOpt
    Optimize --> LockOpt

    style Optimize fill:#e1f5ff
    style HashOpt fill:#fff9c4
    style LockOpt fill:#ffccbc
```

## 5.9 调试与监控

### 5.9.1 关键统计计数器

```mermaid
graph TB
    subgraph Counters["统计计数器"]
        direction TB

        subgraph FlowCounters["Flow计数"]
            FC1[flow.memuse<br/>当前内存使用]
            FC2[flow.spare<br/>备用池Flow数量]
            FC3[flow.active<br/>活跃Flow数量]
            FC4[flow.recycled<br/>回收Flow数量]
        end

        subgraph GetNewCounters["创建统计"]
            GN1[flow.get_used<br/>FlowGetUsedFlow调用次数]
            GN2[flow.get_used_eval<br/>评估次数]
            GN3[flow.get_used_failed<br/>获取失败次数]
            GN4[flow.alloc<br/>直接分配次数]
        end

        subgraph HashCounters["哈希统计"]
            HC1[flow.hash_collisions<br/>哈希碰撞次数]
            HC2[flow.hash_add<br/>添加到哈希表次数]
            HC3[flow.hash_remove<br/>从哈希表移除次数]
        end

        subgraph EmergencyCounters["紧急模式"]
            EC1[flow.emerg_mode_enter<br/>进入次数]
            EC2[flow.emerg_mode_over<br/>退出次数]
            EC3[flow.tcp.syn_no_flow<br/>SYN未创建Flow次数]
        end
    end

    subgraph Tools["调试工具"]
        T1[suricatasc dump-counters]
        T2[GDB断点调试]
        T3[统计日志分析]
    end

    Counters --> Tools

    style FlowCounters fill:#e3f2fd
    style GetNewCounters fill:#fff9c4
    style HashCounters fill:#c5e1a5
    style EmergencyCounters fill:#ffccbc
```

### 5.9.2 GDB调试建议

```bash
# 启动GDB
gdb --args suricata -c suricata.yaml -i eth0

# 流建立相关断点
b FlowSetupPacket              # 设置flow_hash
b FlowHandlePacket             # 流处理入口
b FlowGetFlowFromHash          # 哈希查找
b FlowGetNew                   # 创建新Flow
b FlowSpareSync                # 全局Pool同步
b FlowGetUsedFlow              # 复用旧Flow
b FlowInit                     # Flow初始化
b FlowCompare                  # Flow匹配

# 运行并查看
run

# 查看Packet信息
p p->flow_hash
p p->flags
p/x p->flags & PKT_WANTS_FLOW
p *p->flow

# 查看FlowBucket
p flow_hash[p->flow_hash % flow_config.hash_size]
p *fb->head

# 查看FlowLookupStruct
p *fls
p fls->spare_queue.len
p fls->work_queue.len

# 查看全局Pool
p flow_spare_pool
p flow_spare_pool_flow_cnt
p *flow_spare_pool

# 查看Flow对象
p *f
p f->src
p f->dst
p f->sp
p f->dp
p f->proto
p f->use_cnt
p f->flags
```

## 5.10 常见问题与解答

**Q1: 为什么需要spare_queue和全局Spare Pool两级缓存?**

A: 性能优化策略:
- **spare_queue (线程本地)**: 无锁访问，极快
- **Spare Pool (全局)**: 线程间共享，需要锁
- 优先使用本地队列，空了才同步全局Pool
- 减少锁竞争，提高并发性能

**Q2: FlowGetUsedFlow什么时候被调用?**

A: 极端场景:
- 内存超过配置的memcap限制
- 全局Spare Pool也没有可用Flow
- 系统进入紧急模式
- 从哈希表强制回收use_cnt=0的Flow复用

**Q3: Flow匹配支持双向吗?**

A: 是的，FlowCompare支持双向匹配:
- 正向: Flow(src→dst) == Packet(src→dst)
- 反向: Flow(src→dst) == Packet(dst→src)
- 一个Flow对象同时跟踪两个方向的流量

**Q4: 为什么FlowSpareGetFromPool优先取第二个Pool?**

A: Pool组织策略:
- 第一个Pool可能不满(正在回收Flow)
- 第二个Pool通常是满的(len=100)
- 取满的Pool更高效，一次获得100个Flow
- 避免频繁同步

**Q5: emerg_spare_sync_stamp的作用?**

A: 紧急模式限流:
- 记录上次同步时间戳
- 同一秒内只允许同步一次
- 避免紧急模式下频繁获取全局锁
- 保护系统稳定性

## 5.11 总结

Suricata流建立是一个精心设计的多层次系统:

1. **解码阶段**: 计算flow_hash，设置PKT_WANTS_FLOW
2. **查找阶段**: 在哈希表中查找匹配的Flow
3. **创建阶段**: 未找到则创建新Flow，优先级:
   - 线程本地spare_queue (最快)
   - 全局Spare Pool (需要锁)
   - 直接分配内存 (malloc)
   - 复用旧Flow (紧急模式)
4. **匹配阶段**: FlowCompare支持双向匹配
5. **优化策略**: 预分配、线程本地化、延迟同步、紧急模式

通过这些机制，Suricata能够高效处理大量并发连接，同时保持低延迟和高吞吐量。
