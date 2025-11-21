# CCPM 快速开始指南

## 🚀 立即开始

### 第一次使用

```bash
# 1. 认证 GitHub
gh auth login

# 2. 初始化系统
/pm:init

# 3. 创建项目上下文
/context:create
```

## 📋 常用命令速查

### PRD 管理
```bash
/pm:prd-new <name>        # 创建新 PRD
/pm:prd-list              # 列出所有 PRD
/pm:prd-parse <name>      # 将 PRD 转换为 Epic
/pm:prd-status            # 查看 PRD 状态
```

### Epic 管理
```bash
/pm:epic-list             # 列出所有 Epic
/pm:epic-show <name>      # 显示 Epic 详情
/pm:epic-oneshot <name>   # 一键同步到 GitHub
/pm:epic-status           # 查看 Epic 状态
```

### Issue 管理
```bash
/pm:issue-start <number>  # 开始处理 Issue
/pm:issue-show <number>   # 显示 Issue 详情
/pm:issue-close <number>  # 关闭 Issue
/pm:issue-status          # 查看当前 Issue 状态
```

### 工作流命令
```bash
/pm:next                  # 获取下一个优先任务
/pm:status                # 查看整体项目状态
/pm:standup               # 生成站会报告
/pm:help                  # 查看所有命令
```

## 🔄 典型工作流

### 方式 1: 从 PRD 开始（推荐）

```bash
# 1. 创建 PRD
/pm:prd-new user-authentication

# 2. 转换为 Epic
/pm:prd-parse user-authentication

# 3. 同步到 GitHub
/pm:epic-oneshot user-authentication

# 4. 开始工作
/pm:next
/pm:issue-start <issue-number>
```

### 方式 2: 直接从 Epic 开始

```bash
# 1. 创建 Epic（手动编辑 .claude/epics/feature-name.md）

# 2. 同步到 GitHub
/pm:epic-sync feature-name

# 3. 开始工作
/pm:issue-start <issue-number>
```

## 🎯 最佳实践

1. **始终从 PRD 开始** - 确保需求清晰
2. **使用 /pm:next** - 让系统推荐优先级
3. **定期同步** - 保持 GitHub 和本地一致
4. **小步提交** - 每个 Issue 专注一个任务
5. **使用代理** - 让专业代理处理复杂任务

## 🤖 可用代理

- `@code-analyzer` - 跨文件代码分析和 bug 查找
- `@file-analyzer` - 分析和总结大型文件
- `@test-runner` - 运行测试并分析结果
- `@parallel-worker` - 协调并行工作流

## 📁 目录结构

```
.claude/
├── prds/           # 产品需求文档
├── epics/          # 技术 Epic
├── context/        # 项目上下文
├── agents/         # 专业代理
├── commands/       # 可用命令
├── rules/          # 工作规则
└── scripts/        # 辅助脚本
```

## 🔧 故障排除

### GitHub 认证问题
```bash
gh auth status
gh auth login
```

### 检查配置
```bash
source .claude/ccpm.config
echo $GITHUB_REPO
```

### 查看日志
```bash
cat .claude/context/README.md
```

## 📚 更多资源

- 完整文档: `CCPM_INSTALLATION.md`
- 命令详情: `.claude/commands/pm/`
- 代理说明: `AGENTS.md.ccpm`
- GitHub: https://github.com/automazeio/ccpm