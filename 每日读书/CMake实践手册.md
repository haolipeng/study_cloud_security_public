查看CMake工具版本

cmake --version



ubuntu系统可通过命令sudo apt install build-essential 完成构建系统的安装，包含对 gcc、g++ 和 make 的安装。



使用CMake命令行应该了解的三件事

- 配置
- 构建
- 安装



cmake -G "Unix Makefiles" -S . -B ./build

使用"Unix Makefiles"生成器在当前目录（-S .）生成CMake项目的构建系统(-B ./build)目录。



**修改构建类型（CMAKE_BUILD_TYPE变量）**

cmake -G "Unix Makefiles" -DCMAKE_BUILD_TYPE:STRING=Release -S . -B ./build



**更换生成器类型**

通过-G参数可显式指定生成器。

若想使用Ninja作为构建系统，而不是make，可以这样：

cmake -G "Ninja" -DCMAKE_BUILD_TYPE:STRING=Debug -S . -B ./build



**修改编译器**

对于C/C++项目，通常修改的是

CMAKE_C_COMPILER(C 编译器) 和 CMAKE_CXX_COMPILER(C++ 编译器)。编译器标志由

CMAKE_CXX_FLAGS或CMAKE_C_FLAGS 控制。



缓存变量列表

cmake --build ./build 和下面是相等

cd build && make进行构建



**并行构建**

cmake --build ./build --parallel 2

其中数字2指定了任务数。在多核系统中，建议至少使用比可用硬件线程数少一个线程。



**只构建特定的目标**

CMake允许通过--target子选项构建目标的子集，子选项可以指定多次：

cmake --build ./build/ --target "ch2_framework_component1" --target "ch2_framework_component2"



**构建之前删除以前的构建**

可以使用--clean-first子选项，它会调用一个特殊的目标，该目标清除构建过程中的所有内容。



**调试构建过程**

如果希望检查构建过程中，使用了哪些参数调用了哪些命令，--verbose指示CMake以verbose模式调用所有构建命令。让我们能轻松的调试编译和链接错误。



**通过CLI安装项目**

cmake --install 需要一个已配置和生成的项目。执 行 cmake --install <project_binary_dir> 来安装工程。我们的例子中，build 作为项目二进制目录，所以

<project_binary_dir> 将为 build。

在尝试安装项目之前，必须已经构建了项目。

为了能够成功安装项目，必须拥有适当的权限来写入安装目标目录。



**修改默认安装路径**

cmake --install build --prefix /tmp/example

将安装目录指定为/tmp/example



安装特定组件(基于组件的安装)



## 3、2 创建项目

```
├── CMakeLists.txt
├── build
├── include/project_name
└── src
```

### 3、2、1 嵌套项目

```
── CMakeLists.txt
├── build
├── include/project_name
├── src
└── subproject
├── CMakeLists.txt
├── include
│ └── subproject
└── src
```

当项目嵌套时，每个项目应该编写CMakeLists.txt以便子项目可以独立构建。每个子项目的CMakeLists.txt文件应该指定cmake_minimum_required，以及可选的项目定义。













