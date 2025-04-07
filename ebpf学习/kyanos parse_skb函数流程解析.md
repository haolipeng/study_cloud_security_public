每天确定一个小目标，言出必行的去一点点实现它。

# 一、parse_skb函数概览

关于 skb 的解析主要在 `parse_skb` 函数实现，该函数主要工作是根据 `skb_buffer` 包以及当前阶段进行数据包解析

## 1、1 函数参数

```
static __always_inline int parse_skb(
    void* ctx,                  // BPF 程序上下文
    struct sk_buff *skb,        // 网络数据包缓冲区
    bool sk_not_ready,          // socket是否未就绪
    enum step_t step           // 当前处理阶段(DEV_IN/TCP_IN等)
)
```

parse_skb函数代码的全貌如下：

```
static __always_inline int parse_skb(void* ctx, struct sk_buff *skb, bool sk_not_ready, enum step_t step) {
    //从skb(socket buffer)结构体中读取其sk成员(sock结构指针)。
    struct sock* sk = {0};
    BPF_CORE_READ_INTO(&sk, skb, sk);
    
    // 1. 获取初始序列号
    u32 inital_seq = 0;
    struct sock_key key = {0};
    if (sk) {
        struct sock_common sk_cm = {0};
        BPF_CORE_READ_INTO(&sk_cm, sk, __sk_common);
        if (sk_cm.skc_addrpair && !sk_not_ready) {
            parse_sock_key(skb, &key);
            int *found = bpf_map_lookup_elem(&sock_xmit_map, &key);
            if (found == NULL) {
                return 0;
            }
            bpf_probe_read_kernel(&inital_seq, sizeof(inital_seq), found);
        }
    }

    // 2. 读取数据包头部信息
    u16 network_header = 0, mac_header = 0, trans_header = 0;
	// 获取网络层头部偏移量(相对于skb数据缓冲区开始位置)
    BPF_CORE_READ_INTO(&network_header, skb, network_header);
	// 获取mac层头部偏移量(相对于skb数据缓冲区开始位置)
    BPF_CORE_READ_INTO(&mac_header, skb, mac_header);
	// 获取传输层头部偏移量(相对于skb数据缓冲区开始位置)
    BPF_CORE_READ_INTO(&trans_header, skb, transport_header);

    // 声明一个 void* 类型的指针变量 data 并初始化为 0
	void* data = {0};

	// 读取 skb->head 的值（指向数据包缓冲区的起始地址）并存储到 data 变量中
	BPF_CORE_READ_INTO(&data, skb, head);

	// 通过 data（缓冲区起始地址）加上之前获取的 network_header 偏移量，
	// 得到 IP 头部的实际内存地址，并将结果存储在 ip 指针中
	void* ip = data + network_header;
    void* l3 = NULL;
    void* l4 = NULL;
    u16 l3_proto = 0;

    // 3. 解析L2/L3层
    if (!skb_l2_check(mac_header)) { // L2包处理
        if (mac_header != network_header) {
            struct ethhdr *eth = data + mac_header;
            l3 = (void *)eth + ETH_HLEN;
            BPF_CORE_READ_INTO(&l3_proto, eth, h_proto);
            l3_proto = bpf_ntohs(l3_proto);
        }
    } else { // 非L2包处理
    	//通过 BPF_CORE_READ_INTO 从socket buffer（skb）中安全地读取协议类型到_protocol变量中
        u16 _protocol = 0;
        BPF_CORE_READ_INTO(&_protocol, skb, protocol);
        //从网络字节序转换为主机字节序,并将结果存储到l3_proto
        l3_proto = bpf_ntohs(_protocol);
        
        if (l3_proto == ETH_P_IP || l3_proto == ETH_P_IPV6) {
        	//如果是标准的IPv4和IPv6数据包，
        	//则通过将数据包起始地址（data）加上网络层头部偏移量（network_header）来定位到L3层的起始位置。
            l3 = data + network_header;
        } else if (mac_header >= network_header) {
        	//MAC头部位置大于或等于网络层头部位置的情况
            l3 = data + network_header;
            l3_proto = ETH_P_IP;
        } else {
            return BPF_OK;
        }
    }

    // 4. 检查是否为IP协议
    if (l3_proto != ETH_P_IP && l3_proto != ETH_P_IPV6) {
        return BPF_OK;
    }

    // 5. 解析L4层
    if (!skb_l4_check(trans_header, network_header)) {
        l4 = data + trans_header;
    }

    // 6. 处理IP头和TCP头
    u32 tcp_len;
    u8 proto_l4 = 0;
    
    if (l3_proto == ETH_P_IP) {
        struct iphdr *ipv4 = ip;
        u16 tot_len16 = 0;
		// 获取IP头中的总长度字段（tot_len）
        BPF_CORE_READ_INTO(&tot_len16, ipv4, tot_len);
		// 将IP头中的总长度转换为网络字节序
        u32 len = bpf_ntohs(tot_len16);
        
        u8 ip_hdr_len = 0;
		//计算实际的ip头长度（原始值需要乘以4）
        bpf_probe_read_kernel(&ip_hdr_len, sizeof(((u8 *)ip)[0]), &(((u8 *)ip)[0]));
        ip_hdr_len = get_ip_header_len(ip_hdr_len);
        
		//如果L4指针未设置，则将其设置为IP
        l4 = l4 ? l4 : ip + ip_hdr_len;
        BPF_CORE_READ_INTO(&proto_l4, ipv4, protocol);
        tcp_len = len - ip_hdr_len;
    } else { // IPv6
        struct ipv6hdr *ipv6 = ip;
        proto_l4 = _(ipv6->nexthdr);
        tcp_len = _C(ipv6, payload_len);
        l4 = l4 ? l4 : ip + sizeof(*ipv6);
    }

    // 7. 处理TCP协议
    if (proto_l4 != IPPROTO_TCP) {
        return BPF_OK;
    }

    struct tcphdr *tcp = l4;
    
    // 8. 处理序列号
    if (!inital_seq) {
        if (l3_proto == ETH_P_IP) {
            parse_sock_key_from_ipv4_tcp_hdr(&key, ip, tcp);
        } else {
            parse_sock_key_from_ipv6_tcp_hdr(&key, ip, tcp);
        }

        if (step == DEV_IN || step == DEV_OUT) {
            struct sock_key *translated_flow = bpf_map_lookup_elem(&nat_flow_map, &key);
            if (translated_flow != NULL) {
                key = *translated_flow;
            }
        }

        int *found = bpf_map_lookup_elem(&sock_xmit_map, &key);
        if (found == NULL) {
            if (step == DEV_IN) {
                BPF_CORE_READ_INTO(&inital_seq, tcp, seq);
                inital_seq = bpf_ntohl(inital_seq);
                uint8_t flag = 0;
                bpf_probe_read_kernel(&flag, sizeof(uint8_t), &(((u8 *)tcp)[13]));
                if ((flag & (1 << 1)) == 0) {
                    inital_seq--;
                }
                bpf_map_update_elem(&sock_xmit_map, &key, &inital_seq, BPF_NOEXIST);
            } else {
                return BPF_OK;
            }
        } else {
            bpf_probe_read_kernel(&inital_seq, sizeof(inital_seq), found);
        }
    }

    // 9. 生成事件
    struct parse_kern_evt_body body = {0};
    body.ctx = ctx;
    body.inital_seq = inital_seq;
    body.key = &key;
    body.tcp = tcp;
    body.len = tcp_len;
    body.step = step;

    struct net_device *dev = _C(skb, dev);
    body.ifindex = dev ? _C(dev, ifindex) : _C(skb, skb_iif);

    if (step >= NIC_IN) {
        reverse_sock_key_no_copy(&key);
    }
    
    report_kern_evt(&body);
    return 1;
}
```

