## 新手入门Suricata实战课表（精简版）

### 第1课：快速上手（30分钟）

- 安装Suricata：`apt install suricata`
- 验证安装：`suricata --build-info` [6]
- 启动第一次监控：`sudo suricata -i eth0`



### 第2课：抓包分析实战（45分钟）

- 用现成的pcap文件：`suricata -r sample.pcap`
- 查看告警日志：`tail -f /var/log/suricata/fast.log`
- **实战演示**：分析一个真实恶意流量样本



### 第3课：写你的第一条规则（30分钟）

- 规则基本语法：`alert tcp any any -> any 80`
- 添加自定义规则到`/etc/suricata/rules/local.rules`
- **实战演示**：检测特定HTTP请求



### 第4课：实时监控网络（45分钟）

- 配置监控接口：编辑`suricata.yaml`中的interface
- 启动实时监控：`sudo suricata -i eth0`
- **实战演示**：监控真实网络流量并分析告警



### 第5课：日志分析技巧（30分钟）

- 理解EVE JSON日志格式
- 使用`jq`工具分析日志：`cat eve.json | jq`
- **实战演示**：从日志中发现攻击痕迹



### 第6课：性能调优基础（30分钟）

- 使用Rule Profiling分析性能 [1]
- 基础性能参数调整
- **实战演示**：优化一个慢规则



### 第7课：综合实战项目（60分钟）

- 搭建完整的检测环境
- 模拟攻击场景
- 从检测到响应的完整流程



## 每课结构（严格控制时间）：

- **理论讲解**：5分钟
- **动手实操**：20-50分钟
- **总结要点**：5分钟