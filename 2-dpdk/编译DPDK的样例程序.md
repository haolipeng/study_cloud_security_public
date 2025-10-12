# 一、使用Make构建样例程序

进入dpdk example目录，然后执行如下命令编译dpdk的样例程序，

```
make
```

如果要构建可调式的程序，请使用DEBUG选项。

```
make DEBUG=1
```

分析dpdk的helloworld样子程序的Makefile脚本



当在系统中安装dpdk库后，dpdk会提供一个名为libdpdk.pc的pkg-config文件，供应用程序在构建过程中进行查询。

建议使用pkg-config文件，而不是将dpdk参数(cflags/ldflags)硬编码到应用程序的构建过程中。



下面是一个简化版的示例，其中目标二进制名称存储在变量$(APP)中，用于构建的源文件则存储在$(SRCS-y)变量中。

```
# 应用程序名称
APP = helloworld

# 定义源文件列表，SRCS-y变量中包含所有需要编译的C类型源文件
# all source are stored in SRCS-y
SRCS-y := main.c

# ？表示如果变量未定义才赋值
PKGCONF ?= pkg-config

# 检查系统是否安装了dpdk，未安装则停止编译
ifneq ($(shell $(PKGCONF) --exists libdpdk && echo 0),0)
$(error "no installation of DPDK found")
endif

# 默认构建共享库版本
# shared是构建共享库版本，并创建符号链接
# static是构建静态库版本，并创建符号链接
all: shared
.PHONY: shared static
shared: build/$(APP)-shared
	ln -sf $(APP)-shared build/$(APP)
static: build/$(APP)-static
	ln -sf $(APP)-static build/$(APP)

# 获取DPDK的pkg-config文件路径
PC_FILE := $(shell $(PKGCONF) --path libdpdk 2>/dev/null)
# 获取DPDK的编译标志
CFLAGS += -O3 $(shell $(PKGCONF) --cflags libdpdk)
# 获取共享库链接标志
LDFLAGS_SHARED = $(shell $(PKGCONF) --libs libdpdk)
# 获取静态库链接标志
LDFLAGS_STATIC = $(shell $(PKGCONF) --static --libs libdpdk)

# 静态链接的特殊检查，可忽略
ifeq ($(MAKECMDGOALS),static)
# check for broken pkg-config
ifeq ($(shell echo $(LDFLAGS_STATIC) | grep 'whole-archive.*l:lib.*no-whole-archive'),)
$(warning "pkg-config output list does not contain drivers between 'whole-archive'/'no-whole-archive' flags.")
$(error "Cannot generate statically-linked binaries with this version of pkg-config")
endif
endif

# 添加宏定义，允许使用DPDK的实验性API函数
CFLAGS += -DALLOW_EXPERIMENTAL_API

# 构建共享库版本
build/$(APP)-shared: $(SRCS-y) Makefile $(PC_FILE) | build
	$(CC) $(CFLAGS) $(SRCS-y) -o $@ $(LDFLAGS) $(LDFLAGS_SHARED)

# 构建静态库版本
build/$(APP)-static: $(SRCS-y) Makefile $(PC_FILE) | build
	$(CC) $(CFLAGS) $(SRCS-y) -o $@ $(LDFLAGS) $(LDFLAGS_STATIC)

# 创建build目录，-p表示父目录不存在也创建
build:
	@mkdir -p $@

#清理目标，删除所有生成的中间产物
.PHONY: clean
clean:
	rm -f build/$(APP) build/$(APP)-static build/$(APP)-shared
	test -d build && rmdir -p build || true

```

LDFLAGS和CFLAGS的内容是如何获取的？

答：都是采用pkg-config命令从libdpdk.pc文件中获取的。



# 二、使用meson构建样例程序

进入dpdk的build目录(比如为dpdkbuild)

```
cd dpdkbuild/
```



开启样例程序编译

```
meson configure -Dexamples=all
```



编译构建

```
ninja
```



# 三、运行helloworld程序

我把服务器上的大也内存关闭了，helloworld程序启动后显示如下：

```
root@r630-PowerEdge-R630:/home/work/dpdk-stable-24.11.1/examples/helloworld# ./build/helloworld -c 3 -n 2
EAL: Detected CPU lcores: 72
EAL: Detected NUMA nodes: 2
EAL: Detected shared linkage of DPDK
EAL: Multi-process socket /var/run/dpdk/rte/mp_socket
EAL: Selected IOVA mode 'VA'
EAL: No free 1048576 kB hugepages reported on node 0
EAL: No free 1048576 kB hugepages reported on node 1
EAL: Cannot get hugepage information.
EAL: PANIC in main():
Cannot init EAL
0: /home/work/dpdk-stable-24.11.1/dpdklib/lib/x86_64-linux-gnu/librte_eal.so.25 (rte_dump_stack+0x42) [718b22d1a502]
1: /home/work/dpdk-stable-24.11.1/dpdklib/lib/x86_64-linux-gnu/librte_eal.so.25 (__rte_panic+0xd0) [718b22cf33f7]
2: ./build/helloworld (571f97d3a000+0x115c) [571f97d3b15c]
3: /lib/x86_64-linux-gnu/libc.so.6 (718b22a00000+0x2a1ca) [718b22a2a1ca]
4: /lib/x86_64-linux-gnu/libc.so.6 (__libc_start_main+0x8b) [718b22a2a28b]
5: ./build/helloworld (571f97d3a000+0x1235) [571f97d3b235]
已中止 (核心已转储)
```

其报错信息也很明显，没有可用的大页内存使用，所以DPDK库初始化EAL失败，导致程序退出。

```
EAL: No free 1048576 kB hugepages reported on node 0
EAL: No free 1048576 kB hugepages reported on node 1
EAL: Cannot get hugepage information.
EAL: PANIC in main():
Cannot init EAL
```



**如果你的服务器的确不支持大页内存的话，那么可以添加下--no-huge命令，让程序先启动起来。**

```
root@r630-PowerEdge-R630:/home/work/dpdk-stable-24.11.1/examples/helloworld# ./build/helloworld -c 3 -n 2 --no-huge
EAL: Detected CPU lcores: 72
EAL: Detected NUMA nodes: 2
EAL: Static memory layout is selected, amount of reserved memory can be adjusted with -m or --socket-mem
EAL: Detected shared linkage of DPDK
EAL: Multi-process socket /var/run/dpdk/rte/mp_socket
EAL: Selected IOVA mode 'VA'
EAL: VFIO support initialized
EAL: Using IOMMU type 1 (Type 1)
EAL: lib.eal log level changed from info to debug
worker lcore id:1
hello from core 1
hello from core 0
```

从图上可以看到，程序能够正常启动了。
