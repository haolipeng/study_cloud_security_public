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



# 二、程序启动命令行参数

## 1、DPDK EAL命令行参数(-l 0-3 -n 4)

作用：告诉dpdk在哪些物理CPU核心上运行

范围：指定系统物理CPU核心的范围



## 2、配置文件的worker_defs

作用：定义程序内部工作线程的逻辑ID和功能分配

范围：程序内部的工作线程编号和队列分配

示例：cpu_id 0表示NetDedender的第0个工作线程

**重要说明：**cpu_id只是NetDedender内部的工作线程标识符，**不直接对应物理CPU核心**。



简单映射关系

```
# 命令行：使用系统 CPU 0-3
./dpvs -- -l 0-3

# 配置文件：DPVS 工作线程 0-3
worker_defs {
    worker cpu0 { cpu_id 0 }  # 映射到系统 CPU 0
    worker cpu1 { cpu_id 1 }  # 映射到系统 CPU 1
    worker cpu2 { cpu_id 2 }  # 映射到系统 CPU 2
    worker cpu3 { cpu_id 3 }  # 映射到系统 CPU 3
}
```



复杂映射关系

```
# 命令行：指定映射关系
./dpvs -- --lcores 0@1,1@2,2@3,3@4

# 配置文件：DPVS 工作线程配置
worker_defs {
    worker cpu0 { cpu_id 0 }  # 映射到系统 CPU 1
    worker cpu1 { cpu_id 1 }  # 映射到系统 CPU 2
    worker cpu2 { cpu_id 2 }  # 映射到系统 CPU 3
    worker cpu3 { cpu_id 3 }  # 映射到系统 CPU 4
}
```



### 为什么需要双重配置？

#### 1. 职责分离

- DPDK EAL 参数：负责底层 CPU 资源分配和隔离

- DPVS 配置文件：负责应用层工作线程逻辑分配

#### 2. 灵活性

- 可以在不修改配置文件的情况下，通过命令行参数调整 CPU 使用

- 支持复杂的 CPU 映射关系（如跳过某些 CPU 核心）

#### 3. 性能优化

- 可以避免使用系统繁忙的 CPU 核心（如 CPU 0）

- 支持 NUMA 感知的 CPU 分配