## 1、2 核心逻辑

TODO：这里需要画图了。



# 二、sk_buff网络数据包解析

skb数据包的解析逻辑都在parse_skb()函数中。

## 2、1 获取skb中各协议头的偏移量

想获取skb中各协议头的偏移量，首先我们查看下相关结构体：

先熟悉下sk_buff结构体中关于以太网层/网络层/传输层的成员

```
struct sk_buff {
    /* These two members must be first. */
    struct sk_buff        *next;
    struct sk_buff        *prev;

    ...

    __u16            transport_header;
    __u16            network_header;
    __u16            mac_header;

    ...
};
```

上述引用的代码，摘自https://elixir.bootlin.com/linux/v6.12.6/source/include/linux/skbuff.h#L1062

从上述内容中，我们能看到结构体中名为transport_header、network_header、mac_header的__u16类型的成员



```
+------------------+
|    MAC Header    | ← mac_header
+------------------+
|     IP Header    | ← network_header
+------------------+
|    TCP Header    | ← transport_header
+------------------+
|      Data        |
+------------------+

static __always_inline int parse_skb(void* ctx, struct sk_buff *skb, bool sk_not_ready, enum step_t step) {
	......
	// 2. 读取数据包头部信息
	u16 network_header = 0, mac_header = 0, trans_header = 0;
	// 获取网络层头部偏移量(相对于skb数据缓冲区开始位置)
    BPF_CORE_READ_INTO(&network_header, skb, network_header);
	// 获取mac层头部偏移量(相对于skb数据缓冲区开始位置)
    BPF_CORE_READ_INTO(&mac_header, skb, mac_header);
	// 获取传输层头部偏移量(相对于skb数据缓冲区开始位置)
    BPF_CORE_READ_INTO(&trans_header, skb, transport_header);
    ......
}
```

