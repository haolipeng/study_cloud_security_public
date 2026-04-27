#状态/进行中

创建时间：2026-04-12
背景：
目标：

| 计划 | 行动 | 检查 | 改善 |
| ---- | ---- | ---- | ---- |
|||||

```
alert tcp any any -> any 102 (msg:"IEC61850_MARK_AARQ"; flow:to_server,established; content:"|03 00|"; depth:2; content:"|60|"; distance:0; within:128; flowbits:set,iec_aarq_seen; noalert;
sid:618500100; rev:1;)
alert tcp any 102 -> any any (msg:"IEC61850_MARK_AARE"; flow:from_server,established; flowbits:isset,iec_aarq_seen; content:"|03 00|"; depth:2; content:"|61|"; distance:0; within:128;
flowbits:set,iec_aare_seen; noalert; sid:618500101; rev:1;)
alert tcp any any -> any 102 (msg:"IEC61850_MARK_INIT_REQ"; flow:to_server,established; flowbits:isset,iec_aare_seen; content:"|03 00|"; depth:2; content:"|a8|"; distance:0; within:256;
flowbits:set,iec_init_req_seen; noalert; sid:618500102; rev:1;)
alert tcp any 102 -> any any (msg:"IEC61850_JUDGEMENT_SESSION_READY"; flow:from_server,established; flowbits:isset,iec_init_req_seen; content:"|03 00|"; depth:2; content:"|a9|"; distance:0;
within:256; flowbits:set,iec_session_ready; sid:618500103; rev:1;)
```


## IEC61850_JUDGEMENT_SESSION_READY

可判断：该 TCP 流已经完成 ACSE 建链和 MMS 初始化，可以认为 MMS 会话已建立，后续的流程在该会话中完成。

1. 看见 AARQ
2. 看见 AARE
3. 看见 Initiate-Request
4. 看见 Initiate-Response

### 对应项目里的
- connect



个人解读：最初状态 -> iec_aarq_seen -> iec_aare_seen -> iec_init_req_seen -> iec_session_ready

只有完整且有序地走完 4 步握手，才会在第 4 步产生告警并设置 `iec_session_ready` 标记。



```
alert tcp any any -> any 102 (msg:"IEC61850_MARK_GET_LD_REQ"; flow:to_server,established; flowbits:isset,iec_session_ready; content:"|03 00|"; depth:2; content:"|a0|"; distance:0; within:256;
content:"|a1|"; distance:0; within:32; content:"|a0 03 80 01 09|"; distance:0; within:64; content:"|a1 02 80 00|"; distance:0; within:32; flowbits:set,iec_get_ld_req; noalert; sid:618500110;
rev:1;)
alert tcp any 102 -> any any (msg:"IEC61850_JUDGEMENT_LOGICAL_DEVICES_DISCOVERY"; flow:from_server,established; flowbits:isset,iec_get_ld_req; content:"|03 00|"; depth:2; content:"|a1|";
distance:0; within:256; content:"|a1|"; distance:0; within:32; sid:618500111; rev:1;)
```

## IEC61850_JUDGEMENT_LOGICAL_DEVICES_DISCOVERY

可判断：该流已经向 IED 发起逻辑设备列表枚举，并收到正常的 getNameList 响应。

- 先看到一个 GetLogicalDevices 形态的 request
- getNameList
- objectClass = domain
- objectScope = vmdSpecific
- 然后又看到了正常 getNameList response

对应项目代码 `GetLogicalDevices`



解读：MMS 会话建立后，客户端通常的第一个操作就是 **获取服务器上有哪些逻辑设备**：

```
Client                                    Server (IED)
  │                                          │
  │  ═══ MMS 会话已建立（iec_session_ready）═══│
  │                                          │
  │  GetNameList-Request                     │
  │  (objectClass=domain, scope=vmd)         │
  │  ─────────────────────────────────────→  │   ← 规则 110
  │                                          │
  │  GetNameList-Response                    │
  │  (listOfIdentifier: "PROT","CTRL",...)   │
  │  ←─────────────────────────────────────  │   ← 规则 111
  │                                          │
  │  客户端现在知道了 IED 上所有逻辑设备名称       │
```

getNameList 服务来获取上述信息。



**110规则解读：**

