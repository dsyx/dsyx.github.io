---
title: 'Nordic IoT 开发：起点'
date: 2022-07-25 13:58:09
category: 'IoT'
tags: ['nRF Connect SDK', 'Nordic Semiconductor']
---

[Nordic Semiconductor](https://www.nordicsemi.com/) 是一家总部位于挪威的无厂半导体公司，其专注于为物联网领域提供无线通信技术。截止到本文的编写日，Nordic 的物联网产品已经十分丰富，以下是一张从官网中截取的产品介绍图：

![Nordic Product Guide](/images/nordic-iot-dev-starting-point/nordic-product-guide.png)

Nordic 提供了 [nRF Connect SDK](https://www.nordicsemi.com/Products/Development-software/nrf-connect-sdk) 以方便开发者在 Nordic 的 nRF 系列芯片上进行应用开发。

> nRF Connect SDK 也托管在 GitHub 上：[https://github.com/nrfconnect/sdk-nrf](https://github.com/nrfconnect/sdk-nrf)。

nRF Connect SDK 包含了蜂窝式物联网（[LTE-M](https://www.gsma.com/iot/long-term-evolution-machine-type-communication-lte-mtc-cat-m1/) 和 [NB-IoT](https://www.gsma.com/iot/narrow-band-internet-of-things-nb-iot/)）、[Bluetooth Low Energy](https://www.bluetooth.com/learn-about-bluetooth/tech-overview/)、[Thread](https://www.threadgroup.org/)、[Zigbee](https://csa-iot.org/all-solutions/zigbee/) 和 [Bluetooth Mesh](https://www.bluetooth.com/learn-about-bluetooth/recent-enhancements/mesh/) 的示例和参考实现，以及 nRF 系列芯片的驱动实现。nRF Connect SDK 还包含了为资源受限的连接设备设计的 [Zephyr](https://zephyrproject.org/) 实时操作系统。

nRF Connect SDK 由各种代码库组成，并使用 [West](https://developer.nordicsemi.com/nRF_Connect_SDK/doc/latest/zephyr/develop/west/index.html#west) 工具进行管理。在这些代码库中值得注意的是：

* [sdk-nrf](https://github.com/nrfconnect/sdk-nrf) - 包含专门针对 nRF 系列芯片的应用程序、示例、库和驱动程序。
* [sdk-nrfxlib](https://github.com/nrfconnect/sdk-nrfxlib) - 包含二进制格式的闭源库和模块。
* [sdk-zephyr](https://github.com/nrfconnect/sdk-zephyr) - 包含 Zephyr 项目的一个分支，它提供了一个实时操作系统。
* [sdk-mcuboot](https://github.com/nrfconnect/sdk-mcuboot) - 包含 MCUboot 项目的一个分支，它提供了一个安全的引导加载程序。

nRF Connect SDK 使用了 Zephyr 项目的构建方法，下图展示了 nRF Connect SDK 的构建方法和使用的工具：

![nRF Connect SDK tools and configuration methods](/images/nordic-iot-dev-starting-point/ncs-toolchain.svg)

* [Kconfig](https://docs.zephyrproject.org/latest/build/kconfig/index.html) - 生成配置库和子系统的定义。
* [Devicetree](https://docs.zephyrproject.org/latest/build/dts/index.html) - 描述硬件。
* [CMake](https://docs.zephyrproject.org/latest/build/cmake/index.html) - 基于 `CMakeLists.txt` 文件生成构建文件。
* [Ninja](https://ninja-build.org/) - 类似于 make，使用构建文件来构建程序。
* [GCC](https://gcc.gnu.org/) - 创建可执行文件。

nRF Connect SDK 提供两种安装方式：[手动](https://developer.nordicsemi.com/nRF_Connect_SDK/doc/latest/nrf/gs_installing.html)和[自动](https://developer.nordicsemi.com/nRF_Connect_SDK/doc/latest/nrf/gs_assistant.html)。除非你熟悉 nRF Connect SDK 所使用的工具链，否则更建议你使用自动安装。

---

参考文档：

* [Welcome to the nRF Connect SDK!](https://developer.nordicsemi.com/nRF_Connect_SDK/doc/latest/nrf/index.html)
