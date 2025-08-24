## Suricata新增协议解析

个人总结，新增协议解析分3步进行

-   生成新协议的解析模板
-   修改协议解析器实现
-   修改输出模块实现

## 1\. 创建协议解析模板

对于新增协议解析，Suricata提供脚本自动生成框架代码patch，示例解析器，示例输出模块。一般在使用脚本生成模板后，仅需修改解析器实现和输出模块实现即可。

例：创建pop3协议的解析模板：

```
python scripts/setup-app-layer.py Pop3
```

命令执后如下文件被修改或创建：（具体内容见附件）

框架代码文件

| 文件名 | 修改内容 |
| --- | --- |
| src/Makefile.am | 添加新解析器源码文件和输出模块源码文件 |
| src/app-layer-detect-proto.c | 新增协议名与协议号映射 |
| src/app-layer-parser.c | 新增解析器注册入口 |
| src/app-layer-protos.c | 新增协议名与协议号映射 |
| src/app-layer-protos.h | 添加新协议号枚举 |
| src/output.c | 添加输出模块注册入口 |
| src/suricata-common.h | 添加输出模块号枚举 |
| src/util-profiling.c | ？ |
| suricata.yaml.in | 开启协议解析配置，开启输出输出配置 |

协议解析器代码

| 文件名 | 修改内容 |
| --- | --- |
| src/app-layer-pop3.c | 解析器实现 |
| src/app-layer-pop3.h | \- |

输出模块代码

| 文件名 | 修改内容 |
| --- | --- |
| src/output-json-pop3.c | 输出模块实现 |
| src/output-json-pop3.h | \- |

## 2\. 解析器实现

解析器可以视为Suricata实现协议解析的插件。Suricata在处理流量时，通过解析框架调用解析器预先注册的相关接口，完成从流量到协议字段的还原过程。

解析器一般由如下部分构成：

-   初始化接口
-   上下文管理类接口
-   解析接口

初始化接口用于注册各种框架机制的回调函数，上下文管理类接口用于框架内部获取和设置协议解析状态，解析接口用于接受框架输入的流量数据，解析协议并记录信息到上下文中，以供检测引擎和日志输出模块使用。

### 附件源码 app-[layer](https://so.csdn.net/so/search?q=layer&spm=1001.2101.3001.7020)\-pop3.c

附件中RegisterPop3Parsers为解析器初始化接口，其向Suricata应用层框架注册了如下接口：

| 回调注册接口 | 回调接口 | 回调注册接口描述 |
| --- | --- | --- |
| AppLayerProtoDetectPPRegister | Pop3ProbingParserTs/Pop3ProbingParserTc | 注册协议识别端口，方向，过滤接口等 |
| AppLayerParserRegisterStateFuncs | Pop3StateAlloc/Pop3StateFree | 注册协议解析上下文，内存申请释放接口 |
| AppLayerParserRegisterParser | Pop3ParseRequest/Pop3ParseResponse | 注册数据入口函数 |
| AppLayerParserRegisterTxFreeFunc | Pop3StateTxFree | 注册交互上下文内存释放接口 |
| AppLayerParserRegisterLoggerFuncs | Pop3GetTxLogged/Pop3SetTxLogged | 注册交互上下文，已输出标志获取和设置接口 |
| AppLayerParserRegisterGetTxCnt | Pop3GetTxCnt | 注册交互上下文当前数量获取接口 |
| AppLayerParserRegisterGetStateProgressCompletionStatus | Pop3GetAlstateProgressCompletionStatus | 注册交互上下文结束状态获取接口 |
| AppLayerParserRegisterGetStateProgressFunc | Pop3GetStateProgress | 注册交互上下文当前状态获取接口 |
| AppLayerParserRegisterGetTx | Pop3GetTx | 注册交互上下文获取接口 |
| AppLayerParserRegisterDetectStateFuncs | Pop3GetTxDetectState/Pop3SetTxDetectState | 注册交互上下文中记录的，检测结果获取和设置接口 |

**调用顺序**  
app-layer:

-   Pop3ProbingParserTs/Pop3ProbingParserTc 识别应用层协议
-   Pop3StateAlloc 创建应用层协议解析上下文
-   Pop3ParseRequest/Pop3ParseResponse 解析应用层数据

output:

-   Pop3GetTxCnt 获取当前应用层交互个数
-   Pop3GetTx 获取一个应用层交互
-   Pop3GetAlstateProgressCompletionStatus & Pop3GetStateProgress 获取应用层交互当前状态和结束状态
-   Pop3GetTxLogged & Pop3SetTxLogged 获取应用层交互是否已输出，设置应用层交互已输出

对于一般协议，仅需根据协议格式，修改Pop3ParseRequest/Pop3ParseResponse 解析实现即可。主要是将packet中信息记录到应用层交互上下文中，以供检测模块或输出模块使用。需要注意，交互上下文的状态会影响其内存释放，异常检测和日志输出。

## 3\. 输出模块实现

输出模块可以视为Suricata的日志输出插件。Suricata会在每包解析结束后检查各类输出模块是否达到输出条件，对于符合输出条件的输出模块，调用其回调接口。

Suricata还对输出模块进行了分类，用于输出不同类型的日志。个人理解其是按照输出时机进行分类的：

-   OutputRegisterPacketModule 每包解析完成后调用的包输出模块
-   OutputRegisterTxModule 每个应用层协议交互完成后调用的交互输出模块
-   OutputRegisterFlowModule 每个Flow结束（超时）后或重用时调用的流输出模块
-   OutputRegisterFileModule 每个文件还原开始后到输出模块返回大于1之前调用的文件信息（不包括文件内容）输出模块
-   OutputRegisterFiledataModule 每个文件还原开始后到文件还原完成或终止前调用的文件内容输出模块
-   OutputRegisterStreamingModule tcp重组完成后（ack触发？）调用的stream数据输出模块
-   OutputRegisterStatsModule StatsMgmtThread线程按时间间隔调用的状态输出模块

输出模块一般由如下三部分构成：