```
alert tcp any any -> any 102 (
    msg:"IEC61850_MARK_GET_LD_REQ";
    flow:to_server,established;
    flowbits:isset,iec_session_ready;           # 前置条件
    content:"|03 00|"; depth:2;                 # TPKT
    content:"|a0|"; distance:0; within:256;     # Confirmed-RequestPDU
    content:"|a1|"; distance:0; within:32;      # getNameList 服务
    content:"|a0 03 80 01 09|"; distance:0; within:64;  # objectClass=domain
    content:"|a1 02 80 00|"; distance:0; within:32;     # objectScope=vmd
    flowbits:set,iec_get_ld_req;
    noalert;
    sid:618500110; rev:1;
)
```

objectClass=domain，表示逻辑设备（Logical Device）

objectScope=vmd，表示服务器全局（查所有逻辑设备）

`vmdSpecific` + `domain` = **"在服务器级别，列出所有逻辑设备"** = GetServerDirectory



```
alert tcp any any -> any 102 (msg:"IEC61850_MARK_GET_LD_VARS_REQ"; flow:to_server,established; flowbits:isset,iec_session_ready; content:"|03 00|"; depth:2; content:"|a0|"; distance:0; within:256;
content:"|a1|"; distance:0; within:32; content:"|a0 03 80 01 00|"; distance:0; within:64; content:"|81|"; distance:0; within:16; flowbits:set,iec_get_ld_vars_req; noalert; sid:618500112; rev:1;) //112规则

alert tcp any 102 -> any any (msg:"IEC61850_JUDGEMENT_NAMED_VARIABLE_DISCOVERY"; flow:from_server,established; flowbits:isset,iec_get_ld_vars_req; content:"|03 00|"; depth:2; content:"|a1|";
distance:0; within:256; content:"|a1|"; distance:0; within:32; sid:618500113; rev:1;) //113规则
```


## IEC61850_JUDGEMENT_NAMED_VARIABLE_DISCOVERY


可判断：该流已经对某个逻辑设备执行 named variable 列表发现，并收到正常响应。

### 对应项目代码

- GetLogicalDeviceVariables

**规则112解读：**

```
alert tcp any any -> any 102 (
    msg:"IEC61850_MARK_GET_LD_VARS_REQ";
    flow:to_server,established;
    flowbits:isset,iec_session_ready;
    content:"|03 00|"; depth:2;                     # TPKT
    content:"|a0|"; distance:0; within:256;         # Confirmed-RequestPDU
    content:"|a1|"; distance:0; within:32;          # getNameList 服务
    content:"|a0 03 80 01 00|"; distance:0; within:64;  # objectClass=namedVariable
    content:"|81|"; distance:0; within:16;          # objectScope=domainSpecific
    flowbits:set,iec_get_ld_vars_req;
    noalert;
    sid:618500112; rev:1;
)
```

objectClass=namedVariable，表示查找查命名变量列表

objectScope=domainSpecific，表示指定逻辑设备内

如果命中后则设置iec_get_ld_vars_req标记



**规则113解读：**

```
alert tcp any 102 -> any any (
    msg:"IEC61850_JUDGEMENT_NAMED_VARIABLE_DISCOVERY";
    flow:from_server,established;
    flowbits:isset,iec_get_ld_vars_req;           # 前置：请求已标记
    content:"|03 00|"; depth:2;                   # TPKT
    content:"|a1|"; distance:0; within:256;       # Confirmed-ResponsePDU
    content:"|a1|"; distance:0; within:32;        # getNameList 响应
    sid:618500113; rev:1;
)
```

收到Confirmed-ResponsePDU，并且类型为getNameList，响应中的 `listOfIdentifier` 包含了该逻辑设备下所有逻辑节点（LN）的名称。



上述两条检测流程，都依赖于其会话状态iec_session_ready

```
AARQ(100) → AARE(101) → InitReq(102) → InitResp(103)
                                            │
                         ┌──────────────────┤ iec_session_ready
                         │                  │
                         ▼                  ▼
               GetNameList(domain)  GetNameList(namedVariable)
                   Req(110)             Req(112)
                    │                    │
                    ▼                    ▼
                   Resp(111)           Resp(113)
                    │                    │
                    ▼                    ▼
              ✅ 告警1：               ✅ 告警2：
           "逻辑设备发现"          "命名变量发现"
```



