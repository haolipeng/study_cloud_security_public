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

**helloworld程序编译方法**

进入到example



**helloworld程序运行方法**

```
root@r630-PowerEdge-R630:/home/work/dpdk-stable-24.11.1/examples/helloworld# ./build/helloworld -c 0x3
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



**helloworld代码剖析**

```
/* 在逻辑核心上启动一个处理函数 */
static int
lcore_hello(__rte_unused void *arg)
{
	unsigned lcore_id;
	lcore_id = rte_lcore_id();
	printf("hello from core %u\n", lcore_id);
	return 0;
}

int main(int argc, char **argv)
{
	int ret;
	unsigned lcore_id;
	
	//eal初始化
	ret = rte_eal_init(argc, argv);
	if (ret < 0)
		rte_panic("Cannot init EAL\n");

	/* Launches the function on each lcore. 8< */
	RTE_LCORE_FOREACH_WORKER(lcore_id) {
		rte_eal_remote_launch(lcore_hello, NULL, lcore_id);
	}

	/* 在main核心上也调用lcore_hello 函数*/
	lcore_hello(NULL);

	rte_eal_mp_wait_lcore();

	/* clean up the EAL */
	rte_eal_cleanup();

	return 0;
}
```

lcore_hello如果通俗点理解的话，就是线程函数就行。



需要大家熟悉下以下的api函数：

rte_eal_init

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



rte_eal_remote_launch

https://doc.dpdk.org/api/rte__launch_8h.html#a2bf98eda211728b3dc69aa7694758c6d

## rte_eal_mp_remote_launch和rte_eal_remote_launch的区别



rte_eal_mp_wait_lcore（和rte_eal_remote_launch函数的关联是什么）

rte_eal_cleanup



## 1、5 如何编写自己的dpdk程序

解读dpdk的样例程序的Makefile文件和内容。



# 二、内存管理

**1、Lcore变量**

**2、Memory Pool内存池库**

**3、Mbuf库**

**4、多进程支持**



# 三、CPU Packet Processing数据包处理

**1、Toeplitz Hash库**

https://doc.dpdk.org/guides/prog_guide/toeplitz_hash_lib.html



**2、Hash库**

https://doc.dpdk.org/guides/prog_guide/hash_lib.html



**3、IP分片和重组库**

https://doc.dpdk.org/guides/prog_guide/ip_fragment_reassembly_lib.html



**4、Packet Classification And Access Control(ACL)库**

https://doc.dpdk.org/guides/prog_guide/packet_classif_access_ctrl.html



**5、Packet 转发库**

https://doc.dpdk.org/guides/prog_guide/packet_distrib_lib.html



**6、Reorder库**

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