-   输出模块初始化接口（用于初始化输出模块）（注册各种框架机制的回调函数和配置参数）
-   输出模块上下文管理接口（用于初始化输出模块的上下文）（初始化输出通道和相关配置参数）
-   日志拼接与输出接口（用于格式化和输出）（将协议解析上下文格式化后，写入本模块对应的通道）

### 附件源码 output-json-pop3.c

通常应用层协议输出元数据通常采用 OutputRegisterTxModule 注册应用交互输出模块。附件源码 output-json-pop3.c 中，调用OutputRegisterTxSubModule 注册了交互输出子模块。其与普通交互输出模块区别是，可以共享父模块的输出通道和配置参数。

output-json-pop3.c 中JsonPop3LogRegister为输出模块初始化接口。其注册的信息介绍如下：

-   输出模块号 LOGGER\_JSON\_POP3
-   父输出模块名 eve-log
-   输出模块名 JsonPop3Log
-   输出模块配置选项关键字 eve-log.pop3
-   输出模块初始化接口 OutputPop3LogInitSub
-   输出模块对应的应用层协议号 ALPROTO\_POP3
-   输出入口函数 JsonPop3Logger
-   输出模块线程上下文初始化接口 JsonPop3LogThreadInit
-   输出模块线程上下文销毁接口 JsonPop3LogThreadDeinit
-   输出模块线程结束时状态打印接口 NULL

**调用顺序**：

1.  **OutputPop3LogInitSub 初始化输出通道相关设置**
    -   初始化此输出模块的上下文
    -   复用父模块的输出通道
    -   注册输出模块上下文销毁接口
    -   指明传输层为IPPROTO\_TCP，应用层ALPROTO\_POP3的协议存在对应的输出模块
    -   返回使用此输出模块上下文封装的，统一的输出模块结构 OutputInitResult
2.  **JsonPop3LogThreadInit 初始化此输出模块的线程专用上下文**
    -   构建线程上下文结构
    -   添加一段buffer用于输出时拼接数据
    -   设置此输出模块的上下文到线程上下文中
3.  **JsonPop3Logger 格式化已解析的数据并写入输出通道**
    -   取得本次要输出的应用协议交互（包含要输出的信息）
    -   取得此输出模块的线程上下文（包含输出通道信息）
    -   拼接数据到线程专用buffer
    -   输出线程专用buffer
    -   销毁线程上下文
4.  **JsonPop3LogThreadDeinit 释放线程上下文**
    -   释放此输出模块的线程上下文

对于注册子输出模块的情况，无需初始化输出模块上下文。其父输出模块已经根据配置创建了输出通道。仅需修改JsonPop3Logger中数据格式化逻辑即可。

## 附件

