## 一、核心方法论：陈天老师的实践案例

### 1.1 核心思路

将开源项目的代码转换为另一种语言时，关键步骤：

1. **需求梳理**：让 Cursor 读取源码并梳理项目需求
2. **设计文档生成**：基于源码和需求生成详细的设计文档



看到好的项目仓库想用另一个语言重写，不同编程语言之间没法直接翻译，需要先干什么呢？

**一个代码库不仅仅是项目中的代码，还有代码的依赖库，第三方库等**



### 1.2 设计文档内容要求

- **数据结构**：核心数据结构定义
- **流程图**：使用 Mermaid 格式绘制
- **依赖库选择**：明确原项目使用的库，以及目标语言的对应库
- **架构设计**：整体架构和模块关系
- **独特性**：使用的编程语言不同，提供的特性也不同



## 二、具体操作步骤

### 2.0 参考资源

- **视频教程**：[如何用 cursor 阅读 golang 源码，并生成对应的 Rust 代码？](https://www.bilibili.com/video/BV1LPTUz8ExS/)
- **项目地址**：https://github.com/tyrchen/grpc-client

### 2.1 项目准备

1. 使用合适的项目模板（如 rust-lib-template）
2. 将待分析的开源项目导入到项目中
3. 将开源项目路径添加到 `.gitignore` 文件

### 2.2 核心提示词模板

```
Please look into @/[项目路径], examine every [语言] file (except *_test.[扩展名]), and learn the arch & design of the software. Then based on your learnings, generate two files under ./specs:

1. the product requirement doc: list all the features [项目名] supports, and their detailed explanation and usage.
2. the [目标语言] design doc: generate relevant data structure, trait. Also the design charts using mermaid. Make sure you use latest [相关库]. Please follow @[规则文件路径]
```

### 2.3 实际案例：grpcurl 项目分析

**输入给cursor：**

```
Please look into @/grpcurl , examine every go file (except *_test.go), and learn the arch & design of the software. Then based on your learnings, generate two files under ./specs:

1. the product requirement doc: list all the features grpcurl supports, and their detailed explanation and usage.
2. the rust design doc: generate relevant(相关的) rust data structure, trait. Also the design charts using mermaid. Make sure you use latest clap and tonic / tonic-reflection. Please follow @rust/core/design-patterns.mdc @rust/core/code-quality.mdc @dependencies.mdc @rust/core/type-system.mdc
```

**cursor输出的结果：**

- `product-requirements.md` 文档
- `rust-design.md` 文档



### 2.4 文档完善流程

生成初始文档后，需要：

1. 仔细阅读生成的文档
2. 与AI进行沟通和完善
3. 确保需求和技术文档符合预期



## 三、关键技巧与注意事项

### 3.1 质量控制原则

- **代码审查**：每次AI生成代码后必须进行review
- **误差累积**：单次5%误差，20次后误差会显著放大
- **人工干预**：在AI卡壳时及时介入
  - 编译错误卡住
  - 功能实现丑陋
  - 为满足单元测试而写的虚假代码

### 3.2 阅读重点

- **核心功能**：项目的主要功能特性
- **架构设计思路**：架构设计和实现理念
- **数据结构关系**：使用 Mermaid 图表可视化



### 3.3 文档组织规范

- 所有规格文档存储在 `./specs/` 目录
- 提示词模板存储在 `./specs/instructions/` 目录
- 遵循项目规范文件（design-patterns.mdc、code-quality.mdc 等）