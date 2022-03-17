---
title: 'VSCode 嵌入式项目设置'
date: 2022-03-17 09:21:28
category: VSCode
tags: ['VSCode', 'Embedded', 'STM32CubeMX']
---

现今，每家嵌入式芯片厂商基本上都推出了自家的 Eclipse-based IDE 以方便开发者进行快速开发，如 [ST](https://www.st.com/) 的 [STM32CubeIDE](https://www.st.com/en/development-tools/stm32cubeide.html)、[NXP](https://www.nxp.com/) 的 [MCUXpresso IDE](https://www.nxp.com/design/software/development-software/mcuxpresso-software-and-tools-/mcuxpresso-integrated-development-environment-ide:MCUXpresso-IDE)。

然而，Eclipse 的编辑功能个人认为并不好用，并且这些 Eclipse-based IDE 运行得并不流畅。因此，我决定自己搭建一个开发环境来替代这些 Eclipse-based IDE。本文以 ST 方案为例，因为我的机器上正好有 [STM32CubeMX](https://www.st.com/en/development-tools/stm32cubemx.html)。

## 先决条件

假定已安装了如下软件：

* [VSCode](https://code.visualstudio.com/)
* [STM32CubeMX](https://www.st.com/en/development-tools/stm32cubemx.html)
* [GNU Make](https://www.gnu.org/software/make/)
* [GNU Arm Embedded Toolchain](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-rm)
* [J-Link](https://www.segger.com/downloads/jlink/)

> 注：
> 
> 请确保将 [GNU Arm Embedded Toolchain](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-rm) 添加到环境变量 `PATH` 中。
> 
> 在 Windows 下，不建议将 [J-Link](https://www.segger.com/downloads/jlink/) 添加到环境变量 `PATH` 中，因为它可能会被 JDK 的 `jlink` 命令覆盖。

## 创建项目

使用 [STM32CubeMX](https://www.st.com/en/development-tools/stm32cubemx.html) 创建一个新项目，配置好你板子上的外设及时钟，确保启用了调试功能（根据实际选择调试方式）：

![](https://i.postimg.cc/FH3mm8FR/debug-mode-and-configuration.png)

在 **Project Manager** 页下命名项目，并将 **Toolchain/IDE** 改为 **Makefile**：

![](https://i.postimg.cc/28qr93Pn/project-manager.png)

点击 **GENERATE CODE** 以生成项目。项目的目录结构应该如下：

```bash
$ tree -aCL 2 .
.
├── .mxproject
├── Core
│   ├── Inc
│   └── Src
├── Drivers
│   ├── CMSIS
│   └── STM32H7xx_HAL_Driver
├── Makefile
├── STM32H743IITx_FLASH.ld
├── led-demo.ioc
└── startup_stm32h743xx.s
```

## 安装 VSCode 插件

使用 VSCode 打开生成的项目目录（示例中为 `C:\workspace\led-demo`）。点击左侧栏中的扩展，搜索并安装以下插件：

* [C/C++](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools) —— 用于 C/C++ 的智能提示、调试和代码查阅。
* [Embedded Tools](https://marketplace.visualstudio.com/items?itemName=ms-vscode.vscode-embedded-tools) —— 用于调试嵌入式设备时查看其寄存器及 RTOS 数据。

![](https://i.postimg.cc/J7sXP2Fd/vscode-plugin-embedded-tools.png)

## 配置项目

在项目的根目录下创建 `.vscode` 目录，该目录用于存放工作区设置。此时，项目的目录结构应该如下：

```bash
$ tree -aCL 1 .
.
├── .mxproject
├── .vscode
├── Core
├── Drivers
├── Makefile
├── STM32H743IITx_FLASH.ld
├── led-demo.ioc
└── startup_stm32h743xx.s
```

### 添加 CMSIS-SVD 文件

[CMSIS-SVD](https://www.keil.com/pack/doc/CMSIS/SVD/html/index.html) 文件用于描述 Arm Cortex-M 芯片的详细系统信息，主要是外设的内存映射寄存器。目前没有一个用于统一存储所有 Arm Cortex-M 芯片的 CMSIS-SVD 文件的仓库。但是有一个可用的 Github 仓库，其存储了大量的 CMSIS-SVD 文件：[https://github.com/posborne/cmsis-svd](https://github.com/posborne/cmsis-svd)，你可以按需下载。

将需要的 CMSIS-SVD 文件保存到项目的根目录下，如本示例（`STM32H743x.svd`）：

```bash
$ tree -aCL 1 .
.
├── .mxproject
├── .vscode
├── Core
├── Drivers
├── Makefile
├── STM32H743IITx_FLASH.ld
├── STM32H743x.svd
├── led-demo.ioc
└── startup_stm32h743xx.s
```

### 添加 J-Link 命令文件

[J-Link Commander](https://wiki.segger.com/J-Link_Commander) 可以使用 [J-Link 命令文件](https://wiki.segger.com/J-Link_Commander#Using_J-Link_Command_Files) 以在批处理模式下执行烧写固件任务。在项目的根目录下创建文件 `flash.jlink`，并根据实际情况填写如下内容：

```
r
h
loadfile build/led-demo.hex
q
```

### 添加 gdbinit 文件

在项目的根目录下创建文件 `reset.gdb`，并填写如下内容：

```
reset
```

[`reset`](https://wiki.segger.com/J-Link_GDB_Server#reset) 命令用于重置并停止目标 CPU。确保在使用此命令之前选择设备以使用正确的复位策略。

### 添加调试设置文件

在 `.vscode` 目录下创建 [`launch.json`](https://code.visualstudio.com/docs/editor/debugging) 文件，并根据实际情况填写如下内容，该文件用于自定义工作区的运行和调试：

```json
{
    "configurations": [
        {
            "name": "Build, Flash and Debug",
            "type": "cppdbg",
            "request": "launch",
            "cwd": "${workspaceFolder}",
            "program": "${workspaceFolder}/build/led-demo.elf",
            "stopAtEntry": true,
            "MIMode": "gdb",
            "miDebuggerPath": "C:/gcc-arm-none-eabi/bin/arm-none-eabi-gdb.exe",
            "miDebuggerServerAddress": "localhost:2331",
            "debugServerPath": "C:/Program Files/SEGGER/JLink/JLinkGDBServerCL.exe",
            "debugServerArgs": "-if swd -speed 4000 -endian little -device STM32H743II -x ${workspaceFolder}/reset.gdb",
            "svdPath": "${workspaceFolder}/STM32H743x.svd",
            "preLaunchTask": "Build and Flash"
        }
    ]
}
```

### 添加任务设置文件

在 `.vscode` 目录下创建 [`tasks.json`](https://code.visualstudio.com/docs/editor/tasks) 文件，并根据实际情况填写如下内容，该文件用于定义工作区的任务：

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "Build",
            "type": "shell",
            "command": "C:/msys64/usr/bin/make.exe",
            "group": {
                "kind": "build",
                "isDefault": true
            }
        },
        {
            "label": "Build and Flash",
            "type": "shell",
            "command": "C:/Program Files/SEGGER/JLink/JLink.exe",
            "args": [
                "-Device",
                "STM32H743II",
                "-If",
                "SWD",
                "-Speed",
                "4000",
                "-AutoConnect",
                "1",
                "-CommandFile",
                "flash.jlink"
            ],
            "dependsOn": "Build"
        }
    ]
}
```

### 添加 C/C++ 设置文件

在 `.vscode` 目录下创建 [`c_cpp_properties.json`](https://code.visualstudio.com/docs/cpp/customize-default-settings-cpp) 文件，并根据实际情况填写如下内容，该文件用于配置工作区的 C/C++ 插件设置：

```json
{
    "configurations": [
        {
            "name": "Win32",
            "includePath": [
                // 可以使用如下匹配进行目录递归，这将包含工作区的所有目录：
                //"${workspaceFolder}/**"

                // 单独列出路径可能性能更佳（猜测！）
                "${workspaceFolder}/Core/Inc",
                "${workspaceFolder}/Drivers/STM32H7xx_HAL_Driver/Inc",
                "${workspaceFolder}/Drivers/STM32H7xx_HAL_Driver/Inc/Legacy",
                "${workspaceFolder}/Drivers/CMSIS/Include",
                "${workspaceFolder}/Drivers/CMSIS/Device/ST/STM32H7xx/Include"
            ],
            "defines": [
                "USE_HAL_DRIVER",
                "STM32H743xx"
            ],
            "compilerPath": "C:/gcc-arm-none-eabi/bin/arm-none-eabi-gcc.exe",
            "cStandard": "c11",
            "cppStandard": "c++11",
            "intelliSenseMode": "gcc-arm"
        }
    ],
    "version": 4
}
```

## 调试项目

现在，项目的目录结构应该如下：

```bash
$ tree -aCL 2 .
.
├── .mxproject
├── .vscode
│   ├── c_cpp_properties.json
│   ├── launch.json
│   └── tasks.json
├── Core
│   ├── Inc
│   └── Src
├── Drivers
│   ├── CMSIS
│   └── STM32H7xx_HAL_Driver
├── Makefile
├── STM32H743IITx_FLASH.ld
├── STM32H743x.svd
├── flash.jlink
├── led-demo.ioc
├── reset.gdb
└── startup_stm32h743xx.s
```

点击 VSCode 左侧栏的调试，并点击调试按钮：

![](https://i.postimg.cc/28413ZVm/debug.png)

这将会编译项目，并将固件烧写到目标设备上，然后启动调试：

![](https://i.postimg.cc/P5TPD5wD/debugging.png)

你可以通过点击源文件的行号左侧以设置断点，也可以观察程序暂停时的变量值、寄存器及调用栈：

![](https://i.postimg.cc/GmK9xnF5/pause-debugging.png)
