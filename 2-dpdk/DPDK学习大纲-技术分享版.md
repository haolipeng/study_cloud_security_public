学习计划安排

| 内容      | 时间       | 是否完成   |
| --------- | ---------- | ---------- |
| Hash库    | 2025-07-10 | **已完成** |
| Reorder库 | 2025-07-14 | **未完成** |
|           |            |            |

推荐大家花3000块钱买个二手的服务器（intel还是arm），网卡支持dpdk就可以，如果有条件，我推荐买x710，性价比高，而且稳定bug少。



# 一、环境搭建和初体验

## 1、1 dpdk源代码的编译

选用的dpdk版本是24.11.1的版本，这个是stable版本。



需要安装一些依赖库，具体要安装的库在这里。

https://doc.dpdk.org/guides/linux_gsg/sys_reqs.html#compilation-of-the-dpdk

编译dpdk库，需要安装如下基础库

- build-essential (包含gcc、make等基本编译工具)

- meson (DPDK使用meson构建系统)

- ninja-build (meson使用的构建工具)
- pyelftools库
- pkg-config (管理库的编译和链接标志)




以ubuntu 22.04版本的系统为例：

```
apt install build-essential
```



如果meson的版本低于最低版本，则使用pip3进行安装

```
pip3 install meson ninja
```



安装pyelftools库

```
apt install python3-pyelftools
```



使用 meson-ninja 构建 DPDK，并为 DPDK 应用程序（eta加密流量分析）导出环境变量 PKG_CONFIG_PATH

```
#下载dpdk源代码后进入源码目录
cd dpdk-24.11

mkdir dpdklib                 # user desired install folder
mkdir dpdkbuild               # user desired build folder

meson -Denable_kmods=true -Dprefix=/home/dpdk_course/dpdk-stable-24.11.1/dpdklib dpdkbuild
注意：-Dprefix要求使用绝对路径，/home/dpdk_course/dpdk-stable-24.11.1/dpdklib是我的dpdklib的绝对路径，

#使用ninja编译
ninja -C dpdkbuild
cd dpdkbuild; 
ninja install

#导出环境变量
在/etc/profile文件中添加如下内容：
export PKG_CONFIG_PATH=/home/dpdk_course/dpdk-stable-24.11.1/dpdklib/lib/x86_64-linux-gnu/pkgconfig/:$PKG_CONFIG_PATH

export LD_LIBRARY_PATH=/home/work/dpdk-stable-24.11.1/dpdklib/lib/x86_64-linux-gnu:$LD_LIBRARY_PATH

#使配置生效
source /etc/profile
```

解释下meson命令行参数

**-Denable_kmods=true：**指示Meson启用DPDK内核模块的构建

**-Dprefix=/home/dpdk_course/dpdk-stable-24.11.1/dpdklib：** 指定DPDK库和头文件的安装路径，在执行`ninja install`时，DPDK的库和头文件将被安装到`/home/dpdk_course/dpdk-stable-24.11.1/dpdklib`目录下

**dpdkbuild:**将在执行命令的当前目录创建这个目录，Meson将所有编译产物都放在这个目录中，以实现源代码树和构建树的分离。



**重要声明：真实环境中绑定的网卡的pci号为0000:01:00.0**



## 1、2 为网卡加载dpdk驱动

**加载vfio驱动（首选）**

modprobe vfio-pci



然后再执行dpdk-devbind.py脚本
./dpdk-devbind.py --bind=vfio-pci 0000:01:00.0



此时再次执行./dpdk-devbind.py --status命令，输出内容如下:

```
Network devices using DPDK-compatible driver （绑定到dpdk网卡的）
============================================
0000:01:00.0 'Ethernet Controller 10-Gigabit X540-AT2 1528' numa_node=0 drv=vfio-pci unused=ixgbe

Network devices using kernel driver (此处是绑定内核驱动的网卡)
===================================
0000:01:00.1 'Ethernet Controller 10-Gigabit X540-AT2 1528' numa_node=0 if=eno2 drv=ixgbe unused=vfio-pci 
0000:07:00.0 'I350 Gigabit Network Connection 1521' numa_node=0 if=eno3 drv=igb unused=vfio-pci *Active*
0000:07:00.1 'I350 Gigabit Network Connection 1521' numa_node=0 if=eno4 drv=igb unused=vfio-pci
```



成功加载后，如果需要卸载网口

```
./dpdk-devbind.py -u 0000:01:00.0
```

然后再将网卡绑定到之前的驱动上

```
./dpdk-devbind.py -b ixgbe 0000:01:00.0
```



### 3）不同平台架构开启IOMMU(intel为例)

Intel平台开启IOMMU配置

先检测是否为intel平台架构

```
lscpu | grep "Vendor ID"
BIOS Vendor ID:                       Intel
```



