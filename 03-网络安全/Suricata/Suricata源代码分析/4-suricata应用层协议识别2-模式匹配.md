

# 一、suricata协议识别方法概述

## 1、1 PP (Probing Parser) - 探测解析器

PP的特点：

- 包含协议特定的检测逻辑
- 可以处理可变格式和协议协商
- 通常绑定到特定端口
- 返回`ALPROTO_FAILED`、`ALPROTO_UNKNOWN`或具体协议类型



## 1、2 PM (Pattern Matching) - 模式匹配

指多模式匹配器

PM的特点：

- 使用固定字节模式进行快速匹配
- O(1)时间复杂度，性能优异
- 端口无关检测
- 通过`SCAppLayerProtoDetectPMRegisterPatternCS`等函数注册



## 1·、3 PE (Protocol Expectation) - 协议期望



# 二、协议识别入口函数

协议识别函数

```
AppProto AppLayerProtoDetectGetProto(AppLayerProtoDetectThreadCtx *tctx, Flow *f,
        const uint8_t *buf, uint32_t buflen, uint8_t ipproto, uint8_t flags, bool *reverse_flow)
{
    SCEnter();
    SCLogDebug("buflen %u for %s direction", buflen,
            (flags & STREAM_TOSERVER) ? "toserver" : "toclient");

    AppProto alproto = ALPROTO_UNKNOWN;

    if (!FLOW_IS_PM_DONE(f, flags)) {
        AppProto pm_results[g_alproto_max];
        uint16_t pm_matches = AppLayerProtoDetectPMGetProto(
                tctx, f, buf, buflen, flags, pm_results, reverse_flow);
        if (pm_matches > 0) {
            DEBUG_VALIDATE_BUG_ON(pm_matches > 1);
            alproto = pm_results[0];

            // rerun probing parser for other direction if it is unknown
            uint8_t reverse_dir = (flags & STREAM_TOSERVER) ? STREAM_TOCLIENT : STREAM_TOSERVER;
            if (FLOW_IS_PP_DONE(f, reverse_dir)) {
                AppProto rev_alproto = (flags & STREAM_TOSERVER) ? f->alproto_tc : f->alproto_ts;
                if (rev_alproto == ALPROTO_UNKNOWN) {
                    FLOW_RESET_PP_DONE(f, reverse_dir);
                }
            }
            SCReturnUInt(alproto);
        }
    }

    if (!FLOW_IS_PP_DONE(f, flags)) {
        DEBUG_VALIDATE_BUG_ON(*reverse_flow);
        alproto = AppLayerProtoDetectPPGetProto(f, buf, buflen, ipproto, flags, reverse_flow);
        if (AppProtoIsValid(alproto)) {
            SCReturnUInt(alproto);
        }
    }

    /* Look if flow can be found in expectation list */
    if (!FLOW_IS_PE_DONE(f, flags)) {
        DEBUG_VALIDATE_BUG_ON(*reverse_flow);
        alproto = AppLayerProtoDetectPEGetProto(f, flags);
    }

    SCReturnUInt(alproto);
}
```

其中有三个宏是很重要的

FLOW_IS_PM_DONE（pattern match）

FLOW_IS_PP_DONE（probing parser）

FLOW_IS_PE_DONE（probing expectation）



## 2、1 PP探测解析器-场景分析

这里举一个例子



## 2、2 PM模式匹配-场景分析

这里举一个例子



## 2、3 PE协议期望-场景分析

这里举一个例子



一定要对代码进行精简，来很好的达成自己的目标。

分别是对应了三种不同的协议识别的方法。如果我们移植时，先采用pattern match的方式即可。



AppLayerProtoDetectPMGetProto函数中调用FLOW_SET_PM_DONE

AppLayerProtoDetectPPGetProto函数中调用FLOW_SET_PP_DONE

AppLayerProtoDetectPEGetProto函数中调用FLOW_SET_PE_DONE



SSL协议的注册流程

协议插件必须注册的函数：

1、协议识别注册

- `AppLayerProtoDetectRegisterProtocol`：注册协议名称

- `SCAppLayerProtoDetectPPRegister()` - 注册探测解析器

```
void AppLayerProtoDetectRegisterProtocol(AppProto alproto, const char *alproto_name)
{
    //将<协议-协议名称>存储在名称映射表中
    alpd_ctx.alproto_names[alproto] = alproto_name;
    SCReturn;
}
```

SCAppLayerProtoDetectPPRegister函数

