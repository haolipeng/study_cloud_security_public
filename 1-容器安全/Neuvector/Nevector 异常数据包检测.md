# 零、异常数据包类型概述

所有的异常数据包记录都在dpi_log.h头文件中

```
enum {
    DPI_THRT_NONE = 0,
    DPI_THRT_TCP_FLOOD,
    DPI_THRT_ICMP_FLOOD,
    DPI_THRT_IP_SRC_SESSION,
    DPI_THRT_BAD_PACKET,
    DPI_THRT_IP_TEARDROP,
    DPI_THRT_TCP_SYN_DATA,
    DPI_THRT_TCP_SPLIT_HDSHK,
    DPI_THRT_TCP_NODATA,
    DPI_THRT_TCP_SMALL_WINDOW,
    DPI_THRT_TCP_SMALL_MSS,
    DPI_THRT_PING_DEATH,
    DPI_THRT_DNS_LOOP_PTR,
    DPI_THRT_SSH_VER_1,
    DPI_THRT_SSL_HEARTBLEED,
    DPI_THRT_SSL_CIPHER_OVF,
    DPI_THRT_SSL_VER_2OR3,
    DPI_THRT_SSL_TLS_1DOT0,
    DPI_THRT_HTTP_NEG_LEN,
    DPI_THRT_HTTP_SMUGGLING,
    DPI_THRT_HTTP_SLOWLORIS,
    DPI_THRT_DNS_OVERFLOW,
    DPI_THRT_MYSQL_ACCESS_DENY,
    DPI_THRT_DNS_ZONE_TRANSFER,
    DPI_THRT_ICMP_TUNNELING,
    DPI_THRT_DNS_TYPE_NULL,
    DPI_THRT_SQL_INJECTION,
    DPI_THRT_APACHE_STRUTS_RCE,
    DPI_THRT_K8S_EXTIP_MITM,
    DPI_THRT_MAX,
};
```



异常包分为很多种类型，此文档只分析了ipv4数据包的情况，至于ipv6数据包的检测逻辑是类似的，无需赘述。

# 一、DDOS Flood异常

## 1、1 统计tcp synflood包

dpi_meter_synflood_inc函数是统计synflood攻击数据包的。