BPF_CORE_READ_INTO 宏的使用方法：

**基础语法：**

```
BPF_CORE_READ_INTO(dst, src, field)
```

参数解释：

**dst参数:** dst 是目标地址，它指向要存储读取结果的变量的指针。在使用时必须使用 & 操作符，且该变量的类型必须与要读取的字段类型相匹配，比如 &mac_header、&data、&length 等。

**src参数:** src 是源对象，通常是一个结构体指针。在网络编程中最常见的是 struct sk_buff *skb，也可以是其他结构体指针如 sock、tcp_sock 等。

**field参数：**  field 是要读取的字段名，它必须是源对象结构体中实际存在的字段，可以是基本类型或指针类型，例如 head、mac_header、len 等。

这个宏会安全地从 src->field 读取数据，并将结果存储到 dst 指向的内存位置。



BPF_CORE_READ_INTO 宏的好处：

- 能妥善处理不同内核版本之间可能存在的结构体偏移的差异

- 确保内存访问的安全性

  

下面逐行分析下这几个函数：

```
skb->head
    |
    v
    +----------------+----------------+----------------+----------------+
    |    MAC 头      |     IP 头      |    TCP/UDP头   |     数据       |
    +----------------+----------------+----------------+----------------+
    ^                ^                ^
    |                |                |
mac_header = 0    network_header    trans_header
                    = 14              = 34
```

当 BPF_CORE_READ_INTO(&network_header, skb, network_header) 执行时，它会读取skb->network_header 的值，这个值通常是14（因为以太网的长度通常是14）， 结果会被存储到network_header变量中。



当 BPF_CORE_READ_INTO(&mac_header, skb, mac_header) 执行时，它会读取 skb->mac_header 的值，这个值通常是 0（普通的以太网帧为0，但如果有VLAN标签的话，这个值不会是0），结果会被存储到 mac_header 变量中。



当 BPF_CORE_READ_INTO(&trans_header, skb, transport_header) 执行时，它会读取 skb->transport_header 的值，这个值通常是 34（因为传输层头部在标准 IPv4 数据包中位于 MAC 头部 14 字节 + IP 头部 20 字节之后），结果会被存储到 trans_header 变量中。

## 2、2 以太网数据解析

linux内核中以太网的结构体定义如下：

```
struct ethhdr {
	unsigned char	h_dest[ETH_ALEN];	/* destination eth addr	*/
	unsigned char	h_source[ETH_ALEN];	/* source ether addr	*/
	__be16		h_proto;		/* packet type ID field	*/
} __attribute__((packed));
```



```
static __always_inline int parse_skb(void* ctx, struct sk_buff *skb, bool sk_not_ready, enum step_t step) {
	......省略N多代码......
	// 3. 解析L2/L3层
    if (!skb_l2_check(mac_header)) { // L2包处理（正常情况）
        if (mac_header != network_header) {
            // 获取mac层头部
            struct ethhdr *eth = data + mac_header;
			// 获取网络层头部
            l3 = (void *)eth + ETH_HLEN;
			// 从以太网头部结构体中读取 h_proto 字段（协议类型），获取网络层协议类型
            BPF_CORE_READ_INTO(&l3_proto, eth, h_proto);
			// 将网络层协议类型转换为网络字节序
            l3_proto = bpf_ntohs(l3_proto);
        }
    } else { // 非L2包处理（异常情况）
        //通过 BPF_CORE_READ_INTO 从socket buffer（skb）中安全地读取协议类型到_protocol变量中
        u16 _protocol = 0;
        BPF_CORE_READ_INTO(&_protocol, skb, protocol);
        //从网络字节序转换为主机字节序,并将结果存储到l3_proto
        l3_proto = bpf_ntohs(_protocol);
        
        if (l3_proto == ETH_P_IP || l3_proto == ETH_P_IPV6) {
        	//如果是标准的IPv4和IPv6数据包，
        	//则通过将数据包起始地址（data）加上网络层头部偏移量（network_header）来定位到L3层的起始位置。
            l3 = data + network_header;
        } else if (mac_header >= network_header) {
        	//MAC头部位置大于或等于网络层头部位置的情况
            l3 = data + network_header;
            l3_proto = ETH_P_IP;
        } else {
        	//兜底方案，前面方案都不符合，则无序进行下一步分处理
            return BPF_OK;
        }
    }
	......省略N多代码......
}
```

