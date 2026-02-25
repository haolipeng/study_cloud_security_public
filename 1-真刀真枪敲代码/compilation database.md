# 一、简介	

​	如果您正在处理不基于 CMake、Gradle 或 Makefiles 的项目，您仍然可以从 CLion 提供的高级 IDE 功能中受益。一种方法是导入非 CMake 项目，让 CLion 将其转换为简单的 CMake 结构。另一种选择是通过加载项目的编译数据库来打开项目。

​	

​	编译数据库允许 CLion 检测项目文件并提取所有必要的编译器信息，例如包含路径和编译标志。

​	

​	编译数据库是一个名为 compile_commands.json 的 JSON 格式文件，其中包含有关项目中每个编译单元的结构化数据。

以下片段显示了 JSON 编译数据库的示例：

```json
{
"directory": "/Users/me/prj/Calendar/",
"command": "/usr/local/bin/g++-7 -I/Users/me/prj/Calendar/calendars -g -std=c++11 -o calendar_run.dir/main.cpp.o -c /Users/me/prj/Calendar/main.cpp",
"file": "/Users/me/prj/Calendar/main.cpp"
},
{
"directory": "/Users/me/prj/Calendar/calendars",
"command": "/usr/local/bin/g++-7 -I/Users/me/prj/Calendar/calendars -g -std=c++11 -o calendars.dir/calendar_defs.cpp.o -c /Users/me/prj/Calendar/calendars/calendar_defs.cpp",
"file": "/Users/me/prj/Calendar/calendars/calendar_defs.cpp"
}
```

您可以看到一组称为命令对象的条目。每个命令对象代表翻译单元的主文件、工作目录、实际的编译命令（或参数列表），以及可选的编译步骤创建的输出名称。

# 二、生成编译数据库

要为您的项目获取编译数据库，您有多种选择：它可以由编译器、构建系统和专用工具生成，下面给出了一些例子：

## CMake:﻿

- 使用CMAKE_EXPORT_COMPILE_COMMANDS]标志. 你可以运行

    ```bash
    cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON…
    ```

    或将以下行添加到您的 CMakeLists.txt 脚本中：

    

    ```bash
    set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
    ```

    compile_commands.json 文件将被放入构建目录。

注意：CMAKE_EXPORT_COMPILE_COMMANDS 仅由 Makefile 和 Ninja 生成器实现。对于其他生成器，此选项将被忽略。



## Clang (version 5.0 and later):

-Mj 选项为每个输入文件写入一个编译条目。

可以应用于项目中的每个文件，然后将输出合并到 JSON 格式的编译数据库中。



## Ninja (version 1.2 and later):﻿

要获取编译数据库，请使用 -t compdb 选项。请注意，它需要规则名称作为参数： -t compdb rule1 rule2...

例如：

```
rule cc
    command = gcc -c -o $out $in
    description = CC $out

rule link
    command = gcc -o $out $in
    description = LINK $out
```

要在只有一个名为 cc 的规则的情况下生成编译数据库，请指定：

```
-t compdb cc > compile_commands.json
```

但是对于多个规则，您需要从构建文件中获取它们的确切名称，并将它们传递给 compdb。



## 基于make的项目：

compiledb-generator 工具为基于 make 的构建系统创建编译数据库。

https://github.com/nickdiego/compiledb



## SourceTrail Visual Studio 扩展：

SourceTrail Extension 为 MS Visual Studio 解决方案生成一个编译数据库。



# 三、在 CLion 中使用编译数据库

## 3、1 加载项目

为项目创建编译数据库后，可以在 CLion 中加载它。

导航到 File | Open 主菜单上，选择 compile_commands.json 文件或包含它的目录，然后单击 Open as Project。

结果，项目文件被检测到，compile_commands.json 中所有命令的状态显示在![App toolwindows tool window build](picture/app.toolwindows.toolWindowBuild.svg) Build工具窗口中：



未完成，待续！！！

未完成，待续！！！

未完成，待续！！！



## 3、2 特殊场景（技巧）

比如titan-agent工程的makefile文件位于/home/work/guanji/titan-agent/build/linux/TitanAgent，如果直接执行如下命令：

```
bear -- make #使用bear构建编译数据库
compiledb make #使用compiledb构建编译数据库
```

那么生成的编译数据库compile_commands.json是和makefile在同级目录中，CLion 默认在项目根目录中查找 `compile_commands.json` 文件，并根据其中的编译信息进行代码导航和代码补全等操作。

要解决这个问题，您可以尝试以下几种方法：

1. 复制 compile_commands.json 文件到项目根目录
  将生成的 compile_commands.json 文件复制到包含源代码的项目根目录中。这样，CLion 将能够找到并正确解析该文件。

2. 使用软链接（symbolic link）
  如果您不想将 compile_commands.json 文件复制到项目根目录，您可以尝试使用软链接将该文件链接到项目根目录。

  打开终端，并将当前目录切换到项目根目录，然后执行以下命令（假设 compile_commands.json 文件位于其他位置）

  ```
  ln -s /path/to/compile_commands.json compile_commands.json
  ```

  这将在项目根目录中创建一个名为 `compile_commands.json` 的软链接，指向实际的 `compile_commands.json` 文件。然后重新加载项目或重新启动 CLion，它应该能够找到并解析软链接指向的文件。

这算是一种技巧。



然后设置完毕后，可以进行debug单步调试了。



参考资料

https://www.jetbrains.com/help/clion/compilation-database.html