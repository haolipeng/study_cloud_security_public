# 03 - 配置文件快速上手

> `suricata.yaml` 很长，但入门阶段不需要逐项记住。本文只保留工程实践最常用的配置：网络变量、日志输出、AF_PACKET IDS、应用层协议开关、TCP 流处理、规则文件和配置验证。

## 本篇目标

学完后应能完成 5 件事：

- 知道 `HOME_NET` 对规则和现场网段的影响
- 能确认 `eve.json`、`suricata.log`、`stats.log` 是否启用
- 能配置 AF_PACKET IDS 的最小参数
- 能检查应用层协议是否启用，理解 `detection-ports`
- 能用 `-T`、`--dump-config`、`--set` 验证配置是否生效

建议按这个顺序阅读和实践：

| 步骤 | 重点 | 对应章节 |
|------|------|----------|
| 1 | 先知道配置文件里常看哪些区域 | 最小结构 |
| 2 | 配好基础输入、输出和协议解析 | `vars`、`outputs`、`af-packet`、`app-layer.protocols` |
| 3 | 理解影响 MMS 解析的 TCP 长连接和规则加载 | `stream` / `flow`、`rule-files` |
| 4 | 用命令验证配置是否生效 | `--set`、配置验证 |

## 1. 配置文件最小结构

先把 `suricata.yaml` 理解成 5 个常用区域。入门阶段不需要从头到尾背配置，能快速定位这些区域即可：

| 区域 | 作用 |
|------|------|
| `vars` | 网络和端口变量 |
| `outputs` | EVE JSON、stats、fast.log 等输出 |
| `af-packet` | 现场旁路抓包配置 |
| `app-layer.protocols` | 应用层协议启用、端口识别 |
| `stream` / `flow` | TCP 流跟踪、重组和 flow 管理 |
| `rule-files` | 规则文件路径 |

## 2. 基础变量：vars

本节先解决两个问题：

- 规则里的内外网变量怎么展开
- 工控协议常用端口变量在哪里

`vars` 主要影响规则中的变量展开。入门阶段最重要的是 `HOME_NET`。

```yaml
vars:
  address-groups:
    HOME_NET: "[10.1.0.0/16,10.2.0.0/16]"
    EXTERNAL_NET: "!$HOME_NET"

    DNP3_SERVER: "$HOME_NET"
    DNP3_CLIENT: "$HOME_NET"
    MODBUS_CLIENT: "$HOME_NET"
    MODBUS_SERVER: "$HOME_NET"
    ENIP_CLIENT: "$HOME_NET"
    ENIP_SERVER: "$HOME_NET"
```

关键点：

- `HOME_NET` 应写成真实现场或实验网段，不要长期用 `any`
- `EXTERNAL_NET: "!$HOME_NET"` 表示非内网地址
- 工控现场通常先把站控层、工程师站、IED、网关等网段纳入 `HOME_NET`
- 规则不触发时，`HOME_NET` 配错是常见原因之一

端口变量只需先知道规则会引用它们：

```yaml
vars:
  port-groups:
    DNP3_PORTS: 20000
    MODBUS_PORTS: 502
    SSH_PORTS: 22
```

IEC 61850 MMS 常见 TCP 端口是 `102`。如果插件或规则依赖特定端口，需要确认对应协议配置或规则中是否包含该端口。

## 3. 输出日志：outputs

本节先确认 3 类输出：

- `eve.json`：结构化事件，最重要
- `stats.log`：统计计数，用来判断包、flow、app-layer 是否进入引擎
- `suricata.log`：Suricata 自身运行日志，用来查启动、配置、规则、输出错误

### 日志目录

```yaml
default-log-dir: /var/log/suricata/
```

命令行 `-l <dir>` 会覆盖这个目录。学习和回归验证时，为防止日志互相干扰，建议每次用 `-l` 指定独立目录：

```bash
suricata -c suricata.yaml -r test.pcap -l /tmp/suri-test -k none
```



### stats日志

```yaml
stats:
  enabled: yes
  interval: 8
```

`stats.log` 用来确认包是否进入引擎、TCP flow 是否建立、应用层是否有计数。排查“没有 MMS 事件”时，先看 `stats.log` 很有用。



### EVE JSON日志

EVE JSON 是最重要的输出。入门阶段只关心普通文件输出：

```yaml
outputs:
  - eve-log:
      enabled: yes
      filetype: regular
      filename: eve.json
      types:
        - flow:
        - stats:
            totals: yes
            threads: no
            deltas: no
```



### suricata.log

Suricata 自身运行日志通常这样配置：

```yaml
logging:
  default-log-level: notice
  outputs:
    - console:
        enabled: yes
    - file:
        enabled: yes
        level: info
        filename: suricata.log
```

排查时重点搜：

```bash
rg -n "error|warning|failed|app-layer|eve|rule|pcap" /tmp/suri-test/suricata.log
```

`debug` 日志量很大，只在定位具体问题时临时打开。



## 4. 输入方式：af-packet

AF_PACKET 用于 Linux 上的旁路监听，常见输入来源是镜像口、TAP 或测试网卡。

```yaml
af-packet:
  - interface: eth0
    cluster-id: 99
    cluster-type: cluster_flow
    defrag: yes
    use-mmap: yes
    tpacket-v3: yes
    ring-size: 2048
    checksum-checks: kernel
    bpf-filter: ""
```