开始解析L2/L3二层和三层的协议字段。

skb_l2_check函数的详情如下：

```
static __always_inline bool skb_l2_check(u16 header) 
{
	return !header || header == (u16)~0U;
}
```

函数功能：skb_l2_check 这个函数是用来检查二层（L2）网络头部的有效性的。

函数会在以下两种情况返回 true：

- header 为 0
- header 为 0xFFFF（二进制为 1111 1111 1111 1111）

这个函数通常用于网络数据包处理中，用来判断L2头部是否有效或是否存在。如果返回 true，通常表示这个头部无效或不存在，需要特殊处理。

| mac_header 值 | skb_l2_check 结果 | 含义     |
| ------------- | ----------------- | -------- |
| 0             | true              | 非二层包 |
| 0xFFFF        | true              | 非二层包 |
| 其他值        | false             | 二层包   |

但是更具体点的话，什么类型的网络数据包，其mac_header值为0或者0xFFFF呢？

| **场景**                     | **说明**                                                     |
| ---------------------------- | ------------------------------------------------------------ |
| **本地回环接口（Loopback）** | 在本地回环接口（如 `lo`）中，数据包不经过实际的链路层        |
| **隧道封装协议**             | 使用隧道协议（如 GRE、IP-in-IP、VXLAN 等）时，数据包可能包含多层封装 |
| **虚拟网络接口**             | 某些虚拟网络接口（如 VLAN、桥接接口、TAP/TUN 设备）可能调整头部的位置 |



正常情况下的L2层协议字段的解析流程：

1. 获取mac层头部
2. 获取网络层头部
3. BPF_CORE_READ_INTO宏来获取下网络层协议类型
4. 将网络层协议类型转换为网络字节序，转换后的值表示上层协议类型，如0x0800(IPv4)，0x86DD(IPv6)



异常情况下L2层协议的解析流程：

1. 当协议类型是IPv4或IPv6时，直接通过数据包起始地址data加上网络层偏移量network_header来定位L3层起始位置。
2. 当MAC头部位置大于或等于网络层头部位置时，说明数据包结构异常。这种情况下，程序仍然使用网络层偏移量来定位L3层，并强制将协议类型设置为IPv4。
3. 第三种是兜底处理，如果数据包不属于上述两种情况，则直接返回BPF_OK，表示无需进行后续处理。这种设计确保了程序能够安全地处理各种异常的网络数据包，避免了潜在的解析错误。



## 2、3  网络层数据解析

```
static __always_inline int parse_skb(void* ctx, struct sk_buff *skb, bool sk_not_ready, enum step_t step) {
	......省略N多代码......
	// 4. 检查是否为IP协议
    if (l3_proto != ETH_P_IP && l3_proto != ETH_P_IPV6) {
        return BPF_OK;
    }

    // 5. 解析L4层
    if (!skb_l4_check(trans_header, network_header)) {
        l4 = data + trans_header;
    }
	......省略N多代码......
}
```

检查是否为IP协议，如果既不是IPv4协议，也不是IPv6协议的话，直接返回。

skb_l4_check函数的实现如下：

```
static __always_inline bool skb_l4_check(u16 l4, u16 l3)
{
	return l4 == 0xFFFF || l4 <= l3;
}
```

skb_l4_check函数的参数有两个：

l4:表示传输层（L4）的长度

l3:表示网络层（L3）的长度



当传输层长度等于0xFFFF，或者传输层的长度小于等于网络层的长度时，函数返回true。

__always_inline修饰符表示这个函数会被强制内联，它可以提升程序性能，或者满足ebpf验证器的要求。



开始解析网络层的字段，对应的parse_skb代码如下所示：

