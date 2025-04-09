Tracee事件需要访问内核符号表，检查你的系统环境中/proc/kallsyms是否存在。



以下事件取决于kallsyms，你可以禁用他们：

- dirty_pipe_splice 
- hooked_syscall （检测系统调用拦截技术）
- hidden_kernel_module（检测内核模块隐藏技术）
- hooked_proc_fops（检测procfs文件操作拦截技术）
- print_net_seq_ops（相关的hooked_seq_ops 事件）
- hooked_seq_ops（检测网络数据包拦截技术）
- print_mem_dump（支持从符号到签名的内存转储功能）



# 一、tracee检测syscall table hook

## 1、1 模拟工具Diamorphine





## 1、2 检测工具

**新版本的执行命令**

```
./tracee-ebpf -e hooked_syscall
```

实验截图：

![image-20250325182935823](https://gitee.com/codergeek/picgo-image/raw/master/image/202503251829311.png)



## 1、3 源代码分析

### 1、3、1 ebpf部分代码

```
// syscall_table_check
SEC("uprobe/syscall_table_check")
int uprobe_syscall_table_check(struct pt_regs *ctx)
{
	//构建SYSCALL_TABLE_CHECK事件的相关数据
    program_data_t p = {};
    if (!init_program_data(&p, ctx, SYSCALL_TABLE_CHECK))
        return 0;

    // 检查触发uprobe进程的是否为tracee自身。
    if (p.config->tracee_pid != p.task_info->context.pid &&
        p.config->tracee_pid != p.task_info->context.host_pid)
        return 0;

    // Uprobes不是由syscalls触发的, 所以需要设置标记.
    p.event->context.syscall = NO_SYSCALL;
    
	// 调用syscall_table_check去检查系统调用表
    syscall_table_check(&p);

    return 0;
}
```

syscall_table_check函数实现如下：

```
statfunc void syscall_table_check(program_data_t *p)
{
    char sys_call_table_symbol[15] = "sys_call_table";
    u64 *sys_call_table = (u64 *) get_symbol_addr(sys_call_table_symbol);//获取sys_call_table符号的地址
#pragma unroll
    for (int i = 0; i < 500; i++) {
        index = i;
        //使用index索引值（范围0～499）来查看正确的系统调用的地址
        syscall_table_entry_t *expected_entry =
            bpf_map_lookup_elem(&expected_sys_call_table, &index);

		//遍历系统调用表，并读取sys_call_table中符号的地址
        u64 effective_address;
        bpf_probe_read_kernel(&effective_address, sizeof(u64), sys_call_table + index);

		//将每个系统调用的实际地址与预期地址（存储在 expected_sys_call_table 映射中）进行比较
		//如果相同则跳过
        if (expected_entry->address == effective_address)
            continue;

        save_to_submit_buf(&(p->event->args_buf), &index, sizeof(int), 0);
        save_to_submit_buf(&(p->event->args_buf), &effective_address, sizeof(u64), 1);
        events_perf_submit(p, 0);
    }
}
```



问题1：uprobe_syscall_table_check何时被调用，以及这个 uprobe 是如何工作的？

问题2：uprobe是针对用户态某个应用程序的，那么针对哪个应用程序呢？

Tracee 采用了一种巧妙的设计模式来实现系统调用表完整性检查。



### 1、3、2 用户态代码

**自我挂载：**

从 `probe_group.go` 的第185行可以看到，uprobe 挂载在 Tracee 自己的二进制文件上：

```
SyscallTableCheck: NewUprobe("syscall_table_check", "uprobe_syscall_table_check", binaryPath, "github.com/aquasecurity/tracee/pkg/ebpf.(*Tracee).triggerSyscallTableIntegrityCheckCall")
```



**空函数触发器：**

转到Tracee结构体的triggerSyscallTableIntegrityCheckCall方法，发现这是个空方法啊！！！

```
//go:noinline
func (t *Tracee) triggerSyscallTableIntegrityCheckCall() {
}
```

triggerSyscallTableIntegrityCheckCall函数唯一作用就是作为一个挂载点。注意它有一个特殊的注解 `//go:noinline`，这确保了该函数不会被编译器内联优化，从而保证它在二进制文件中有一个明确的入口点。



**自我触发：**

当 Tracee 调用这个空函数时，会触发之前注册的 uprobe，从而执行 eBPF 程序 `uprobe_syscall_table_check`。



**安全检查：**

```
// uprobe was triggered from other tracee instance
if (p.config->tracee_pid != p.task_info->context.pid &&
    p.config->tracee_pid != p.task_info->context.host_pid)
    return 0;
```

检查触发uprobe进程的是否为tracee自身。



```
func (t *Tracee) populateExpectedSyscallTableArray(tableMap *bpf.BPFMap) error {
	// Get address to the function that defines the not implemented sys call
	niSyscallSymbol, err := t.getKernelSymbols().GetSymbolByOwnerAndName("system", events.SyscallPrefix+"ni_syscall")
	niSyscallAddress := niSyscallSymbol[0].Address

	for i, kernelRestrictionArr := range events.SyscallSymbolNames {
		syscallName := t.getSyscallNameByKerVer(kernelRestrictionArr)
		var index = uint32(i)

		kernelSymbol, err := t.getKernelSymbols().GetSymbolByOwnerAndName("system", events.SyscallPrefix+syscallName)

		var expectedAddress = kernelSymbol[0].Address
		//将系统调用的索引和对应的地址存储到expected_sys_call_table映射表中
		err = tableMap.Update(unsafe.Pointer(&index), unsafe.Pointer(&expectedAddress))
	}

	return nil
}
```

tracee会将当前系统对应的每个系统调用的地址都填充到expected_sys_call_table中。



**总结：**

`uprobe_syscall_table_check` 是通过 Tracee 自己调用 `triggerSyscallTableIntegrityCheckCall` 函数来触发的，这个 uprobe 挂载在 Tracee 自己的二进制文件上，形成了一种"自我监控"机制。



参考链接：

https://aquasecurity.github.io/tracee/v0.22/docs/events/builtin/extra/hooked_syscall/

https://www.aquasec.com/blog/linux-syscall-hooking-using-tracee/

# 二、检测proc fops hook

## 2、1 工具检测

./tracee-ebpf -e hooked_proc_fops

![image-20250325184808076](https://gitee.com/codergeek/picgo-image/raw/master/image/202503251848236.png)



## 2、2 源代码分析

**hooked_proc_fops事件分析**

从 `core.go` 中的定义可以看到：

```
HookedProcFops: {
    id:      HookedProcFops,
    id32Bit: Sys32Undefined,
    name:    "hooked_proc_fops",
    version: NewVersion(1, 0, 0),
    dependencies: Dependencies{
        probes: []Probe{
            {handle: probes.SecurityFilePermission, required: true},
        },
        kSymbols: []KSymbol{
            {symbol: "_stext", required: true},
            {symbol: "_etext", required: true},
        },
        ids: []ID{
            DoInitModule,
        },
        capabilities: Capabilities{
            base: []cap.Value{
                cap.SYSLOG, // read /proc/kallsyms
            },
        },
    },
},
```

这个事件依赖于：

- `security_file_permission` 内核探针
- 内核符号 `_stext` 和 `_etext`（用于确定内核文本段的边界）
- `DoInitModule` 事件
- `cap.SYSLOG` 能力（用于读取 `/proc/kallsyms`）

### 2、2、1 ebpf部分代码

eBPF 部分的实现在 `trace_security_file_permission` 函数中：

```
SEC("kprobe/security_file_permission")
int BPF_KPROBE(trace_security_file_permission)
{
    // 判断是否是 procfs 文件系统
    if (s_magic != PROC_SUPER_MAGIC) {
        return 0;
    }

    // 获取文件操作函数表
    struct file_operations *fops = (struct file_operations *) BPF_CORE_READ(f_inode, i_fop);

    // 获取 iterate 和 iterate_shared 函数指针
    unsigned long iterate_addr = (unsigned long) BPF_CORE_READ(fops, iterate);
    unsigned long iterate_shared_addr = (unsigned long) BPF_CORE_READ(fops, iterate_shared);

    // 获取内核文本段边界
    void *stext_addr = get_stext_addr();
    void *etext_addr = get_etext_addr();

    // 如果函数指针在内核文本段内，标记为 0（表示正常）
    if (iterate_shared_addr >= (u64) stext_addr && iterate_shared_addr < (u64) etext_addr)
        iterate_shared_addr = 0;
    if (iterate_addr >= (u64) stext_addr && iterate_addr < (u64) etext_addr)
        iterate_addr = 0;

    // 将可能被钩子劫持的函数指针地址保存到数组中
    unsigned long fops_addresses[2] = {iterate_shared_addr, iterate_addr};

    // 将数组保存到事件参数缓冲区并提交事件
    save_u64_arr_to_buf(&p.event->args_buf, (const u64 *) fops_addresses, 2, 0);
    events_perf_submit(&p, 0);
    return 0;
}
```

这段 eBPF 代码的工作原理是：

1. 跟踪 `security_file_permission` 内核函数调用
2. 检查是否是 procfs 文件系统的操作
3. 获取文件的 `file_operations` 结构体
4. 检查 `iterate` 和 `iterate_shared` 函数指针
5. 判断函数指针是否在内核文本段内
   - 如果在文本段内，认为是正常的（标记为 0）
   - 如果在文本段外，可能被钩子劫持
6. 如果发现可能被劫持的函数指针，将其地址提交给用户空间处理



### 2、2、2 用户态代码

用户态部分的实现在 `processHookedProcFops` 函数中：

```

// processHookedProcFops 处理hooked_proc_fops事件，用于检测procfs文件操作函数是否被钩子劫持
func (t *Tracee) processHookedProcFops(event *trace.Event) error {
    // 定义参数名称常量，与eBPF程序中使用的参数名称对应
    const hookedFopsPointersArgName = "hooked_fops_pointers"
    
    // 从事件参数中解析出uint64类型的地址数组，这些地址是可能被钩子劫持的函数指针
    fopsAddresses, err := parse.ArgVal[[]uint64](event.Args, hookedFopsPointersArgName)
    
    // 初始化结果数组，用于存储被钩子劫持的函数信息
    hookedFops := make([]trace.HookedSymbolData, 0)
    
    // 遍历从eBPF程序传递过来的地址数组
    for idx, addr := range fopsAddresses {
        // 查找地址对应的内核符号和模块信息
        hookingFunction := t.getKernelSymbols().GetPotentiallyHiddenSymbolByAddr(addr)[0]
        
        // 如果函数的拥有者是"system"，表示这是系统正常的函数，不是恶意劫持，直接跳过
        if hookingFunction.Owner == "system" {
            continue
        }
        
        // 根据索引确定具体的函数名称
        functionName := "unknown"
        switch idx {
        case IterateShared: // 索引为0
            functionName = "iterate_shared"
        case Iterate: // 索引为1
            functionName = "iterate"
        }
        
        // 将被劫持函数的名称和拥有者模块的信息填充到HookedSymbolData结构体，并添加到结果数组中
        hookedFops = append(hookedFops, trace.HookedSymbolData{SymbolName: functionName, ModuleOwner: hookingFunction.Owner})
    }
    
    // 更新事件参数，将原始的地址数组替换为更有意义的结构化数据
    err = events.SetArgValue(event, hookedFopsPointersArgName, hookedFops)
    if err != nil {
        return err
    }
    
    // 处理成功，返回nil表示没有错误
    return nil
}
```

用户态代码的处理逻辑：

- **解析函数地址：**从eBPF上报的事件中提取被hook劫持的函数地址
- **符号解析：**将地址转换为内核符号信息，明确函数名称和所属模块
- **过滤正常函数：**排除拥有者是"system"的正常系统函数
- **构建劫持信息：**基于索引识别具体被劫持的函数类型(iterate, iterate_shared)
- **更新事件数据：**将被劫持函数的名称和所属模块信息填充后进行上报



# 三、tracee检测模块加载/隐藏内核模块

```
#define TRACE_ENT_FUNC(name, id)                                                                   \
    int trace_##name(struct pt_regs *ctx)                                                          \
    {                                                                                              \
        args_t args = {};                                                                          \
        args.args[0] = PT_REGS_PARM1(ctx);                                                         \
        args.args[1] = PT_REGS_PARM2(ctx);                                                         \
        args.args[2] = PT_REGS_PARM3(ctx);                                                         \
        args.args[3] = PT_REGS_PARM4(ctx);                                                         \
        args.args[4] = PT_REGS_PARM5(ctx);                                                         \
        args.args[5] = PT_REGS_PARM6(ctx);                                                         \
                                                                                                   \
        return save_args(&args, id);                                                               \
    }
```

TRACE_ENT_FUNC(do_init_module, DO_INIT_MODULE)会根据TRACE_ENT_FUNC宏定义，生成一个trace_do_init_module 的函数。

这个函数的主要功能是：

- 获取 kprobe 触发时的寄存器状态（ctx）

- 从寄存器中提取函数参数（从PT_REGS_PARM1 到 PT_REGS_PARM6）

- 将这些参数保存到一个名为 args 的结构体中

- 调用 save_args 函数，将参数和指定的 ID（这里是 DO_INIT_MODULE）一起保存起来，以便后续处理



DO_INIT_MODULE：标识do_init_module函数相关事件的宏。

```
SEC("kretprobe/do_init_module")
int BPF_KPROBE(trace_ret_do_init_module)
{
    // 1. 恢复之前保存的参数
    args_t saved_args;
    if (load_args(&saved_args, DO_INIT_MODULE) != 0) {
        // 如果无法找到之前保存的参数（可能是因为错过了入口点或该函数没有被跟踪）
        return 0;
    }
    // 删除已加载的参数，释放资源
    del_args(DO_INIT_MODULE);

    // 2. 初始化程序数据结构
    program_data_t p = {};
    if (!init_program_data(&p, ctx, HIDDEN_KERNEL_MODULE_SEEKER))
        return 0;

    // 3. 获取模块指针（从之前保存的参数中）
    struct module *mod = (struct module *) saved_args.args[0];

    // 4. 触发隐藏内核模块查找器（LKM Seeker）
    if (evaluate_scope_filters(&p)) {
        u64 addr = (u64) mod;
        u32 flags = FULL_SCAN;
        lkm_seeker_send_to_userspace((struct module *) addr, &flags, &p);
    }

    // 5. 从模块中读取版本信息
    const char *version = BPF_CORE_READ(mod, version);
    const char *srcversion = BPF_CORE_READ(mod, srcversion);

    // 6. 重置事件数据结构，准备新的事件
    if (!reset_event(p.event, DO_INIT_MODULE))
        return 0;

    // 7. 再次检查过滤条件（这可能是基于事件 ID 的不同过滤器）
    if (!evaluate_scope_filters(&p))
        return 0;

    // 8. 将模块名称和版本信息保存到事件缓冲区
    save_str_to_buf(&p.event->args_buf, &mod->name, 0);
    save_str_to_buf(&p.event->args_buf, (void *) version, 1);
    save_str_to_buf(&p.event->args_buf, (void *) srcversion, 2);

    // 9. 获取函数的返回值
    int ret_val = PT_REGS_RC(ctx);
    
    // 10. 提交事件到用户空间
    return events_perf_submit(&p, ret_val);
}
```



进程执行。

```
SEC("kprobe/call_usermodehelper") // 定义一个kprobe，监视call_usermodehelper函数的调用
int BPF_KPROBE(trace_call_usermodehelper) // kprobe的处理函数
{
    program_data_t p = {}; // 初始化程序数据结构体p
    if (!init_program_data(&p, ctx, CALL_USERMODE_HELPER)) // 初始化程序数据，检查是否成功
        return 0; // 如果初始化失败，返回0

    if (!evaluate_scope_filters(&p)) // 评估作用域过滤器，检查是否满足条件
        return 0; // 如果不满足条件，返回0

    void *path = (void *) PT_REGS_PARM1(ctx); // 获取第一个参数：用户模式助手的路径
    unsigned long argv = PT_REGS_PARM2(ctx); // 获取第二个参数：argv（参数数组）
    unsigned long envp = PT_REGS_PARM3(ctx); // 获取第三个参数：envp（环境变量数组）
    int wait = PT_REGS_PARM4(ctx); // 获取第四个参数：wait（等待标志）

    save_str_to_buf(&p.event->args_buf, path, 0); // 将路径保存到事件参数缓冲区
    save_str_arr_to_buf(&p.event->args_buf, (const char *const *) argv, 1); // 将argv数组保存到事件参数缓冲区
    save_str_arr_to_buf(&p.event->args_buf, (const char *const *) envp, 2); // 将envp数组保存到事件参数缓冲区
    save_to_submit_buf(&p.event->args_buf, (void *) &wait, sizeof(int), 3); // 将wait标志保存到事件参数缓冲区

    return events_perf_submit(&p, 0); // 提交事件并返回结果
}
```

该代码段定义了一个kprobe，用于监视call_usermodehelper函数的调用。

首先，初始化一个数据结构以存储程序的上下文信息；

其次，提取函数参数（路径、参数数组、环境变量数组和等待标志），并将这些信息保存到事件参数缓冲区中；

最后，提供事件以供后续处理；

# 四、通信通道

可实现的手段：

- Network packets
- New device
- New proc device
- Override device
- Override syscalls

Diamorphine例子中使用的Override syscalls的手法来构建通信通道，和内核模块进行通信，来达到和恶意软件通信来下发comand指令的目的。



Tracee主要通过以下几种方式检测针对tcp4_seq_show的hook：

- 文件操作钩子检测：通过proc_fops_hooking.go检测对proc文件系统的文件操作钩子

- 多源数据对比：通过eBPF直接捕获网络数据包并与proc文件系统报告的连接进行对比

- 内核函数完整性检查：检查关键函数（如tcp4_seq_show）是否被修改

## 具体检测机制

### 1. proc文件系统操作钩子检测

Tracee会检测/proc/net/tcp文件的文件操作结构体是否被篡改：

```
// tracee中检测文件操作钩子的部分代码
if (hooked_fops_pointers) {
    // 检查/proc/net/*文件的文件操作结构体
    // 包括tcp, tcp6, udp等
}
```

当rootkit修改了tcp4_seq_show函数（该函数负责读取/proc/net/tcp文件内容）时，Tracee会发现文件操作结构体中的函数指针被更改。



### 2. 网络数据包与proc文件系统对比

Tracee使用eBPF程序直接从内核捕获网络连接数据：

```
// 在tracee.bpf.c中的代码
u32 ret = CGROUP_SKB_HANDLE(proto);
```

这行代码的作用是调用对应协议的处理函数，如cgroup_skb_handle_proto_tcp，获取真实的网络连接信息。随后Tracee将这些信息与从/proc/net/tcp读取的信息进行对比，如果发现差异（例如实际有8080端口连接但在/proc/net/tcp中看不到），就会认为存在隐藏的网络连接。



### 3. 内核函数完整性检查

Tracee还会检查关键内核函数的完整性：

```
// 检查系统调用表和关键内核函数是否被钩子
if (original_function_address != current_function_address) {
    // 发现钩子
}
```

这种方法可以发现通过ftrace或内核模块直接替换函数指针的钩子技术。



参考链接：

https://aquasecurity.github.io/tracee/dev/docs/events/builtin/extra/net_tcp_connect/

# 五、检测进程执行



参考资料：

TODO:必须好好的翻译

https://www.aquasec.com/blog/ebpf-container-tracing-malware-detection/



# 六、检测隐藏内核模块

```
./tracee -e hidden_kernel_module
```



ftrace hook手法

./tracee -e ftrace_hook

1. SEC("raw_tracepoint/sys_enter") 和 SEC("raw_tracepoint/sys_exit"):

这些函数用于监控系统调用的进入和退出，可以帮助检测异常的系统调用行为。

SEC("raw_tracepoint/sched_process_fork"):

该函数监控进程的创建（fork），可以用于检测是否有恶意进程被创建。

3. SEC("raw_tracepoint/sched_process_exec"):

监控进程执行，可以帮助检测是否有可疑的可执行文件被运行。

SEC("uprobe/lkm_seeker") 和相关的尾调用函数:

这些函数用于检测内核模块的加载和卸载，能够识别隐藏的内核模块，这是 rootkit 常用的隐蔽手段。

5. SEC("uprobe/lkm_seeker_kset_tail") 和 SEC("uprobe/lkm_seeker_mod_tree_tail"):

这些函数进一步检查内核模块的状态，确保没有隐藏的模块存在。

6. SEC("uprobe/lkm_seeker_new_mod_only_tail"):

监控新加载的模块，确保它们是合法的。

SEC("uprobe/syscall_table_check"):

检查系统调用表的完整性，防止恶意代码篡改系统调用。

SEC("kprobe/do_exit"):

监控进程退出，可以帮助检测是否有异常的进程退出行为。



# 六、参考资料

# BlackHat Arsenal 2022: Detecting Linux kernel rootkits with Aqua Tracee

https://www.youtube.com/watch?v=EATX8g3sh-0&t=702s

上面这个资料太重要了，一定要多看。



**Detecting Drovorub’s File Operations Hooking with Tracee**

https://www.aquasec.com/blog/detect-drovorub-kernel-rootkit-attack-tracee/



**在tracee目录执行make help，**这个太重要了，在编译go程序时，必须要生成调试符号才能行