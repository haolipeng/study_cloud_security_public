功能简介和实现原理

对来说，



# 一、Dashboard仪表盘

## 1、1 统计信息

![image-20220512132317631](https://gitee.com/codergeek/picgo-image/raw/master/image/202306301215261.png)

## 1、2 暴露的ingress和egress

![image-20220512132547296](https://gitee.com/codergeek/picgo-image/raw/master/image/202306301215617.png)

右上角可以切换Ingress和Egress按钮，可以查看ingress和egress的pod情况。

## 1、3 Critical Run-Time Security Events

关键的运行时安全事件

![image-20220512132656150](https://gitee.com/codergeek/picgo-image/raw/master/image/202306301215925.png)

## 1、4 Top Security Events - Source and Destination

安全事件top排名，源和目的。

![image-20220512132840980](https://gitee.com/codergeek/picgo-image/raw/master/image/202306301215030.png)

## 1、5 Top Vulnerable Pods and Nodes

有漏洞风险的Pods和Nodes

![image-20220512132908769](https://gitee.com/codergeek/picgo-image/raw/master/image/202306301215277.png)

## 1、6 Policy Mode of Services and Pods

![image-20220512132946267](https://gitee.com/codergeek/picgo-image/raw/master/image/202306301215534.png)

## 1、7 Application Protocols Detected and Protected by Layer 7 Network Rules

![image-20220512133055415](https://gitee.com/codergeek/picgo-image/raw/master/image/202306301216235.png)



# 二、网络连接

![image-20220512143906930](https://gitee.com/codergeek/picgo-image/raw/master/image/202306301217382.png)

可以看到不同实体（node或虚机）之间的连接关系

![image-20220512144140441](https://gitee.com/codergeek/picgo-image/raw/master/image/202306301217593.png)

点击后还会出现不同的

# 三、资产

## 3、1 Platform平台

![image-20220512144427537](https://gitee.com/codergeek/picgo-image/raw/master/image/202306301218691.png)

## 3、2 Nodes节点

![image-20220512144914313](picture/image-20220512144914313.png)

## 3、3 Container容器

![image-20220512144933635](picture/image-20220512144933635.png)

## 3、4 Registries仓库

![image-20220512145739973](picture/image-20220512145739973.png)

添加仓库

![image-20220512145753364](picture/image-20220512145753364.png)

## 3、5 System Components系统组件



# 四、Policy策略

## 4、1 DLP sensor

![image-20220512133915977](picture/image-20220512133915977.png)



## 4、2 waf sensor

![image-20220512134226562](picture/image-20220512134226562.png)

值得关注的是添加waf规则的方式

![image-20220512134132151](picture/image-20220512134132151.png)

编写好的Pattern可以点击右侧的"TEST"按钮进行测试。

context上下文支持packet、url、header、body。（TODO：有可能不止四种）

![image-20220512134310214](picture/image-20220512134310214.png)

waf的规则如何来？

1、crd是什么？

在k8s中pod、node、service都是一种内置资源对象，crd = custom resource define。

kubectl get pod

kubectl get rules



1、语法

2、范围 局部 全局

3、角色和角色绑定



- 在应用程序开发期间定义安全策略，推向生产环境。
- 在投入生产之前，使用 NeuVector 了解行为并导出 CRD 以供审查。
- 将安全策略从暂存集群迁移到生产集群。
- 在多云、多集群、混合云中复制规则。
- 执行全局安全政策。



支持的资源类型

NvSecurityRule

NvClusterSecurityRule

两种资源的区别就是边界不同。NvSecurityRule 资源的范围在命名空间级别，NvClusterSecurityRule 的范围是集群级别的。任何一种资源类型都可以在 yaml 文件中配置，并且可以在部署期间创建。



一般来说，crd的规则的制定者会公布crd的语法是什么

https://open-docs.neuvector.com/policy/usingcrd#crd-policy-rules-syntax-and-semantics



使用CRD的RBAC支持



参考链接

https://open-docs.neuvector.com/policy/usingcrd

https://open-docs.neuvector.com/policy/dlp



## 4、3 Response Rules 响应规则

![image-20220512134636235](picture/image-20220512134636235.png)

右侧按钮分别是Edit Rule，Add Rule，Enable Rule，Delete Rule。

添加规则：

![image-20220512135016311](picture/image-20220512135016311.png)

**Category**：策略，支持的类型有Security Event，Event，CVE-Report，Compliance，Admission

![image-20220512135254888](picture/image-20220512135254888.png)

**Group**：分组

**Criteria Type**:标准类型

Criteria Type根据Category策略的变化而变化。

Security Event策略

![image-20220512135552389](picture/image-20220512135552389.png)

Event策略

![image-20220512135611840](picture/image-20220512135611840.png)

CVE-Report策略

![image-20220512135632200](picture/image-20220512135632200.png)

Compliance策略

![image-20220512135725856](picture/image-20220512135725856.png)

Admission策略

![image-20220512135757301](picture/image-20220512135757301.png)

**Action**: 操作

**Status**：启用或禁用状态



## 4、4 Network Rules 网络规则

![image-20220512140148708](picture/image-20220512140148708.png)

From:源组，指定连接的源GROUP

TO:指定允许或拒绝这些连接的目标 GROUP

Application：指定允许或拒绝的应用程序，Neuvector将分析有效负载以确定应用程序协议；要选择任何/全部，					请将此字段留空。

Ports：端口，如tcp/6443代表tcp协议的6443端口

Action：动作，Allow表示允许

Type：类型，Learned表示学习

Update at：更新时间



**重要：**网络规则按照它们在列表中出现的顺序从上到下执行。要重新排序规则，请选择要移动的规则，然后您会看到顶部出现“移动到”框，您可以将选定的规则移动到指定规则之前或之后的位置。



**重要：**如果您编辑（添加、删除、更改）规则，您的更改将不会应用，直到您单击顶部的保存按钮。如果您在未部署更改的情况下退出此页面，它们将会丢失。

## 4、5 Groups组

![image-20220512141337047](picture/image-20220512141337047.png)

对于nv.calico-apiserver.calico-apiserver组而言，点击后查看下面的Members成员（2个）

Group命名 = nv + 命名空间 + 服务名



因为Neuvector的规则都是针对Group组而言的，有进程规则、文件规则、网络规则、响应规则、waf规则、dlp规则等。

### 1） 进程规则

![image-20220512141704730](picture/image-20220512141704730.png)

Process Name：进程名称

Path：进程路径



### 2）文件规则

![image-20220512141728223](picture/image-20220512141728223.png)

**Recursive**：是否递归

**Action**：监控文件变化 或者 阻断未授权的访问

**Allowed Applications Type**:允许的应用类型



### 3）网络规则

![image-20220512142250941](picture/image-20220512142250941.png)

### 4） 响应规则

![image-20220512142406802](picture/image-20220512142406802.png)



## 4、6 Admission Control访问控制

![image-20220512142459756](picture/image-20220512142459756.png)

添加规则

![image-20220512142636276](picture/image-20220512142636276.png)

**Criterion**：标准，支持以下条目：

Allow Privilege Escalation

Count of high severity CVE

Count of high severity CVE with fix

Count of medium severity CVE

CVE names

CVE score

Environment variables with secrets

Environment variables

![image-20220512142823091](picture/image-20220512142823091.png)



Image

Image compliance violations

Image without OS information

Image registry

Image scanned

Labels

Mount Volumes

Namespace

![image-20220512142843506](picture/image-20220512142843506.png)



PSP Best Practice

Resource Limit Configuration

Run as privileged

Run as root

Share host ipc namespaces

Share host's Network

Share host's PID namespaces

User

User groups

![image-20220512142917188](picture/image-20220512142917188.png)



# 五、原理篇

## 5、1 容器安全的出口控制

https://blog.neuvector.com/article/enforce-egress-control-containers

备注：以下文字只是翻译了部分文字。



​	一旦在容器主机上部署 NeuVector Enforcer 容器后，它使用 DPI 技术（深度数据包检测）自动开始监视该主机上的容器网络连接。NeuVector使用DPI技术 能够发现网络中的攻击。它还可用于识别应用程序并在应用程序级别执行网络策略。



​	一旦将拥有DNS白名单的网络策略推送到 Enforcer，Enforcer 的data path进程开始默默地检查容器发出的 DNS 请求。Enforcer 不会主动发送任何 DNS 请求，因此 NeuVector 不会引入额外的网络开销。它解析 DNS 请求和响应，并将解析的 IP 与指定容器建立的网络连接相关联。根据定义的策略，Enforcer 可以允许、警告或拒绝网络连接。

![Network Connections](picture/neuvector_k8-egress.png)

这种方法适用于未加密和加密的连接，并且不会被任何伪造的标头欺骗。它可用于对任何协议强制执行策略，而不仅仅是 HTTP 或 HTTPS协议。更重要的是，由于 DPI 技术，Enforcer 可以检查连接并确保仅使用正确的 PostgreSQL 协议来访问 PostgreSQL 数据库。



使用 DPI/DLP 通过传统代理执行出口控制

另一个更高级的用例是通过代理（例如在集群外运行的 squid 代理）路由流量。

这带来了挑战，即如何根据连接的目的地区分,连接是应该被允许或应该被阻止，但在源容器或pod 上执行策略。



在下面的示例中，允许连接访问morningstart.com 上的外部资源，而 对于oracle.com 的连接应该被阻止。但是，所有连接都必须通过在集群外运行的 squid 代理。

![Cluster](picture/egress_proxy.png)

挑战在于我们希望在集群内、在源容器或pod 上强制执行出口控制，定义allow允许哪些容器源访问哪些出口目的地将更加灵活。



为了实现这一点，我们可以使用 NeuVector 深度数据包检测 (DPI)功能和数据丢失防护 (DLP) 功能来检查出站连接中的 http 标头，并在阻止 oracle.com 的同时允许 morninstar.com。