```
static __always_inline int parse_skb(void* ctx, struct sk_buff *skb, bool sk_not_ready, enum step_t step) {
	......省略N多代码......
	if (l3_proto == ETH_P_IP) {
        struct iphdr *ipv4 = ip;
        u16 tot_len16 = 0;
		// 获取IP头中的总长度字段（tot_len）
        BPF_CORE_READ_INTO(&tot_len16, ipv4, tot_len);
		// 将IP头中的总长度转换为网络字节序
        u32 len = bpf_ntohs(tot_len16);
        
        u8 ip_hdr_len = 0;
		//计算实际的ip头长度（原始值需要乘以4）
        bpf_probe_read_kernel(&ip_hdr_len, sizeof(((u8 *)ip)[0]), &(((u8 *)ip)[0]));
        ip_hdr_len = get_ip_header_len(ip_hdr_len);
        
		//如果L4指针未设置，则将其设置为IP层结束位置（即传输层起始位置）
        l4 = l4 ? l4 : ip + ip_hdr_len;
		// 获取传输层协议类型
        BPF_CORE_READ_INTO(&proto_l4, ipv4, protocol);
		// 计算传输层数据长度（IP头长度+传输层头长度），即负载数据长度
        tcp_len = len - ip_hdr_len;
    } else { // IPv6
    	
        struct ipv6hdr *ipv6 = ip;
        proto_l4 = _(ipv6->nexthdr);
        tcp_len = _C(ipv6, payload_len);
        l4 = l4 ? l4 : ip + sizeof(*ipv6);
    }
    
    // 7. 处理TCP协议
    if (proto_l4 != IPPROTO_TCP) {
        return BPF_OK;
    }
    ......省略N多代码......
}
```

**IPv4场景：**

- 获取IP头中的总长度字段（tot_len）,并将其转换为网络字节序；
- 计算实际的ip头部长度
- 如果l4指针未设置，则将其设置未IP层结束位置（即传输层起始位置）
- 获取传输层协议类型
- 计算传输层数据长度，即负载数据长度

针对读取ip头部长度字段的代码bpf_probe_read_kernel(&ip_hdr_len, sizeof(((u8 *)ip)[0]), &(((u8 *)ip)[0]));需要好好解读下

**bpf_probe_read_kernel** 是 eBPF 的一个辅助函数，用于安全地从内核空间读取数据。它有三个参数：

第一个参数：目标缓冲区的指针（写入数据的位置），ip_hdr_len的类型为u8

第二个参数：要读取的字节数

第三个参数：源数据的指针（要读取的数据的位置）



**针对具体的实参进行解析：**

&ip_hdr_len：目标位置，将读取的数据存储到 ip_hdr_len 变量中

sizeof(((u8 *)ip)[0])：是读取一个字节的大小

&(((u8 *)ip)[0])：从 IP 头部的第一个字节的地址读取数据



**IPv6场景：**

struct ipv6hdr 是 Linux 内核中定义的 IPv6 头部结构体，包含了版本号、负载长度、下一个头部类型等信息。

_(ipv6->nexthdr) 中的 _() 是一个宏，用于安全地读取内存。这是 eBPF 程序的特殊要求，因为 eBPF 需要确保所有内存访问都是安全的。nexthdr 字段指示了下一个协议头的类型。

_C(ipv6, payload_len) 同样是安全的内存读取，获取 IPv6 数据包的负载长度。

最后一行是在计算传输层协议（如 TCP 或 UDP）头部的起始位置。如果 l4 已经设置则使用现有值，否则将 ip 指针加上 IPv6 头部的大小来定位到传输层头部的开始位置。



当传输层协议不是TCP协议时，则直接返回BPF_OK不做进一步的处理了。



# 三、TCP序列号的初始化和后续处理

## 3、0 函数概览

```
static __always_inline int parse_skb(void* ctx, struct sk_buff *skb, bool sk_not_ready, enum step_t step) {
    ......省略N多代码......
    // 1. 获取初始序列号
    struct sock* sk = {0};
    BPF_CORE_READ_INTO(&sk, skb, sk);
    
    u32 inital_seq = 0;
    struct sock_key key = {0};
    if (sk) {
        struct sock_common sk_cm = {0};
        BPF_CORE_READ_INTO(&sk_cm, sk, __sk_common);
        if (sk_cm.skc_addrpair && !sk_not_ready) {
            parse_sock_key(skb, &key);
            int *found = bpf_map_lookup_elem(&sock_xmit_map, &key);
            if (found == NULL) {
                return 0;
            }
            bpf_probe_read_kernel(&inital_seq, sizeof(inital_seq), found);
        }
    }
    ......省略N多代码......
    // 7. 处理TCP协议
    if (proto_l4 != IPPROTO_TCP) {
        return BPF_OK;
    }

    struct tcphdr *tcp = l4;
    
    // 8. 处理序列号
    if (!inital_seq) {
        if (l3_proto == ETH_P_IP) {
            parse_sock_key_from_ipv4_tcp_hdr(&key, ip, tcp);
        } else {
            parse_sock_key_from_ipv6_tcp_hdr(&key, ip, tcp);
        }

        if (step == DEV_IN || step == DEV_OUT) {
            struct sock_key *translated_flow = bpf_map_lookup_elem(&nat_flow_map, &key);
            if (translated_flow != NULL) {
                key = *translated_flow;
            }
        }

        int *found = bpf_map_lookup_elem(&sock_xmit_map, &key);
        if (found == NULL) {
            if (step == DEV_IN) {
                BPF_CORE_READ_INTO(&inital_seq, tcp, seq);
                inital_seq = bpf_ntohl(inital_seq);
                uint8_t flag = 0;
                bpf_probe_read_kernel(&flag, sizeof(uint8_t), &(((u8 *)tcp)[13]));
                if ((flag & (1 << 1)) == 0) {
                    inital_seq--;
                }
                bpf_map_update_elem(&sock_xmit_map, &key, &inital_seq, BPF_NOEXIST);
            } else {
                return BPF_OK;
            }
        } else {
            bpf_probe_read_kernel(&inital_seq, sizeof(inital_seq), found);
        }
    }
    ......省略N多代码......
}
```