```
void SCAppLayerProtoDetectPPRegister(uint8_t ipproto, const char *portstr, AppProto alproto,
        uint16_t min_depth, uint16_t max_depth, uint8_t direction, ProbingParserFPtr ProbingParser1,
        ProbingParserFPtr ProbingParser2)
{
    DetectPort *head = NULL; //端口链表的头指针
    if (portstr == NULL) {
        // WebSocket没有固定关口，端口设置为0，use_ports设置为false，直接插入协议探测器
        // WebSocket has a probing parser, but no port
        // as it works only on HTTP1 protocol upgrade
        AppLayerProtoDetectInsertNewProbingParser(&alpd_ctx.ctx_pp, ipproto, false, 0, alproto,
                min_depth, max_depth, direction, ProbingParser1, ProbingParser2);
        return;
    }
    //解析端口字符串，生成端口链表
    DetectPortParse(NULL,&head, portstr);
    DetectPort *temp_dp = head;
    //遍历端口链表，循环处理每个端口范围
    while (temp_dp != NULL) {
        uint16_t port = temp_dp->port;
        if (port == 0 && temp_dp->port2 != 0)
            port++;
        for (;;) {
            //为每个端口插入新的协议探测器
            AppLayerProtoDetectInsertNewProbingParser(&alpd_ctx.ctx_pp, ipproto, true, port,
                    alproto, min_depth, max_depth, direction, ProbingParser1, ProbingParser2);
            if (port == temp_dp->port2) {
                break;
            } else {
                port++;
            }
        }
        temp_dp = temp_dp->next;//移动到链表中的下一个节点
    }
    DetectPortCleanupList(NULL,head);
}
```

AppLayerProtoDetectInsertNewProbingParser的函数解析如下

```
static void AppLayerProtoDetectInsertNewProbingParser(AppLayerProtoDetectProbingParser **pp,
        uint8_t ipproto, bool use_ports, uint16_t port, AppProto alproto, uint16_t min_depth,
        uint16_t max_depth, uint8_t direction, ProbingParserFPtr ProbingParser1,
        ProbingParserFPtr ProbingParser2)
{
    /* get the top level ipproto pp */
    //查找或创建ip协议级别的协议探测器
    AppLayerProtoDetectProbingParser *curr_pp = *pp;
    while (curr_pp != NULL) {
        if (curr_pp->ipproto == ipproto)
            break;
        curr_pp = curr_pp->next;
    }
    //找不到则创建新的IP协议类型的探测器
    if (curr_pp == NULL) {
        AppLayerProtoDetectProbingParser *new_pp = AppLayerProtoDetectProbingParserAlloc();
        new_pp->ipproto = ipproto;
        AppLayerProtoDetectProbingParserAppend(pp, new_pp);
        curr_pp = new_pp;
    }

    /* get the top level port pp */
    //查找或创建端口级别的协议探测器
    AppLayerProtoDetectProbingParserPort *curr_port = curr_pp->port;
    while (curr_port != NULL) {
        // when not use_ports, always insert a new AppLayerProtoDetectProbingParserPort
        if (curr_port->port == port && use_ports)
            break;
        curr_port = curr_port->next;
    }
    //找不到则创建新的端口类型的探测器
    if (curr_port == NULL) {
        AppLayerProtoDetectProbingParserPort *new_port = AppLayerProtoDetectProbingParserPortAlloc();
        new_port->port = port;
        new_port->use_ports = use_ports;
        AppLayerProtoDetectProbingParserPortAppend(&curr_pp->port, new_port);
        curr_port = new_port;
        if (direction & STREAM_TOSERVER) {
            curr_port->dp_max_depth = max_depth;
        } else {
            curr_port->sp_max_depth = max_depth;
        }

        AppLayerProtoDetectProbingParserPort *zero_port;

        zero_port = curr_pp->port;
        // get special run on any port if any, to add it to this port
        while (zero_port != NULL && !(zero_port->port == 0 && zero_port->use_ports)) {
            zero_port = zero_port->next;
        }
        if (zero_port != NULL) {
            AppLayerProtoDetectProbingParserElement *zero_pe;
            //获取当前端口的dst目标端口探测器
            zero_pe = zero_port->dp;
            for ( ; zero_pe != NULL; zero_pe = zero_pe->next) {
                if (curr_port->dp == NULL)
                    curr_port->dp_max_depth = zero_pe->max_depth;
                if (zero_pe->max_depth == 0)
                    curr_port->dp_max_depth = zero_pe->max_depth;
                if (curr_port->dp_max_depth != 0 &&
                    curr_port->dp_max_depth < zero_pe->max_depth) {
                    curr_port->dp_max_depth = zero_pe->max_depth;
                }

                AppLayerProtoDetectProbingParserElement *dup_pe =
                    AppLayerProtoDetectProbingParserElementDuplicate(zero_pe);
                AppLayerProtoDetectProbingParserElementAppend(&curr_port->dp, dup_pe);
            }

            //获取当前端口的src源端口探测器
            zero_pe = zero_port->sp;
            for ( ; zero_pe != NULL; zero_pe = zero_pe->next) {
                if (curr_port->sp == NULL)
                    curr_port->sp_max_depth = zero_pe->max_depth;
                if (zero_pe->max_depth == 0)
                    curr_port->sp_max_depth = zero_pe->max_depth;
                if (curr_port->sp_max_depth != 0 &&
                    curr_port->sp_max_depth < zero_pe->max_depth) {
                    curr_port->sp_max_depth = zero_pe->max_depth;
                }

                AppLayerProtoDetectProbingParserElement *dup_pe =
                    AppLayerProtoDetectProbingParserElementDuplicate(zero_pe);
                AppLayerProtoDetectProbingParserElementAppend(&curr_port->sp, dup_pe);
            }
        } /* if (zero_port != NULL) */
    } /* if (curr_port == NULL) */

    /* insert the pe_pp */
    AppLayerProtoDetectProbingParserElement *curr_pe;
    if (direction & STREAM_TOSERVER)
        curr_pe = curr_port->dp;
    else
        curr_pe = curr_port->sp;
    while (curr_pe != NULL) {
        if (curr_pe->alproto == alproto) {
            SCLogError("Duplicate pp registered - "
                       "ipproto - %" PRIu8 " Port - %" PRIu16 " "
                       "App Protocol - NULL, App Protocol(ID) - "
                       "%" PRIu16 " min_depth - %" PRIu16 " "
                       "max_dept - %" PRIu16 ".",
                    ipproto, port, alproto, min_depth, max_depth);
            goto error;
        }
        curr_pe = curr_pe->next;
    }
    /* Get a new parser element */
    AppLayerProtoDetectProbingParserElement *new_pe =
            AppLayerProtoDetectProbingParserElementCreate(alproto, min_depth, max_depth);
    if (new_pe == NULL)
        goto error;
    curr_pe = new_pe;
    AppLayerProtoDetectProbingParserElement **head_pe;
    if (direction & STREAM_TOSERVER) {
        curr_pe->ProbingParserTs = ProbingParser1;
        curr_pe->ProbingParserTc = ProbingParser2;
        head_pe = &curr_port->dp;
    } else {
        curr_pe->ProbingParserTs = ProbingParser2;
        curr_pe->ProbingParserTc = ProbingParser1;
        head_pe = &curr_port->sp;
    }
    AppLayerProtoDetectProbingParserElementAppend(head_pe, new_pe);
}
```

