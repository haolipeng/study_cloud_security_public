# CCPM 使用指南

本文档基于实际使用经验，总结了 CCPM (Claude Code Project Manager) 工具的完整使用流程和方法。

## 目录

1. [概述](#概述)
2. [前置准备](#前置准备)
3. [核心工作流](#核心工作流)
4. [实战案例](#实战案例)
5. [常见问题](#常见问题)
6. [最佳实践](#最佳实践)

---

## 概述

### 什么是 CCPM？

CCPM (Claude Code Project Manager) 是一个基于 GitHub Issues 和 Git Worktrees 的项目管理系统，旨在实现代码的可追溯性和规范化的开发流程。

**核心理念**: 每一行代码都应该能够追溯到相应的规格说明。

**项目地址**: https://github.com/automazeio/ccpm

### 核心概念

- **PRD (Product Requirements Document)**: 产品需求文档，详细描述功能的背景、目标、需求和设计
- **Epic**: 史诗，将 PRD 分解为可执行的技术实现计划
- **Task**: 任务，Epic 的具体执行单元，每个任务对应一个可独立完成的工作项
- **GitHub Issue**: 用于跟踪 PRD、Epic 和 Task 的进度
- **Git Worktree**: 独立的工作目录，用于并行开发

### CCPM 工作流

```
PRD (需求文档)
  ↓ /pm:prd-parse
Epic (技术计划)
  ↓ /pm:epic-decompose
Tasks (具体任务)
  ↓ /pm:epic-sync
GitHub Issues (问题跟踪)
  ↓ /pm:task-work
Git Worktree (代码实现)
  ↓ commit & PR
Code Review & Merge
```

---

## 前置准备

### 1. GitHub 认证

CCPM 使用 GitHub CLI (`gh`) 进行 GitHub 操作，需要先进行认证：

```bash
# 使用 Personal Access Token 认证
gh auth login

# 验证认证状态
gh auth status
```

**预期输出**:
```
✓ Logged in to github.com as <your-username> (oauth_token)
```

### 2. 配置检查

确认 CCPM 已正确配置：

```bash
# 检查 ccpm.config 文件
cat .claude/ccpm.config
```

该文件会自动检测 GitHub 仓库：
```bash
# Extract GitHub repository from git remote
REPO=$(git remote get-url origin | sed 's/.*github.com[:/]\(.*\)\.git/\1/')
```

### 3. 目录结构

CCPM 使用以下目录结构：

```
.claude/
├── ccpm.config          # CCPM 配置文件
├── settings.local.json  # 本地设置
├── prds/                # PRD 文档目录
│   ├── feature-1.md
│   └── feature-2.md
├── epics/               # Epic 目录
│   ├── feature-1/
│   │   ├── epic.md      # Epic 主文档
│   │   ├── 001.md       # Task 1
│   │   ├── 002.md       # Task 2
│   │   └── ...
│   └── feature-2/
├── commands/            # 自定义命令
└── rules/               # 项目规则
```

---

## 核心工作流

### 步骤 1: 创建 PRD

**命令**: `/pm:prd-new <feature-name>`

**作用**: 创建一个新的产品需求文档。

**示例**:
```bash
/pm:prd-new tcp-state-machine
```

**生成的文件**: `.claude/prds/tcp-state-machine.md`

**文件结构**:

```yaml
---
name: tcp-state-machine
description: 功能简短描述
status: draft | in_progress | completed | archived
created: 2025-11-15T01:40:19Z
completed: 2025-11-15T01:40:19Z  # 如果已完成
---

# PRD: TCP State Machine

## Executive Summary
高层概述：为什么需要这个功能，价值是什么

## Problem Statement
当前存在什么问题，为什么需要解决

## User Stories
作为 <角色>，我希望 <功能>，以便 <价值>

## Functional Requirements
1. 功能需求 1
2. 功能需求 2
...

## Non-Functional Requirements
1. 性能要求
2. 安全要求
...

## Technical Design
技术设计方案

## Implementation Status
当前实现状态（如果已完成）

## Success Criteria
成功标准和验收条件
```

**注意事项**:
- 如果代码已经实现，可以设置 `status: completed`
- 可以在 PRD 中链接到相关的 git commit

### 步骤 2: 将 PRD 解析为 Epic

**命令**: `/pm:prd-parse <feature-name>`

**作用**: 读取 PRD 并创建对应的 Epic 文档。

**示例**:
```bash
/pm:prd-parse tcp-state-machine
```

**生成的文件**: `.claude/epics/tcp-state-machine/epic.md`

**文件结构**:
```yaml
---
name: tcp-state-machine
status: draft | in_progress | completed
created: 2025-11-15T01:42:55Z
completed: 2025-11-15T01:42:55Z
progress: 100%
prd: .claude/prds/tcp-state-machine.md
github: [GitHub Issue URL]
commit: <git-commit-hash>  # 如果已完成
---

# Epic: TCP State Machine Implementation

## Overview
技术实现概述

## Architecture Decisions
关键架构决策

## Technical Approach
技术实现方法

## Implementation Strategy
实现策略和阶段划分

## Risk Mitigation
风险识别和缓解措施

## Dependencies
依赖项

## Success Criteria
成功标准
```

### 步骤 3: 将 Epic 分解为 Tasks

**命令**: `/pm:epic-decompose <epic-name>`

**作用**: 将 Epic 分解为多个可执行的 Task。

**示例**:
```bash
/pm:epic-decompose tcp-state-machine
```

**生成的文件**:
```
.claude/epics/tcp-state-machine/
├── epic.md
├── 001.md  # Task 1
├── 002.md  # Task 2
├── 003.md  # Task 3
├── 004.md  # Task 4
└── 005.md  # Task 5
```

**Task 文件结构**:
```yaml
---
name: Design TCP State Machine Architecture
status: pending | in_progress | completed
created: 2025-11-15T01:54:55Z
completed: 2025-11-15T01:54:55Z
github: [GitHub Issue URL]
depends_on: []  # 依赖的其他任务
parallel: true | false  # 是否可以并行执行
conflicts_with: []  # 冲突的任务
---

# Task: Design TCP State Machine Architecture

## Description
详细的任务描述

## Acceptance Criteria
- [ ] 验收标准 1
- [ ] 验收标准 2

## Technical Details
技术实现细节

## Dependencies
依赖项

## Effort Estimate
- Size: S | M | L
- Hours: X hours
- Parallel: true | false

## Definition of Done
- [ ] 完成条件 1
- [ ] 完成条件 2
```

**分解建议**:
- 每个 Task 应该是独立可完成的工作单元
- Task 之间要明确依赖关系
- 标注哪些 Task 可以并行执行
- 估算每个 Task 的工作量

### 步骤 4: 同步到 GitHub，也可一键同步

**命令**: `/pm:epic-sync <epic-name>`

**作用**: 将 Epic 和 Tasks 同步到 GitHub Issues。

**示例**:
```bash
/pm:epic-sync tcp-state-machine
```

**执行流程**:

1. **创建 Epic Issue**:
   - 读取 `epic.md` 的内容
   - 在 GitHub 创建 Epic Issue
   - 将 Issue URL 写回 `epic.md` 的 frontmatter

2. **创建 Task Issues**:
   - 遍历所有 Task 文件（001.md, 002.md, ...）
   - 为每个 Task 创建 GitHub Issue
   - 将 Issue URL 写回 Task 文件的 frontmatter
   - 建立 Task 与 Epic 的关联关系



**一键同步**

```
/pm:epic-oneshot tcp-state-machine
```



**GitHub Issue 示例**:

Epic Issue #2:
```
Title: Epic: TCP State Machine Implementation
Body: [epic.md 的内容]
Labels: epic, enhancement
```

Task Issue #3:
```
Title: Design TCP State Machine Architecture
Body: [001.md 的内容]
Labels: task, epic:tcp-state-machine
Related to: #2 (Epic)
```

**注意事项**:
- 确保网络连接正常
- 首次使用可能需要在仓库中创建相关 labels
- 如果遇到网络错误，可以稍后重试

### 步骤 5: 执行任务（可选）

如果是新功能开发，可以使用以下命令：

**命令**: `/pm:task-work <epic-name> <task-number>`

**作用**: 为指定任务创建 Git Worktree 并开始开发。

**示例**:
```bash
/pm:task-work tcp-state-machine 001
```

这会：
1. 创建新的 Git Worktree
2. 切换到任务分支
3. 更新 Task 状态为 `in_progress`

---

## 实战案例

### 案例：TCP 状态机功能

以下是一个完整的实战案例，展示如何为已完成的功能创建文档。

#### 背景

- **功能**: 实现完整的 RFC 793 TCP 状态机
- **Commit**: `aada69a42de62c45b86e35c2ebc7a8e82cc3c23b`
- **状态**: 已完成实现和测试

#### 步骤 1: 创建 PRD

```bash
/pm:prd-new tcp-state-machine
```

创建了 `.claude/prds/tcp-state-machine.md`，包含：
- 执行摘要：为什么需要完整的 TCP 状态机
- 问题陈述：当前简单的 FIN/RST 检测的局限性
- 用户故事：内核开发者、网络管理员、安全分析师的需求
- 功能需求：11 种 TCP 状态、所有状态转换、双向支持等
- 技术设计：头文件结构、TC/XDP 集成方案
- 实现状态：链接到 commit aada69a

#### 步骤 2: 创建 Epic

```bash
/pm:prd-parse tcp-state-machine
```

创建了 `.claude/epics/tcp-state-machine/epic.md`，包含：
- 架构决策：
  - 决策 1: 头文件实现（可重用、零开销）
  - 决策 2: 分离入站/出站转换（适配 TC 限制）
  - 决策 3: 热路径集成（最小性能影响）
  - 决策 4: 保持向后兼容（无破坏性更改）
- 技术方案：eBPF 组件、编译配置、测试策略
- 实现阶段：设计、头文件、TC 集成、XDP 集成、测试

#### 步骤 3: 分解任务

```bash
/pm:epic-decompose tcp-state-machine
```

创建了 5 个任务文件：

**Task 001**: Design TCP State Machine Architecture
- 并行执行: 是
- 工作量: 2 小时
- 状态: 已完成

**Task 002**: Implement TCP State Machine Header File
- 依赖: Task 001
- 并行执行: 否
- 工作量: 4 小时
- 文件: `src/bpf/headers/tcp_state_machine.h` (265 行)
- 核心函数:
  - `get_tcp_flags()`
  - `tcp_state_transition_inbound()`
  - `tcp_state_transition_outbound()`
  - `is_tcp_state_closing()`
  - `is_tcp_state_established()`

**Task 003**: Integrate TCP State Machine with TC Program
- 依赖: Task 002
- 并行执行: 否
- 工作量: 3 小时
- 文件修改: `src/bpf/tc_microsegment.bpf.c` (+87 行)
- 关键更改:
  - 新增 `update_tcp_state()` 函数
  - 修改 `create_session()` 签名
  - 更新热路径状态追踪

**Task 004**: Integrate TCP State Machine with XDP Program
- 依赖: Task 002
- 并行执行: 是（与 Task 003 不同文件）
- 工作量: 3 小时
- 文件修改: `src/bpf/xdp_microsegment.bpf.c` (+69 行)
- 限制: 仅支持入站流量

**Task 005**: Testing and Validation
- 依赖: Task 003, Task 004
- 并行执行: 否
- 工作量: 2 小时
- 测试场景:
  - 三次握手
  - 主动关闭
  - 被动关闭
  - 同时关闭
  - RST 处理

#### 步骤 4: 同步到 GitHub

```bash
/pm:epic-sync tcp-state-machine
```

**成功创建的 Issues**:
- Epic #2: Epic: TCP State Machine Implementation
  - URL: https://github.com/haolipeng/ebpf-based-microsegment/issues/2
- Task #3: Design TCP State Machine Architecture
- Task #4: Implement TCP State Machine Header File

**网络错误导致未创建**:
- Task #5: Integrate TCP State Machine with TC Program
- Task #6: Integrate TCP State Machine with XDP Program
- Task #7: Testing and Validation

#### 步骤 5: 手动创建剩余 Issues

对于网络错误导致未创建的 Issues，可以手动创建：

```bash
# 清除代理设置（如果有）
unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY

# 创建 Task 003
awk 'BEGIN{p=0} /^---$/{p++; next} p>=2{print}' \
  .claude/epics/tcp-state-machine/003.md > /tmp/task-003.md

gh issue create \
  --repo haolipeng/ebpf-based-microsegment \
  --title "Integrate TCP State Machine with TC Program" \
  --body-file /tmp/task-003.md

# 创建 Task 004
awk 'BEGIN{p=0} /^---$/{p++; next} p>=2{print}' \
  .claude/epics/tcp-state-machine/004.md > /tmp/task-004.md

gh issue create \
  --repo haolipeng/ebpf-based-microsegment \
  --title "Integrate TCP State Machine with XDP Program" \
  --body-file /tmp/task-004.md

# 创建 Task 005
awk 'BEGIN{p=0} /^---$/{p++; next} p>=2{print}' \
  .claude/epics/tcp-state-machine/005.md > /tmp/task-005.md

gh issue create \
  --repo haolipeng/ebpf-based-microsegment \
  --title "Testing and Validation" \
  --body-file /tmp/task-005.md
```

**说明**:
- `awk` 命令用于移除 YAML frontmatter（前两个 `---` 之间的内容）
- `gh issue create` 使用提取的内容创建 Issue

#### 成果总结

**文档创建**:
- 1 个 PRD 文档（400+ 行）
- 1 个 Epic 文档（300+ 行）
- 5 个 Task 文档（每个 70-90 行）

**GitHub Issues**:
- 1 个 Epic Issue
- 5 个 Task Issues（3 个自动创建，2 个需手动创建）

**代码追溯**:
- 所有代码变更（3 个文件，+402 行）都可以追溯到具体的 Task
- 每个 Task 都链接到相应的设计决策和需求

---

## 常见问题

### 问题 1: `unknown flag: --json`

**错误信息**:
```
unknown flag: --json
```

**原因**: 旧版本的 `gh` CLI 不支持 `--json` 标志。

**解决方法**:
```bash
# 升级 gh CLI
sudo apt update
sudo apt install gh

# 或使用其他方式获取 Issue 编号
gh issue create ... | tee /tmp/issue-url.txt
grep -o '/issues/[0-9]*' /tmp/issue-url.txt | grep -o '[0-9]*'
```

### 问题 2: `could not add label: 'epic' not found`

**错误信息**:
```
label 'epic' not found
```

**原因**: GitHub 仓库中没有预定义的标签。

**解决方法**:

方法 1 - 创建 Issue 时不添加标签:
```bash
# 移除 --label 参数
gh issue create \
  --title "..." \
  --body-file "..."
```

方法 2 - 手动创建标签:
```bash
# 在 GitHub 仓库中手动创建标签
# Settings → Labels → New label
# 创建: epic, task, enhancement, bug 等
```

方法 3 - 使用 gh CLI 创建标签:
```bash
gh label create epic --color "8B4789" --description "Epic issue"
gh label create task --color "0E8A16" --description "Task issue"
```

### 问题 3: 网络连接错误

**错误信息**:
```
error connecting to api.github.com
```

**原因**: 网络问题或代理设置问题。

**解决方法**:

方法 1 - 清除代理设置:
```bash
unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY
```

方法 2 - 检查网络连接:
```bash
# 测试连接
curl -I https://api.github.com

# 检查 GitHub 状态
gh api /rate_limit
```

方法 3 - 稍后重试:
```bash
# 等待几分钟后重试
# GitHub API 可能有速率限制
```

### 问题 4: Git commit 提交时的格式问题

**问题**: CCPM 文档中的 commit 信息应该如何格式化？

**建议格式**:
```bash
# 简洁的提交信息（不包含 emoji 和自动生成标记）
git commit -m "实现完整的 TCP 状态机"

# 如果需要详细说明
git commit -m "实现完整的 TCP 状态机

- 新增 tcp_state_machine.h 头文件（265 行）
- 集成到 TC 程序（+87 行）
- 集成到 XDP 程序（+69 行）
- 支持全部 11 种 TCP 状态
- 符合 RFC 793 标准"
```

### 问题 5: 如何为已完成的功能创建文档？

**场景**: 代码已经实现并提交，现在需要补充 CCPM 文档。

**步骤**:

1. **创建 PRD 时设置状态为 completed**:
```yaml
---
status: completed
completed: 2025-11-15T01:40:19Z
---
```

2. **在 PRD 中链接 commit**:
```markdown
## Implementation Status

This feature has been fully implemented and merged.

- **Commit**: aada69a42de62c45b86e35c2ebc7a8e82cc3c23b
- **Date**: 2025-11-15 00:02:51 +0800
- **Files Changed**: 3 files, +402 lines, -19 lines
```

3. **Epic 中标记实际工作量**:
```markdown
## Estimated Effort

**Actual Effort**: ~4 hours (completed in single session)

Breakdown:
- Design & architecture: 30 minutes
- Header implementation: 1.5 hours
- TC integration: 1 hour
- XDP integration: 45 minutes
- Testing & validation: 15 minutes
```

4. **所有 Tasks 标记为 completed**:
```yaml
---
status: completed
completed: 2025-11-15T01:54:55Z
---
```

---

## 最佳实践

### 1. PRD 编写建议

**详细但简洁**:
- Executive Summary: 2-3 段，说明价值
- Problem Statement: 明确当前问题和解决必要性
- User Stories: 3-5 个关键场景
- Requirements: 清晰的功能和非功能需求
- Technical Design: 足够的技术细节，但不要过度设计

**使用 frontmatter**:
```yaml
---
name: feature-name         # 使用 kebab-case
description: 简短描述      # 1-2 句话
status: draft              # draft | in_progress | completed | archived
priority: high             # high | medium | low
created: 2025-11-15T00:00:00Z
---
```

### 2. Epic 分解原则

**合理的粒度**:
- Epic 应该在 1-2 周内完成
- Task 应该在 1-2 天内完成
- 避免过度分解（太多小任务）
- 避免分解不足（任务太大）

**明确依赖关系**:
```yaml
depends_on: [001, 002]     # 必须先完成 001 和 002
parallel: false            # 不能与其他任务并行
conflicts_with: [004]      # 与 004 冲突（修改同一文件）
```

**清晰的验收标准**:
```markdown
## Acceptance Criteria
- [ ] 功能完整实现
- [ ] 所有单元测试通过
- [ ] 代码审查完成
- [ ] 文档更新完成
- [ ] 无 eBPF verifier 错误
```

### 3. GitHub Issue 管理

**使用有意义的标题**:
```
✓ 好: "Integrate TCP State Machine with TC Program"
✗ 差: "Task 003"
```

**善用 Labels**:
```
epic           # Epic 级别的 Issue
task           # Task 级别的 Issue
enhancement    # 新功能
bug            # Bug 修复
documentation  # 文档更新
```

**建立 Issue 关联**:
```markdown
# 在 Task Issue 中引用 Epic
Related to #2 (Epic)

# 在 Epic Issue 中列出所有 Tasks
**Tasks**:
- #3: Design TCP State Machine Architecture
- #4: Implement TCP State Machine Header File
- #5: Integrate with TC Program
```

### 4. 工作流建议

**从小功能开始**:
- 先用简单的功能熟悉 CCPM 流程
- 逐步应用到更复杂的功能

**保持文档同步**:
- 代码完成后及时更新 Task 状态
- Epic 完成后更新 progress 百分比
- 关闭对应的 GitHub Issues

**定期回顾**:
- 每周回顾 Epic 进度
- 检查是否有遗漏的文档
- 更新技术债务和未来增强

### 5. 团队协作

**并行开发**:
```yaml
# Task 001 和 Task 004 可以并行
Task 001:
  parallel: true
  conflicts_with: []

Task 004:
  parallel: true
  conflicts_with: [003]  # 与 003 修改同一文件
```

**代码审查**:
- 在 Task 的 Definition of Done 中包含 Code Review
- 使用 GitHub PR 关联到 Task Issue
- PR 描述中引用相关 Task 和 Epic

**知识共享**:
- 在 Epic 中记录架构决策（Architecture Decision Records）
- 在 PRD 中记录用户反馈和需求变更
- 在 Task 中记录技术难点和解决方案

---

## 附录

### A. CCPM 命令速查表

| 命令 | 作用 | 示例 |
|------|------|------|
| `/pm:prd-new` | 创建新的 PRD | `/pm:prd-new feature-name` |
| `/pm:prd-parse` | 将 PRD 转换为 Epic | `/pm:prd-parse feature-name` |
| `/pm:epic-decompose` | 将 Epic 分解为 Tasks | `/pm:epic-decompose epic-name` |
| `/pm:epic-sync` | 同步到 GitHub Issues | `/pm:epic-sync epic-name` |
| `/pm:task-work` | 开始处理任务 | `/pm:task-work epic-name 001` |
| `/pm:epic-oneshot` | 一键执行 parse + decompose + sync | `/pm:epic-oneshot prd-name` |

### B. 文件模板

#### PRD 模板

```markdown
---
name: feature-name
description: 简短描述
status: draft
priority: medium
created: 2025-11-15T00:00:00Z
---

# PRD: Feature Name

## Executive Summary

[为什么需要这个功能，价值是什么]

## Problem Statement

[当前存在什么问题]

## User Stories

### Story 1: [角色名称]
**As a** [角色]
**I want** [功能]
**So that** [价值]

**Acceptance Criteria**:
- [ ] 标准 1
- [ ] 标准 2

## Functional Requirements

1. **REQ-F1**: [功能需求描述]
2. **REQ-F2**: [功能需求描述]

## Non-Functional Requirements

1. **REQ-NF1**: [性能/安全/可维护性需求]
2. **REQ-NF2**: [性能/安全/可维护性需求]

## Technical Design

### Architecture
[架构图和说明]

### Components
[组件说明]

### Data Model
[数据模型]

## Implementation Status

[如果已完成，链接到 commit]

## Success Criteria

- [ ] 成功标准 1
- [ ] 成功标准 2
```

#### Epic 模板

```markdown
---
name: epic-name
status: draft
created: 2025-11-15T00:00:00Z
progress: 0%
prd: .claude/prds/feature-name.md
---

# Epic: Epic Name

## Overview

[技术实现概述]

## Architecture Decisions

### Decision 1: [决策名称]
**Choice**: [选择的方案]
**Rationale**: [选择理由]

## Technical Approach

[技术实现方法]

## Implementation Strategy

### Phase 1: [阶段名称]
- Task 001: [任务名称]
- Task 002: [任务名称]

## Dependencies

- [ ] 依赖项 1
- [ ] 依赖项 2

## Success Criteria

- [ ] 成功标准 1
- [ ] 成功标准 2
```

#### Task 模板

```markdown
---
name: Task Name
status: pending
created: 2025-11-15T00:00:00Z
depends_on: []
parallel: false
conflicts_with: []
---

# Task: Task Name

## Description

[详细的任务描述]

## Acceptance Criteria

- [ ] 验收标准 1
- [ ] 验收标准 2

## Technical Details

[技术实现细节，包括代码位置、修改方法等]

## Dependencies

- [ ] 依赖项 1

## Effort Estimate

- Size: M
- Hours: 3 hours
- Parallel: false

## Definition of Done

- [ ] 代码实现完成
- [ ] 单元测试通过
- [ ] 代码审查完成
- [ ] 文档更新完成
```



最佳实践

```
准备好创建实施 Epic 了吗？运行以下命令：

  /pm:prd-parse 进程级别策略关联

  这将会：

    1. 解析 PRD 中的需求
    2. 创建 Epic 文件（包含任务分解）
    3. 生成 GitHub Issues
    4. 建立开发追踪体系
```

