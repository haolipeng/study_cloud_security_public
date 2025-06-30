写作风格：简洁简单，通俗易懂，适合新手无压力阅读

# 一、什么是uprobe？urpobe能干什么？

kprobes机制在事件的基础上为内核态提供了追踪的功能，而uprobes则为用户态提供了追踪调试的功能。



# 二、libbpf示例程序

**概述**

例子中的程序将uprobe和uretprobe的BPF程序绑定到它自己的函数，并使用bpf_printk宏记录函数的输入参数和返回值。用户空间函数每秒触发一次：



## 2、1 内核态程序剖析剖析

内核态代码：

```
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>

char LICENSE[] SEC("license") = "Dual BSD/GPL";

SEC("uprobe")
int BPF_KPROBE(uprobe_add, int a, int b)
{
	bpf_printk("uprobed_add ENTRY: a = %d, b = %d", a, b);
	return 0;
}

SEC("uretprobe")
int BPF_KRETPROBE(uretprobe_add, int ret)
{
	bpf_printk("uprobed_add EXIT: return = %d", ret);
	return 0;
}

SEC("uprobe//proc/self/exe:uprobed_sub")
int BPF_KPROBE(uprobe_sub, int a, int b)
{
	bpf_printk("uprobed_sub ENTRY: a = %d, b = %d", a, b);
	return 0;
}

SEC("uretprobe//proc/self/exe:uprobed_sub")
int BPF_KRETPROBE(uretprobe_sub, int ret)
{
	bpf_printk("uprobed_sub EXIT: return = %d", ret);
	return 0;
}
```

在这个示例中，用户态进程自己探测自己（这个例子举的不太好，实际情况是探测其他进程），进程自己探测自己，所以可以使用/proc/self/exe，后者这个文件是指向访问这个文件的进程本身的符号链接。



```c
SEC("uretprobe")
```

这种没指定目标函数。由用户态bpf程序加载和动态绑定到探测目标函数。

上层用户态bpf程序需要自行计算函数在ELF文件中的偏移量。



```c
SEC("uprobe//proc/self/exe:uprobed_sub")
```

这种指定了目标elf路径和函数。可由用户态libbpf自动加载和绑定到探测目标函数。

上层用户态bpf程序不需要计算函数在ELF文件中的偏移量offset。由libbpf自动计算。





两种方式的区别是什么呢？

一个指明了具体的路径，一个是需要自己来计算偏移量的。



## 2、2 用户态ebpf程序剖析

```
#include <errno.h>
#include <stdio.h>
#include <unistd.h>
#include <sys/resource.h>
#include <bpf/libbpf.h>
#include "uprobe.skel.h"

static int libbpf_print_fn(enum libbpf_print_level level, const char *format, va_list args)
{
	return vfprintf(stderr, format, args);
}

//它是一个全局函数，用于确保编译器不会将其内联。为了更加确保安全，我们还使用了“asm volatile”和 noinline 属性来阻止编译器进行局部内联。
__attribute__((noinline)) int uprobed_add(int a, int b)
{
	asm volatile ("");
	return a + b;
}

__attribute__((noinline)) int uprobed_sub(int a, int b)
{
	asm volatile ("");
	return a - b;
}

int main(int argc, char **argv)
{
	struct uprobe_bpf *skel;
	int err, i;
	LIBBPF_OPTS(bpf_uprobe_opts, uprobe_opts);

	/* Set up libbpf errors and debug info callback */
	libbpf_set_print(libbpf_print_fn);

	/* Load and verify BPF application */
	skel = uprobe_bpf__open_and_load();
	if (!skel) {
		fprintf(stderr, "Failed to open and load BPF skeleton\n");
		return 1;
	}

	//uprobe/uretprobe 需要指定要附加到的目标函数的相对偏移量。
	//如果我们提供函数名，libbpf 将自动为我们计算偏移量。
	//如果没有指定函数名，libbpf 将尝试使用函数偏移量。
	uprobe_opts.func_name = "uprobed_add";
	uprobe_opts.retprobe = false;
	skel->links.uprobe_add = bpf_program__attach_uprobe_opts(skel->progs.uprobe_add,
								 0 /* self pid */, "/proc/self/exe",
								 0 /* offset for function */,
								 &uprobe_opts /* opts */);
	if (!skel->links.uprobe_add) {
		err = -errno;
		fprintf(stderr, "Failed to attach uprobe: %d\n", err);
		goto cleanup;
	}

	/* we can also attach uprobe/uretprobe to any existing or future
	 * processes that use the same binary executable; to do that we need
	 * to specify -1 as PID, as we do here
	 */
	uprobe_opts.func_name = "uprobed_add";
	uprobe_opts.retprobe = true;
	skel->links.uretprobe_add = bpf_program__attach_uprobe_opts(
		skel->progs.uretprobe_add, -1 /* self pid */, "/proc/self/exe",
		0 /* offset for function */, &uprobe_opts /* opts */);
	if (!skel->links.uretprobe_add) {
		err = -errno;
		fprintf(stderr, "Failed to attach uprobe: %d\n", err);
		goto cleanup;
	}

	/* Let libbpf perform auto-attach for uprobe_sub/uretprobe_sub
	 * NOTICE: we provide path and symbol info in SEC for BPF programs
	 */
	err = uprobe_bpf__attach(skel);
	if (err) {
		fprintf(stderr, "Failed to auto-attach BPF skeleton: %d\n", err);
		goto cleanup;
	}

	printf("Successfully started! Please run `sudo cat /sys/kernel/debug/tracing/trace_pipe` "
	       "to see output of the BPF programs.\n");

	for (i = 0;; i++) {
		/* trigger our BPF programs */
		fprintf(stderr, ".");
		uprobed_add(i, i + 1);
		uprobed_sub(i * i, i);
		sleep(1);
	}

cleanup:
	uprobe_bpf__destroy(skel);
	return -err;
}
```

上述代码实现了uprobed_add和uprobed_sub函数，分别是加法和减法。



bpf_program__attach_uprobe_opts这个是不需要计算偏移量了吗？

# 三、编写uprobe代码跟踪c语言程序