第一次看这个代码，我是懵逼的，还是先从数据结构定义来入手吧。

`sock_xmit_map` 是用于存储 TCP 初始序列号信息的 map。

## 3、1 map定义

```
struct {
	__uint(type, BPF_MAP_TYPE_HASH);
	__uint(key_size, sizeof(struct sock_key));
	__uint(value_size, sizeof(u32));
	__uint(max_entries, 65535);
	__uint(map_flags, 0);
} sock_xmit_map SEC(".maps");
```

map的key类型为struct sock_key结构体类型，代表连接的五元组信息；

map的value类型是u32类型，代表TCP连接的初始序列号。

struct sock_key结构体的定义如下：

```
struct sock_key {
	uint64_t sip[2];//源ip
	uint64_t dip[2];//目的ip
	uint16_t sport;//源端口
	uint16_t dport;//目的端口
};
```



## 3、2 sock_xmit_map的应用场景浅析

### 3、2、1 获取初始序列号

```
u32 inital_seq = 0;                        // 初始化序列号变量
struct sock_key key = {0};                 // 初始化sock_key键值结构体

// 如果套接字存在
if (sk) {                                  
    struct sock_common sk_cm = {0};        // 初始化通用套接字结构体
    BPF_CORE_READ_INTO(&sk_cm, sk, __sk_common);  // 安全地读取套接字通用信息
    
    // 检查套接字是否有地址对，且套接字已准备就绪
    if (sk_cm.skc_addrpair && !sk_not_ready) {
    	// 解析网络包，生成套接字键值
        parse_sock_key(skb, &key);         
        
        // 在 sock_xmit_map 中查找对应的条目
        int *found = bpf_map_lookup_elem(&sock_xmit_map, &key);
        if (found == NULL) {               // 如果没找到，返回0
            return 0;
        }
        // 如果找到了对应条目，则读取对应的初始序列号
        bpf_probe_read_kernel(&inital_seq, sizeof(inital_seq), found);
    }
}
```

看这块代码时，有几个疑问点需要解决：

- struct sock_common结构体是什么作用？如何使用呢？
- struct sock_common是struct sock结构体的一部分，所以必须学下struct sock结构体
- struct sock_common结构体中的skc_addrpair是什么作用？
- sk_not_ready对应哪些阶段？
- parse_sock_key函数的具体实现是什么？



**1、socket连接状态检查**

通过 BPF_CORE_READ_INTO 宏读取套接字的通用信息到 sk_cm变量中



判断socket是否已经初始化

### sock_common 结构体

```
struct sock_common {
	union {
		__addrpair	skc_addrpair;// 地址对是否已初始化
		struct {
			__be32	skc_daddr;
			__be32	skc_rcv_saddr;
		};
	};

	/* skc_dport && skc_num must be grouped as well */
	union {
		__portpair	skc_portpair;// 端口对是否已初始化
		struct {
			__be16	skc_dport;
			__u16	skc_num;
		};
	};
};
```



sk_not_ready在不同阶段的状态

| 阶段     | sk_not_ready | 原因         | 可访问信息   |
| -------- | ------------ | ------------ | ------------ |
| 网卡接收 | 1            | socket未绑定 | 基础包信息   |
| IP层     | 1            | 协议栈处理中 | IP头信息     |
| TCP层    | 0            | socket已就绪 | 完整连接信息 |
| Socket层 | 0            | 连接已建立   | 全部信息     |

TODO：上述每一个阶段，分别对应哪些hook点呢？我从源代码来分析的话，感觉不对啊



网卡接收阶段：

```
SEC("tracepoint/net/netif_receive_skb")
int tracepoint__netif_receive_skb(struct trace_event_raw_net_dev_template  *ctx) {
	parse_skb(ctx, skb, 1, DEV_IN); 
	return BPF_OK;
}
```



IP层

