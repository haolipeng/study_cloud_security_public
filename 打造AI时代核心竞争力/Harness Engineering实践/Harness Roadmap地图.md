/////////////////////////////////////////////////////////////////////////////



https://github.com/walkinglabs/learn-harness-engineering



通过如下的课程列表去学习：

https://walkinglabs.github.io/learn-harness-engineering/zh/



第一课 实验

https://walkinglabs.github.io/learn-harness-engineering/zh/projects/project-01-baseline-vs-minimal-harness/



限定30分钟时间和20轮次，从15：00开始实验



好的范例是长什么样的？

feature_list.json文件

```
{
  "project": "project-01",
  "description": "Baseline Electron knowledge base with minimal harness",
  "features": [
    {
      "id": "window-launch",
      "name": "Window Launch",
      "description": "Electron app opens a BrowserWindow with correct dimensions and preload script",
      "status": "pass",
      "evidence": "npm run dev launches window at 1200x800 with contextIsolation=true and nodeIntegration=false",
      "testedAt": "2026-03-30T10:00:00Z"
    },
    {
      "id": "document-list",
      "name": "Document List Panel",
      "description": "Left sidebar shows imported documents with empty state message",
      "status": "pass",
      "evidence": "DocumentList component renders empty state when no documents, shows document cards when data present",
      "testedAt": "2026-03-30T10:05:00Z"
    },
    {
      "id": "question-panel",
      "name": "Question Panel",
      "description": "Bottom input bar accepts questions and submits via IPC",
      "status": "pass",
      "evidence": "QuestionPanel renders text input and Ask button, submits to window.knowledgeBase.qa.ask on Enter or click",
      "testedAt": "2026-03-30T10:08:00Z"
    },
    {
      "id": "data-directory",
      "name": "Data Directory",
      "description": "PersistenceService creates and manages userData/knowledge-base-data directory",
      "status": "pass",
      "evidence": "PersistenceService constructor calls ensureDirectories() creating data, documents, and index subdirectories",
      "testedAt": "2026-03-30T10:10:00Z"
    }
  ]
}
```

四个具体功能是窗口启动、文档列表、问答面板、本地数据目录。



起码包括上述内容：

- 单元测试 — 能跑 make test 或 cargo test 或 go test
- 集成测试/端到端测试 — 能跑完整流程
- Linter — 比如 clippy、golangci-lint、静态分析工具
- 编译检查 — 确保能编译通过（这个肯定有，但我想确认你是否会在 CI 或本地脚本里显式检查）
- CI/CD — GitHub Actions / GitLab CI / Jenkins 之类的



**AI Coding Agent Readiness Checklist**

https://gist.github.com/gmoigneu/a963b595ac238ad2d2260ebb8b29f048



**Making Your Repository AI-Ready**

https://medium.com/@chalyi/making-your-repository-ai-ready-1ab45b05222b



**项目仓库地址搜集**

https://github.com/agent-next/agent-ready



https://github.com/superduck-ai/agent-readiness



# 2026年最新AIAgent应用开发学习路线零基础到精通

https://github.com/liyupi/codefather/issues/51