关键点：

- `interface` 必须是现场镜像口、TAP 或测试网卡
- `cluster-type: cluster_flow` 可以尽量保证同一 TCP 流进入同一处理线程
- `checksum-checks: kernel` 适合实时抓包
- `bpf-filter` 可以临时缩小流量范围，例如只看 TCP 102：

```yaml
bpf-filter: "tcp port 102"
```

最小启动命令：

```bash
suricata -c suricata.yaml -i eth0 -l /tmp/suri-afpacket
```

## 5. 应用层协议：app-layer.protocols

本节重点确认 3 件事：

- 协议是否启用
- `detection-ports` 是否覆盖样本端口
- 对应协议 logger 是否能写入 `eve.json`

`app-layer.protocols` 控制协议识别和解析。

```yaml
app-layer:
  protocols:
    <protocol-name>:
      enabled: yes|no|detection-only
      detection-ports:
        dp: <port-list>
        sp: <port-list>
```

`enabled` 的含义：

| 值 | 说明 |
|----|------|
| `yes` | 启用协议识别、解析和日志 |
| `no` | 禁用协议 |
| `detection-only` | 只识别协议，不做完整解析 |

工控相关协议示例：

```yaml
app-layer:
  protocols:
    modbus:
      enabled: yes
      detection-ports:
        dp: 502
    dnp3:
      enabled: yes
      detection-ports:
        dp: 20000
    enip:
      enabled: yes
      detection-ports:
        dp: 44818
        sp: 44818
```

因为公司业务需求开发了 IEC 61850 MMS 插件，如果插件有自己的配置段，应重点确认：

```yaml
iec61850-mms:
      enabled: yes
      detection-ports:
        dp: 102
```

- 协议是否 `enabled: yes`
- `detection-ports` 是否覆盖 TCP 102 或测试样本端口
- EVE logger 是否已注册并启用

如果协议配置不确定，先用命令确认当前构建的 Suricata 程序支持的协议：

```bash
suricata --list-app-layer-protos
```

## 6. TCP 长连接和重组：stream / flow

IEC 61850 MMS 跑在 TCP 上，配置时只需先分清 `flow` 和 `stream` 的职责：

| 概念 | 通俗理解 | 主要关注 |
|------|----------|----------|
| `flow` | 一张连接登记表 | 五元组、方向、时间、包数、字节数、应用层协议识别结果 |
| `stream` | TCP 数据拼接器 | TCP 状态、乱序、重传、分片、重组、缺包、校验和 |

简单说：`flow` 说明 Suricata 看到了这条通信；`stream` 决定 TCP 里的 MMS 数据能不能被拼完整。没有 `iec61850_mms` 事件时，优先看 TCP 重组、缺包、单向流量、校验和和 logger 配置。

### stream

`stream` 负责 TCP 状态跟踪和数据重组。

```yaml
stream:
  memcap: 64 MiB
  checksum-validation: yes
  reassembly:
    memcap: 256 MiB
    depth: 1 MiB
```

关键点：

- `checksum-validation` 会影响校验和异常的数据包
- `reassembly.depth` 太小可能导致应用层数据不完整
- `reassembly.memcap` 不足可能导致重组失败或异常

### flow

`flow` 负责跟踪一条通信流。

```yaml
flow:
  memcap: 128 MiB
  hash-size: 65536
  prealloc: 10000
```

关键点：

- flow 表负责跟踪连接
- 现场长连接多时，需要关注 flow 相关 memcap
- `stats.log` 里 flow、tcp、app-layer 计数能帮助判断流量是否走到应用层

## 7. 规则文件：rule-files

入门阶段只需要知道规则从哪里加载：

```yaml
default-rule-path: /var/lib/suricata/rules

rule-files:
  - suricata.rules

classification-file: /etc/suricata/classification.config
reference-config-file: /etc/suricata/reference.config
```

临时测试规则时，可以不用改配置文件：

```bash
# 只加载这个规则文件
suricata -c suricata.yaml -S my-rules.rules -r test.pcap

# 在配置中的规则基础上额外加载
suricata -c suricata.yaml -s extra-rules.rules -r test.pcap
```

**规则不触发时，先看 `suricata.log` 是否有规则加载错误。**



## 8. 配置验证

配置改完后按这个顺序确认：

| 顺序 | 命令 | 目的 |
|------|------|------|
| 1 | `suricata -T` | 检查语法和配置可加载 |
| 2 | `suricata --dump-config` | 查看最终生效配置 |

修改配置后先跑语法检查：

```bash
suricata -T -c /etc/suricata/suricata.yaml
```

查看最终生效配置：

```bash
suricata --dump-config -c /etc/suricata/suricata.yaml
```



## 小结

本文只保留入门和工控插件实践所需的配置路径：

- `vars`：确认现场网段和规则变量
- `outputs`：确保 EVE JSON、运行日志、统计日志可用
- `af-packet`：覆盖现场旁路监听
- `app-layer.protocols`：确认协议识别和解析开关
- `stream` / `flow`：理解 TCP 长连接和重组对 MMS 的影响
- `rule-files`：知道规则从哪里加载
