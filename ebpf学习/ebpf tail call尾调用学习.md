# 一、尾调用简介

从 4.2 内核版本开始，eBPF 支持了尾调用特性。 该特性的主要特点是需要使用 `bpf_tail_call()` 这个帮助函数。

尾调用，是调用另外一个 eBPF 程序，而不再返回。

这就允许将多个 eBPF 程序串联起来，从而能够突破 4096 个指令的限制（在 5.2 内核之前，每个 eBPF 程序的指令数量限制是 4096，之后则是 100 万）。

尾调用是有调用次数限制的，目前限制为 32 次，由内核宏 **MAX_TAIL_CALL_CNT** 指定。



# 二、尾调用函数及代码示例

```
long bpf_tail_call(void *ctx, struct bpf_map *prog_array_map, u32 index)
```

ctx：指向上下文的



采用下面的代码进行讲解：

```
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_core_read.h>
#include <bpf/bpf_tracing.h>

char LICENSE[] SEC("license") = "GPL";

SEC("raw_tracepoint/sys_enter")
int enter_fchmodat(struct bpf_raw_tracepoint_args *ctx) {	
    struct pt_regs* regs;
    regs = (struct pt_regs*)ctx->args[0];

    //int fchmodat(int dirfd, const char* pathname, mode_t mode, int flags);
    char pathname[256];
    char* pathname_ptr = (char*)PT_REGS_PARM2_CORE(regs);
    bpf_core_read_user_str(pathname, sizeof(pathname), pathname_ptr);

    char fmt[] = "fchmodat %s\n";
    bpf_trace_printk(fmt, sizeof(fmt), &pathname);
    return 0;
}

// 映射表，存储程序的尾调用的
struct {
    __uint(type, BPF_MAP_TYPE_PROG_ARRAY);
    __uint(max_entries, 1024);
    __uint(key_size, sizeof(u32));
    __uint(value_size, sizeof(u32));
    __array(values,int (void*));
}tail_jump_map SEC(".maps") = {
    .values = {
        [268] = (void*)&enter_fchmodat,
    },
};

SEC("raw_tracepoint/sys_enter")
int raw_tracepoint__sys_enter(struct bpf_raw_tracepoint_args *ctx) {
    //call another ebpf program
    u32 syscall_id = 268;
    bpf_tail_call(ctx,&tail_jump_map,syscall_id);

    char fmt[] = "no bpf program for syscall %d\n";
    bpf_trace_printk(fmt, sizeof(fmt), syscall_id);

    return 0;
}

```

代码链接在https://github.com/haolipeng/ebpf-playground/blob/master/rootkit.bpf.c

欢迎star。

# 三、参考链接

https://github.com/mozillazg/hello-libbpfgo/blob/master/22-tail-calls/main.bpf.c



b站《Go cilium/ebpf尾递归示例》

https://www.bilibili.com/video/BV1zy411h72B/?spm_id_from=333.337.search-card.all.click&vd_source=553cfea04504936858081191fc1520dc



**eBPF Talk: eBPF 尾调用简介**

https://asphaltt.github.io/post/ebpf-tailcall-intro/