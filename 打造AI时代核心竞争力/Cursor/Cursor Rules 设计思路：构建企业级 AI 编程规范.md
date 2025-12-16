## 前言

![img](https://mmbiz.qpic.cn/mmbiz_svg/icXtjp9jsR5TE97cIic3FWW05fZnk5Aft2rYNkpAjvPk7EGDfFp0gXTX2hr2vc9n4Q1WN5NNz0htKibKxoeZOyD1VicpjQwzFZHT/640?wx_fmt=svg&from=appmsg#imgIndex=0)

![img](https://mmbiz.qpic.cn/mmbiz_svg/icXtjp9jsR5TE97cIic3FWW05fZnk5Aft2nQIx2vVuTFv32BNIM48PiaGLSpialwzicLx7RVGHszK1JZd3oVnQoox8POtYBibSFeem/640?wx_fmt=svg&from=appmsg#imgIndex=1)

在 AI 编程时代，如何让 AI 编程工具更好地理解项目规范、遵循团队约定，已成为提升开发效率的关键。Cursor Rules 为我们提供了一个强大的解决方案。本文将深入分享我们在后台管理系统中的 Cursor Rules 设计思路，特别是规则类型策略的考量。

01

# Cursor Rules 基础概念

![img](https://mmbiz.qpic.cn/mmbiz_svg/icXtjp9jsR5TE97cIic3FWW05fZnk5Aft2lV3jF8a9LZNxxcuFyNHpPZ4Z5QIC51A8icQs5NjRTjenBz7YzjynG0ySBsx0mfqL9/640?wx_fmt=svg&from=appmsg#imgIndex=2)

![img](https://mmbiz.qpic.cn/mmbiz_svg/icXtjp9jsR5TE97cIic3FWW05fZnk5Aft2SUpJKgdwiaHrSAaLheAgwHKEgePkA5rpgsPYtpPrqpRL7Sbxav3pfpN26k0U0oSPic/640?wx_fmt=svg&from=appmsg#imgIndex=3)

## 什么是 Cursor Rules

Cursor Rules 是通过.cursor/rules/ 目录下的 mdc 文件来定义 AI 编程时的行为规范。这些规则文件可以：

- 定义项目特定的编码规范
- 提供开发流程指导
- 约束 AI 的代码生成行为
- 确保团队开发的一致性

## Cursor 规则应用类型

Cursor 支持四种规则应用级别，通过不同的字段组合来决定规则的应用模式。



1. Always Apply 类型 - 始终包含在模型上下文中

- **字段配置**：仅设置 "alwaysApply": true

  **应用时机**：每个聊天和 cmd-k 会话都自动应用

  **适用场景**：项目基础信息、核心编码规范、技术栈约束

  **典型用途**：核心代码示例，始终加载到上下文



2. Apply to Specific Files 类型 - 当引用与某个 glob 模式匹配的文件时自动包含

- **字段配置**：仅设置 "globs": "pattern"

  **应用时机**：当文件匹配指定模式时自动应用

  **适用场景**：特定文件类型的处理规范（如 .go 文件的 Go 语言规范）

  **典型用途**：基于文件匹配模式自动应用



3. Apply Intelligently 类型 - 提供给 AI，由其决定是否包含。必须提供描述

- **字段配置**：仅设置 "description": "描述"

  **应用时机**：当 AI 基于描述智能判断相关时应用

  **适用场景**：特定开发流程、业务场景规范

  **典型用途**：供 AI 代理按需请求的规则



4. Apply Manually 类型 - 仅在使用 @ruleName 明确提及时才会包含

- **字段配置**：不设置任何规则字段

  **应用时机**：当用户手动 @ 提及时应用

  **适用场景**：可选工具、辅助功能

  **典型用途**：用户手动选择应用的规则

## mdc 文件声明格式

![img](https://mmbiz.qpic.cn/sz_mmbiz_png/O4hdbRPSHheSWgSFTK0AKb4Sh0u2cqI0ibY7xbHx099l6llKu31iabJuUCfLD6EdibhGZOnkFcCSoGF4KgRj4G1gw/640?wx_fmt=png&from=appmsg#imgIndex=4)

## 字段组合规则总结

![img](https://mmbiz.qpic.cn/sz_mmbiz_png/O4hdbRPSHheSWgSFTK0AKb4Sh0u2cqI0cPmicAU4McMjIKxd40BqQDgoiaicoJj6IcIZ9XOriadjQbllFIBro86K4g/640?wx_fmt=png&from=appmsg#imgIndex=5)

重要提醒：每个规则只能使用一种类型，不能同时设置多个规则字段，否则会导致规则冲突。

02

# 后台管理项目 Cursor Rules 体系设计

![img](https://mmbiz.qpic.cn/mmbiz_svg/icXtjp9jsR5TE97cIic3FWW05fZnk5Aft2lV3jF8a9LZNxxcuFyNHpPZ4Z5QIC51A8icQs5NjRTjenBz7YzjynG0ySBsx0mfqL9/640?wx_fmt=svg&from=appmsg#imgIndex=6)

![img](https://mmbiz.qpic.cn/mmbiz_svg/icXtjp9jsR5TE97cIic3FWW05fZnk5Aft2SUpJKgdwiaHrSAaLheAgwHKEgePkA5rpgsPYtpPrqpRL7Sbxav3pfpN26k0U0oSPic/640?wx_fmt=svg&from=appmsg#imgIndex=7)

## 整体架构设计

我们的后台管理系统采用了分层、模块化的规则设计架构：

![img](https://mmbiz.qpic.cn/sz_mmbiz_png/O4hdbRPSHheSWgSFTK0AKb4Sh0u2cqI04lAR3H8KjPQ3wLica6SaPl1RB9EU9OSK5zPVTwYnicQuGmhmsIxuKhfw/640?wx_fmt=png&from=appmsg#imgIndex=8)

## 规则分类设计理念

基于 Cursor 的四种规则类型，我们将后台管理项目的规则体系设计为三种应用类型：



2.2.1 Always Apply 类型 - 基础约束层

设计目标：

确保 AI 始终遵循项目的基础约定和核心开发模式

包含内容：

- tcf-general.mdc：项目基本信息、技术栈、角色定义

  tcf-admin-code-style.mdc：编码规范、错误处理、日志规范等

  workflow/curd/：标准CRUD开发流程（完整示例）



设计为 Always Apply 类型的原因：

- **一致性保证**：无论开发什么功能，都需要遵循基础规范

  **角色定位**：让 AI 始终以"经验丰富的后端研发工程师"身份工作

  **技术栈声明**：给AI传达使用的技术栈信息

  **编码标准**：统一的代码风格、错误处理、日志记录方式

  **典型的代码完整示例**（效果的关键保障）：

  

  - 使用标准CRUD作为最具代表性的完整开发流程，可以展示 api→controller→service→logic→dao 的完整开发链路，与自然语言和文档相比，代码示例更具准确表达性

    无论做什么需求都至少有该层的代码作为参照



2.2.2 Apply Intelligently 类型 - 智能匹配层

设计目标：

通过 AI 识别 description 字段，智能判断是否需要相关规范，有效减少上下文干扰

包含内容：

- workflow/belongs-to-curd/：多对一关系开发流程

  workflow/has-many-curd/：一对多关系开发流程

  api-client/：第三方API文档封装调用流程

  database/：数据库设计和优化

  auto-test/：自动化测试规范，确保 AI 在完成代码开发后，知悉项目标准的测试流程而非自己理解的测试方式，通过测试验证代码的生成结果并进行AI自我修复

设计为 Apply Intelligently 类型的原因：

- **智能识别**：AI 通过分析 description 字段，判断当前开发场景是否需要引入相关规范

  **减少干扰**：只在相关场景匹配上才使用，避免无关规范占用上下文空间

  **准确匹配**：通过 description 描述，AI 能准确识别开发需求并激活对应规范



2.2.3 Apply Manually 类型 - 手动选择层

设计目标：

为用户提供可选的参考和工具，由用户根据实际需求手动选择使用

包含内容：

- tools/commit/：提交工具规范

设计为 Apply Manually 类型的原因：

- **按需使用**：用户可根据实际需要手动引用相关规范或工具

  **工具导向**：主要提供具体工具的使用方法和操作指引

  **避免自动干扰**：不会被自动使用，完全由用户自主控制

> 自 Cursor 1.6 起已引入 commands 概念，建议将工具类内容放入 commands 目录，生成代码相关规范仍放在 rules 目录。此前由于没有 commands，部分工具类内容被放在 rules 下也能达到目的，但有时会导致 AI 将 mdc 文件当作文本理解而非执行指令。后续应将工具类规范迁移至 commands，能提升 AI 的理解和执行准确性，同时也可以被 Claude Code、Gemini CLI等通过 git submodule 统一引入。

补充说明

03

# 确保 Cursor Rules 通用性的验证策略

![img](https://mmbiz.qpic.cn/mmbiz_svg/icXtjp9jsR5TE97cIic3FWW05fZnk5Aft2QA6XmrtwSTiakbm09CyE6A17tkKulKxgU7YlNTzGf4G9sIBLhkQAGBlbhWkKZuP82/640?wx_fmt=svg&from=appmsg#imgIndex=9)

![img](https://mmbiz.qpic.cn/mmbiz_svg/icXtjp9jsR5TE97cIic3FWW05fZnk5Aft2V6sVgWOuFmicyuFpibMKZpXBuhxPOsXfibDtNpLMmQghcy0l7CX3FC38J1JxyXHvf6x/640?wx_fmt=svg&from=appmsg#imgIndex=10)

## 引入中立性场景验证

为了确保 rules 的效果具备通用性，方便不同的业务项目通过 git submodule 引入到 .cursor/rules 目录并达到相同的AI生成代码效果，我们建立了中立性场景进行测试验证：

典型数据结构验证

我们采用以下四种典型数据结构进行规范验证：

- **单表结构**：用户管理（User）- 独立实体，包含基础字段和加密字段处理

  **一对多结构**：订单与订单项（Order -> OrderItem）- 主表包含多个子表记录

  **多对一结构**：组织架构（Employee -> Team -> Department）- 三层级联，体现归属关系

  **树形结构**：分类管理（Category）- 自关联的递归树形结构

## 中立性场景验证的重要性

1. 避免业务偏向

如果基于特定业务项目而非中立性项目来验证 Cursor Rules 的代码生成效果，容易制定出仅适用于该业务的规范，从而削弱其通用性。

2. 标准化验证

通过多种典型数据结构类型的验证，确保规范适用于各种常见的业务场景

04

# 跨平台编程工具的 Cursor Rules 适配策略

![img](https://mmbiz.qpic.cn/mmbiz_svg/icXtjp9jsR5TE97cIic3FWW05fZnk5Aft20ywcMGrSFpkpnmRegQLGzmpicsQ0QG46AWdHqCwm1HvnrAw8qwrO4fY1X3YDtoiap4/640?wx_fmt=svg&from=appmsg#imgIndex=11)

![img](https://mmbiz.qpic.cn/mmbiz_svg/icXtjp9jsR5TE97cIic3FWW05fZnk5Aft20PuqxR01ekLPFiakFcWKKNiaqvOeibnj67JrvkZM4mgT9icUP465xjNib28UenNPPYTYP/640?wx_fmt=svg&from=appmsg#imgIndex=12)

为了让 Claude Code 和 Gemini CLI 等编程工具达到与 Cursor 相同的规范化开发体验，我们设计了三种策略来分别处理不同类型的 rules：

## Apply Manually（手动应用）规则适配

**对应场景：**工具类规范 - 如代码生成、测试、提交等高频操作工具

**设计理念：**采用类似 Linux alias 的思路，通过简短命令触发特定规范的读取和执行。自 Cursor 1.6 起已引入 commands 概念，使用方式比这种方式增加了 "/"，例如：/ai-commit

**实现方式：**在 AI 工具的配置文件中预定义命令映射关系

典型场景：

当用户输入 `ai-commit` 请读取规范 `.cursor/rules/tcf-admin-backend/tools/commit/ai-commit.mdc` 并执行

**优势：**操作简便，记忆成本低，提高开发效率

## Always Apply（始终应用）规则适配

**对应场景：**基础规范 - 标记为 alwaysApply: true 的核心编码规范

**识别方式：**通过 alwaysApply: true 标识的核心规范文件

**加载机制：**会话首次发送消息，AI将会读取 .cursor/rules 里包含 alwaysApply: true 的mdc文件加入上下文

查找指令：

\# 查找必读规范文件 find .cursor/rules -name "*.mdc" -exec grep -l "alwaysApply: true" {} \;

**作用机制：**确保所有代码生成都严格遵守项目的基础编码规范

## Apply Intelligently（匹配应用）规则适配

**对应场景：**业务规范 - 根据具体需求按需应用的规范文件

**适用场景：**不确定具体需要哪些规范时的智能推荐

**加载机制：**会话首次发送消息，AI将会读取 .cursor/rules 里所有文件的 description 的mdc文件加入上下文。根据用户需求分析，由 AI 匹配相关度再决定是否深入阅读相关规范文件

分析指令：

\# 获取所有规范的描述信息grep -r "description:" .cursor/rules/ --include="*.mdc"

**智能化优势：**避免手动筛选，精准匹配业务场景需求

## 完整配置示例（以CLAUDE.md 为例）

![img](https://mmbiz.qpic.cn/sz_mmbiz_png/O4hdbRPSHheSWgSFTK0AKb4Sh0u2cqI0KB3rQDMSrFjfpGs3CTUkoJ26zd0kQqgYh98aINOcOcETqT8EWT2WeA/640?wx_fmt=png&from=appmsg#imgIndex=13)

通过这三种策略，Claude Code 和 Gemini CLI 等工具能够复用 Cursor 的 rules 规范，实现相同的规范化开发体验。



### END

![img](https://mmbiz.qpic.cn/mmbiz_svg/icXtjp9jsR5TE97cIic3FWW05fZnk5Aft2OaZA6oDPnSwzWqFyWaOEEJRh5OPc0OFDgStKEGY15X0pffbgjicNDPGfH5VlEqazr/640?wx_fmt=svg&from=appmsg#imgIndex=14)



   