```
SEC("kprobe/ip_rcv_core")
int BPF_KPROBE(ip_rcv_core, struct sk_buff *skb) {
	parse_skb(ctx, skb, 1, IP_IN);
	return BPF_OK;
} 
```



TCP层

```
SEC("kprobe/tcp_v4_do_rcv")
int BPF_KPROBE(tcp_v4_do_rcv, struct sock *sk, struct sk_buff *skb) { 
	parse_skb(ctx, skb, 1, TCP_IN);
	return BPF_OK;
}
```

TODO： tcp_v4_do_rcv这个内核函数的流程，这个阶段，tcp的socket连接已经建立了，为什么	parse_skb函数的第三个参数还要填写1（1代表sk未就绪）

tcp_v4_do_rcv 函数处理的 TCP socket 状态是比较复杂的，不一定都是已连接状态。

调用tcp_v4_do_rcv 函数，TCP socket 状态可能是如下情况：

1、已连接状态

当 socket 处于 ESTABLISHED 状态时，确实是已连接状态，这时tcp_v4_do_rcv 会处理正常的数据传输；



2、未完全连接状态

在三次握手过程中，也会调用 tcp_v4_do_rcv，比如收到 SYN+ACK 或 ACK 包时，此时 socket 可能处于 SYN_SENT、SYN_RECV 等状态

tcp_v4_do_rcv 是 TCP 协议栈中的核心接收函数，它会处理各种 TCP 状态下的数据包，不仅限于已连接状态。



Socket层

SEC("raw_tp/tcp_destroy_sock")

int BPF_PROG(tcp_destroy_sock, struct sock *sk)



parse_sock_key函数的实现

```
static void __always_inline parse_sock_key(struct sk_buff *skb, struct sock_key* key) {
	struct sock* _sk = {0};
	BPF_CORE_READ_INTO(&_sk,skb,sk);
	parse_sock_key_sk(_sk, key);
}
```

parse_sock_key函数内部会调用parse_sock_key_sk(_sk, key);函数。

```
static bool __always_inline parse_sock_key_sk(struct sock* sk, struct sock_key* key) {
	struct sock_common *skc = {0};
	skc = (struct sock_common *)sk;
	bool supportIpv6 = use_ipv6(skc);
	//获取协议族
	switch (_C(skc, skc_family)) {
		case AF_INET: // IPv4
			key->dip[0] = _C(skc, skc_daddr);//源ip
			key->sip[0] = _C(skc, skc_rcv_saddr);//目的ip
			break;
		case AF_INET6:// IPv6
			if (supportIpv6) {
				bpf_probe_read_kernel((void *)(key->dip), sizeof(struct in6_addr), (const void *)__builtin_preserve_access_index(&((typeof((skc)))((skc)))->skc_v6_daddr)); 
				bpf_probe_read_kernel((void *)(key->sip), sizeof(struct in6_addr), (const void *)__builtin_preserve_access_index(&((typeof((skc)))((skc)))->skc_v6_rcv_saddr));
			} else {
				key->dip[0] = _C(skc, skc_daddr);//目的ip
				key->sip[0] = _C(skc, skc_rcv_saddr);//源ip
			}
		break;
		default:
			return false;
	}
	u16 sport = 0;
	u16 dport = 0;
	BPF_CORE_READ_INTO(&sport,sk,__sk_common.skc_num);//源端口: skc_num
	BPF_CORE_READ_INTO(&dport,sk,__sk_common.skc_dport);//目的端口: skc_dport (需要网络字节序转换)
	dport = bpf_ntohs(dport);
	
	//将源端口和目的端口都赋值到key值中
	key->dport = dport;
	key->sport = sport;
	return true;
}
```

3、2、2 处理序列号