检测系统是否支持IOMMU的通用方法

```
# 检查IOMMU组（最可靠的方法）
ls /sys/kernel/iommu_groups/
```

如果结果中存在多个数字命名的文件夹，则代表支持iommu



## 1、3 配置大页内存

### 1、3、1 临时设置大页内存

echo 4 > /sys/devices/system/node/node0/hugepages/hugepages-1048576kB/nr_hugepages



设置大页内存后，显示结果如下

```
cat /proc/meminfo | grep Huge
AnonHugePages:         0 kB
ShmemHugePages:        0 kB
FileHugePages:         0 kB
HugePages_Total:       3
HugePages_Free:        3
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:    1048576 kB
Hugetlb:         3145728 kB
root@haolipeng-ThinkPad-T450:/home/work/dpdk-stable-24.11.1/usertools#
```



如果系统不支持1G大小的大页内存，可以先使用2M大小的大页内存。

```
root@r630-PowerEdge-R630:~# cat /proc/meminfo | grep Huge
AnonHugePages:         0 kB
ShmemHugePages:        0 kB
FileHugePages:         0 kB
HugePages_Total:    4096
HugePages_Free:     4096
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlb:        18874368 kB
```



### 1、3、2 永久设置大页内存，开机启动

大页内存也可以在开机自启动时，永久的设置到系统中。

编辑GRUB配置文件：

```
vim /etc/default/grub
```



修改`GRUB_CMDLINE_LINUX_DEFAULT`行，添加大页内存参数

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash hugepagesz=1G hugepages=4"
```



更新GRUB配置

```
update-grub
```



验证大页内存的配置

```
cat /proc/meminfo | grep Huge
```



## 1、4 helloworld程序的流程解读

### 1、4、1 **程序编译方法**

进入到example

比如我的dpdk库的example目录为/home/work/dpdk-stable-24.11.1/examples/helloworld

```
cd /home/work/dpdk-stable-24.11.1/examples/helloworld
```

执行make编译，编译源代码

```
root@r630-PowerEdge-R630:/home/work/dpdk-stable-24.11.1/examples/helloworld# make
cc -O3 -I/home/work/dpdk-stable-24.11.1/dpdklib/include -include rte_config.h -march=native -mrtm  -DALLOW_EXPERIMENTAL_API main.c -o build/helloworld-shared  -Wl,--as-needed -L/home/work/dpdk-stable-24.11.1/dpdklib/lib/x86_64-linux-gnu -lrte_node -lrte_graph -lrte_pipeline -lrte_table -lrte_pdump -lrte_port -lrte_fib -lrte_pdcp -lrte_ipsec -lrte_vhost -lrte_stack -lrte_security -lrte_sched -lrte_reorder -lrte_rib -lrte_mldev -lrte_regexdev -lrte_rawdev -lrte_power -lrte_pcapng -lrte_member -lrte_lpm -lrte_latencystats -lrte_jobstats -lrte_ip_frag -lrte_gso -lrte_gro -lrte_gpudev -lrte_dispatcher -lrte_eventdev -lrte_efd -lrte_dmadev -lrte_distributor -lrte_cryptodev -lrte_compressdev -lrte_cfgfile -lrte_bpf -lrte_bitratestats -lrte_bbdev -lrte_acl -lrte_timer -lrte_hash -lrte_metrics -lrte_cmdline -lrte_pci -lrte_ethdev -lrte_meter -lrte_net -lrte_mbuf -lrte_mempool -lrte_rcu -lrte_ring -lrte_eal -lrte_telemetry -lrte_argparse -lrte_kvargs -lrte_log 
ln -sf helloworld-shared build/helloworld
root@r630-PowerEdge-R630:/home/work/dpdk-stable-24.11.1/examples/helloworld#
```

可执行程序生成在build文件夹下。

### 1、4、2 程序运行方法

程序支持很多参数，但想最简单的运行，只需要很少的命令行参数。

```
root@r630-PowerEdge-R630:/home/work/dpdk-stable-24.11.1/examples/helloworld# ./build/helloworld -c 0x3 -n 4
EAL: Detected CPU lcores: 72
EAL: Detected NUMA nodes: 2
EAL: Detected shared linkage of DPDK
EAL: Multi-process socket /var/run/dpdk/rte/mp_socket
EAL: Selected IOVA mode 'VA'
EAL: VFIO support initialized
EAL: Using IOMMU type 1 (Type 1)
hello from core 1   
hello from core 0
```

其中

```
hello from core 1   
hello from core 0
```

为程序的输出。

### 1、4、3 helloworld代码剖析

```
/* 在逻辑核心上启动一个处理函数 */
static int
lcore_hello(__rte_unused void *arg)
{
	unsigned lcore_id;
	lcore_id = rte_lcore_id();//获取当前函数所使用的lcore逻辑核心
	printf("hello from core %u\n", lcore_id);
	return 0;
}