```
diff --git a/src/Makefile.am b/src/Makefile.am
index 8999f5d..f458d75 100755
--- a/src/Makefile.am
+++ b/src/Makefile.am
@@ -47,6 +47,7 @@ app-layer-tftp.c app-layer-tftp.h \
 app-layer-ikev2.c app-layer-ikev2.h \
 app-layer-krb5.c app-layer-krb5.h \
 app-layer-dhcp.c app-layer-dhcp.h \
+app-layer-pop3.c app-layer-pop3.h \
 app-layer-template.c app-layer-template.h \
 app-layer-template-rust.c app-layer-template-rust.h \
 app-layer-rdp.c app-layer-rdp.h \
@@ -350,6 +351,7 @@ output-json-ikev2.c output-json-ikev2.h \
 output-json-krb5.c output-json-krb5.h \
 output-json-dhcp.c output-json-dhcp.h \
 output-json-snmp.c output-json-snmp.h \
+output-json-pop3.c output-json-pop3.h \
 output-json-template.c output-json-template.h \
 output-json-template-rust.c output-json-template-rust.h \
 output-json-rdp.c output-json-rdp.h \
diff --git a/src/app-layer-detect-proto.c b/src/app-layer-detect-proto.c
index 7d6f985..3f016f3 100644
--- a/src/app-layer-detect-proto.c
+++ b/src/app-layer-detect-proto.c
@@ -869,6 +869,8 @@ static void AppLayerProtoDetectPrintProbingParsers(AppLayerProtoDetectProbingPar
                         printf("            alproto: ALPROTO_SIP\n");
                     else if (pp_pe->alproto == ALPROTO_TEMPLATE_RUST)
                         printf("            alproto: ALPROTO_TEMPLATE_RUST\n");
+                    else if (pp_pe->alproto == ALPROTO_POP3)
+                        printf("            alproto: ALPROTO_POP3\n");
                     else if (pp_pe->alproto == ALPROTO_TEMPLATE)
                         printf("            alproto: ALPROTO_TEMPLATE\n");
                     else if (pp_pe->alproto == ALPROTO_DNP3)
@@ -942,6 +944,8 @@ static void AppLayerProtoDetectPrintProbingParsers(AppLayerProtoDetectProbingPar
                     printf("            alproto: ALPROTO_SIP\n");
                 else if (pp_pe->alproto == ALPROTO_TEMPLATE_RUST)
                     printf("            alproto: ALPROTO_TEMPLATE_RUST\n");
+                else if (pp_pe->alproto == ALPROTO_POP3)
+                    printf("            alproto: ALPROTO_POP3\n");
                 else if (pp_pe->alproto == ALPROTO_TEMPLATE)
                     printf("            alproto: ALPROTO_TEMPLATE\n");
                 else if (pp_pe->alproto == ALPROTO_DNP3)
diff --git a/src/app-layer-parser.c b/src/app-layer-parser.c
index 660289e..900b8ac 100644
--- a/src/app-layer-parser.c
+++ b/src/app-layer-parser.c
@@ -68,6 +68,7 @@
 #include "app-layer-dhcp.h"
 #include "app-layer-snmp.h"
 #include "app-layer-sip.h"
+#include "app-layer-pop3.h"
 #include "app-layer-template.h"
 #include "app-layer-template-rust.h"
 #include "app-layer-rdp.h"
@@ -1553,6 +1554,7 @@ void AppLayerParserRegisterProtocolParsers(void)
     RegisterSNMPParsers();
     RegisterSIPParsers();
     RegisterTemplateRustParsers();
+    RegisterPop3Parsers();
     RegisterTemplateParsers();
     RegisterRdpParsers();
 
diff --git a/src/app-layer-pop3.c b/src/app-layer-pop3.c
new file mode 100644
index 0000000..e60d8b5
--- /dev/null
+++ b/src/app-layer-pop3.c
@@ -0,0 +1,584 @@
+/* Copyright (C) 2015 Open Information Security Foundation
+ *
+ * You can copy, redistribute or modify this Program under the terms of
+ * the GNU General Public License version 2 as published by the Free
+ * Software Foundation.
+ *
+ * This program is distributed in the hope that it will be useful,
+ * but WITHOUT ANY WARRANTY; without even the implied warranty of
+ * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
+ * GNU General Public License for more details.
+ *
+ * You should have received a copy of the GNU General Public License
+ * version 2 along with this program; if not, write to the Free Software
+ * Foundation, Inc., 51 Franklin Street, Fifth Floor, Boston, MA
+ * 02110-1301, USA.
+ */
+
+/*
+ * TODO: Update \author in this file and app-layer-pop3.h.
+ * TODO: Implement your app-layer logic with unit tests.
+ * TODO: Remove SCLogNotice statements or convert to debug.
+ */
+
+/**
+ * \file
+ *
+ * \author FirstName LastName <yourname@domain>
+ *
+ * Pop3 application layer detector and parser for learning and
+ * pop3 pruposes.
+ *
+ * This pop3 implements a simple application layer for something
+ * like the echo protocol running on port 7.
+ */
+
+#include "suricata-common.h"
+#include "stream.h"
+#include "conf.h"
+#include "app-layer-detect-proto.h"
+#include "app-layer-parser.h"
+#include "app-layer-pop3.h"
+
+#include "util-unittest.h"
+
+
+/* The default port to probe for echo traffic if not provided in the
+ * configuration file. */
+#define POP3_DEFAULT_PORT "7"
+
+/* The minimum size for a message. For some protocols this might
+ * be the size of a header. */
+#define POP3_MIN_FRAME_LEN 1
+
+/* Enum of app-layer events for the protocol. Normally you might
+ * have events for errors in parsing data, like unexpected data being
+ * received. For pop3 we'll make something up, and log an app-layer
+ * level alert if an empty message is received.
+ *
+ * Example rule:
+ *
+ * alert pop3 any any -> any any (msg:"SURICATA Pop3 empty message"; \
+ *    app-layer-event:pop3.empty_message; sid:X; rev:Y;)
+ */
+enum {
+    POP3_DECODER_EVENT_EMPTY_MESSAGE,
+};
+
+SCEnumCharMap pop3_decoder_event_table[] = {
+    {"EMPTY_MESSAGE", POP3_DECODER_EVENT_EMPTY_MESSAGE},
+
+    // event table must be NULL-terminated
+    { NULL, -1 },
+};
+
+static Pop3Transaction *Pop3TxAlloc(Pop3State *state)
+{
+    Pop3Transaction *tx = SCCalloc(1, sizeof(Pop3Transaction));
+    if (unlikely(tx == NULL)) {
+        return NULL;
+    }
+
+    /* Increment the transaction ID on the state each time one is
+     * allocated. */
+    tx->tx_id = state->transaction_max++;
+
+    TAILQ_INSERT_TAIL(&state->tx_list, tx, next);
+
+    return tx;
+}
+
+static void Pop3TxFree(void *txv)
+{
+    Pop3Transaction *tx = txv;
+
+    if (tx->request_buffer != NULL) {
+        SCFree(tx->request_buffer);
+    }
+
+    if (tx->response_buffer != NULL) {
+        SCFree(tx->response_buffer);
+    }
+
+    AppLayerDecoderEventsFreeEvents(&tx->decoder_events);
+
+    SCFree(tx);
+}
+
+static void *Pop3StateAlloc(void)
+{
+    SCLogNotice("Allocating pop3 state.");
+    Pop3State *state = SCCalloc(1, sizeof(Pop3State));
+    if (unlikely(state == NULL)) {
+        return NULL;
+    }
+    TAILQ_INIT(&state->tx_list);
+    return state;
+}
+
+static void Pop3StateFree(void *state)
+{
+    Pop3State *pop3_state = state;
+    Pop3Transaction *tx;
+    SCLogNotice("Freeing pop3 state.");
+    while ((tx = TAILQ_FIRST(&pop3_state->tx_list)) != NULL) {
+        TAILQ_REMOVE(&pop3_state->tx_list, tx, next);
+        Pop3TxFree(tx);
+    }
+    SCFree(pop3_state);
+}
+
+/**
+ * \brief Callback from the application layer to have a transaction freed.
+ *
+ * \param state a void pointer to the Pop3State object.
+ * \param tx_id the transaction ID to free.
+ */
+static void Pop3StateTxFree(void *statev, uint64_t tx_id)
+{
+    Pop3State *state = statev;
+    Pop3Transaction *tx = NULL, *ttx;
+
+    SCLogNotice("Freeing transaction %"PRIu64, tx_id);
+
+    TAILQ_FOREACH_SAFE(tx, &state->tx_list, next, ttx) {
+
+        /* Continue if this is not the transaction we are looking
+         * for. */
+        if (tx->tx_id != tx_id) {
+            continue;
+        }
+
+        /* Remove and free the transaction. */
+        TAILQ_REMOVE(&state->tx_list, tx, next);
+        Pop3TxFree(tx);
+        return;
+    }
+
+    SCLogNotice("Transaction %"PRIu64" not found.", tx_id);
+}
+
+static int Pop3StateGetEventInfo(const char *event_name, int *event_id,
+    AppLayerEventType *event_type)
+{
+    *event_id = SCMapEnumNameToValue(event_name, pop3_decoder_event_table);
+    if (*event_id == -1) {
+        SCLogError(SC_ERR_INVALID_ENUM_MAP, "event \"%s\" not present in "
+                   "pop3 enum map table.",  event_name);
+        /* This should be treated as fatal. */
+        return -1;
+    }
+
+    *event_type = APP_LAYER_EVENT_TYPE_TRANSACTION;
+
+    return 0;
+}
+
+static int Pop3StateGetEventInfoById(int event_id, const char **event_name,
+                                         AppLayerEventType *event_type)
+{
+    *event_name = SCMapEnumValueToName(event_id, pop3_decoder_event_table);
+    if (*event_name == NULL) {
+        SCLogError(SC_ERR_INVALID_ENUM_MAP, "event \"%d\" not present in "
+                   "pop3 enum map table.",  event_id);
+        /* This should be treated as fatal. */
+        return -1;
+    }
+
+    *event_type = APP_LAYER_EVENT_TYPE_TRANSACTION;
+
+    return 0;
+}
+
+static AppLayerDecoderEvents *Pop3GetEvents(void *tx)
+{
+    return ((Pop3Transaction *)tx)->decoder_events;
+}
+
+/**
+ * \brief Probe the input to server to see if it looks like pop3.
+ *
+ * \retval ALPROTO_POP3 if it looks like pop3,
+ *     ALPROTO_FAILED, if it is clearly not ALPROTO_POP3,
+ *     otherwise ALPROTO_UNKNOWN.
+ */
+static AppProto Pop3ProbingParserTs(Flow *f, uint8_t direction,
+        const uint8_t *input, uint32_t input_len, uint8_t *rdir)
+{
+    /* Very simple test - if there is input, this is pop3. */
+    if (input_len >= POP3_MIN_FRAME_LEN) {
+        SCLogNotice("Detected as ALPROTO_POP3.");
+        return ALPROTO_POP3;
+    }
+
+    SCLogNotice("Protocol not detected as ALPROTO_POP3.");
+    return ALPROTO_UNKNOWN;
+}
+
+/**
+ * \brief Probe the input to client to see if it looks like pop3.
+ *     Pop3ProbingParserTs can be used instead if the protocol
+ *     is symmetric.
+ *
+ * \retval ALPROTO_POP3 if it looks like pop3,
+ *     ALPROTO_FAILED, if it is clearly not ALPROTO_POP3,
+ *     otherwise ALPROTO_UNKNOWN.
+ */
+static AppProto Pop3ProbingParserTc(Flow *f, uint8_t direction,
+        const uint8_t *input, uint32_t input_len, uint8_t *rdir)
+{
+    /* Very simple test - if there is input, this is pop3. */
+    if (input_len >= POP3_MIN_FRAME_LEN) {
+        SCLogNotice("Detected as ALPROTO_POP3.");
+        return ALPROTO_POP3;
+    }
+
+    SCLogNotice("Protocol not detected as ALPROTO_POP3.");
+    return ALPROTO_UNKNOWN;
+}
+
+static int Pop3ParseRequest(Flow *f, void *statev,
+    AppLayerParserState *pstate, const uint8_t *input, uint32_t input_len,
+    void *local_data, const uint8_t flags)
+{
+    Pop3State *state = statev;
+
+    SCLogNotice("Parsing pop3 request: len=%"PRIu32, input_len);
+
+    /* Likely connection closed, we can just return here. */
+    if ((input == NULL || input_len == 0) &&
+        AppLayerParserStateIssetFlag(pstate, APP_LAYER_PARSER_EOF)) {
+        return 0;
+    }
+
+    /* Probably don't want to create a transaction in this case
+     * either. */
+    if (input == NULL || input_len == 0) {
+        return 0;
+    }
+
+    /* Normally you would parse out data here and store it in the
+     * transaction object, but as this is echo, we'll just record the
+     * request data. */
+
+    /* Also, if this protocol may have a "protocol data unit" span
+     * multiple chunks of data, which is always a possibility with
+     * TCP, you may need to do some buffering here.
+     *
+     * For the sake of simplicity, buffering is left out here, but
+     * even for an echo protocol we may want to buffer until a new
+     * line is seen, assuming its text based.
+     */
+
+    /* Allocate a transaction.
+     *
+     * But note that if a "protocol data unit" is not received in one
+     * chunk of data, and the buffering is done on the transaction, we
+     * may need to look for the transaction that this newly recieved
+     * data belongs to.
+     */
+    Pop3Transaction *tx = Pop3TxAlloc(state);
+    if (unlikely(tx == NULL)) {
+        SCLogNotice("Failed to allocate new Pop3 tx.");
+        goto end;
+    }
+    SCLogNotice("Allocated Pop3 tx %"PRIu64".", tx->tx_id);
+
+    /* Make a copy of the request. */
+    tx->request_buffer = SCCalloc(1, input_len);
+    if (unlikely(tx->request_buffer == NULL)) {
+        goto end;
+    }
+    memcpy(tx->request_buffer, input, input_len);
+    tx->request_buffer_len = input_len;
+
+    /* Here we check for an empty message and create an app-layer
+     * event. */
+    if ((input_len == 1 && tx->request_buffer[0] == '\n') ||
+        (input_len == 2 && tx->request_buffer[0] == '\r')) {
+        SCLogNotice("Creating event for empty message.");
+        AppLayerDecoderEventsSetEventRaw(&tx->decoder_events,
+            POP3_DECODER_EVENT_EMPTY_MESSAGE);
+    }
+
+end:
+    return 0;
+}
+
+static int Pop3ParseResponse(Flow *f, void *statev, AppLayerParserState *pstate,
+    const uint8_t *input, uint32_t input_len, void *local_data,
+    const uint8_t flags)
+{
+    Pop3State *state = statev;
+    Pop3Transaction *tx = NULL, *ttx;
+
+    SCLogNotice("Parsing Pop3 response.");
+
+    /* Likely connection closed, we can just return here. */
+    if ((input == NULL || input_len == 0) &&
+        AppLayerParserStateIssetFlag(pstate, APP_LAYER_PARSER_EOF)) {
+        return 0;
+    }
+
+    /* Probably don't want to create a transaction in this case
+     * either. */
+    if (input == NULL || input_len == 0) {
+        return 0;
+    }
+
+    /* Look up the existing transaction for this response. In the case
+     * of echo, it will be the most recent transaction on the
+     * Pop3State object. */
+
+    /* We should just grab the last transaction, but this is to
+     * illustrate how you might traverse the transaction list to find
+     * the transaction associated with this response. */
+    TAILQ_FOREACH(ttx, &state->tx_list, next) {
+        tx = ttx;
+    }
+
+    if (tx == NULL) {
+        SCLogNotice("Failed to find transaction for response on state %p.",
+            state);
+        goto end;
+    }
+
+    SCLogNotice("Found transaction %"PRIu64" for response on state %p.",
+        tx->tx_id, state);
+
+    /* If the protocol requires multiple chunks of data to complete, you may
+     * run into the case where you have existing response data.
+     *
+     * In this case, we just log that there is existing data and free it. But
+     * you might want to realloc the buffer and append the data.
+     */
+    if (tx->response_buffer != NULL) {
+        SCLogNotice("WARNING: Transaction already has response data, "
+            "existing data will be overwritten.");
+        SCFree(tx->response_buffer);
+    }
+
+    /* Make a copy of the response. */
+    tx->response_buffer = SCCalloc(1, input_len);
+    if (unlikely(tx->response_buffer == NULL)) {
+        goto end;
+    }
+    memcpy(tx->response_buffer, input, input_len);
+    tx->response_buffer_len = input_len;
+
+    /* Set the response_done flag for transaction state checking in
+     * Pop3GetStateProgress(). */
+    tx->response_done = 1;
+
+end:
+    return 0;
+}
+
+static uint64_t Pop3GetTxCnt(void *statev)
+{
+    const Pop3State *state = statev;
+    SCLogNotice("Current tx count is %"PRIu64".", state->transaction_max);
+    return state->transaction_max;
+}
+
+static void *Pop3GetTx(void *statev, uint64_t tx_id)
+{
+    Pop3State *state = statev;
+    Pop3Transaction *tx;
+
+    SCLogNotice("Requested tx ID %"PRIu64".", tx_id);
+
+    TAILQ_FOREACH(tx, &state->tx_list, next) {
+        if (tx->tx_id == tx_id) {
+            SCLogNotice("Transaction %"PRIu64" found, returning tx object %p.",
+                tx_id, tx);
+            return tx;
+        }
+    }
+
+    SCLogNotice("Transaction ID %"PRIu64" not found.", tx_id);
+    return NULL;
+}
+
+static void Pop3SetTxLogged(void *state, void *vtx, LoggerId logged)
+{
+    Pop3Transaction *tx = (Pop3Transaction *)vtx;
+    tx->logged = logged;
+}
+
+static LoggerId Pop3GetTxLogged(void *state, void *vtx)
+{
+    const Pop3Transaction *tx = (Pop3Transaction *)vtx;
+    return tx->logged;
+}
+
+/**
+ * \brief Called by the application layer.
+ *
+ * In most cases 1 can be returned here.
+ */
+static int Pop3GetAlstateProgressCompletionStatus(uint8_t direction) {
+    return 1;
+}
+
+/**
+ * \brief Return the state of a transaction in a given direction.
+ *
+ * In the case of the echo protocol, the existence of a transaction
+ * means that the request is done. However, some protocols that may
+ * need multiple chunks of data to complete the request may need more
+ * than just the existence of a transaction for the request to be
+ * considered complete.
+ *
+ * For the response to be considered done, the response for a request
+ * needs to be seen.  The response_done flag is set on response for
+ * checking here.
+ */
+static int Pop3GetStateProgress(void *txv, uint8_t direction)
+{
+    Pop3Transaction *tx = txv;
+
+    SCLogNotice("Transaction progress requested for tx ID %"PRIu64
+        ", direction=0x%02x", tx->tx_id, direction);
+
+    if (direction & STREAM_TOCLIENT && tx->response_done) {
+        return 1;
+    }
+    else if (direction & STREAM_TOSERVER) {
+        /* For the pop3, just the existence of the transaction means the
+         * request is done. */
+        return 1;
+    }
+
+    return 0;
+}
+
+/**
+ * \brief retrieve the detection engine per tx state
+ */
+static DetectEngineState *Pop3GetTxDetectState(void *vtx)
+{
+    Pop3Transaction *tx = vtx;
+    return tx->de_state;
+}
+
+/**
+ * \brief get the detection engine per tx state
+ */
+static int Pop3SetTxDetectState(void *vtx,
+    DetectEngineState *s)
+{
+    Pop3Transaction *tx = vtx;
+    tx->de_state = s;
+    return 0;
+}
+
+void RegisterPop3Parsers(void)
+{
+    const char *proto_name = "pop3";
+
+    /* Check if Pop3 TCP detection is enabled. If it does not exist in
+     * the configuration file then it will be enabled by default. */
+    if (AppLayerProtoDetectConfProtoDetectionEnabled("tcp", proto_name)) {
+
+        SCLogNotice("Pop3 TCP protocol detection enabled.");
+
+        AppLayerProtoDetectRegisterProtocol(ALPROTO_POP3, proto_name);
+
+        if (RunmodeIsUnittests()) {
+
+            SCLogNotice("Unittest mode, registeringd default configuration.");
+            AppLayerProtoDetectPPRegister(IPPROTO_TCP, POP3_DEFAULT_PORT,
+                ALPROTO_POP3, 0, POP3_MIN_FRAME_LEN, STREAM_TOSERVER,
+                Pop3ProbingParserTs, Pop3ProbingParserTc);
+
+        }
+        else {
+
+            if (!AppLayerProtoDetectPPParseConfPorts("tcp", IPPROTO_TCP,
+                    proto_name, ALPROTO_POP3, 0, POP3_MIN_FRAME_LEN,
+                    Pop3ProbingParserTs, Pop3ProbingParserTc)) {
+                SCLogNotice("No pop3 app-layer configuration, enabling echo"
+                    " detection TCP detection on port %s.",
+                    POP3_DEFAULT_PORT);
+                AppLayerProtoDetectPPRegister(IPPROTO_TCP,
+                    POP3_DEFAULT_PORT, ALPROTO_POP3, 0,
+                    POP3_MIN_FRAME_LEN, STREAM_TOSERVER,
+                    Pop3ProbingParserTs, Pop3ProbingParserTc);
+            }
+
+        }
+
+    }
+
+    else {
+        SCLogNotice("Protocol detecter and parser disabled for Pop3.");
+        return;
+    }
+
+    if (AppLayerParserConfParserEnabled("tcp", proto_name)) {
+
+        SCLogNotice("Registering Pop3 protocol parser.");
+
+        /* Register functions for state allocation and freeing. A
+         * state is allocated for every new Pop3 flow. */
+        AppLayerParserRegisterStateFuncs(IPPROTO_TCP, ALPROTO_POP3,
+            Pop3StateAlloc, Pop3StateFree);
+
+        /* Register request parser for parsing frame from server to client. */
+        AppLayerParserRegisterParser(IPPROTO_TCP, ALPROTO_POP3,
+            STREAM_TOSERVER, Pop3ParseRequest);
+
+        /* Register response parser for parsing frames from server to client. */
+        AppLayerParserRegisterParser(IPPROTO_TCP, ALPROTO_POP3,
+            STREAM_TOCLIENT, Pop3ParseResponse);
+
+        /* Register a function to be called by the application layer
+         * when a transaction is to be freed. */
+        AppLayerParserRegisterTxFreeFunc(IPPROTO_TCP, ALPROTO_POP3,
+            Pop3StateTxFree);
+
+        AppLayerParserRegisterLoggerFuncs(IPPROTO_TCP, ALPROTO_POP3,
+            Pop3GetTxLogged, Pop3SetTxLogged);
+
+        /* Register a function to return the current transaction count. */
+        AppLayerParserRegisterGetTxCnt(IPPROTO_TCP, ALPROTO_POP3,
+            Pop3GetTxCnt);
+
+        /* Transaction handling. */
+        AppLayerParserRegisterGetStateProgressCompletionStatus(ALPROTO_POP3,
+            Pop3GetAlstateProgressCompletionStatus);
+        AppLayerParserRegisterGetStateProgressFunc(IPPROTO_TCP,
+            ALPROTO_POP3, Pop3GetStateProgress);
+        AppLayerParserRegisterGetTx(IPPROTO_TCP, ALPROTO_POP3,
+            Pop3GetTx);
+
+        /* What is this being registered for? */
+        AppLayerParserRegisterDetectStateFuncs(IPPROTO_TCP, ALPROTO_POP3,
+            Pop3GetTxDetectState, Pop3SetTxDetectState);
+
+        AppLayerParserRegisterGetEventInfo(IPPROTO_TCP, ALPROTO_POP3,
+            Pop3StateGetEventInfo);
+        AppLayerParserRegisterGetEventInfoById(IPPROTO_TCP, ALPROTO_POP3,
+            Pop3StateGetEventInfoById);
+        AppLayerParserRegisterGetEventsFunc(IPPROTO_TCP, ALPROTO_POP3,
+            Pop3GetEvents);
+    }
+    else {
+        SCLogNotice("Pop3 protocol parsing disabled.");
+    }
+
+#ifdef UNITTESTS
+    AppLayerParserRegisterProtocolUnittests(IPPROTO_TCP, ALPROTO_POP3,
+        Pop3ParserRegisterTests);
+#endif
+}
+
+#ifdef UNITTESTS
+#endif
+
+void Pop3ParserRegisterTests(void)
+{
+#ifdef UNITTESTS
+#endif
+}
diff --git a/src/app-layer-pop3.h b/src/app-layer-pop3.h
new file mode 100644
index 0000000..22ea286
--- /dev/null
+++ b/src/app-layer-pop3.h
@@ -0,0 +1,73 @@
+/* Copyright (C) 2015-2018 Open Information Security Foundation
+ *
+ * You can copy, redistribute or modify this Program under the terms of
+ * the GNU General Public License version 2 as published by the Free
+ * Software Foundation.
+ *
+ * This program is distributed in the hope that it will be useful,
+ * but WITHOUT ANY WARRANTY; without even the implied warranty of
+ * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
+ * GNU General Public License for more details.
+ *
+ * You should have received a copy of the GNU General Public License
+ * version 2 along with this program; if not, write to the Free Software
+ * Foundation, Inc., 51 Franklin Street, Fifth Floor, Boston, MA
+ * 02110-1301, USA.
+ */
+
+/**
+ * \file
+ *
+ * \author FirstName LastName <yourname@domain>
+ */
+
+#ifndef __APP_LAYER_POP3_H__
+#define __APP_LAYER_POP3_H__
+
+#include "detect-engine-state.h"
+
+#include "queue.h"
+
+void RegisterPop3Parsers(void);
+void Pop3ParserRegisterTests(void);
+
+typedef struct Pop3Transaction
+{
+    /** Internal transaction ID. */
+    uint64_t tx_id;
+
+    /** Application layer events that occurred
+     *  while parsing this transaction. */
+    AppLayerDecoderEvents *decoder_events;
+
+    uint8_t *request_buffer;
+    uint32_t request_buffer_len;
+
+    /** flags indicating which loggers that have logged */
+    LoggerId logged;
+
+    uint8_t *response_buffer;
+    uint32_t response_buffer_len;
+
+    uint8_t response_done; /*<< Flag to be set when the response is
+                            * seen. */
+
+    DetectEngineState *de_state;
+
+    TAILQ_ENTRY(Pop3Transaction) next;
+
+} Pop3Transaction;
+
+typedef struct Pop3State {
+
+    /** List of Pop3 transactions associated with this
+     *  state. */
+    TAILQ_HEAD(, Pop3Transaction) tx_list;
+
+    /** A count of the number of transactions created. The
+     *  transaction ID for each transaction is allocted
+     *  by incrementing this value. */
+    uint64_t transaction_max;
+} Pop3State;
+
+#endif /* __APP_LAYER_POP3_H__ */
diff --git a/src/app-layer-protos.c b/src/app-layer-protos.c
index 2a72051..cb25f07 100644
--- a/src/app-layer-protos.c
+++ b/src/app-layer-protos.c
@@ -102,6 +102,9 @@ const char *AppProtoToString(AppProto alproto)
         case ALPROTO_SIP:
             proto_name = "sip";
             break;
+        case ALPROTO_POP3:
+            proto_name = "pop3";
+            break;
         case ALPROTO_TEMPLATE:
             proto_name = "template";
             break;
@@ -150,6 +153,7 @@ AppProto StringToAppProto(const char *proto_name)
     if (strcmp(proto_name,"dhcp")==0) return ALPROTO_DHCP;
     if (strcmp(proto_name,"snmp")==0) return ALPROTO_SNMP;
     if (strcmp(proto_name,"sip")==0) return ALPROTO_SIP;
+    if (strcmp(proto_name,"pop3")==0) return ALPROTO_POP3;
     if (strcmp(proto_name,"template")==0) return ALPROTO_TEMPLATE;
     if (strcmp(proto_name,"template-rust")==0) return ALPROTO_TEMPLATE_RUST;
     if (strcmp(proto_name,"rdp")==0) return ALPROTO_RDP;
diff --git a/src/app-layer-protos.h b/src/app-layer-protos.h
index 9ee8631..23b469c 100644
--- a/src/app-layer-protos.h
+++ b/src/app-layer-protos.h
@@ -51,6 +51,7 @@ enum AppProtoEnum {
     ALPROTO_DHCP,
     ALPROTO_SNMP,
     ALPROTO_SIP,
+    ALPROTO_POP3,
     ALPROTO_TEMPLATE,
     ALPROTO_TEMPLATE_RUST,
     ALPROTO_RDP,
diff --git a/src/output-json-pop3.c b/src/output-json-pop3.c
new file mode 100644
index 0000000..1a7885c
--- /dev/null
+++ b/src/output-json-pop3.c
@@ -0,0 +1,170 @@
+/* Copyright (C) 2015 Open Information Security Foundation
+ *
+ * You can copy, redistribute or modify this Program under the terms of
+ * the GNU General Public License version 2 as published by the Free
+ * Software Foundation.
+ *
+ * This program is distributed in the hope that it will be useful,
+ * but WITHOUT ANY WARRANTY; without even the implied warranty of
+ * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
+ * GNU General Public License for more details.
+ *
+ * You should have received a copy of the GNU General Public License
+ * version 2 along with this program; if not, write to the Free Software
+ * Foundation, Inc., 51 Franklin Street, Fifth Floor, Boston, MA
+ * 02110-1301, USA.
+ */
+
+/*
+ * TODO: Update \author in this file and in output-json-pop3.h.
+ * TODO: Remove SCLogNotice statements, or convert to debug.
+ * TODO: Implement your app-layers logging.
+ */
+
+/**
+ * \file
+ *
+ * \author FirstName LastName <yourname@domain>
+ *
+ * Implement JSON/eve logging app-layer Pop3.
+ */
+
+#include "suricata-common.h"
+#include "debug.h"
+#include "detect.h"
+#include "pkt-var.h"
+#include "conf.h"
+
+#include "threads.h"
+#include "threadvars.h"
+#include "tm-threads.h"
+
+#include "util-unittest.h"
+#include "util-buffer.h"
+#include "util-debug.h"
+#include "util-byte.h"
+
+#include "output.h"
+#include "output-json.h"
+
+#include "app-layer.h"
+#include "app-layer-parser.h"
+
+#include "app-layer-pop3.h"
+#include "output-json-pop3.h"
+
+typedef struct LogPop3FileCtx_ {
+    LogFileCtx *file_ctx;
+    uint32_t    flags;
+} LogPop3FileCtx;
+
+typedef struct LogPop3LogThread_ {
+    LogPop3FileCtx *pop3log_ctx;
+    uint32_t            count;
+    MemBuffer          *buffer;
+} LogPop3LogThread;
+
+static int JsonPop3Logger(ThreadVars *tv, void *thread_data,
+    const Packet *p, Flow *f, void *state, void *tx, uint64_t tx_id)
+{
+    Pop3Transaction *pop3tx = tx;
+    LogPop3LogThread *thread = thread_data;
+
+    SCLogNotice("Logging pop3 transaction %"PRIu64".", pop3tx->tx_id);
+
+    /* Convert the request buffer to a string then log. */
+
+    /* Convert the response buffer to a string then log. */
+
+    OutputJSONBuffer(js, thread->pop3log_ctx->file_ctx, &thread->buffer);
+
+    return TM_ECODE_OK;
+    
+error:
+    return TM_ECODE_FAILED;
+}
+
+static void OutputPop3LogDeInitCtxSub(OutputCtx *output_ctx)
+{
+    LogPop3FileCtx *pop3log_ctx = (LogPop3FileCtx *)output_ctx->data;
+    SCFree(pop3log_ctx);
+    SCFree(output_ctx);
+}
+
+static OutputInitResult OutputPop3LogInitSub(ConfNode *conf,
+    OutputCtx *parent_ctx)
+{
+    OutputInitResult result = { NULL, false };
+    OutputJsonCtx *ajt = parent_ctx->data;
+
+    LogPop3FileCtx *pop3log_ctx = SCCalloc(1, sizeof(*pop3log_ctx));
+    if (unlikely(pop3log_ctx == NULL)) {
+        return result;
+    }
+    pop3log_ctx->file_ctx = ajt->file_ctx;
+
+    OutputCtx *output_ctx = SCCalloc(1, sizeof(*output_ctx));
+    if (unlikely(output_ctx == NULL)) {
+        SCFree(pop3log_ctx);
+        return result;
+    }
+    output_ctx->data = pop3log_ctx;
+    output_ctx->DeInit = OutputPop3LogDeInitCtxSub;
+
+    SCLogNotice("Pop3 log sub-module initialized.");
+
+    AppLayerParserRegisterLogger(IPPROTO_TCP, ALPROTO_POP3);
+
+    result.ctx = output_ctx;
+    result.ok = true;
+    return result;
+}
+
+static TmEcode JsonPop3LogThreadInit(ThreadVars *t, const void *initdata, void **data)
+{
+    LogPop3LogThread *thread = SCCalloc(1, sizeof(*thread));
+    if (unlikely(thread == NULL)) {
+        return TM_ECODE_FAILED;
+    }
+
+    if (initdata == NULL) {
+        SCLogDebug("Error getting context for EveLogPop3.  \"initdata\" is NULL.");
+        SCFree(thread);
+        return TM_ECODE_FAILED;
+    }
+
+    thread->buffer = MemBufferCreateNew(JSON_OUTPUT_BUFFER_SIZE);
+    if (unlikely(thread->buffer == NULL)) {
+        SCFree(thread);
+        return TM_ECODE_FAILED;
+    }
+
+    thread->pop3log_ctx = ((OutputCtx *)initdata)->data;
+    *data = (void *)thread;
+
+    return TM_ECODE_OK;
+}
+
+static TmEcode JsonPop3LogThreadDeinit(ThreadVars *t, void *data)
+{
+    LogPop3LogThread *thread = (LogPop3LogThread *)data;
+    if (thread == NULL) {
+        return TM_ECODE_OK;
+    }
+    if (thread->buffer != NULL) {
+        MemBufferFree(thread->buffer);
+    }
+    SCFree(thread);
+    return TM_ECODE_OK;
+}
+
+void JsonPop3LogRegister(void)
+{
+    /* Register as an eve sub-module. */
+    OutputRegisterTxSubModule(LOGGER_JSON_POP3, "eve-log", "JsonPop3Log",
+        "eve-log.pop3", OutputPop3LogInitSub, ALPROTO_POP3,
+        JsonPop3Logger, JsonPop3LogThreadInit,
+        JsonPop3LogThreadDeinit, NULL);
+
+    SCLogNotice("Pop3 JSON logger registered.");
+}
diff --git a/src/output-json-pop3.h b/src/output-json-pop3.h
new file mode 100644
index 0000000..e475101
--- /dev/null
+++ b/src/output-json-pop3.h
@@ -0,0 +1,29 @@
+/* Copyright (C) 2015 Open Information Security Foundation
+ *
+ * You can copy, redistribute or modify this Program under the terms of
+ * the GNU General Public License version 2 as published by the Free
+ * Software Foundation.
+ *
+ * This program is distributed in the hope that it will be useful,
+ * but WITHOUT ANY WARRANTY; without even the implied warranty of
+ * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
+ * GNU General Public License for more details.
+ *
+ * You should have received a copy of the GNU General Public License
+ * version 2 along with this program; if not, write to the Free Software
+ * Foundation, Inc., 51 Franklin Street, Fifth Floor, Boston, MA
+ * 02110-1301, USA.
+ */
+
+/**
+ * \file
+ *
+ * \author FirstName LastName <name@domain>
+ */
+
+#ifndef __OUTPUT_JSON_POP3_H__
+#define __OUTPUT_JSON_POP3_H__
+
+void JsonPop3LogRegister(void);
+
+#endif /* __OUTPUT_JSON_POP3_H__ */
diff --git a/src/output.c b/src/output.c
index c486999..719f3f6 100644
--- a/src/output.c
+++ b/src/output.c
@@ -76,6 +76,7 @@
 #include "output-json-dhcp.h"
 #include "output-json-snmp.h"
 #include "output-json-sip.h"
+#include "output-json-pop3.h"
 #include "output-json-template.h"
 #include "output-json-template-rust.h"
 #include "output-json-rdp.h"
@@ -1113,6 +1114,8 @@ void OutputRegisterLoggers(void)
     JsonSNMPLogRegister();
     /* SIP JSON logger. */
     JsonSIPLogRegister();
+    /* Pop3 JSON logger. */
+    JsonPop3LogRegister();
     /* Template JSON logger. */
     JsonTemplateLogRegister();
     /* Template Rust JSON logger. */
diff --git a/src/suricata-common.h b/src/suricata-common.h
index b55509c..d13ba6f 100644
--- a/src/suricata-common.h
+++ b/src/suricata-common.h
@@ -440,6 +440,7 @@ typedef enum {
     LOGGER_JSON_SNMP,
     LOGGER_JSON_SIP,
     LOGGER_JSON_TEMPLATE_RUST,
+    LOGGER_JSON_POP3,
     LOGGER_JSON_TEMPLATE,
     LOGGER_JSON_RDP,
 
diff --git a/src/util-profiling.c b/src/util-profiling.c
index 3fbcd6b..0db48de 100644
--- a/src/util-profiling.c
+++ b/src/util-profiling.c
@@ -1320,6 +1320,7 @@ const char * PacketProfileLoggertIdToString(LoggerId id)
         CASE_CODE (LOGGER_JSON_TLS);
         CASE_CODE (LOGGER_JSON_SIP);
         CASE_CODE (LOGGER_JSON_TEMPLATE_RUST);
+        CASE_CODE (LOGGER_JSON_POP3);
         CASE_CODE (LOGGER_JSON_TEMPLATE);
         CASE_CODE (LOGGER_JSON_RDP);
         CASE_CODE (LOGGER_TLS_STORE);
diff --git a/suricata.yaml.in b/suricata.yaml.in
index aacad74..6d10fb7 100644
--- a/suricata.yaml.in
+++ b/suricata.yaml.in
@@ -143,6 +143,7 @@ outputs:
         header: X-Forwarded-For
 
       types:
+        - pop3
         - alert:
             # payload: yes             # enable dumping payload in Base64
             # payload-buffer-size: 4kb # max size of payload buffer to output in eve-log
@@ -713,6 +714,8 @@ pcap-file:
 # "detection-only" enables protocol detection only (parser disabled).
 app-layer:
   protocols:
+    pop3:
+      enabled: yes
     krb5:
       enabled: yes
     snmp:

```