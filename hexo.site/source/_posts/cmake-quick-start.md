---
title: 'CMake 快速入坑'
date: 2020-12-30 19:05:49
tags: 'CMake'
---

[CMake](https://cmake.org/) 是一个开源、跨平台的工具系列，可用于构建、测试和打包软件。它通过一个名为 `CMakeLists.txt` 的配置文件来管理软件的编译流程，并根据用户所选择的目标平台生成构建软件所需的本地化 makefile 或 workspace。通俗来讲，使用 CMake 可以生成 UNIX-like 上构建软件所需的 Makefile 和 Windows 上构建软件所需的 vcxproj，而无需为它们单独写一份 Makefile/vcxproj 。

[CMake 官方文档](https://cmake.org/documentation/)中有详细的使用手册，可以帮助用户更深入地了解 CMake。另外，还有一个详细的使用教程：[CMake Tutorial](https://cmake.org/cmake/help/latest/guide/tutorial/index.html)。

本文以 Ubuntu 20.04 为例，介绍一些 CMake 的日常用法。

# 安装

Ubuntu 20.04 下安装 CMake 是十分方便的。通过以下命令安装 GNU C++ compiler、GNU Make 和 CMake：

```bash
sudo apt update
sudo apt install g++ make cmake
```

# Hello World

创建一个最小组织结构 `hello`：

```bash
mkdir hello && cd hello && mkdir build && touch CMakeLists.txt hello.cpp
```

该组织结构的树图为：

```
hello/
├── build
├── CMakeLists.txt
└── hello.cpp
```

> 利用 `build` 目录来存放 `cmake` 的输出可以避免项目组织结构被污染。

将 `hello.cpp` 的内容改为：

```cpp
#include <iostream>

int main()
{
    std::cout << "Hello, World!" << std::endl;
}
```

`CMakeLists.txt` 的内容改为：

```cmake
# 最低的 CMake 版本要求
cmake_minimum_required(VERSION 3.0.0)

# 设置项目名
project(hello)

# 添加一个可执行文件
add_executable(hello hello.cpp)
```

> CMake 使用 `#` 来注释一行。

将工作目录切换到 `build` 目录，然后运行 `cmake` 命令生成本地化的 makefile：

```bash
cd build/
cmake ../
```

生成完成后，通过 `ls` 命令可以看到构建软件所需的 `Makefile` 文件。此时可以使用以下命令来构建软件：

```bash
cmake --build .
```

执行构建所得的可执行文件 `hello`，将得到预期的输出：

```bash
$ ./hello 
Hello, World!
```

# 各种项目组织形式

## 单个源文件

单个源文件的场景请参考 [Hello World](#hello-world)，如果源文件与 `CMakeLists.txt` 不在同一个目录下，例如：

```
hello2/
├── build
├── CMakeLists.txt
└── src
    └── hello.cpp
```

只需要将：

```cmake
add_executable(hello hello.cpp)
```

改为：

```cmake
add_executable(hello src/hello.cpp)
```

## 多个源文件

多个源文件的场景分两种：单个目录和多个目录。

### 单个目录

创建一个具有多个源文件-单个目录的组织结构 `adder`：

```bash
mkdir adder && cd adder && mkdir build && touch CMakeLists.txt main.cpp adder.cpp adder.h
```

该组织结构的树图为：

```
adder/
├── adder.cpp
├── adder.h
├── build
├── CMakeLists.txt
└── main.cpp
```

将 `adder.h` 的内容改为：

```cpp
#ifndef ADDER_H
#define ADDER_H
double add(double a, double b);
#endif
```

`adder.cpp` 的内容改为：

```cpp
double add(double a, double b)
{
    return a + b;
}
```

`main.cpp` 的内容改为：

```cpp
#include "adder.h"
#include <iostream>

int main()
{
    std::cout << "1 + 2 = "
              << add(1, 2)
              << std::endl;
}
```

`CMakeLists.txt` 的内容改为：

```cmake
# 最低的 CMake 版本要求
cmake_minimum_required(VERSION 3.0.0)

# 设置项目名
project(adder)

# 添加一个可执行文件
add_executable(adder main.cpp adder.cpp)
```

将工作目录切换到 `build` 目录，然后运行 `cmake` 命令生成本地化的 makefile，并构建软件：

```bash
cd build/
cmake ../
cmake --build .
```

执行构建所得的可执行文件 `adder`，将得到预期的输出：

```bash
$ ./adder 
1 + 2 = 3
```

这里稍微说明一下 CMake 的 `add_executable` 命令，它的完整语法为：

```
add_executable(<name> [WIN32] [MACOSX_BUNDLE]
               [EXCLUDE_FROM_ALL]
               [source1] [source2 ...])
```

它的作用为从命令所指定的源文件/列表（`[source1] [source2 ...]`）中构建一个可执行文件目标（`<name>`）。

将源文件手工地一个个添加到 `add_executable` 中虽然可行，但不是一种好的办法。CMake 提供了一个 `aux_source_directory` 命令，它可以帮助用户解决这种手工烦恼，其语法为：

```
aux_source_directory(<dir> <variable>)
```

该命令会将指定目录（`<dir>`）下的所有源文件名收集成一个列表，并存放在变量（`<variable>`）中。所以，我们可以将 `CMakeLists.txt` 的内容改为：

```cmake
# 最低的 CMake 版本要求
cmake_minimum_required(VERSION 3.0.0)

# 设置项目名
project(adder)

# 获取目录下的所有源文件
aux_source_directory(. SRC_LIST)

# 添加一个可执行文件
add_executable(adder ${SRC_LIST})
```

> CMake 中，引用变量的语法为 `${variable_name}`。

### 多个目录

创建一个具有多个源文件-多个目录的组织结构 `adder2`：

```bash
mkdir adder2 && cd adder2 && mkdir build math && touch CMakeLists.txt main.cpp && touch math/adder.cpp math/adder.h
```

该组织结构的树图为：

```
adder2/
├── build
├── CMakeLists.txt
├── main.cpp
└── math
    ├── adder.cpp
    └── adder.h
```

`main.cpp`、`adder.h` 和 `adder.cpp` 的内容与[单个源文件](#单个源文件)中的相同。

尝试参考[单个源文件](#单个源文件)中的做法，将 `CMakeLists.txt` 的内容改为：

```cmake
# 最低的 CMake 版本要求
cmake_minimum_required(VERSION 3.0.0)

# 设置项目名
project(adder)

# 获取目录下的所有源文件
aux_source_directory(. SRC_LIST)
aux_source_directory(math MATH_SRC_LIST)

# 添加一个可执行文件
add_executable(adder ${SRC_LIST} ${MATH_SRC_LIST})
```

将工作目录切换到 `build` 目录，然后运行 `cmake` 命令生成本地化的 makefile，并构建软件：

```bash
cd build/
cmake ../
cmake --build .
```

然而，这次构建发生了错误：

```bash
$ cmake --build .
Scanning dependencies of target adder
[ 33%] Building CXX object CMakeFiles/adder.dir/main.cpp.o
/home/dsyx/cmake.demo/adder2/main.cpp:1:10: fatal error: adder.h: No such file or directory
    1 | #include "adder.h"
      |          ^~~~~~~~~
compilation terminated.
make[2]: *** [CMakeFiles/adder.dir/build.make:63: CMakeFiles/adder.dir/main.cpp.o] Error 1
make[1]: *** [CMakeFiles/Makefile2:76: CMakeFiles/adder.dir/all] Error 2
make: *** [Makefile:84: all] Error 2
```

根据输出的错误信息，可以知道找不到头文件 `adder.h`。在 `CMakeLists.txt` 中添加以下命令可以修正这个错误：

```cmake
# 将 math 目录添加到编译器的 include 搜索列表中
include_directories(math)
```

然而，在 CMake 中更推荐的组织方法是将子目录作为库来看待。创建一个具有多个源文件-多个目录的组织结构 `adder3`：

```bash
mkdir adder3 && cd adder3 && mkdir build math && touch CMakeLists.txt main.cpp && touch math/adder.cpp math/adder.h math/CMakeLists.txt
```

该组织结构的树图为：

```
adder3
├── build
├── CMakeLists.txt
├── main.cpp
└── math
    ├── adder.cpp
    ├── adder.h
    └── CMakeLists.txt
```

`main.cpp`、`adder.h` 和 `adder.cpp` 的内容与[单个源文件](#单个源文件)中的相同。

将 `math/CMakeLists.txt` 的内容修改为：

```cmake
# 获取目录下的所有源文件
aux_source_directory(. MATH_SRC_LIST)
# 添加一个库
add_library(math ${MATH_SRC_LIST})
```

`CMakeLists.txt` 的内容修改为：

```cmake
# 最低的 CMake 版本要求
cmake_minimum_required(VERSION 3.0.0)

# 设置项目名
project(adder)

# 添加子目录
add_subdirectory(math)

# 获取目录下的所有源文件
aux_source_directory(. SRC_LIST)

# 添加一个可执行文件
add_executable(adder ${SRC_LIST})

# 指定目标要链接的库
target_link_libraries(adder math)

# 指定在编译给定目标时要使用到的 include 目录
target_include_directories(adder PUBLIC
                          "${PROJECT_SOURCE_DIR}/math"
                          )
```

将工作目录切换到 `build` 目录，然后运行 `cmake` 命令生成本地化的 makefile，并构建软件：

```bash
cd build/
cmake ../
cmake --build .
```

执行构建所得的可执行文件 `adder`，将得到预期的输出：

```bash
$ ./adder 
1 + 2 = 3
```

# 安装和测试

本节以[多个目录](#多个目录)中的 `adder3` 作为基础。

在 `math/CMakeLists.txt` 中添加以下内容：

```cmake
# 将 math 库安装到 ${CMAKE_INSTALL_PREFIX}/lib 中
install(TARGETS math DESTINATION lib)
# 将 adder.h 文件安装到 adder.h ${CMAKE_INSTALL_PREFIX}/include 中
install(FILES adder.h DESTINATION include)
```

在 `CMakeLists.txt` 中添加以下内容：

```cmake
# 将可执行文件 adder 安装到 ${CMAKE_INSTALL_PREFIX}/bin 中
install(TARGETS adder DESTINATION bin)
```

`install` 命令用来指定要在安装时运行的规则，它的语法如下：

```
install(TARGETS <target>... [...])
install({FILES | PROGRAMS} <file>... [...])
install(DIRECTORY <dir>... [...])
install(SCRIPT <file> [...])
install(CODE <code> [...])
install(EXPORT <export-name> [...])
```

该命令可用的规则十分之多，这里仅说明目前用到的：

* `TARGETS` 指定从项目中安装目标的规则，一般用于指定可执行文件和库。
* `DESTINATION` 指定磁盘上要安装文件的目录。参数可以是相对或绝对路径。如果是相对路径则会使用 `CMAKE_INSTALL_PREFIX` 变量作为该目录的前缀。
* `FILES` 指定安装项目文件的规则。如果使用的是相对路径则该路径将相对于当前源目录进行解释。

> `CMAKE_INSTALL_PREFIX` 有一个默认值，在 UNIX-like 上为 `/usr/local`；在 Windows 上为 `c:/Program Files/${PROJECT_NAME}`。

将工作目录切换到 `build` 目录，然后运行 `cmake` 命令生成本地化的 makefile，并构建软件：

```bash
cd build/
cmake ../
cmake --build .
```

使用 `cmake --install .` 进行安装，这里使用 `--prefix` 来替代 `CMAKE_INSTALL_PREFIX` 的值，放置污染系统目录：

```bash
cmake --install . --prefix ~/cmake.installdir
```

安装后可以看到用户目录下存在 `cmake.installdir`，其组织树图为：

```
cmake.installdir
├── bin
│   └── adder
├── include
│   └── adder.h
└── lib
    └── libmath.a
```

这里冒出了一个问题，如何删除 CMake 安装的软件呢？CMake 并没有提供 `uninstall` 命令，但是在 `cmake --install` 后，会生成一个 `install_manifest.txt` 文件，这个文件描述了安装的详情。

为了演示测试功能，我们修改 `main.cpp` 的内容，使其接受命令行参数（为了方便不做任何异常处理）：

```cpp
#include "adder.h"
#include <iostream>
#include <string>

int main(int argc, char *argv[])
{
    double a = std::stod(argv[1]);
    double b = std::stod(argv[2]);

    std::cout << a
              << " + "
              << b
              << " = "
              << add(a, b)
              << std::endl;
}
```

在 `CMakeLists.txt` 的末尾添加以下内容：

```cmake
# 启用对此目录及其子目录的测试
enable_testing()

# 添加一个名为 Run 的测试，以测试程序能否正常运行
add_test(NAME Run COMMAND adder 1 2)

# 添加一个名为 Test_0_0 的测试，以测试 0 + 0 的结果是否符合预期
add_test(NAME Test_0_0 COMMAND adder 0 0)
set_tests_properties(Test_0_0 PROPERTIES PASS_REGULAR_EXPRESSION "0 \\+ 0 = 0")

# 添加一个名为 Test_n1_1 的测试，以测试 -1 + 1 的结果是否符合预期
add_test(NAME Test_n1_1 COMMAND adder -1 1)
set_tests_properties(Test_n1_1 PROPERTIES PASS_REGULAR_EXPRESSION "-1 \\+ 1 = 0")

# 添加一个名为 Test_1_100 的测试，以测试 1 + 100 的结果是否符合预期
add_test(NAME Test_1_100 COMMAND adder 1 100)
set_tests_properties(Test_1_100 PROPERTIES PASS_REGULAR_EXPRESSION "1 \\+ 100 = 101")

# 添加一个名为 Test_n1_n100 的测试，以测试 -1 + -100 的结果是否符合预期
add_test(NAME Test_n1_n100 COMMAND adder -1 -100)
set_tests_properties(Test_n1_n100 PROPERTIES PASS_REGULAR_EXPRESSION "-1 \\+ -100 = -101")
```

`add_test` 命令将测试添加到要由 `ctest` 运行的项目中，它的语法是：

```
add_test(NAME <name> COMMAND <command> [<arg>...]
         [CONFIGURATIONS <config>...]
         [WORKING_DIRECTORY <dir>]
         [COMMAND_EXPAND_LISTS])
```

`set_tests_properties` 命令用于设置测试的属性，它的语法是：

```
set_tests_properties(test1 [test2...] PROPERTIES prop1 value1 prop2 value2)
```

其中 `PROPERTIES` 可以为：

* `WILL_FAIL`：如果设置了该属性，则将反转测试的通过/失败标志。
* `PASS_REGULAR_EXPRESSION`：如果设置了该属性，则会根据给出的正则表达式检查测试输出，如果匹配则通过。
* `FAIL_REGULAR_EXPRESSION`：如果设置了该属性，则会根据给出的正则表达式检查测试输出，如果匹配则失败。
* `TIMEOUT`：设置该属性会将测试运行时间限制为指定的秒数。

将工作目录切换到 `build` 目录，然后运行 `cmake` 命令生成本地化的 makefile，并构建软件：

```bash
cd build/
cmake ../
cmake --build .
```

使用 `ctest` 命令执行测试：

> `ctest` 可执行文件是 CMake 测试驱动程序。

```bash
$ ctest
Test project /home/dsyx/cmake.demo/adder3/build
    Start 1: Run
1/5 Test #1: Run ..............................   Passed    0.00 sec
    Start 2: Test_0_0
2/5 Test #2: Test_0_0 .........................   Passed    0.00 sec
    Start 3: Test_n1_1
3/5 Test #3: Test_n1_1 ........................   Passed    0.00 sec
    Start 4: Test_1_100
4/5 Test #4: Test_1_100 .......................   Passed    0.00 sec
    Start 5: Test_n1_n100
5/5 Test #5: Test_n1_n100 .....................   Passed    0.00 sec

100% tests passed, 0 tests failed out of 5

Total Test time (real) =   0.01 sec
```