```
alert tcp any any -> any 102 (msg:"IEC61850_MARK_GET_DATASET_LIST_REQ"; flow:to_server,established; flowbits:isset,iec_session_ready; content:"|03 00|"; depth:2; content:"|a0|"; distance:0;
within:256; content:"|a1|"; distance:0; within:32; content:"|a0 03 80 01 02|"; distance:0; within:64; content:"|81|"; distance:0; within:16; flowbits:set,iec_get_dataset_list_req; noalert;
sid:618500114; rev:1;) //114规则

alert tcp any 102 -> any any (msg:"IEC61850_JUDGEMENT_DATASET_LIST_DISCOVERY"; flow:from_server,established; flowbits:isset,iec_get_dataset_list_req; content:"|03 00|"; depth:2; content:"|a1|";
distance:0; within:256; content:"|a1|"; distance:0; within:32; sid:618500115; rev:1;) //115规则
```

## IEC61850_JUDGEMENT_DATASET_LIST_DISCOVERY



可判断：该流已经对某个逻辑设备执行数据集（namedVariableList）列表发现，并收到正常响应。

### 对应项目代码

- GetLogicalDeviceDataSets



**规则114解读：**

```
alert tcp any any -> any 102 (
    msg:"IEC61850_MARK_GET_DATASET_LIST_REQ";
    flow:to_server,established;
    flowbits:isset,iec_session_ready;
    content:"|03 00|"; depth:2;                          #  TPKT
    content:"|a0|"; distance:0; within:256;              #  Confirmed-RequestPDU
    content:"|a1|"; distance:0; within:32;               #  getNameList 请求服务
    content:"|a0 03 80 01 02|"; distance:0; within:64;   #  objectClass=namedVariableList ★
    content:"|81|"; distance:0; within:16;               #  objectScope=domainSpecific
    flowbits:set,iec_get_dataset_list_req;
    noalert;
    sid:618500114; rev:1;
)
```

objectClass=namedVariableList，表示查找数据集

objectScope=domainSpecific，表示指定逻辑设备内的范围

如果命中后则设置iec_get_dataset_list_req标记



**规则115解读：**

```
alert tcp any 102 -> any any (
    msg:"IEC61850_JUDGEMENT_DATASET_LIST_DISCOVERY";
    flow:from_server,established;
    flowbits:isset,iec_get_dataset_list_req;      # 前置：数据集请求已标记
    content:"|03 00|"; depth:2;                   # TPKT
    content:"|a1|"; distance:0; within:256;       # Confirmed-ResponsePDU
    content:"|a1|"; distance:0; within:32;        # getNameList 响应
    sid:618500115; rev:1;
)
```

响应中返回的是该 LD 下所有数据集的名称，如 `dsProtMeas`（保护测量数据集）、`dsGOOSEOut`（GOOSE 输出数据集）等。

115规则中的getNameList 响应和114规则中的getNameList 请求服务是一体的。



```
alert tcp any any -> any 102 (msg:"IEC61850_MARK_SPEC_REQ"; flow:to_server,established; flowbits:isset,iec_session_ready; content:"|03 00|"; depth:2; content:"|a0|"; distance:0; within:256;
content:"|a6|"; distance:0; within:48; flowbits:set,iec_spec_req; noalert; sid:618500116; rev:1;) //116规则

alert tcp any 102 -> any any (msg:"IEC61850_JUDGEMENT_VARIABLE_SPEC_RESPONSE"; flow:from_server,established; flowbits:isset,iec_spec_req; content:"|03 00|"; depth:2; content:"|a1|"; distance:0;
within:256; content:"|a6|"; distance:0; within:48; sid:618500117; rev:1;) //117规则
```

## IEC61850_JUDGEMENT_VARIABLE_SPEC_RESPONSE



可判断：该流已经执行变量规格/类型查询，并收到正常的 GetVariableAccessAttributes 响应。

### 对应项目代码

主要对应：

- GetNamedVariableSpecification
- GetNamedVariableLeafChildren
- 以及 ReadNamedVariableCaptureValues 的结构辅助查询


说明当前不是单纯“读值”，而是在做：

- 类型发现
- 结构发现
- 规格查询

**116规则解读：**

