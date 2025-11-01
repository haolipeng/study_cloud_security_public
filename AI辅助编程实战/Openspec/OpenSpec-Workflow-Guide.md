# OpenSpec 工作流程指南

> 💡 **快速参考**：本文档是 OpenSpec 工作流程的中文说明，方便在使用 OpenSpec 工具时快速查阅。

---

## 📋 目录

- [快速检查清单](#快速检查清单)
- [三阶段工作流程](#三阶段工作流程)
- [常用命令速查](#常用命令速查)
- [实战示例](#实战示例)
- [常见错误与解决](#常见错误与解决)

---

## 快速检查清单

每次开始工作前：

```bash
# 1. 查看现有能力（避免重复）
openspec list --specs

# 2. 查看进行中的变更（避免冲突）
openspec list

# 3. 查看项目约定
cat openspec/project.md

# 4. 搜索相关规格（如果需要）
rg -n "Requirement:|Scenario:" openspec/specs
```

---

## 三阶段工作流程

### Stage 1: 创建变更提案 📝

#### 什么时候需要创建提案？

✅ **需要创建提案：**
- 添加新功能或能力
- 进行破坏性变更（API、架构、数据结构）
- 性能优化（改变现有行为）
- 安全模式更新
- 架构或设计模式变更

❌ **跳过提案（直接修改）：**
- Bug 修复（恢复规格中的预期行为）
- 拼写、格式、注释修正
- 非破坏性的依赖更新
- 配置文件调整
- 为现有行为补充测试

#### 创建提案的完整流程

**步骤 1：探索现有状态**

```bash
# 查看所有现有能力
openspec list --specs

# 查看进行中的变更
openspec list

# 查看具体规格详情
openspec show <spec-name>

# 全文搜索（查找相关需求）
rg -n "Requirement:|Scenario:" openspec/specs
```

**步骤 2：选择唯一的 change-id**

命名规则：
- 使用 kebab-case（小写+连字符）
- 动词开头：`add-`, `update-`, `remove-`, `refactor-`
- 简短且描述性强
- 确保唯一性（如重复则加 `-2`, `-3` 等）

示例：
```
✅ 好的命名：
- add-tcp-state-machine
- update-session-tracking
- refactor-policy-map-structure
- remove-legacy-protocol-support

❌ 不好的命名：
- tcp-state           # 缺少动词
- AddTCPState         # 不是 kebab-case
- update_map          # 不是连字符
```

**步骤 3：创建目录结构**

```bash
CHANGE_ID="add-tcp-state-machine"

# 创建变更目录
mkdir -p openspec/changes/$CHANGE_ID/specs/session-tracking

# 创建必需文件
touch openspec/changes/$CHANGE_ID/proposal.md
touch openspec/changes/$CHANGE_ID/tasks.md
touch openspec/changes/$CHANGE_ID/specs/session-tracking/spec.md

# 可选：复杂变更需要 design.md
touch openspec/changes/$CHANGE_ID/design.md
```

目录结构：
```
openspec/changes/add-tcp-state-machine/
├── proposal.md              # 提案说明（必需）
├── tasks.md                 # 实施任务清单（必需）
├── design.md                # 技术设计文档（可选）
└── specs/                   # 规格增量（必需）
    └── session-tracking/
        └── spec.md          # 具体的增量变更
```

**步骤 4：编写 proposal.md**

```markdown
# Add TCP State Machine Tracking

## Why
当前的会话跟踪缺乏 TCP 状态机支持，无法准确跟踪连接的生命周期，导致以下问题：
1. 无法区分半开连接
2. 无法检测异常的 FIN/RST 包
3. 会话超时策略不够精确

## What Changes
- 添加 TCP 11 状态跟踪（CLOSED, LISTEN, SYN_SENT, SYN_RECV, ESTABLISHED, FIN_WAIT1, FIN_WAIT2, CLOSE_WAIT, CLOSING, LAST_ACK, TIME_WAIT）
- 实现状态转换逻辑（基于 TCP flags）
- 添加状态超时策略（不同状态不同超时时间）
- 更新 session_map 数据结构，包含状态字段
- **BREAKING**: 修改 `struct tcp_session` 结构（添加 `state` 字段）

## Impact
- **受影响的规格**：`session-tracking`
- **受影响的代码**：
  - `src/bpf/session_tracking.bpf.c` - eBPF 程序
  - `src/user/session_viewer.c` - 用户态工具
  - `tests/test_tcp_statemachine.sh` - 测试脚本
- **向后兼容性**：需要重新加载所有 eBPF 程序（数据结构变更）
- **性能影响**：每包增加约 20-30 条指令（状态判断逻辑）
```

**步骤 5：编写 tasks.md**

```markdown
# Implementation Tasks

## 1. Data Structure Changes
- [ ] 1.1 定义 TCP 状态枚举（11 个状态）
- [ ] 1.2 更新 `struct tcp_session` 添加 `state` 字段
- [ ] 1.3 添加状态超时时间常量表

## 2. eBPF Program Implementation
- [ ] 2.1 实现 `get_tcp_flags()` 辅助函数
- [ ] 2.2 实现 `tcp_state_transition()` 状态机逻辑
- [ ] 2.3 在 ingress hook 集成状态跟踪
- [ ] 2.4 添加状态超时检查逻辑

## 3. User-Space Tools
- [ ] 3.1 更新 `session_viewer` 显示状态信息
- [ ] 3.2 添加按状态过滤功能

## 4. Testing
- [ ] 4.1 编写状态转换单元测试
- [ ] 4.2 添加集成测试（完整 TCP 握手）
- [ ] 4.3 测试异常情况（RST/超时）
- [ ] 4.4 性能基准测试

## 5. Documentation
- [ ] 5.1 更新 API 文档
- [ ] 5.2 添加状态机流程图
- [ ] 5.3 更新用户指南
```

**步骤 6：编写规格增量 (spec delta)**

`openspec/changes/add-tcp-state-machine/specs/session-tracking/spec.md`:

```markdown
## ADDED Requirements

### Requirement: TCP State Machine Tracking
The system SHALL track TCP connection states according to RFC 793 state machine.

#### Scenario: SYN packet creates new session
- **WHEN** a SYN packet is received
- **THEN** create a new session entry with state = SYN_SENT
- **AND** set initial timeout to 30 seconds

#### Scenario: SYN-ACK transitions to ESTABLISHED
- **WHEN** a SYN-ACK packet is received for an existing SYN_SENT session
- **THEN** transition session state to ESTABLISHED
- **AND** update timeout to 7200 seconds (2 hours)

#### Scenario: FIN initiates graceful close
- **WHEN** a FIN packet is received for an ESTABLISHED session
- **THEN** transition session state to FIN_WAIT1
- **AND** update timeout to 60 seconds

#### Scenario: RST forces immediate close
- **WHEN** a RST packet is received for any active session
- **THEN** transition session state to CLOSED
- **AND** mark session for immediate deletion

### Requirement: State-Based Timeout Policy
The system SHALL apply different timeout values based on TCP state.

#### Scenario: Timeout values per state
- **WHEN** checking session timeout
- **THEN** apply the following timeout values:
  - SYN_SENT / SYN_RECV: 30 seconds
  - ESTABLISHED: 7200 seconds (2 hours)
  - FIN_WAIT1 / FIN_WAIT2: 60 seconds
  - CLOSE_WAIT: 60 seconds
  - TIME_WAIT: 120 seconds (2 minutes)
  - All other states: 30 seconds

## MODIFIED Requirements

### Requirement: TCP Session Data Structure
The system SHALL store TCP connection information with state tracking.

**Previous structure:**
```c
struct tcp_session {
    __u64 last_seen;
    __u32 packet_count;
    __u32 byte_count;
};
```

**New structure:**
```c
struct tcp_session {
    __u64 last_seen;
    __u32 packet_count;
    __u32 byte_count;
    __u8  state;        // NEW: TCP state (enum tcp_state)
    __u8  flags;        // NEW: Additional flags
    __u16 reserved;     // Padding for alignment
};
```

#### Scenario: Session creation with initial state
- **WHEN** a new TCP session is created
- **THEN** initialize all fields including state = SYN_SENT
- **AND** set last_seen to current timestamp
- **AND** set packet_count = 1, byte_count = packet_size

#### Scenario: Session lookup returns state info
- **WHEN** querying an existing session
- **THEN** return complete session information including current state
- **AND** allow filtering by state in user-space tools
```

**步骤 7：可选 - 编写 design.md（复杂变更需要）**

何时需要 `design.md`：
- ✅ 跨多个服务/模块的变更
- ✅ 引入新的外部依赖
- ✅ 重大数据模型变更
- ✅ 安全、性能或迁移复杂性
- ✅ 需要技术决策讨论的模糊需求

`openspec/changes/add-tcp-state-machine/design.md` 示例：

```markdown
## Context
当前会话跟踪仅记录最后活动时间和统计信息，无法感知 TCP 连接的实际状态。这导致：
1. 半开连接无法被清理
2. 异常连接（RST）无法被快速检测
3. 超时策略过于粗糙（所有连接统一超时）

## Goals / Non-Goals

**Goals:**
- 实现 RFC 793 标准的 TCP 11 状态机
- 支持状态感知的超时策略
- 提供状态统计和可视化

**Non-Goals:**
- 不实现窗口大小跟踪（暂不需要）
- 不处理 TCP 选项（SACK, Window Scaling 等）
- 不支持 TCP 时间戳选项

## Decisions

### Decision 1: 使用完整 11 状态机
**选择**：实现完整的 11 个 TCP 状态，而非简化版（3-5 个状态）

**理由**：
- 精确跟踪有助于调试和异常检测
- 状态转换逻辑清晰，符合 RFC 标准
- 后续扩展（如窗口跟踪）需要完整状态信息

**替代方案**：
- 简化为 3 状态（SYN, ESTABLISHED, CLOSE）- 放弃，粒度太粗
- 使用 conntrack 内核模块 - 放弃，增加外部依赖

### Decision 2: eBPF 内核态实现状态机
**选择**：在 eBPF 程序中直接实现状态转换逻辑

**理由**：
- 零延迟（无需用户态查询）
- 数据包级别的即时状态更新
- 减少用户态-内核态通信开销

**替代方案**：
- 用户态轮询更新状态 - 放弃，延迟太高

### Decision 3: 状态存储在 LRU_HASH Map
**选择**：在现有 `tcp_map` (LRU_HASH) 中添加 `state` 字段

**理由**：
- 避免维护两个 map（减少 map 查找开销）
- LRU 自动淘汰旧连接，适合状态跟踪
- 结构体对齐后仅增加 4 字节（state + flags + reserved）

**替代方案**：
- 单独的 state_map - 放弃，增加查找次数

## Risks / Trade-offs

### Risk: eBPF 指令数增加
- **风险**：状态机逻辑增加约 100-150 条 eBPF 指令，可能接近 verifier 限制
- **缓解**：使用内联函数优化，必要时拆分为 tail call

### Risk: 数据结构变更破坏兼容性
- **风险**：修改 `struct tcp_session` 导致旧程序无法读取新数据
- **缓解**：需要完全重启所有 eBPF 程序，清空 map

### Trade-off: 精度 vs 复杂度
- **选择**：11 状态提供高精度，但增加代码复杂度
- **权衡**：可维护性换取准确性（值得）

## Migration Plan

### Phase 1: 部署前准备
1. 备份现有配置和统计数据
2. 编译新版本 eBPF 程序
3. 准备回滚脚本

### Phase 2: 部署
1. 停止现有 eBPF 程序（`kill` loader 进程）
2. 清空旧 map 数据（`rm -rf /sys/fs/bpf/tcp_map`）
3. 加载新 eBPF 程序
4. 验证状态跟踪正常工作

### Phase 3: 验证
1. 运行集成测试（完整 TCP 握手）
2. 检查状态统计是否正常
3. 观察性能指标（延迟、CPU 使用率）

### Rollback Plan
如果出现问题：
1. 停止新 eBPF 程序
2. 恢复旧版本程序和 map 定义
3. 重新加载
4. 预计回滚时间：<2 分钟

## Open Questions

1. ❓ 是否需要导出状态转换事件到 ring buffer？（用于实时监控）
   - **初步决定**：暂不需要，先看需求

2. ❓ TIME_WAIT 状态的会话是否需要特殊清理逻辑？
   - **初步决定**：使用标准 120 秒超时即可

3. ❓ 是否支持 IPv6 的状态跟踪？
   - **初步决定**：暂不支持，留待后续扩展
```

**步骤 8：验证提案**

```bash
# 运行严格验证
openspec validate add-tcp-state-machine --strict

# 如果有错误，查看详细输出
openspec show add-tcp-state-machine --json --deltas-only

# 修复所有错误后重新验证
```

**步骤 9：请求批准**

⚠️ **重要**：在提案被审核批准前，**不要开始实施**！

---

### Stage 2: 实施变更 🔨

批准后，按照以下步骤逐步完成：

#### 实施清单（作为 TODO 跟踪）

1. ✅ **阅读 proposal.md** - 理解要构建什么
2. ✅ **阅读 design.md**（如果存在）- 审查技术决策
3. ✅ **阅读 tasks.md** - 获取实施任务清单
4. ✅ **按顺序实施任务** - 逐个完成 `tasks.md` 中的任务
5. ✅ **确认完成** - 确保 `tasks.md` 中的每个项目都已完成
6. ✅ **更新清单** - 将所有任务标记为 `- [x]`（完成后再标记，不要提前）
7. ✅ **验证功能** - 运行测试，确保所有功能正常工作

#### 实施过程中的最佳实践

```bash
# 1. 创建特性分支（可选）
git checkout -b feature/add-tcp-state-machine

# 2. 实施第一个任务
# ... 编写代码 ...

# 3. 提交代码（遵循项目约定）
git add src/bpf/session_tracking.bpf.c
git commit -m "Add TCP state machine data structures

- Define 11 TCP states enum
- Update tcp_session struct with state field
- Add state timeout constants

🤖 Generated with Claude Code
Co-Authored-By: Claude <noreply@anthropic.com>"

# 4. 继续下一个任务...
```

#### 更新 tasks.md 示例

```markdown
## 1. Data Structure Changes
- [x] 1.1 定义 TCP 状态枚举（11 个状态）
- [x] 1.2 更新 `struct tcp_session` 添加 `state` 字段
- [x] 1.3 添加状态超时时间常量表

## 2. eBPF Program Implementation
- [x] 2.1 实现 `get_tcp_flags()` 辅助函数
- [x] 2.2 实现 `tcp_state_transition()` 状态机逻辑
- [x] 2.3 在 ingress hook 集成状态跟踪
- [ ] 2.4 添加状态超时检查逻辑  ← 当前正在进行

## 3. User-Space Tools
- [ ] 3.1 更新 `session_viewer` 显示状态信息
- [ ] 3.2 添加按状态过滤功能
```

---

### Stage 3: 归档变更 📦

功能部署到生产环境后：

```bash
# 1. 归档变更（将 changes/ 移动到 archive/）
openspec archive add-tcp-state-machine --yes

# 这会执行以下操作：
# - 移动 changes/add-tcp-state-machine/ → changes/archive/2025-10-29-add-tcp-state-machine/
# - 将 spec deltas 合并到 specs/session-tracking/spec.md
# - 更新规格文件为最终状态

# 2. 验证归档后的规格
openspec validate --strict

# 3. 提交归档变更
git add openspec/
git commit -m "Archive add-tcp-state-machine change

Feature successfully deployed to production.

🤖 Generated with Claude Code
Co-Authored-By: Claude <noreply@anthropic.com>"
```

归档选项：
- `--yes` / `-y` - 跳过确认提示（自动化脚本使用）
- `--skip-specs` - 仅归档，不更新 specs（工具性变更）

---

## 常用命令速查

### 探索命令

```bash
# 查看所有现有能力（规格）
openspec list --specs
openspec spec list --long         # 详细列表

# 查看进行中的变更
openspec list
openspec change list --json       # JSON 格式（脚本使用）

# 查看具体项目详情
openspec show <spec-name>         # 查看规格
openspec show <change-id>         # 查看变更
openspec show                     # 交互式选择

# 查看规格的 JSON 格式（过滤使用）
openspec show <spec-name> --type spec --json

# 查看变更的 delta 详情
openspec show <change-id> --json --deltas-only
```

### 验证命令

```bash
# 验证单个变更（严格模式）
openspec validate <change-id> --strict

# 验证所有（交互式批量验证）
openspec validate

# 验证特定规格
openspec validate <spec-name> --type spec

# 非交互式验证
openspec validate --no-interactive
```

### 归档命令

```bash
# 归档变更（交互式确认）
openspec archive <change-id>

# 归档变更（跳过确认）
openspec archive <change-id> --yes

# 仅归档，不更新规格（工具性变更）
openspec archive <change-id> --skip-specs --yes

# ⚠️ 注意：始终显式传递 change-id，不要依赖自动检测
```

### 项目管理命令

```bash
# 初始化 OpenSpec（新项目）
openspec init [path]

# 更新指令文件（升级 OpenSpec）
openspec update [path]
```

### 搜索命令

```bash
# 全文搜索规格
rg -n "Requirement:|Scenario:" openspec/specs

# 搜索变更
rg -n "^#|Requirement:" openspec/changes

# 搜索特定关键词
rg -n "TCP state" openspec/
```

---

## 实战示例

### 示例 1：添加新功能（完整流程）

假设我们要添加 "DDoS 防护" 功能：

```bash
# Step 1: 检查是否已存在
openspec list --specs | grep -i ddos
openspec list | grep -i ddos

# Step 2: 选择 change-id
CHANGE_ID="add-ddos-protection"

# Step 3: 创建目录结构
mkdir -p openspec/changes/$CHANGE_ID/specs/packet-filtering

# Step 4: 创建文件
cat > openspec/changes/$CHANGE_ID/proposal.md << 'EOF'
# Add DDoS Protection

## Why
当前系统缺乏对 DDoS 攻击的防护，容易受到 SYN Flood 等攻击。

## What Changes
- 添加 SYN Flood 检测逻辑
- 实现源 IP 频率限制（rate limiting）
- 添加 DDoS 统计和告警
- **BREAKING**: 添加新的 BPF map `ddos_state_map`

## Impact
- 受影响的规格: packet-filtering
- 受影响的代码: src/bpf/microsegment.bpf.c
- 性能影响: 每包增加约 10-15 条指令
EOF

cat > openspec/changes/$CHANGE_ID/tasks.md << 'EOF'
# Implementation Tasks

## 1. Data Structures
- [ ] 1.1 定义 ddos_state 结构体
- [ ] 1.2 创建 ddos_state_map (LRU_HASH)
- [ ] 1.3 定义速率限制阈值常量

## 2. Detection Logic
- [ ] 2.1 实现 SYN 包计数逻辑
- [ ] 2.2 实现时间窗口滑动
- [ ] 2.3 实现阈值检查和丢包逻辑

## 3. Statistics
- [ ] 3.1 添加 DDoS 事件计数器
- [ ] 3.2 导出统计到 ring buffer

## 4. Testing
- [ ] 4.1 SYN Flood 模拟测试
- [ ] 4.2 正常流量不受影响测试
- [ ] 4.3 性能影响测试
EOF

cat > openspec/changes/$CHANGE_ID/specs/packet-filtering/spec.md << 'EOF'
## ADDED Requirements

### Requirement: SYN Flood Protection
The system SHALL detect and mitigate SYN Flood attacks.

#### Scenario: Detect SYN flood from single source
- **WHEN** receiving >100 SYN packets/second from a single IP
- **THEN** drop subsequent SYN packets from that IP
- **AND** log DDoS event

#### Scenario: Normal traffic unaffected
- **WHEN** SYN packet rate is <100/second
- **THEN** allow all packets to pass
- **AND** not trigger any rate limiting

### Requirement: Rate Limiting State Management
The system SHALL track per-source-IP SYN packet rates.

#### Scenario: State expiration
- **WHEN** no SYN packets received from an IP for 60 seconds
- **THEN** remove rate limiting state for that IP
- **AND** allow future connections normally
EOF

# Step 5: 验证
openspec validate $CHANGE_ID --strict

# Step 6: 查看提案（确认格式正确）
openspec show $CHANGE_ID

# Step 7: 等待批准...
```

### 示例 2：修改现有功能

假设要修改现有的会话超时逻辑：

```bash
CHANGE_ID="update-session-timeout"

# 创建结构
mkdir -p openspec/changes/$CHANGE_ID/specs/session-tracking

# 创建 spec delta
cat > openspec/changes/$CHANGE_ID/specs/session-tracking/spec.md << 'EOF'
## MODIFIED Requirements

### Requirement: Session Timeout Policy
The system SHALL automatically remove inactive sessions after a timeout period.

**修改说明**：从固定 300 秒超时改为根据协议和状态动态超时。

#### Scenario: TCP ESTABLISHED timeout
- **WHEN** a TCP session in ESTABLISHED state is inactive for 7200 seconds (2 hours)
- **THEN** remove the session from session_map
- **AND** log session expiration event

#### Scenario: UDP timeout
- **WHEN** a UDP session is inactive for 180 seconds (3 minutes)
- **THEN** remove the session from session_map

#### Scenario: TCP handshake timeout
- **WHEN** a TCP session in SYN_SENT state is inactive for 30 seconds
- **THEN** remove the session from session_map
- **AND** mark as potential half-open connection
EOF

openspec validate $CHANGE_ID --strict
```

### 示例 3：删除废弃功能

```bash
CHANGE_ID="remove-legacy-ipv4-only"

cat > openspec/changes/$CHANGE_ID/specs/network-stack/spec.md << 'EOF'
## REMOVED Requirements

### Requirement: IPv4-Only Mode
**Reason**: IPv6 support is now mandatory, IPv4-only mode is deprecated.

**Migration**: Update all configurations to enable dual-stack (IPv4 + IPv6).

**Original Requirement**:
The system SHALL support IPv4-only deployment mode.

#### Scenario: IPv4-only filtering (REMOVED)
- This scenario is removed as IPv6 is now required

**Migration Path**:
1. Ensure kernel supports IPv6
2. Update eBPF programs to handle both address families
3. Test dual-stack configuration
4. Remove IPv4-only config options
EOF
```

---

## 常见错误与解决

### 错误 1：Scenario 格式不正确

**错误信息**：
```
Error: Requirement must have at least one scenario
```

**原因**：Scenario 格式不正确，必须使用 `#### Scenario:` 格式。

**错误示例**：
```markdown
- **Scenario**: User login        ❌
**Scenario:** User login          ❌
### Scenario: User login          ❌
```

**正确示例**：
```markdown
#### Scenario: User login success ✅
- **WHEN** valid credentials provided
- **THEN** return JWT token
```

**调试方法**：
```bash
# 查看解析的 scenarios
openspec show <change-id> --json --deltas-only | jq '.deltas[].scenarios'
```

### 错误 2：Change 缺少 delta

**错误信息**：
```
Error: Change must have at least one delta
```

**原因**：
- `changes/<id>/specs/` 目录为空
- 或者 spec.md 文件没有操作前缀（`## ADDED`, `## MODIFIED`, `## REMOVED`）

**解决方法**：
```bash
# 检查目录结构
ls -la openspec/changes/<change-id>/specs/

# 确保至少有一个 spec.md 文件
cat openspec/changes/<change-id>/specs/*/spec.md

# 确保文件有正确的操作头
grep "^## ADDED\|^## MODIFIED\|^## REMOVED" openspec/changes/<change-id>/specs/*/spec.md
```

### 错误 3：MODIFIED Requirement 不完整

**错误场景**：归档时发现原需求的某些内容丢失。

**原因**：`MODIFIED` 操作需要包含完整的新需求内容，不是部分更新。

**错误做法**：
```markdown
## MODIFIED Requirements
### Requirement: User Authentication
Added support for 2FA.
```

**正确做法**：
```markdown
## MODIFIED Requirements
### Requirement: User Authentication
The system SHALL authenticate users via username/password AND two-factor authentication (2FA).

#### Scenario: Login with 2FA
- **WHEN** valid credentials and OTP provided
- **THEN** issue JWT token

#### Scenario: Login without 2FA fails
- **WHEN** valid credentials but missing OTP
- **THEN** reject login with error
```

### 错误 4：Validation 静默失败

**现象**：`openspec validate` 通过，但 scenario 没有被解析。

**调试步骤**：
```bash
# 1. 查看 JSON 输出
openspec show <change-id> --json --deltas-only

# 2. 检查 scenarios 数组是否为空
openspec show <change-id> --json --deltas-only | jq '.deltas[].scenarios | length'

# 3. 检查格式
grep -n "^#### Scenario:" openspec/changes/<change-id>/specs/*/spec.md

# 4. 确保没有额外的空格或特殊字符
cat -A openspec/changes/<change-id>/specs/*/spec.md | grep Scenario
```

### 错误 5：归档时 spec 冲突

**错误信息**：
```
Error: Cannot find requirement "XXX" in spec
```

**原因**：`MODIFIED` 或 `REMOVED` 的需求名称与 spec 中不匹配。

**解决方法**：
```bash
# 1. 查看当前 spec 中的需求名称
openspec show <spec-name> --type spec | grep "^### Requirement:"

# 2. 确保 delta 中的名称完全匹配（空格不敏感）
grep "^### Requirement:" openspec/changes/<change-id>/specs/<capability>/spec.md

# 3. 使用 RENAMED 操作（如果改名了）
```

---

## 最佳实践

### 1. 保持简单

- 默认实现 <100 行新代码
- 单文件实现，直到证明不够用
- 避免不必要的框架和抽象
- 选择成熟、无聊的技术模式

### 2. 明确参考

在文档中使用清晰的引用：
- 代码位置：`src/bpf/hello.bpf.c:42`
- 规格引用：`specs/auth/spec.md`
- 相关变更：`changes/add-tcp-state-machine/`

### 3. 能力命名

- 使用动词-名词：`user-auth`, `packet-filtering`, `session-tracking`
- 单一职责（不要用 "AND" 连接两个目的）
- 10 分钟可理解性规则

### 4. Change ID 命名

- kebab-case，简短描述性
- 动词开头：`add-`, `update-`, `remove-`, `refactor-`
- 确保唯一性

### 5. 复杂度触发器

只在以下情况添加复杂度：
- 性能数据证明当前方案太慢
- 具体的规模需求（>1000 用户，>100MB 数据）
- 多个已证明的用例需要抽象

---

## 工具选择指南

| 任务 | 工具 | 原因 |
|------|------|------|
| 查找文件模式 | Glob | 快速模式匹配 |
| 搜索代码内容 | Grep/rg | 优化的正则搜索 |
| 读取特定文件 | Read | 直接文件访问 |
| 探索未知范围 | Task | 多步骤调查 |

---

## 快速参考卡片

### 阶段指示器
- `openspec/changes/` - 提议中，尚未构建
- `openspec/specs/` - 已构建并部署
- `openspec/changes/archive/` - 已完成的变更

### 文件用途
- `proposal.md` - 为什么和是什么
- `tasks.md` - 实施步骤
- `design.md` - 技术决策（可选）
- `spec.md` - 需求和行为

### CLI 核心命令
```bash
openspec list              # 进行中的变更？
openspec show [item]       # 查看详情
openspec validate --strict # 是否正确？
openspec archive <change-id> --yes  # 标记完成
```

---

## 记住

> **规格是真理。变更是提案。保持它们同步。**

- ✅ 在创建提案前先搜索现有规格
- ✅ 每个 Requirement 至少要有一个 Scenario
- ✅ 使用 `#### Scenario:` 格式（4 个井号）
- ✅ MODIFIED 需要包含完整的新内容
- ✅ 在实施前等待批准
- ✅ 实施完成后归档变更

---

## 相关文档

- `openspec/AGENTS.md` - 完整的 OpenSpec 指令（英文）
- `openspec/project.md` - 项目上下文和约定
- 官方文档：https://github.com/your-org/openspec

---

**最后更新**：2025-10-29
**版本**：1.0