int main(int argc, char **argv)
{
	int ret;
	unsigned lcore_id;
	
	//EAL初始化，简单理解就是环境初始化就行，刚开始学技术不用想太多，先使用起来，例子运行起来。
	ret = rte_eal_init(argc, argv);
	if (ret < 0)
		rte_panic("Cannot init EAL\n");

	//遍历每个worker逻辑核心，并在worker核心上运行lcore_hello函数
	RTE_LCORE_FOREACH_WORKER(lcore_id) {
		rte_eal_remote_launch(lcore_hello, NULL, lcore_id);
	}

	/* 在main核心上也调用lcore_hello 函数*/
	lcore_hello(NULL);

	rte_eal_mp_wait_lcore();

	/* 清理EAL资源，对应上面的EAL初始化 */
	rte_eal_cleanup();

	return 0;
}
```

lcore_hello如果通俗点理解的话，就是线程函数就行。

需要大家熟悉下以下的api函数：

**1、RTE_LCORE_FOREACH_WORKER宏定义**

RTE_LCORE_FOREACH_WORKER宏定义如下

```
/**
 * Macro to browse all running lcores except the main lcore.
 */
#define RTE_LCORE_FOREACH_WORKER(i)					\
	for (i = rte_get_next_lcore(-1, 1, 0);				\
	     i < RTE_MAX_LCORE;						\
	     i = rte_get_next_lcore(i, 1, 0))
```



**2、rte_eal_remote_launch**

函数原型

```
int rte_eal_remote_launch(lcore_function_t * f,void * arg, unsigned worker_id)
```

参数f：要被调用的函数

参数arg：函数的参数

参数worker_id：f指定的函数将要在哪个cpu核心上执行，worker_id来进行标识

返回值：

- 0: 成功. 函数 f 的执行已在远程 lcore 上开始
- (-EBUSY): 远程核心未处于WAIT状态
- (-EPIPE): 读取或写入管道到工作线程时出错

描述：

rte_eal_remote_launch函数只能在main核心上被调用。

发送一个消息到worker_id指定的处于WAIT状态的cpu核心，

当worker_id指定的cpu核心接收到消息后，它会切换到RUNNING 状态，然后调用注册的回调函数f。

一旦函数执行完成，worker_id指定的cpu核心又重新切换回WAIT状态，并且 f 的返回值存储在本地变量中，以便使用 rte_eal_wait_lcore() 来读取。

https://doc.dpdk.org/api/rte__launch_8h.html#a2bf98eda211728b3dc69aa7694758c6d



**3、rte_eal_mp_wait_lcore**

等待所有 lcore 完成其工作。

rte_eal_mp_wait_lcore函数只能在main核心上被调用。

调用 rte_eal_mp_wait_lcore() 后，调用者可以假定所有工作 lcore 都处于 WAIT 状态。



温馨提示：虽然现在有AI辅助编程，但是程序员还是要锻炼自身阅读文档的能力，因为官网文档更新速度快，解释很清晰，能够让你掌握第一手的信息。



### 1、4、4 再深入一点点

**启动线程时，调用rte_eal_remote_launch**

发送一个消息到worker_id指定的处于WAIT状态的cpu核心，

当worker_id指定的cpu核心接收到消息后，它会切换到RUNNING 状态，然后调用注册的回调函数f。

一旦函数执行完成，worker_id指定的cpu核心又重新切换回WAIT状态，并且 f 的返回值存储在本地变量中，以便使用。



**等待线程结束时，调用rte_eal_mp_wait_lcore**

等待所有 lcore 完成其工作。



不知道小伙伴们对这句话的表述是否犯迷糊？相信我，我第一个学习时，也犯迷糊，很正常。

让我们深入源代码探究下这块原理，源码面前，没有秘密。

#### 1. lcore函数启动流程

```
// 在 main.c 中调用
rte_eal_remote_launch(lcore_hello, NULL, lcore_id);

