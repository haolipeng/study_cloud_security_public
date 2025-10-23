

# 一、提出问题

公司项目有时候会借鉴多个开源项目的核心功能（有主机安全也有网络安全的），组成一个新的牛逼功能，那么我在使用cursor写代码时，要如何使用它来借鉴多个开源项目的核心技术呢？



之前没有ai工具时，程序实现上述需求的流程大致如下
1、首先，公司项目借鉴A项目的某个功能模块，我先调研下此功能模块的技术选型，使用了什么技术，为啥使用这个技术
2、其次，梳理这个功能模块的实现思路，整理成文档
3、再次，根据梳理的技术实现思路和文档，再实现一个最小化的demo
4、最后，把测试通过的功能模块demo代码，移植到公司项目中



通过听陈天老师的b站视频《如何用 cursor 阅读 golang 源码，并生成对应的 Rust 代码？》，https://www.bilibili.com/video/BV1LPTUz8ExS/?spm_id_from=333.1387.top_right_bar_window_history.content.click&vd_source=553cfea04504936858081191fc1520dc



陈天老师的开源项目地址为https://github.com/tyrchen/grpc-client



具体操作：

rust项目使用的是rust-lib-template

将待重写的python开源项目导入到项目中，并添加到.gitignore文件中。



和cursor 进行交互的提示词如下：

**输入给cursor：**

Please look into @/grpcurl , examine every go file (except *_test.go), and learn the arch & design of the software. Then based on your learnings, generate two files under ./specs:

1. the product requirement doc: list all the features grpcurl supports, and their detailed explanation and usage.
2. the rust design doc: generate relevant(相关的) rust data structure, trait. Also the design charts using mermaid. Make sure you use latest clap and tonic / tonic-reflection. Please follow @rust/core/design-patterns.mdc @rust/core/code-quality.mdc @dependencies.mdc @rust/core/type-system.mdc



**cursor输出的结果：**

product-requirements.md文档

rust-design.md文档

这时候需要用户好好的阅读生成的文档，然后和ai进行沟通和完善文档，确保需求和技术文档都符合心里所想的样子。



cursor每次生成的代码，可能会有一些误差，所以每次它写完以后，一定要记得review下代码，即使每次task执行有5%的误差，那么执行20次task后，这个误差就已经很大了。



在遇到AI一直卡壳的部分，选择性的给予人工干预，比如编译错误卡住，功能实现的很丑陋，或者为了满足单元测试而写的虚假代码等。



列出开源项目支持的特性features

想将开源项目的repo代码，转换为另一个语言的项目repo代码时，第一件事是：

先让cusor去读开源项目的源码来**梳理项目需求**，再根据项目源码和需求再生成一个**设计文档**。

设计文档的内容包括：数据结构，流程图（mermaid格式），明确当前项目使用的库和依赖，以及转到另一种语言，要采用什么库和依赖。

以grpc库举例，python语言有很多grpc库可以选择，go语言也有很多grpc库可以选择，那么选择哪个开源库呢？





最终要生成specs文档，存储在specs目录下。

please follow @rust/core/design-patterns.mdc @rust/core/code-quality.mdc @dependencies.mdc 规则文件



具体操作



# 一、使用AI阅读开源代码

deepwiki网站适合对项目进行初步的了解，其内容还是显得太单薄了点。

看什么？

- 核心功能
- 核心设计思路



重点：将以上内容写入./specs/design.md中，并添加mermaid chart来揭示数据结构之间的关系。



在阅读开源代码上，AI相比程序员的优势在哪里呢？



好用的提问技巧有哪些？



不同的项目有不同的阅读方式。



在家里的机器上，使用claude code帮我来阅读tracee的源代码，让自己能够更快的理解其核心技术实现，从而提升自己的技术能力。





具体方法：

对于从0到1的项目，使用speckit工具来来进行要更好

https://github.com/github/spec-kit



cursor memory bank功能:

https://cursor.zone/faq/how-to-use-cursor-memory-bank.html



陈天



磨刀不误砍柴功，平时多学点软技能，关键时候才能用的上。