![image-20230628164326352](https://gitee.com/codergeek/picgo-image/raw/master/image-20230628164326352.png)

此函数的调用是在dpi_tcp_tracker函数中。



## 1、2 统计icmp synflood

![image-20230628174034705](https://gitee.com/codergeek/picgo-image/raw/master/image-20230628174034705.png)



```
void dpi_icmp_tracker(dpi_packet_t *p)
{
    dpi_session_t *s = p->session;
    dpi_meter_action_t act;

    if (p->ip_proto == IPPROTO_ICMP) {
        struct icmphdr *icmph = (struct icmphdr *)(p->pkt + p->l4);
        if (icmph->type == ICMP_DEST_UNREACH) {
            if (dpi_parse_embed_icmp(p) < 0){//解析icmp协议包
                dpi_set_action(p,DPI_ACTION_DROP);
            } else {
                dpi_set_action(p,DPI_ACTION_BYPASS);
            }
            return;
        }
    }
    ...省略代码...

    if (dpi_threat_status(DPI_THRT_ICMP_FLOOD) && dpi_is_client_pkt(p)) {
        act = dpi_meter_packet_inc(DPI_METER_ICMP_FLOOD, p);
        if (unlikely(act != DPI_METER_ACTION_NONE)) {
            return;
        }
    }
```

dpi_threat_status函数：判断DPI_THRT_ICMP_FLOOD对应的配置是否开启，后面代码可以做成动态配置的。



## 1、3 ddos攻击检测代码实现

neuvector中dp组件对于ddos攻击的检测以**会话统计**为主。

通过dpi_meter_session_inc函数来统计会话相关的数据。

其调用堆栈如下图所示：

![image-20230628144126586](https://gitee.com/codergeek/picgo-image/raw/master/image-20230628144126586.png)

根据协议种类来进行相应的统计，有icmp、ip、tcp、udp协议等。

```
int dpi_meter_session_inc(dpi_packet_t *p, dpi_session_t *s)
{
    //只统计入向流量
    if (!(s->flags & DPI_SESS_FLAG_INGRESS)) return DPI_METER_ACTION_NONE;

    bool ipv4 = FLAGS_TEST(s->flags, DPI_SESS_FLAG_IPV4);
    uint8_t *peer_ip = (uint8_t *)&s->client.ip;

    if (dpi_threat_status(DPI_THRT_IP_SRC_SESSION)) {
        bool fire = false, create = false;
        //meter_inc是个关键函数
        dpi_meter_t *m = meter_inc(DPI_METER_IP_SRC_SESSION, s->server.mac, peer_ip, ipv4, &fire, &create);
        if (likely(m != NULL)) {
        	//给会话设置meter标记
            FLAGS_SET(s->meter_flags, DPI_SESS_METER_FLAG_IP_SESSION);

            if (unlikely(create || fire)) {
                DPMsgThreatLog *log = &m->log;
                memset(log, 0, sizeof(*log));
                log_common(log, DPI_THRT_IP_SRC_SESSION);//用日志记录基础信息
                log_packet_flags(log, p, false);//用日志记录数据包标志
                log_packet_detail(log, p, false);//用日志记录数据包详情
            }

            if (unlikely(fire)) {
                // Aggregate log
                meter_info_t *info = &meter_info[m->type];
                if (th_snap.tick - m->last_log >= info->log_timeout && m->log_count > 0) {
                    if (likely(ipv4)) {
                    	//ipv4的ddos数据包
                        dpi_ddos_log(info->log_id, m,
                                     "Session rate %u from "DBG_IPV4_FORMAT" exceeds the shreshold %u",
                                     m->count, DBG_IPV4_TUPLE(m->peer_ip.ip4), info->lower_limit);
                    } else 
                    	//记录ipv6的ddos
                        dpi_ddos_log(info->log_id, m,
                                     "Session rate %u from "DBG_IPV6_FORMAT" exceeds the shreshold %u",
                                     m->count, DBG_IPV6_TUPLE(m->peer_ip.ip6), info->lower_limit);
                    }

                    m->last_log = th_snap.tick;
                    m->log_count = 0;
                }
            }
        }
    }

    return DPI_METER_ACTION_NONE;
}
```

上述流程请参考注释，核心函数是充当统计功能的meter_inc函数。

```
static dpi_meter_t *meter_inc(uint8_t type, uint8_t *ep_mac, uint8_t *peer_ip, bool ipv4, bool *fire, bool *create)
{
    dpi_meter_t key, *m;
    meter_info_t *info = &meter_info[type];

    memset(&key, 0, sizeof(key));
    make_key(&key, type, ep_mac, peer_ip, ipv4);//构造查找所需的key
    m = rcu_map_lookup(&th_meter_map, &key);//根据构造的key在th_meter_map表中进行查找
    if (m == NULL) {
    	//首次进行meter统计，则创建仪表盘,将其加入到时间轮中进行超时
        m = meter_alloc(type, ep_mac, peer_ip, ipv4);

        timer_wheel_entry_init(&m->ts_entry);
        timer_wheel_entry_start(&th_timer, &m->ts_entry,
                                meter_release, info->timeout, th_snap.tick);
        *create = true;
    } else {
    	//刷新时间轮的时间戳
        timer_wheel_entry_refresh(&th_timer, &m->ts_entry, th_snap.tick);
        *create = false;
    }

    if (info->rate) {
        uint32_t span = th_snap.tick - m->start_tick;

        if (span >= info->span) {
            m->last_count = m->count;
            m->count *= (float)(info->span - 1) / span;
            m->start_tick = th_snap.tick - (info->span - 1);
        } else if (m->count > info->upper_limit) {
            // count if reached before the span is reached, for example, a burst of incident. !! span can be 0.
            m->last_count = m->count / (span + 1);
            m->count *= (float)(info->span - span) / info->span;
        }

        m->count ++;

        if (likely(m->last_count < info->lower_limit)) {
            FLAGS_UNSET(m->flags, DPI_METER_FLAG_ON);
            *fire = false;
            return m;
        } else if (m->last_count >= info->upper_limit) {//超过设置的阈值，则设置fire为true，表示触发ddos检测
            m->log_count ++;
            FLAGS_SET(m->flags, DPI_METER_FLAG_ON);
            *fire = true;
            return m;
        } else if (m->flags & DPI_METER_FLAG_ON) {
            m->log_count ++;
            *fire = true;
            return m;
        }
    } else {
        m->count ++;
        //超过设置的阈值
        if (unlikely(m->count >= info->upper_limit)) {
            m->log_count ++;
            *fire = true;
            return m;
        }
    }

    *fire = false;
    return m;
}
```

th_meter_map为用于仪表盘统计的数据结构

```
#define th_meter_map    (g_dpi_thread_data[THREAD_ID].meter_map) //每个线程拥有自己的meter_map
```



# 二、损坏的数据包

损坏的数据包，其对应的宏为DPI_THRT_BAD_PACKET

```
#define LOG_BAD_PKT(p, format, args...) \
        dpi_threat_trigger(DPI_THRT_BAD_PACKET, p, format, ##args)
```

![image-20230628183124548](https://gitee.com/codergeek/picgo-image/raw/master/image-20230628183124548.png)

损坏的数据包分为四种协议，icmp、ip、tcp、udp协议。



## 2、1 icmp异常数据包

icmp数据包长度小于正常协议头部长度

```
static int dpi_parse_icmp(dpi_packet_t *p)
{
    struct icmphdr *icmph = (struct icmphdr *)(p->pkt + p->l4);
    uint16_t icmp_len = p->len - p->l4;
    uint16_t sport, dport;

	//icmp数据包长度小于icmp正常协议头部长度
    if (icmp_len < sizeof(*icmph)) {
        DEBUG_LEVEL(DBG_PACKET, "Bad icmp packet length %u\n", icmp_len);
        LOG_BAD_PKT(p, "Bad icmp packet length %u", icmp_len);
        return -1;
    }
}
```



## 2、2 ip异常数据包

分为三种情况

- ip数据包长度小于正常ip协议头部长度
- ipv4数据包的version字段不为4
- ip头部长度不符合条件

```

static int dpi_parse_ipv4_hdr(dpi_packet_t *p)
{
    struct iphdr *iph;
    uint16_t ip_caplen, ip_len, iph_len;
	
    ip_caplen = p->cap_len - sizeof(struct ethhdr);
    //ip数据包长度小于正常ip协议头部长度
    if (ip_caplen < sizeof(struct iphdr)) {
        DEBUG_LEVEL(DBG_PACKET, "Bad ipv4 packet length %u\n", ip_caplen);
        LOG_BAD_PKT(p, "Bad ipv4 packet length %u", ip_caplen);
        return -1;
    }

    iph = (struct iphdr *)(p->pkt + sizeof(struct ethhdr));

	//ipv4数据包的version字段不为4
    if (iph->version != 4) {
        DEBUG_LEVEL(DBG_PACKET, "Bad ip version %u\n", iph->version);
        LOG_BAD_PKT(p, "Bad ip version %u", iph->version);
        return -1;
    }

    ip_len = ntohs(iph->tot_len);
    iph_len = get_iph_len(iph);

	//以下条件不合法：
	//头部长度小于协议头长度
	//头部长度大于ip 总长度
	//ip总长度 大于 数据捕获长度
    if (iph_len < sizeof(*iph) || ip_len < iph_len || ip_len > ip_caplen) {
        DEBUG_LEVEL(DBG_PACKET,
                    "Bad ipv4 header length: ip_len=%u, hdr_len=%u caplen=%u\n",
                    ip_len, iph_len, ip_caplen);
        LOG_BAD_PKT(p, "Bad ipv4 header length: ip_len=%u, hdr_len=%u caplen=%u\n",
                       ip_len, iph_len, ip_caplen);
        return -1;
    }

    ...省略代码...

    return 0;
}
```



## 2、3 tcp异常数据包

```
static int dpi_parse_tcp(dpi_packet_t *p)
{
    struct tcphdr *tcph = (struct tcphdr *)(p->pkt + p->l4);
    uint16_t tcp_len = p->len - p->l4;
    uint16_t tcph_len;

	//tcp数据包长度小于正常tcp协议头部长度
    if (tcp_len < sizeof(*tcph)) {
        DEBUG_LEVEL(DBG_PACKET, "Bad tcp packet length %u\n", tcp_len);
        LOG_BAD_PKT(p, "Bad tcp packet length %u", tcp_len);
        return -1;
    }

	//tcp头部长度小于正常tcp协议头部长度
    tcph_len = get_tcph_len(tcph);
    if (tcph_len < sizeof(*tcph) || tcp_len < tcph_len) {
        DEBUG_LEVEL(DBG_PACKET, "Bad tcp header length: tcp_len=%u tcph_len=%u\n",
                                tcp_len, tcph_len);
        LOG_BAD_PKT(p, "Bad tcp header length: tcp_len=%u tcph_len=%u", tcp_len, tcph_len);
        return -1;
    }

	//tcp所有标志位是否合法
    if (BITMASK_TEST(tcp_bad_flag_mask, tcph->th_flags & TCP_FLAG_MASK)) {
        char flags[10];
        DEBUG_LEVEL(DBG_PACKET, "Bad tcp flags %s\n", get_tcp_flag_string(tcph, flags));
        LOG_BAD_PKT(p, "Bad tcp flags %s", get_tcp_flag_string(tcph, flags));
        return -1;
    }

    // Checksum
    if (unlikely(g_io_config->enable_cksum)) {
        if (pkt_cksum != cksum) {
        	//tcp校验和验证失败
            DEBUG_LEVEL(DBG_PACKET, "Bad tcp checksum 0x%x, should be 0x%x\n",
                                    htons(pkt_cksum), htons(cksum));
            tcph->check = cksum; 
        }
    }

    if (tcph_len > sizeof(*tcph)) {
        if (dpi_parse_tcp_options(p) < 0) {
            return -1;
        }

        // Check TCP TS option can't be 0 for non-SYN packet
        if ((p->flags & DPI_PKT_FLAG_TCP_TS) &&
            p->tcp_ts_value == 0 && p->tcp_ts_echo ==0 &&
            tcph->ack && !tcph->syn) {
            DEBUG_LEVEL(DBG_PACKET, "Bad tcp zero timestamp\n");
            LOG_BAD_PKT(p, "Bad tcp zero timestamp");
            return -1;
        }

#define DPI_MIN_TCP_MSS 256
        if (p->tcp_mss > 0 && p->tcp_mss < DPI_MIN_TCP_MSS) {
            DEBUG_LEVEL(DBG_PACKET, "Small TCP MSS, mss=%u (<%u)\n", p->tcp_mss, DPI_MIN_TCP_MSS);
            dpi_threat_trigger(DPI_THRT_TCP_SMALL_MSS, p, "mss=%u (<%u)", p->tcp_mss, DPI_MIN_TCP_MSS);
        }
    }

    ...省略代码...

    return 0;
}
```

## 2、4 udp异常数据包

```
static int dpi_parse_udp(dpi_packet_t *p)
{
    struct udphdr *udph = (struct udphdr *)(p->pkt + p->l4);
    uint16_t udp_len = p->len - p->l4;
    
	//udp数据包长度小于正常udp协议头部长度
    if (udp_len < sizeof(*udph)) {
        DEBUG_LEVEL(DBG_PACKET, "Bad udp packet length %u\n", udp_len);
        LOG_BAD_PKT(p, "Bad udp packet length %u", udp_len);
        return -1;
    }
    ...省略代码...
}
```





# 三、应用层协议 异常数据包检测

## 3、1 SSH协议

DPI_THRT_SSH_VER_1

```
// Return offset of end of line; return 0 if more data is needed, -1 if not SSH
static int ssh_banner(dpi_packet_t *p, uint8_t *ptr, uint32_t len)
{
    ssh_data_t *data = dpi_get_parser_data(p);
    uint8_t *end = ptr + len, *eol = ptr, *ver, *space = NULL;

    ...省略相关解析代码...
    
    if ((eol - ptr) > 7 && ptr[4] == '1' && ptr[6] == '9' && ptr[7] == '9') {
        data->ver = (data->ver == 1) ? 1 : 2;
    } else {
        data->ver = (data->ver == 1) ? 1 : ptr[4] - '0';
    }

    if (data->ver < 2) {
    	//ssh的协议版本小于2，则上报异常
        DEBUG_ERROR(DBG_PARSER, "SSH version 1\n");
        dpi_threat_trigger(DPI_THRT_SSH_VER_1, p, NULL);
    }

    return eol - ptr;
}
```



## 3、2 SSL协议

以DPI_THRT_SSL_HEARTBLEED异常检测为例，其函数堆栈

ssl_parser

​	ssl_parse_v3

```
int ssl_parse_v3(dpi_packet_t *p, ssl_wing_t *w, uint8_t *ptr, ssl_record_t *rec)
{
    int ret = FORMAT_MATCH;

    switch (rec->type) {
    //心跳包
    case SSL3_RT_HEARTBEAT:
        DEBUG_LOG(DBG_PARSER, p, "SSLv3: heartbeat\n");

        if (ptr[0] == SSL3_HBT_REQUEST) {
            uint16_t hbt_len = GET_BIG_INT16(ptr + 1);
            // 1024 is a min size to reduce false positive.
            if (hbt_len - (rec->len - 3) > 1024) {
                DEBUG_ERROR(DBG_PARSER, "SSLv3: heartbleed, heartbeat=%u record=%u\n", hbt_len, rec->len);
                dpi_threat_trigger(DPI_THRT_SSL_HEARTBLEED, p, "heartbeat=%u, record=%u", hbt_len, rec->len);
            }
        }
        break;
    }

    return ret;
}
```

剩下的几种分别搜索对应代码即可。

DPI_THRT_SSL_CIPHER_OVF,

DPI_THRT_SSL_VER_2OR3,

DPI_THRT_SSL_TLS_1DOT0,



## 3、3 HTTP协议异常包

HTTP协议异常包分为以下三种：

DPI_THRT_HTTP_SLOWLORIS,

DPI_THRT_HTTP_NEG_LEN,

DPI_THRT_HTTP_SMUGGLING,

以DPI_THRT_HTTP_SLOWLORIS为例，其函数调用堆栈如下：

```
int dpi_http_tick_timeout(dpi_session_t *s, void *parser_data)
{
    http_data_t *data = parser_data;

    DEBUG_LOG_FUNC_ENTRY(DBG_SESSION | DBG_PARSER | DBG_TIMER, NULL);

    if (data->url_start_tick > 0) {
        if (th_snap.tick - data->url_start_tick >= HTTP_HEADER_COMPLETE_TIMEOUT) {
            //异常包检测
            dpi_threat_log_by_session(DPI_THRT_HTTP_SLOWLORIS, s,
                      "Header duration=%us, threshold=%us",
                      th_snap.tick - data->url_start_tick, HTTP_HEADER_COMPLETE_TIMEOUT);
            return DPI_SESS_TICK_RESET;
        }
    } else if (data->last_body_tick > 0) {
        switch (data->client.section) {
        case HTTP_SECTION_FIRST_BODY:
            if (th_snap.tick - data->last_body_tick >= HTTP_BODY_FIRST_TIMEOUT) {
                //异常包检测
                dpi_threat_log_by_session(DPI_THRT_HTTP_SLOWLORIS, s,
                          "First body packet interval=%us, threshold=%us",
                          th_snap.tick - data->last_body_tick, HTTP_BODY_INTERVAL_TIMEOUT);
                return DPI_SESS_TICK_RESET;
            }
            break;
        case HTTP_SECTION_BODY:
            /* Easy to get false positive, maybe 3s is too short.
            if (th_snap.tick - data->last_body_tick >= HTTP_BODY_INTERVAL_TIMEOUT) {
 				//异常包检测
                dpi_threat_log_by_session(DPI_THRT_HTTP_SLOWLORIS, s,
                          "Body packet interval=%us, threshold=%us",
                          th_snap.tick - data->last_body_tick, HTTP_BODY_INTERVAL_TIMEOUT);
                return DPI_SESS_TICK_RESET;
            }
            */
            break;
        }
    }

    return DPI_SESS_TICK_CONTINUE;
}
```



## 3、4 DNS协议

HTTP协议异常包分为以下四种：

DPI_THRT_DNS_LOOP_PTR,

DPI_THRT_DNS_OVERFLOW,

DPI_THRT_DNS_ZONE_TRANSFER,

DPI_THRT_DNS_TYPE_NULL,

下面我们以DPI_THRT_DNS_ZONE_TRANSFER为例，解析dns请求并检测到DPI_THRT_DNS_ZONE_TRANSFER。

```
static int dns_question(dpi_packet_t *p, uint8_t *ptr, int len, int shift, int count, dns_question_t *questions, int *qt_count)
{
    char labels[MAX_LABEL_LEN];
    while (count > 0) {
        labels[0] = 0;
        int jump = get_dns_name(p, ptr, len, shift, labels);
        if (jump < 0) {
            return jump;
        }

        shift += jump + 4;
        if (shift > len) return -1;

        uint16_t type = ntohs(*(uint16_t *)(ptr+shift-4));

        if (type == DNS_TYPE_A && questions != NULL) {
            if (jump > 1) {
                strcpy(questions[(*qt_count)++].question, labels);
            }
        }else if (type == DNS_TYPE_AXFR) {
        	//检测到DNS Zone Transfer
            DEBUG_LOG(DBG_PARSER, p, "DNS Zone Transfer AXFR.\n");
            dpi_threat_trigger(DPI_THRT_DNS_ZONE_TRANSFER, p, "DNS Zone Transfer AXFR");
        }else if (type == DNS_TYPE_IXFR) {
        	//检测到DNS Zone Transfer
            DEBUG_LOG(DBG_PARSER, p, "DNS Zone Transfer IXFR.\n");
            dpi_threat_trigger(DPI_THRT_DNS_ZONE_TRANSFER, p, "DNS Zone Transfer IXFR");
        }else if (type == DNS_TYPE_NULL) {
        	//检测到DNS NULL type
            DEBUG_LOG(DBG_PARSER, p, "DNS NULL type.\n");
            dpi_threat_trigger(DPI_THRT_DNS_TYPE_NULL, p, "DNS NULL type");
        }
        count --;
    }

    return shift;
}
```



## 3、5 数据库协议

DPI_THRT_MYSQL_ACCESS_DENY,

DPI_THRT_SQL_INJECTION,

以DPI_THRT_SQL_INJECTION为例；

```
//Embedded sql-injection threat detection based on PCRE pattern matching
void check_sql_query(dpi_packet_t *p, uint8_t *query, int len, int app)
{
    int i;
    for (i=0; i<sizeof(sql_injections)/sizeof(sql_injections[0]); i++) {
        if (sql_injections[i].cb(p, sql_injections[i].recompiled, 
                                 sql_injections[i].signature, 
                                 sql_injections[i].substr_values,
                                 query, len)) {
            dpi_threat_trigger(DPI_THRT_SQL_INJECTION, p, 
                    "SQL Injection, application=%d", app);
        }
    }
}

//检测sql注入的正则表达式
static sql_injection_t sql_injections[] ={   
//SELECT * FROM users WHERE name='adam' or 'x' = 'x'
{injection_0, NULL, 0, "(?i)^SELECT.*\'\\s+(?:or|OR)\\s+((?:\'|\")?[0-9a-zA-Z_]+(?:\'|\")?)\\s*=\\s*\\1"},
};
```



## 3、6 Web服务器类

DPI_THRT_APACHE_STRUTS_RCE,

```
static void buffer_body(http_ctx_t *ctx, uint8_t *ptr, int len) {
    http_wing_t *w = ctx->w;

    // This is to specifically detect threats in client-side XML, e.g. CVE-2017-9805
    if (unlikely(is_request(w) && w->ctype == HTTP_CTYPE_APPLICATION_XML && w->encode == HTTP_ENCODE_NONE)) {
        http_data_t *data = ctx->data;
        if (data->body_buffer == NULL) {
            data->body_buffer = malloc(2048);
        }
        if (data->body_buffer != NULL && data->body_buffer_len < 2048) {
            int copy = min(len, 2048 - data->body_buffer_len);
            memcpy(&data->body_buffer[data->body_buffer_len], ptr, copy);
            data->body_buffer_len += copy;

            if (unlikely(th_apache_struts_re_data == NULL)) {
                th_apache_struts_re_data  = pcre2_match_data_create_from_pattern(apache_struts_re, NULL);
            }

			//有struct相关数据，则用pcre正则表达式进行匹配
            if (likely(th_apache_struts_re_data != NULL)) {
                int rc = pcre2_match(apache_struts_re,
                        (PCRE2_SPTR)data->body_buffer, data->body_buffer_len,
                        0, 0, th_apache_struts_re_data, NULL);
                if (rc >= 0) {
                    dpi_threat_trigger(DPI_THRT_APACHE_STRUTS_RCE, ctx->p, NULL);
                }
            }
        }
    }
}
```



## 3、7 杂项

DPI_THRT_ICMP_TUNNELING,

DPI_THRT_K8S_EXTIP_MITM,

以DPI_THRT_ICMP_TUNNELING为例：

```
void dpi_icmp_tunneling_check(dpi_packet_t *p)
{
    uint32_t len = dpi_pkt_len(p);
    uint8_t *ptr = dpi_pkt_ptr(p);
    dpi_wing_t *client = &p->session->client;
    struct icmphdr *icmph = (struct icmphdr *)(p->pkt + p->l4);

    if (client->icmp_times == ICMP_TUNNEL_REPORTED) {
        return;
    }

    if (icmph->type == ICMP_ECHO) {
        uint32_t hash = sdbm_hash(ptr, len);

        client->icmp_echo_hash = hash;
        client->icmp_echo_seq = icmph->un.echo.sequence;
    } else if (icmph->type == ICMP_ECHOREPLY) {
        uint32_t hash = sdbm_hash(ptr, len);

        if (client->icmp_echo_seq == icmph->un.echo.sequence && client->icmp_echo_hash != 0) {
            if (client->icmp_echo_hash == hash) {
                client->icmp_times = 0;
            } else {
                client->icmp_times ++;
                if (client->icmp_times == ICMP_TUNNEL_THRESHOD) {
                    client->icmp_times = ICMP_TUNNEL_REPORTED;
                    dpi_threat_trigger(DPI_THRT_ICMP_TUNNELING, p, "ICMP tunneling");
                }
            }
            client->icmp_echo_hash = 0;
            client->icmp_echo_seq = 0;
        }
    }
}
```

icmp隧道的检测显得略微简单了一点。

# 四、其他异常数据包

**DPI_THRT_TCP_SYN_DATA类型的异常包**

```
void dpi_tcp_tracker(dpi_packet_t *p)
{
    struct tcphdr *tcph = (struct tcphdr *)(p->pkt + p->l4);
    dpi_session_t *s = p->session;
    uint32_t seq = p->raw.seq, ack = ntohl(tcph->th_ack);
    uint32_t len = p->raw.len, end = seq + len;

    DEBUG_LOG_FUNC_ENTRY(DBG_PACKET | DBG_TCP, p);

    if (unlikely(tcph->syn && len != 0)) {
        DEBUG_LEVEL(DBG_SESSION | DBG_TCP, "SYN with data\n");
        dpi_threat_trigger(DPI_THRT_TCP_SYN_DATA, p, NULL);//syn数据包中带有数据
        p->session = NULL;
        return;
    }
    ...省略代码...
}
```

**TCP相关的异常包**

DPI_THRT_TCP_NODATA,
DPI_THRT_TCP_SMALL_WINDOW,
DPI_THRT_TCP_SMALL_MSS,



```
// This is a very simple protection schema, to reset the session if no data sent from client
// for a short period of time. Possible improvements include,
// 1. Set a shorter timeout after TWH done, count how many concurrent sessions reach and stay
//    in this state, start resetting sessions if threashold is reached;
// 2. Instead of counting total sessions, count how fast sessions reach the shorter timeout.
//
// For simplicity now, logic #2 is used.
static void session_nodata_timeout(timer_entry_t *n)
{
    dpi_session_t *s = STRUCT_OF(n, dpi_session_t, ts_entry);

    if (dpi_threat_status(DPI_THRT_TCP_NODATA)) {
        if (dpi_meter_session_rate(DPI_METER_TCP_NODATA, s)) {
            DEBUG_LOG(DBG_SESSION, NULL, "Trigger TCP nodata\n");
            dpi_threat_log_by_session(DPI_THRT_TCP_NODATA, s,
                                      "Client send no data after open connection for %us",
                                      SESS_TIMEOUT_TCP_ACTIVE_NODATA);
            if (!FLAGS_TEST(s->flags, DPI_SESS_FLAG_TAP)) {
                dpi_session_reset(s, DPI_THRT_TCP_NODATA);
                dpi_session_timer_start(s, dpi_session_timeout, SESS_TIMEOUT_TCP_RST);
            } else {
                dpi_session_timer_start(s, dpi_session_timeout, SESS_TIMEOUT_TCP_ACTIVE);
            }
        } else {
            DEBUG_LOG(DBG_SESSION, NULL, "Ignore TCP nodata\n");
            dpi_session_timer_start(s, dpi_session_timeout, SESS_TIMEOUT_TCP_ACTIVE);
        }
    } else {
        DEBUG_LOG(DBG_SESSION, NULL, "Ignore TCP nodata\n");
        dpi_session_timer_start(s, dpi_session_timeout, SESS_TIMEOUT_TCP_ACTIVE);
    }
}
```

