---
title: '为 Qt5 添加 MQTT 模块'
date: 2021-01-13 19:41:00
tags: ['MQTT', 'Qt5']
---

[MQTT](http://mqtt.org/) 是一种利用发布/订阅范式的机对机（M2M）协议，目前在物联网应用中十分流行。

在 Qt5 中，Qt 官方的 MQTT 模块属于 [Qt for Automation](https://doc.qt.io/QtForAutomation/index.html) 的一部分。因此商用版的 Qt5 可以很方便地使用 MQTT 模块，使用方法请参考官方文档 [Qt MQTT](https://doc.qt.io/QtMQTT/index.html) 。

然而大部分个人开发者（比如我）都用不起商用版，所以本文介绍如何为开源版 Qt5 添加 MQTT 模块，使得开源版也可以像商业版一样方便地使用 MQTT 模块。

> 提醒：QtMQTT 采用 [Qt Company](https://www.qt.io/company) 和 [GPLv3](https://www.gnu.org/licenses/gpl-3.0.html) 两种许可进行分发。开源版的分发许可为 GPLv3，因此商业应用需要考虑许可问题。

本文假定读者有基本的 Qt 开发基础和 Git 使用基础，并已安装 Qt5 的开发工具链（建议使用官方的安装包进行安装）和 Git 工具。Windows 环境下请确保已安装 [perl](https://www.perl.org/)，因为 Qt 的命令工具 `syncqt.pl` 需要 perl。

# 下载 QtMQTT 源码

目前有两个地方可以获取 QtMQTT 的源码：

* [Qt Code Review](https://codereview.qt-project.org/admin/repos/qt%2Fqtmqtt)
* [GitHub](https://github.com/qt/qtmqtt)

这里使用 GitHub 地址为例，拉取源码库到本地：

```bash
git clone https://github.com/qt/qtmqtt.git
```

切换工作目录到源码库目录，并变更分支到对应的 Qt 版本（这里以 5.15.0 为例）：

```bash
cd qtmqtt
git checkout 5.15.0
```

# 在 Linux 下编译并安装 QtMQTT

在 `qtmqtt` 目录下使用 `qmake` 命令生成用于构建模块的 Makefile 文件。

```bash
qmake
```

构建模块，并将模块安装到 Qt 的安装目录中（详情见 Makefile 文件）：

```bash
make all
# 此处省略构建输出的信息
# ...

make install
# 此处省略安装输出的信息
# ...
```

## 安装 QtMQTT 文档

```bash
make docs
make install_docs
```

> 注意：在 Linux 下构建的文档可能无法正常显示（由于 `qtmqtt.qch` 文件有错）。可以在 Windows 下构建文档然后将其 `qtmqtt\doc` 中的 `qtmqtt.qch` 文件复制到 Linux 下的 Qt 安装目录的文档目录下（如：`Qt/Docs/Qt-5.15.0`）。

## 安装 QtMQTT 示例

```bash
make sub-examples-install_subtargets
```

# 在 Windows 下编译并安装 QtMQTT

在 Windows 下进行编译需要用到 MSVC。以 VS2019 为例，默认安装下 VS2019 并不会将 MSVC 的运行环境配置到系统的 `PATH` 中，官方给出的建议是使用 VS2019 提供的批处理文件来自动配置 MSVC 的运行环境。这些批处理文件位于 `C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\VC\Auxiliary\Build` 中，其中 `vcvars32.bat` 用于配置 32-bit 的运行环境；`vcvars64.bat` 用于配置 64-bit 的运行环境。

> 注意：`MinGW` 用户请参考 Linux。

以 64-bit 为例，打开 `cmd`，切换其工作目录到 `C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\VC\Auxiliary\Build` 然后执行批处理文件：

```powershell
vcvars64.bat
```

在 Windows 下，Qt 的命令工具同样也需要配置运行环境。Qt 官方提供了批处理文件 `qtenv2.bat` 用于配置 Qt 命令工具的运行环境。这个批处理文件位于 Qt 安装目录下对应编译器目录的 `bin` 目录下。

切换工作目录到 Qt 的 `bin` 目录下（如 `C:\Qt\5.15.0\msvc2019_64\bin`），然后执行 `qtenv2.bat` 以配置 Qt 命令工具的运行环境：

```powershell
qtenv2.bat
```

将工作目录切换到 QtMQTT 的源码库目录，执行 `qmake` 命令生成用于构建模块的 Makefile 文件：

```powershell
qmake
```

构建模块，并将模块安装到 Qt 的安装目录中（详情见 Makefile 文件）：

```powershell
nmake all
# 此处省略构建输出的信息
# ...

nmake install
# 此处省略安装输出的信息
# ...
```

## 安装 QtMQTT 文档

```powershell
nmake docs
nmake install_docs
```

## 安装 QtMQTT 示例

```powershell
nmake sub-examples-install_subtargets
```

# 最后

现在你可以像商业版一样方便地使用 MQTT 模块了。

简单用法可以参考官方文档 [Qt MQTT](https://doc.qt.io/QtMQTT/index.html) 。

更详细的用法可以参考 QtMQTT 文档和示例。