```
static __always_inline int parse_skb(void* ctx, struct sk_buff *skb, bool sk_not_ready, enum step_t step) {
    ......省略N多代码......
    struct tcphdr *tcp = l4;
    
    // 8. 处理序列号
    if (!inital_seq) {
        if (l3_proto == ETH_P_IP) {
        	//从 IPv4 和 TCP 头部提取连接信息并填充到 sock_key 结构中
            parse_sock_key_from_ipv4_tcp_hdr(&key, ip, tcp);//ipv4情况
        } else {
            parse_sock_key_from_ipv6_tcp_hdr(&key, ip, tcp);//ipv6情况
        }

		//NAT场景下的流量追踪
		//当数据包在进入设备或离开设备时，程序会检查这个流量是否经过了 NAT 转换
        if (step == DEV_IN || step == DEV_OUT) {
            struct sock_key *translated_flow = bpf_map_lookup_elem(&nat_flow_map, &key);
            if (translated_flow != NULL) {
                key = *translated_flow;
            }
        }

		//处理tcp序号的逻辑，查找指定连接在sock_xmit_map表中是否有记录
        int *found = bpf_map_lookup_elem(&sock_xmit_map, &key);
        if (found == NULL) {
        	//没有记录，且当数据包时进入设备阶段时
            if (step == DEV_IN) {
            	//读取 TCP 头中的序列号（seq），从网络字节序转换为主机字节序
                BPF_CORE_READ_INTO(&inital_seq, tcp, seq);
                inital_seq = bpf_ntohl(inital_seq);
                uint8_t flag = 0;
                //读取 TCP 标志位（第13字节）
                bpf_probe_read_kernel(&flag, sizeof(uint8_t), &(((u8 *)tcp)[13]));
                //检查 SYN 标志位（第一位是syn位）
                if ((flag & (1 << 1)) == 0) {
                	//如果不是syn数据包，则序列号减少1
                    inital_seq--;
                }
                //将初始序列号存入 map 中
                bpf_map_update_elem(&sock_xmit_map, &key, &inital_seq, BPF_NOEXIST);
            } else {
                return BPF_OK;
            }
        } else {
        	//如果找到记录，直接从 map 中读取之前保存的初始序列号
            bpf_probe_read_kernel(&inital_seq, sizeof(inital_seq), found);
        }
    }
    ......省略N多代码......
}
```

parse_sock_key_from_ipv4_tcp_hdr函数如下：

```
static void __always_inline parse_sock_key_from_ipv4_tcp_hdr(struct sock_key *key, struct iphdr *ipv4, 
	struct tcphdr *tcp) {
	u32 saddr = 0;
	u32 daddr = 0;
	//提取源/目的ip地址
	BPF_CORE_READ_INTO(&saddr, ipv4, saddr);
	BPF_CORE_READ_INTO(&daddr, ipv4, daddr);
	u16 sport = 0;
	u16 dport = 0;
	//提取源/目的port端口
	BPF_CORE_READ_INTO(&sport, tcp, source);
	BPF_CORE_READ_INTO(&dport, tcp, dest);
	key->sip[0] = saddr;
	key->dip[0] = daddr;
	key->sport = bpf_ntohs(sport);
	key->dport = bpf_ntohs(dport);
}
```

此函数的功能是从 IPv4 和 TCP 头部提取连接信息并填充到 sock_key 结构中。



# 四、生成事件

```
static __always_inline int parse_skb(void* ctx, struct sk_buff *skb, bool sk_not_ready, enum step_t step) {
    ......省略N多代码......
	// 9. 生成事件
    struct parse_kern_evt_body body = {0};
    body.ctx = ctx;
    body.inital_seq = inital_seq;
    body.key = &key;
    body.tcp = tcp;
    body.len = tcp_len;
    body.step = step;

	//获取数据包所在的网络设备信息
    struct net_device *dev = _C(skb, dev);
    //获取网络接口的唯一标识
    body.ifindex = dev ? _C(dev, ifindex) : _C(skb, skb_iif);
	//当数据包处于网卡接收阶段（NIC_IN）或之后的阶段
    if (step >= NIC_IN) {
    	//反转连接信息，保持一致的流向视角
        reverse_sock_key_no_copy(&key);
    }
    
    //上报内核事件
    report_kern_evt(&body);
    return 1;
    ......省略N多代码......
}
```

结构体parse_kern_evt_body定义：

```
struct parse_kern_evt_body {
	void* ctx;//上下文信息
	u32 inital_seq;//tcp初始序列号
	struct sock_key *key;//连接标识（源IP、目的IP、源端口、目的端口）
	u32 cur_seq;//当前序列号
	u32 len;//长度？这个是什么长度？
	const char *func_name;//函数名称
	enum step_t step;//当前处理阶段
  	struct tcphdr* tcp;//tcp头部信息
  	u32 ifindex;//网口接口的唯一标识
};
```

reverse_sock_key_no_copy函数实现如下：

```
static __always_inline void reverse_sock_key_no_copy(struct sock_key* key) {
	uint64_t temp = key->sip[0];
	key->sip[0] = key->dip[0];
	key->dip[0] = temp;
	temp = key->sip[1];
	key->sip[1] = key->dip[1];
	key->dip[1] = temp;
	temp = key->sport;
	key->sport = key->dport;
	key->dport = temp;
}
```

此函数实现了将网络连接中的源地址（IP和端口）与目标地址（IP和端口）进行交换的功能。

report_kern_evt函数只是个信息上报的函数，此篇文章我们不详细展开讲，等下期文章再讲解。