// 在 eal_common_launch.c 中
int rte_eal_remote_launch(lcore_function_t *f, void *arg, unsigned int worker_id)
{
    // 1. 检查 worker 是否在 WAIT 状态
    if (rte_atomic_load_explicit(&lcore_config[worker_id].state, 
            rte_memory_order_acquire) != WAIT)
        goto finish;

    // 2. 设置函数参数
    lcore_config[worker_id].arg = arg;
    
    // 3. 设置要执行的函数指针
    rte_atomic_store_explicit(&lcore_config[worker_id].f, f, rte_memory_order_release);

    // 4. 唤醒 worker 线程
    rc = eal_thread_wake_worker(worker_id);
}
```

至于是如何唤醒worker线程，我们马上讲。

#### 2. Worker 线程唤醒机制

```
// 在 eal_unix_thread.c 中
int eal_thread_wake_worker(unsigned int worker_id)
{
    // 通过管道发送唤醒信号
    int m2w = lcore_config[worker_id].pipe_main2worker[1];
    write(m2w, &c, 1);  // 发送一个字节唤醒 worker
}
```

#### 3、Worker 线程主循环

```
uint32_t eal_thread_loop(void *arg)
{
    unsigned int lcore_id = (uintptr_t)arg;
    
    // 初始化线程
    __rte_thread_init(lcore_id, &lcore_config[lcore_id].cpuset);
    
    // 主循环
    while (1) {
        lcore_function_t *f;
        void *fct_arg;

        // 1. 等待主线程的命令
        eal_thread_wait_command();

        // 2. 设置状态为 RUNNING
        rte_atomic_store_explicit(&lcore_config[lcore_id].state, RUNNING,
            rte_memory_order_release);

        // 3. 确认命令已收到
        eal_thread_ack_command();

        // 4. 等待函数指针被设置
        while ((f = rte_atomic_load_explicit(&lcore_config[lcore_id].f, rte_memory_order_acquire)) == NULL)
        	//函数指针为空，则等待
            rte_pause();

        // 5. 执行函数！这里是关键
        fct_arg = lcore_config[lcore_id].arg;
        ret = f(fct_arg);  // <-- 你注册的回调函数f在这里执行！
        
        // 6. 保存返回值并清理
        lcore_config[lcore_id].ret = ret;
        lcore_config[lcore_id].f = NULL;
        lcore_config[lcore_id].arg = NULL;

        // 7. 设置状态为 WAIT，等待下一个任务
        rte_atomic_store_explicit(&lcore_config[lcore_id].state, WAIT,
            rte_memory_order_release);
    }
}
```

你的 lcore 函数（如 lcore_hello）是在每个 worker 线程的 eal_thread_loop 函数中执行的。每个 worker 线程都在一个无限循环中等待主线程分配任务，当收到任务时，就会调用你指定的函数。



总结如下图所示：

![image-20250716102831065](https://gitee.com/codergeek/picgo-image/raw/master/image/202507161028779.png)

master和worker之间的通信机制如下图所示：

![image-20250716105742321](https://gitee.com/codergeek/picgo-image/raw/master/image/202507161057428.png)



## 1、5 如何编写自己的dpdk程序

解读dpdk的样例程序的Makefile文件和内容。



# 二、内存管理

**1、Lcore变量**

**2、Memory Pool内存池库**

**3、Mbuf库**

**4、多进程支持**



# 三、CPU Packet Processing数据包处理

## **3、1 Toeplitz Hash库**

https://doc.dpdk.org/guides/prog_guide/toeplitz_hash_lib.html



## **3、2 Hash库**

Use Case: Flow Classification

通过阅读官网文档，我推荐新手阶段专注在使用场景上，也就是最简单的api使用即可。

然后后面慢慢渗入，比如使用更精准高性能的api，或者理解透彻其中的一些原理。

https://doc.dpdk.org/guides/prog_guide/hash_lib.html



**创建哈希表：rte_hash_create**

**哈希表操作(添加元素)：rte_hash_add_key**

**哈希表操作(查找元素)：rte_hash_lookup**

**哈希表操作(查找元素)：rte_hash_del_key**

**获取哈希表中的元素个数：rte_hash_count**

**遍历哈希表中元素项：rte_hash_iterate**

**清空哈希表中元素项：rte_hash_reset**



## **3、3 IP分片和重组库**

https://doc.dpdk.org/guides/prog_guide/ip_fragment_reassembly_lib.html



## 3、**4 数据报分类和ACL库**

https://doc.dpdk.org/guides/prog_guide/packet_classif_access_ctrl.html



## **3、5 Packet 转发库**

https://doc.dpdk.org/guides/prog_guide/packet_distrib_lib.html



## 3、**6 Reorder库**

https://doc.dpdk.org/guides/prog_guide/reorder_lib.html



# 四、高级库

**1、Packet Framework Library**

https://doc.dpdk.org/guides/prog_guide/packet_framework.html



**2、Graph Library and Inbuilt Nodes**

https://doc.dpdk.org/guides/prog_guide/graph_lib.html

其设计思想是参考了vpp数据包处理框架的。



# 五、组件库

**1、参数解析库**

**2、命令行库**

**3、指针压缩库**

**4、定时器库**

**5、RCU库**

**6、Ring库**

**7、Stack库**

**8、日志库**

**9、Metrics库**

**10、Telemetry Library**

**11、Packet Capture Library**

**12、Packet Capture Next Generation Library**

**13、BPF库**

**14、Trace库**