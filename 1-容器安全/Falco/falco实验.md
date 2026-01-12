# 一、安装部署falco

首先，你拷贝配置文件到/etc/falco目录

```
sudo -s
mkdir /etc/falco
cp falco.yaml falco_rules.yaml /etc/falco
touch /var/log/falco_events.log
```

falco.yaml 配置 Falco 服务

falco_rules.yaml 包含威胁检测模式

falco_events.log 记录事件的日志文件



然后，您可以拉取并启动 Falco 容器，挂载您之前定义的配置文件：

```
docker run -d --name falco \
    --privileged \
    -v /var/run/docker.sock:/host/var/run/docker.sock \
    -v /dev:/host/dev \
    -v /proc:/host/proc:ro \
    -v /boot:/host/boot:ro \
    -v /lib/modules:/host/lib/modules:ro \
    -v /usr:/host/usr:ro \
    -v /etc:/host/etc:ro \
    -v /etc/falco/falco.yaml:/etc/falco/falco.yaml \
    -v /etc/falco/falco_rules.yaml:/etc/falco/falco_rules.yaml \
    -v /var/log/falco_events.log:/var/log/falco_events.log \
    falcosecurity/falco:latest
```



# 实验一、容器运行交互式 shell

在第一个练习中，来检测在任何容器中运行交互式 shell 的攻击者。此警报包含在默认规则集中。

我们先触发它，然后你可以查看规则定义。

在 Docker 主机上运行任何容器，例如 nginx：

```
docker run -d -P --name example1 nginx
```

现在创建一个交互式shell：

```
docker exec -it example1 bash
```

你可以在容器内玩一下，然后执行 exit 离开容器shell。



**日志事件**

如果您使用 tail /var/log/falco_events.log 跟踪日志文件，您应该能够阅读：

```
08:46:43.798834868: Notice A shell was spawned in a container with an attached terminal (user=root user_loginuid=-1 example1 (id=07b87bc077d1) shell=bash parent=runc cmdline=bash terminal=34816 container_id=07b87bc077d1 image=nginx)
```



**规则：容器中的终端shell**

这是触发事件的规则，位于/etc/falco/falco_rules.yaml

```
- rule: Terminal shell in container
  desc: A shell was used as the entrypoint/exec point into a container with an attached terminal.
  condition: >
    spawned_process and container
    and shell_procs and proc.tty != 0
    and container_entrypoint
    and not user_expected_terminal_shell_in_container_conditions
  output: >
    A shell was spawned in a container with an attached terminal (user=%user.name user_loginuid=%user.loginuid %container.info
    shell=%proc.name parent=%proc.pname cmdline=%proc.cmdline terminal=%proc.tty container_id=%container.id image=%container.image.repository)
  priority: NOTICE
  tags: [container, shell, mitre_execution]
```

**规则剖析**

这是一个相当复杂的规则，如果您此时没有完全理解每个部分，请不要担心。您可以识别：

- 规则名称 
- 描述 
- 触发条件 
- 事件输出（带有上下文相关变量，如 %proc.name 或 %container.info） 
- 优先 
- 标签



实验2：未经授权的进程

Docker 最佳实践建议每个容器只运行一个主进程。

例如，nginx 容器应该只执行 nginx 进程。其他任何东西都是异常的指标。

您将在此示例中使用新版本的配置文件：

```
sudo cp falco_rules_3.yaml /etc/falco/falco_rules.yaml
```

注意文件末尾的以下规则：

```
# Our nginx containers should only be running the 'nginx' process
- rule: Unauthorized process on nginx containers
  desc: There is a process running in the nginx container that is not described in the template
  condition: spawned_process and container and container.image startswith nginx and not proc.name in (nginx)
  output: Unauthorized process (%proc.cmdline) running in (%container.id)
  priority: WARNING
```

我们来剖析规则的condition:

spawned_process:

container:

not proc.name in (nginx):

https://katacoda.com/falco/courses/falco/falco



执行命令 kubectl cluster-info

```
controlplane $ kubectl cluster-info
Kubernetes master is running at https://172.17.0.8:6443
KubeDNS is running at https://172.17.0.8:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```



```
controlplane $ kubectl get nodes
NAME           STATUS   ROLES    AGE     VERSION
controlplane   Ready    master   3m      v1.18.0
node01         Ready    <none>   2m26s   v1.18.0
```



```
controlplane $ helm repo add falcosecurity https://falcosecurity.github.io/charts
"falcosecurity" has been added to your repositories

controlplane $ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "falcosecurity" chart repository
Update Complete. ⎈Happy Helming!⎈

controlplane $ kubectl create ns falco
namespace/falco created

controlplane $ helm install falco -n falco falcosecurity/falco
NAME: falco
LAST DEPLOYED: Fri Mar 11 06:54:19 2022
NAMESPACE: falco
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Falco agents are spinning up on each node in your cluster. After a few
seconds, they are going to start monitoring your containers looking for
security issues.
```



大致流程如下

![img](picture/01b_topology.png)

```
kubectl create namespace ping
kubectl create -f mysql-deployment.yaml --namespace=ping
kubectl create -f mysql-service.yaml --namespace=ping
kubectl create -f ping-deployment.yaml --namespace=ping
kubectl create -f ping-service.yaml --namespace=ping
kubectl create -f client-deployment.yaml --namespace=ping
```



```
kubectl exec client -n ping -- curl -F "s=OK" -F "user=bob" -F "passwd=foobar" -F "ipaddr=localhost" -X POST http://ping/ping.php
```



```
helm upgrade falco falcosecurity/falco -f custom_rules.yaml -n falco
```



容器安全 译文

https://www.katacoda.com/falco/courses/falco/falco

尽管它在使用容器时特别有用，因为它支持特定于容器的上下文，例如 container.id、container.image 或命名空间的规则。

Falco 是一种审计工具，与 Seccomp 或 AppArmor 等强制工具不同。

Falco 运行在用户空间，使用内核模块拦截系统调用。



首先，拷贝修改后的配置文件，到/etc/falco目录

```
sudo -s
mkdir /etc/falco
cp falco.yaml falco_rules.yaml /etc/falco
touch /var/log/falco_events.log
```

falco.yaml

falco_rules.yaml



```
docker run -d --name falco \
    --privileged \
    -v /var/run/docker.sock:/host/var/run/docker.sock \
    -v /dev:/host/dev \
    -v /proc:/host/proc:ro \
    -v /boot:/host/boot:ro \
    -v /lib/modules:/host/lib/modules:ro \
    -v /usr:/host/usr:ro \
    -v /etc:/host/etc:ro \
    -v /etc/falco/falco.yaml:/etc/falco/falco.yaml \
    -v /etc/falco/falco_rules.yaml:/etc/falco/falco_rules.yaml \
    -v /var/log/falco_events.log:/var/log/falco_events.log \
    falcosecurity/falco:latest
```