状态准备阶段

```
int AppLayerProtoDetectPrepareState(void)
{
    SCEnter();

    AppLayerProtoDetectPMCtx *ctx_pm;
    int i, j;
    int ret = 0;

    for (i = 0; i < FLOW_PROTO_DEFAULT; i++) {
        for (j = 0; j < 2; j++) {
            ctx_pm = &alpd_ctx.ctx_ipp[i].ctx_pm[j];

            if (AppLayerProtoDetectPMSetContentIDs(ctx_pm) < 0)
                goto error;

            if (ctx_pm->max_sig_id == 0)
                continue;

            if (AppLayerProtoDetectPMMapSignatures(ctx_pm) < 0)
                goto error;
            if (AppLayerProtoDetectPMPrepareMpm(ctx_pm) < 0)
                goto error;
        }
    }
}
```





```
void RegisterSSLParsers(void)
{
    const char *proto_name = "tls";

    //检测基于tcp的tls协议识别功能是否启用
    if (SCAppLayerProtoDetectConfProtoDetectionEnabled("tcp", proto_name)) {
        AppLayerProtoDetectRegisterProtocol(ALPROTO_TLS, proto_name);

		//注册协议识别模式
        if (SSLRegisterPatternsForProtocolDetection() < 0)
            return;
		
		//以单元模式运行
        if (RunmodeIsUnittests()) {
            SCAppLayerProtoDetectPPRegister(
                    IPPROTO_TCP, "443", ALPROTO_TLS, 0, 3, STREAM_TOSERVER, SSLProbingParser, NULL);
        } else {
            if (SCAppLayerProtoDetectPPParseConfPorts("tcp", IPPROTO_TCP, proto_name, ALPROTO_TLS,
                        0, 3, SSLProbingParser, NULL) == 0) {
                SCLogConfig("no TLS config found, "
                            "enabling TLS detection on port 443.");
                SCAppLayerProtoDetectPPRegister(IPPROTO_TCP, "443", ALPROTO_TLS, 0, 3,
                        STREAM_TOSERVER, SSLProbingParser, NULL);
            }
        }
    }

    if (SCAppLayerParserConfParserEnabled("tcp", proto_name)) {
        AppLayerParserRegisterParser(IPPROTO_TCP, ALPROTO_TLS, STREAM_TOSERVER,
                                     SSLParseClientRecord);

        AppLayerParserRegisterParser(IPPROTO_TCP, ALPROTO_TLS, STREAM_TOCLIENT,
                                     SSLParseServerRecord);
  		//注册一系列函数
    }
}
```

