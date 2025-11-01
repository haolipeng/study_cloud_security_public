不管是在公司，还是在个人项目中使用openspec，都几乎有以下三个应用场景：

- 1、新项目
- 2、进行中的项目，需大量开发功能feature
- 3、成熟项目，只需要修复bug啥的





openspec/changes/archive下的目录和文件，和openspec/specs目录下的目录和文件分别起到什么作用？

让我详细解释这两个目录的作用和它们之间的关系：



# 一、每个目录的功能和作用

## `openspec/specs/` 

**作用**：记录系统**当前**已实现的功能规范



- 这是"源头真相"（source of truth），描述系统**现在**具备的能力
- 格式要求：
  - 必须有 `## Purpose` 部分
  - 使用 `## Requirements`（不是 ADDED/MODIFIED/REMOVED）
  - 每个 requirement 必须有 `#### Scenario:`
- 这些文件会随着开发不断更新：
  - 当你完成一个新功能并**archive**归档后，新的 requirements 会合并到这里
  - 当你修改现有功能时，相应的 requirement 会被更新

**例子**：



```
openspec/specs/
├── ebpf-capture/spec.md      # 当前 eBPF 捕获功能的完整规范（8个requirements）
├── llm-analysis/spec.md       # 当前 LLM 分析功能的完整规范（8个requirements）
└── network-monitoring/spec.md # 当前网络监控功能的完整规范（7个requirements）
```

## `openspec/changes/archive/` - 历史变更记录（Change History）

**作用**：记录功能是**如何**、**何时**被添加或修改的历史



- 这是变更的历史档案，按时间顺序记录每次功能开发
- 格式特点：
  - 使用日期前缀：`YYYY-MM-DD-change-name/`
  - 包含完整的变更上下文（proposal.md, tasks.md, design.md）
  - specs 使用**增量格式**：`## ADDED Requirements`, `## MODIFIED Requirements`
- 一旦归档就不再修改，保持历史记录完整性

**例子**：



```
openspec/changes/archive/
├── 2025-10-27-baseline-ebpf-capture/     # 记录 eBPF 功能是何时、为何被添加的
│   ├── proposal.md                       # 为什么要这个功能
│   ├── tasks.md                          # 当时做了哪些工作
│   └── specs/ebpf-capture/spec.md        # 增量：## ADDED Requirements（8个）
├── 2025-10-27-baseline-llm-analysis/     # LLM 功能的历史记录
└── 2025-10-27-baseline-network-monitoring/  # 网络监控功能的历史记录
```



## 两者的关系图

```
工作流程：
┌─────────────────────────────────────────────────────────────────┐
│ 1. 开发新功能 "事件过滤" (add-event-filtering)                     │
│    openspec/changes/add-event-filtering/                        │
│    ├── proposal.md          （为什么要这个功能）                   │
│    ├── tasks.md             （要做的工作）                        │
│    └── specs/event-analysis/spec.md                             │
│        ## ADDED Requirements  （新增3个 requirements）            │
└─────────────────────────────────────────────────────────────────┘
                              ↓
                    【实现功能，完成所有 tasks】
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 2. 归档变更                                                       │
│    openspec archive add-event-filtering --yes                   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
                    ┌─────────┴─────────┐
                    ↓                   ↓
┌──────────────────────────┐  ┌──────────────────────────────┐
│ 归档到历史记录              │  │ 更新当前规范                  │
│ openspec/changes/archive/ │  │ openspec/specs/              │
│                          │  │                              │
│ 2025-10-27-add-event-    │  │ event-analysis/spec.md       │
│   filtering/             │  │ ## Purpose                   │
│ ├── proposal.md          │  │ ## Requirements              │
│ ├── tasks.md (✓完成)     │  │ ### Requirement: 过滤配置     │
│ └── specs/.../spec.md    │  │ ### Requirement: 过滤执行     │
│    ## ADDED (3个新增)     │  │ ### Requirement: 过滤规则管理 │
│                          │  │ (新增的3个requirements合并)   │
└──────────────────────────┘  └──────────────────────────────┘
    历史：它是何时被添加的         当前：系统现在能做什么
```

## 实际例子对比

### `openspec/specs/network-monitoring/spec.md` (你刚打开的文件)

```markdown
# Network Monitoring

## Purpose
This capability monitors network traffic to detect C2 communication...

## Requirements

### Requirement: Network Syscall Monitoring
The system SHALL capture all network-related syscalls...

#### Scenario: Capture outbound connections
- **WHEN** malware attempts to connect to a remote server
- **THEN** the system captures the connect syscall...
```

→ **这是当前规范**：描述系统**现在**具备的网络监控能力（7个完整的 requirements）



### `openspec/changes/archive/2025-10-27-baseline-network-monitoring/specs/network-monitoring/spec.md`

```markdown
# Network Monitoring - Baseline Specification

## ADDED Requirements

### Requirement: Network Syscall Monitoring
The system SHALL capture all network-related syscalls...
```

→ **这是历史记录**：记录这7个 requirements 是在 2025-10-27 作为 baseline 被添加的



### `openspec/changes/archive/2025-10-27-baseline-network-monitoring/proposal.md`

```markdown
# Baseline: Network Monitoring

## Why
Document the existing network traffic monitoring capabilities...

## What Was Implemented
- Network syscall monitoring (connect, bind, accept, socket)
- DNS query interception and logging
- SSL/TLS plaintext capture...
```

→ **这是变更背景**：解释为什么要添加这些功能，当时的决策和影响



## 为什么需要两个目录？

1. **`openspec/specs/`** - 回答"系统能做什么"
   - 开发人员查看：我要修改某个功能，现在的规范是什么？
   - 新成员了解：这个系统有哪些能力？
   - 始终保持最新状态
2. **`openspec/changes/archive/`** - 回答"为什么这样设计"、"何时加入的"
   - Code review：这个功能当时为什么要这样设计？
   - 回溯历史：某个 requirement 是什么时候加入的？
   - 审计追踪：所有变更都有完整的 proposal、tasks、design 文档
