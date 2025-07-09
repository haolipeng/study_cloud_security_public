# 一、port_list端口完整流程分析

## 1、1 配置文件解析阶段

**步骤1：配置文件初始化**

```
void netif_cfgfile_init(void)
{
    INIT_LIST_HEAD(&port_list);
    INIT_LIST_HEAD(&worker_list);
}
```

**步骤2：设备配置处理**

```
static void device_handler(vector_t tokens)
{
    assert(VECTOR_SIZE(tokens) >= 1);

    char *str;
    //申请结构体内存，并赋值
    struct port_conf_stream *port_cfg =
        rte_zmalloc(NULL, sizeof(struct port_conf_stream), RTE_CACHE_LINE_SIZE);

    port_cfg->port_id = -1;
    str = VECTOR_SLOT(tokens, 1);
    RTE_LOG(INFO, NETIF, "netif device config: %s\n", str);
    strncpy(port_cfg->name, str, sizeof(port_cfg->name));
    port_cfg->rx_queue_nb = -1;
    port_cfg->tx_queue_nb = -1;

	//添加到此节点到port_list中
    list_add(&port_cfg->port_list_node, &port_list);
    RTE_LOG(INFO, NETIF, "Added device %s to port_list, total devices: %d\n", 
            port_cfg->name, list_elems(&port_list));
}
```

**步骤3：各种配置参数处理**

- rx_queue_number_handler() - 处理接收队列数量

- tx_queue_number_handler() - 处理发送队列数量

- rss_handler() - 处理RSS配置

- promisc_mode_handler() - 处理混杂模式

- 等等...

所有这些处理函数都通过 list_entry(port_list.next, struct port_conf_stream, port_list_node) 获取当前配置的端口。



总结：

**配置文件解析：**通过device_handler函数解析配置文件中的端口配置

**配置存储：**将解析的配置信息存储在port_conf_stream结构体中

**链表管理：**通过port_list_node将配置项链接到port_list链表中

## 1、2 端口初始化阶段

**步骤1：端口名称分配**

```
int port_name_alloc(portid_t pid, char *pname, size_t buflen)
{
    assert(pname && buflen > 0);
    memset(pname, 0, buflen);
    if (is_physical_port(pid)) {
        struct port_conf_stream *current_cfg;
        list_for_each_entry_reverse(current_cfg, &port_list, port_list_node) {
            if (current_cfg->port_id < 0) {
                current_cfg->port_id = pid;
                if (current_cfg->name[0])
                    snprintf(pname, buflen, "%s", current_cfg->name);
                else
                    snprintf(pname, buflen, "dpdk%d", pid);
                return ENDF_OK;
            }
        }
        RTE_LOG(ERR, NETIF, "%s: not enough ports configured in setting.conf\n", __func__);
        return ENDF_NOTEXIST;
    }

    return ENDF_INVAL;
}
```



**步骤2：端口初始化**

```
static void netif_port_init(void)
{
    // 检查配置的端口数量是否匹配
    nports_cfg = list_elems(&port_list) + list_elems(&bond_list);
    
    for (pid = 0; pid < nports; pid++) {
        // 分配端口名称
        port_name_alloc(pid, ifname, sizeof(ifname));
        
        // 创建端口结构
        port = netif_alloc(pid, sizeof(union netif_bond), ifname, 0, 0, dpdk_port_setup);
        
        // 注册端口
        netif_port_register(port);
    }
}
```

## 1、3 配置到运行时的映射关系

```
配置文件解析 → port_list → 端口初始化 → port_tab/port_ntab
```

数据结构

```
static struct list_head port_tab[NETIF_PORT_TABLE_BUCKETS]; /* hashed by id */
static struct list_head port_ntab[NETIF_PORT_TABLE_BUCKETS]; /* hashed by name */
```

哈希表初始化

```
static inline void port_tab_init(void)
{
    int i;
    for (i = 0; i < NETIF_PORT_TABLE_BUCKETS; i++)
        INIT_LIST_HEAD(&port_tab[i]);
}

static inline void port_ntab_init(void)
{
    int i;
    for (i = 0; i < NETIF_PORT_TABLE_BUCKETS; i++)
        INIT_LIST_HEAD(&port_ntab[i]);
}
```

映射转换过程在netif_port_register函数进行

```
int netif_port_register(struct netif_port *port)
{
    // 1. 计算哈希值
    hash = port_tab_hashkey(port->id);
    nhash = port_ntab_hashkey(port->name, sizeof(port->name));
    
    // 2. 检查重复性
    list_for_each_entry(cur, &port_tab[hash], list) {
        if (cur->id == port->id || strcmp(cur->name, port->name) == 0) {
            return EDPVS_EXIST;
        }
    }
    
    // 3. 添加到哈希表
    list_add_tail(&port->list, &port_tab[hash]);      // 按 ID 索引
    list_add_tail(&port->nlist, &port_ntab[nhash]);   // 按名称索引
    
    g_nports++;
    
    return EDPVS_OK;
}
```