来好好看下SSLRegisterPatternsForProtocolDetection

```
static int SSLRegisterPatternsForProtocolDetection(void)
{
    if (SCAppLayerProtoDetectPMRegisterPatternCSwPP(IPPROTO_TCP, ALPROTO_TLS, "|01 00 02|", 5, 2,
                STREAM_TOSERVER, SSLProbingParser, 0, 3) < 0) {
        return -1;
    }

	/***** 到服务端方向 *****/
    /** TLSv1.2 */
    if (SCAppLayerProtoDetectPMRegisterPatternCS(
                IPPROTO_TCP, ALPROTO_TLS, "|01 03 03|", 3, 0, STREAM_TOSERVER) < 0) {
        return -1;
    }
    if (SCAppLayerProtoDetectPMRegisterPatternCS(
                IPPROTO_TCP, ALPROTO_TLS, "|16 03 03|", 3, 0, STREAM_TOSERVER) < 0) {
        return -1;
    }

    /***** 到客户端方向 *****/
    /** TLSv1.2 */
    if (SCAppLayerProtoDetectPMRegisterPatternCS(
                IPPROTO_TCP, ALPROTO_TLS, "|15 03 03|", 3, 0, STREAM_TOCLIENT) < 0) {
        return -1;
    }
    if (SCAppLayerProtoDetectPMRegisterPatternCS(
                IPPROTO_TCP, ALPROTO_TLS, "|16 03 03|", 3, 0, STREAM_TOCLIENT) < 0) {
        return -1;
    }
    if (SCAppLayerProtoDetectPMRegisterPatternCS(
                IPPROTO_TCP, ALPROTO_TLS, "|17 03 03|", 3, 0, STREAM_TOCLIENT) < 0) {
        return -1;
    }

    return 0;
}
```

SCAppLayerProtoDetectPMRegisterPatternCS函数内部直接调用AppLayerProtoDetectPMRegisterPattern

```
int SCAppLayerProtoDetectPMRegisterPatternCS(uint8_t ipproto, AppProto alproto, const char *pattern,
        uint16_t depth, uint16_t offset, uint8_t direction)
{
    int r = AppLayerProtoDetectPMRegisterPattern(ipproto, alproto,
            pattern, depth, offset,
            direction, 1 /* case-sensitive */,
            NULL, 0, 0);
}
```

继续

```
static int AppLayerProtoDetectPMRegisterPattern(uint8_t ipproto, AppProto alproto,
                                                const char *pattern,
                                                uint16_t depth, uint16_t offset,
                                                uint8_t direction,
                                                uint8_t is_cs,
                                                ProbingParserFPtr PPFunc,
                                                uint16_t pp_min_depth, uint16_t pp_max_depth)
{
    //获取IP协议类型对应的协议识别上下文，分为TCP/UDP/ICMP/Default为基础协议的上下文
    AppLayerProtoDetectCtxIpproto *ctx_ipp = &alpd_ctx.ctx_ipp[FlowGetProtoMapping(ipproto)];
    //模式匹配上下文
    AppLayerProtoDetectPMCtx *ctx_pm = NULL;

    //解析带引号的内容，解析失败则跳转到错误处理
    DetectContentData *cd = DetectContentParseEncloseQuotes(alpd_ctx.spm_global_thread_ctx, pattern);

	//赋值depth和offset
    cd->depth = depth;
    cd->offset = offset;
    //如果模式不区分大小写，销毁
    if (!is_cs) {
        /* 销毁原有的模式匹配上下文 */
        SpmDestroyCtx(cd->spm_ctx);
        
        //重新初始化模式匹配上下文(不区分大小写)
        cd->spm_ctx = SpmInitCtx(cd->content, cd->content_len, 1,
                                 alpd_ctx.spm_global_thread_ctx);
        cd->flags |= DETECT_CONTENT_NOCASE;
    }

    //根据方向选择模式匹配上下文
    if (direction & STREAM_TOSERVER)
        ctx_pm = (AppLayerProtoDetectPMCtx *)&ctx_ipp->ctx_pm[0];
    else
        ctx_pm = (AppLayerProtoDetectPMCtx *)&ctx_ipp->ctx_pm[1];

    /* 将解析的内容转换为协议识别签名，并添加到模式匹配上下文中 */
    AppLayerProtoDetectPMAddSignature(ctx_pm, cd, alproto, direction,
            PPFunc, pp_min_depth, pp_max_depth);
}
```





以tcp协议为基础的协议识别

AppLayerHandleTCPData -> 





# 五、参考链接

suricata的应用层协议识别

http://www.hyuuhit.com/2018/06/20/suricata-4-0-3-app-layer-protocol-detect/

看看suricata是如何进行协议的识别的