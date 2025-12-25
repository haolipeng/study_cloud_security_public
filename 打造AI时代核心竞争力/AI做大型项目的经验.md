PTC 专业术语是什么来着？





这个是一个大佬分享的使用AI做真实大项目的宝贵经验：

1、在根目录主 MD 中强调：任何功能/架构/写法更新后，必须同步更新相关目录的子文档。

2、每个文件夹新增（或补充）一个极简架构说明（3 行以内），并在其下列出该文件夹内每个文件的名称、地位、功能；文件开头要声明“若所属文件夹变化请更新我”。

3、每个文件开头增加 3 行极简注释：input / output / pos，并追加一句“若文件更新必须同步更新我的开头注释和所属文件夹的 MD”。

4、以上形成自指/分形式的多级索引结构，确保局部-整体联动。

本质上就是不断强化agent对于项目的理解，防止跑偏，你帮我在此项目中实现下这个功能。

但是此项目的实现不包括项目根目录下的source-references目录，因为这个目录是我所参考的开源项目代码的路径



第一次尝试直接coredump了

```
1: 0x1015271  [claude]
 2: 0x29d3acb V8_Fatal(char const*, ...) [claude]
 3: 0x14db119 v8::internal::Sweeper::RawSweep(v8::internal::PageMetadata*, v8::internal::FreeSpaceTreatmentMode, v8::internal::Sweeper::SweepingMode, bool, bool) [claude]
 4: 0x14dd992  [claude]
 5: 0x14ddb9c v8::internal::Sweeper::MajorSweeperJob::RunImpl(v8::JobDelegate*, bool) [claude]
 6: 0x232e498 v8::platform::DefaultJobWorker::Run() [claude]
 7: 0x1015444  [claude]
 8: 0x7f8d0bc94ac3  [/usr/lib/x86_64-linux-gnu/libc.so.6]
 9: 0x7f8d0bd26850  [/usr/lib/x86_64-linux-gnu/libc.so.6]
Illegal instruction (core dumped)
```



如何写一个更好的CLAUDE.md文件呢？

https://www.humanlayer.dev/blog/writing-a-good-claude-md

https://www.anthropic.com/engineering/claude-code-best-practices