```
alert tcp any any -> any 102 (
    msg:"IEC61850_MARK_SPEC_REQ";
    flow:to_server,established;
    flowbits:isset,iec_session_ready;
    content:"|03 00|"; depth:2;                     # TPKT
    content:"|a0|"; distance:0; within:256;         # Confirmed-RequestPDU
    content:"|a6|"; distance:0; within:48;          # getVariableAccessAttributes
    flowbits:set,iec_spec_req;
    noalert;
    sid:618500116; rev:1;
)
```

本规则匹配的是MMS服务**getVariableAccessAttributes**



**117规则解读：**

```
alert tcp any 102 -> any any (
    msg:"IEC61850_JUDGEMENT_VARIABLE_SPEC_RESPONSE";
    flow:from_server,established;
    flowbits:isset,iec_spec_req;                       # 前置：规格请求已标记
    content:"|03 00|"; depth:2;                        # TPKT
    content:"|a1|"; distance:0; within:256;            # Confirmed-ResponsePDU
    content:"|a6|"; distance:0; within:48;             # getVariableAccessAttributes 响应
    sid:618500117; rev:1;
)
```

**getVariableAccessAttributes响应报文中的数据需要尽可能解析出来不？**



```
alert tcp any any -> any 102 (msg:"IEC61850_MARK_DATASET_DIR_REQ"; flow:to_server,established; flowbits:isset,iec_session_ready; content:"|03 00|"; depth:2; content:"|a0|"; distance:0; within:256;
content:"|ac|"; distance:0; within:48; flowbits:set,iec_dataset_dir_req; noalert; sid:618500118; rev:1;)//118规则

alert tcp any 102 -> any any (msg:"IEC61850_JUDGEMENT_DATASET_DIRECTORY_RESPONSE"; flow:from_server,established; flowbits:isset,iec_dataset_dir_req; content:"|03 00|"; depth:2; content:"|a1|";
distance:0; within:256; content:"|ac|"; distance:0; within:48; sid:618500119; rev:1;)//119规则
```

### IEC61850_JUDGEMENT_DATASET_DIRECTORY_RESPONSE



可判断：该流已经执行数据集目录查询（GetNamedVariableListAttributes），并收到正常响应。

### 对应项目代码

- GetNamedVariableListDirectory
- GetNamedVariableListDirectories

### 它说明什么

说明项目正在取：

- 某个 DataSet 的成员列表



**规则118解读：**

```
css
alert tcp any any -> any 102 (
    msg:"IEC61850_MARK_DATASET_DIR_REQ";
    flow:to_server,established;
    flowbits:isset,iec_session_ready;
    content:"|03 00|"; depth:2;                      # TPKT
    content:"|a0|"; distance:0; within:256;          # Confirmed-RequestPDU
    content:"|ac|"; distance:0; within:48;           # getNamedVariableListAttributes
    flowbits:set,iec_dataset_dir_req;
    noalert;
    sid:618500118; rev:1;
)
```

在解析时，需重点关注下getNamedVariableListAttributes服务



**119规则解读：**

```
alert tcp any 102 -> any any (
    msg:"IEC61850_JUDGEMENT_DATASET_DIRECTORY_RESPONSE";
    flow:from_server,established;
    flowbits:isset,iec_dataset_dir_req;                # 前置：数据集目录请求已标记
    content:"|03 00|"; depth:2;                        # TPKT
    content:"|a1|"; distance:0; within:256;            # Confirmed-ResponsePDU
    content:"|ac|"; distance:0; within:48;             # getNamedVariableListAttributes 响应
    sid:618500119; rev:1;
)
```



```
alert tcp any any -> any 102 (msg:"IEC61850_MARK_READ_REQ"; flow:to_server,established; flowbits:isset,iec_session_ready; content:"|03 00|"; depth:2; content:"|a0|"; distance:0; within:256;
content:"|a4|"; distance:0; within:32; flowbits:set,iec_read_req; noalert; sid:618500120; rev:1;)

alert tcp any 102 -> any any (msg:"IEC61850_JUDGEMENT_READ_RESPONSE_RETURNED"; flow:from_server,established; flowbits:isset,iec_read_req; content:"|03 00|"; depth:2; content:"|a1|"; distance:0;
within:256; content:"|a4|"; distance:0; within:32; sid:618500121; rev:1;)
```

## IEC61850_JUDGEMENT_READ_RESPONSE_RETURNED


可判断：该流已经发起 MMS read，并且服务器返回了正常 read response。

### 对应项目代码

覆盖：

- ReadValue
- ReadValuesBatch
- ReadBatchRequests
- ReadNamedVariableCaptureValues

它能判断到：

已经发生真实读值交互，并且服务端返回了正常 read response



**120规则解读：**

```
alert tcp any any -> any 102 (
    msg:"IEC61850_MARK_READ_REQ";
    flow:to_server,established;
    flowbits:isset,iec_session_ready;
    content:"|03 00|"; depth:2;                      #  TPKT 魔数
    content:"|a0|"; distance:0; within:256;          #  Confirmed-RequestPDU
    content:"|a4|"; distance:0; within:32;           #  read [4] 
    flowbits:set,iec_read_req;
    noalert;
    sid:618500120; rev:1;
)
```

关心的是read这个服务，read有两种方式

```
方式A: listOfVariable (逐个变量)         方式B: variableListName (整个数据集)
┌──────────────────────────────┐        ┌──────────────────────────────┐
│ Read:                        │        │ Read:                        │
│   PTOC1$ST$Op$general        │        │   dsProtection               │
│                              │        │     → 一次返回所有成员值        │
│ 返回: 1个值                   │        │ 返回: N个值                    │
└──────────────────────────────┘        └──────────────────────────────┘
精确但低效                               高效但信息量大 → 更危险
```



**121规则解读：**

```
alert tcp any 102 -> any any (
    msg:"IEC61850_JUDGEMENT_READ_RESPONSE_RETURNED";
    flow:from_server,established;
    flowbits:isset,iec_read_req;                       # 前置：读请求已标记
    content:"|03 00|"; depth:2;                        #  TPKT
    content:"|a1|"; distance:0; within:256;            #  Confirmed-ResponsePDU
    content:"|a4|"; distance:0; within:32;             #  read response [4]
    sid:618500121; rev:1;
)
```



总结：

```
PDU 层:
  ✅ AARQ / AARE               (ACSE 关联建立)
  ✅ Initiate-Request/Response  (MMS 初始化)
  ✅ Confirmed-RequestPDU  [A0] (请求方向)
  ✅ Confirmed-ResponsePDU [A1] (响应方向)

MMS 服务层:
  ✅ getNameList            [A1] × 3组  (objectClass: domain/variable/variableList)
  ✅ getVariableAccessAttr  [A6] × 1组
  ✅ getNamedVarListAttr    [AC] × 1组
  ✅ read                   [A4] × 1组
```



在完整探测链中的位置：

```
第一步  GetServerDirectory        → 获取有哪些逻辑设备（LD）        ← 规则 110/111
第二步  GetLogicalDeviceDirectory → 获取 LD 内有哪些命名变量       ← 规则 112/113
第三步  GetDataSetDirectory       → 获取 LD 内有哪些数据集         ← 规则 114/115 ★
第四步  GetDataSetValues          → 读取数据集内的实际值           ← 更深层操作
```



检测 MMS 客户端执行 Read 操作——从侦察阶段正式跨入数据获取阶段，这是攻击链的关键转折点。

```
═══════════════ 侦察阶段 (Discovery) ═══════════════
  第一步  GetServerDirectory         → LD 列表            ← 110/111
  第二步  GetLogicalDeviceDirectory  → 变量名列表         ← 112/113
  第三步  GetDataSetNames            → 数据集名称列表     ← 114/115
  第四步  GetDataDirectory           → 变量类型结构       ← 116/117
  第五步  GetDataSetDirectory        → 数据集成员列表     ← 118/119

═══════════════ 操作阶段 (Action) ═══════════════════
  第六步  Read                       → 读取实际数据值     ← 120/121  ->本组
  第七步  Write                      → 修改/篡改数据值    ← (预计下一组)
```

要很熟悉工控环境下攻击者的攻击过程。


---





以这种偏移量来识别mms协议的会话状态以及字段的值可能会有一定的误报的问题。

而且由于经过了串行的链式规则匹配，效率要低一些。

### 规则 100

alert tcp any any -> any 102 (msg:"IEC61850_MARK_AARQ"; flow:to_server,established; content:"|03 00|"; depth:2; content:"|60|"; distance:0; within:128; flowbits:set,iec_aarq_seen; noalert;
sid:618500100; rev:1;)

- 前 2 字节是 TPKT：03 00
- 后续 128 字节内出现 0x60
	  - 0x60 在 libiec61850 ACSE 解析里就是 AARQ

### 规则 101

alert tcp any 102 -> any any (msg:"IEC61850_MARK_AARE"; flow:from_server,established; flowbits:isset,iec_aarq_seen; content:"|03 00|"; depth:2; content:"|61|"; distance:0; within:128;
flowbits:set,iec_aare_seen; noalert; sid:618500101; rev:1;)


- 只在同一 flow 已经看到 AARQ 的前提下
- 找到 0x61 -> AARE


### 规则 102

alert tcp any any -> any 102 (msg:"IEC61850_MARK_INIT_REQ"; flow:to_server,established; flowbits:isset,iec_aare_seen; content:"|03 00|"; depth:2; content:"|a8|"; distance:0; within:256;
flowbits:set,iec_init_req_seen; noalert; sid:618500102; rev:1;)



- 要求同一 flow 已经到了 AARE 阶段
- 然后匹配 a8 -> MMS initiateRequestPdu 

### 规则 103

alert tcp any 102 -> any any (msg:"IEC61850_JUDGEMENT_SESSION_READY"; flow:from_server,established; flowbits:isset,iec_init_req_seen; content:"|03 00|"; depth:2; content:"|a9|"; distance:0;
within:256; flowbits:set,iec_session_ready; sid:618500103; rev:1;)

1. 要求同一 flow 已经到了 Initiate-Request 戒断
2. 当前又看到 Initiate-Response



### 规则 110 / 111

#### 110

alert tcp any any -> any 102 (msg:"IEC61850_MARK_GET_LD_REQ"; flow:to_server,established; flowbits:isset,iec_session_ready; content:"|03 00|"; depth:2; content:"|a0|"; distance:0; within:256;
content:"|a1|"; distance:0; within:32; content:"|a0 03 80 01 09|"; distance:0; within:64; content:"|a1 02 80 00|"; distance:0; within:32; flowbits:set,iec_get_ld_req; noalert; sid:618500110;
rev:1;)

#### 111

alert tcp any 102 -> any any (msg:"IEC61850_JUDGEMENT_LOGICAL_DEVICES_DISCOVERY"; flow:from_server,established; flowbits:isset,iec_get_ld_req; content:"|03 00|"; depth:2; content:"|a1|";
distance:0; within:256; content:"|a1|"; distance:0; within:32; sid:618500111; rev:1;)


110 要求 request 中同时满足：

- 顶层 MMS request：a0
- 服务是 getNameList：a1
- objectClass = domain：a0 03 80 01 09
- objectScope = vmdSpecific：a1 02 80 00



111 回复侧检查：

- 顶层 confirmedResponse = a1
- service getNameList response = a1


### 规则 112 / 113

#### 112 request

匹配：

- getNameList
- objectClass = namedVariable
- objectScope = domainSpecific

其中：

- a0 03 80 01 00 表示 namedVariable
- 81 是 inner domainSpecific tag
	-  domainSpecific 后面紧跟的是：
	  - 域名长度
	  - 域名内容（LD 名称）

#### 113 response
- 顶层 confirmedResponse = a1
- service getNameList response = a1


### 规则 114 / 115

#### 114 request

匹配：

- getNameList
- objectClass = namedVariableList
- objectScope = domainSpecific
- namedVariableList = 2

#### 115 response



### 规则 116 / 117

#### 116 request

匹配：

- 顶层 confirmed request：a0
- 服务 getVariableAccessAttributes：a6


#### 117 response

- 顶层 confirmed response：a1
- 服务 getVariableAccessAttributes response：a6



### 规则 118 / 119

#### 118 request

匹配：

- 顶层 confirmed request：a0
- 服务 getNamedVariableListAttributes：ac

#### 119 response

匹配：

- 顶层 confirmed response：a1
- 服务 getNamedVariableListAttributes response：ac

### 规则 120 / 121

#### 120 request

匹配：

- 顶层 confirmed request：a0
- 服务 read：a4


#### 121 response

匹配：

- 顶层 confirmed response：a1
- 服务 read response：a4



