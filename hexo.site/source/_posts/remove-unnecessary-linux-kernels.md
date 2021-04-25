---
title: '移除不必要的 Linux 内核'
date: 2021-04-25 21:34:01
tags: ['Linux', 'Ubuntu']
---

在基于 Linux 内核的系统中，随着内核的更新，磁盘中可能会存留大量的旧内核。由于这些旧内核几乎不会再被使用，所以我们可以手动清理它们以腾出更多的磁盘空间。本文介绍 Ubuntu Linux 下的操作流程。

> 注：移除内核具有一定的风险，执行这些操作的前提是你明白自己在干什么！！！

# 查看当前内核信息

在移除旧内核前，我们应该查看当前正在使用的内核：

```bash
uname -r
```

该命令将输出如下相似的内容：

```
5.4.0-72-generic
```

# 查看已安装的内核

使用如下命令可以获得系统中已安装的内核列表：

```bash
dpkg --list | grep linux-image
```

该命令将输出如下相似的内容（不包含前五行，这是为了方便读者理解每列的含义而添加的）：

```
Desired=Unknown/Install/Remove/Purge/Hold
| Status=Not/Inst/Conf-files/Unpacked/halF-conf/Half-inst/trig-aWait/Trig-pend
|/ Err?=(none)/Reinst-required (Status,Err: uppercase=bad)
||/ Name                                       Version                                          Architecture Description
+++-==========================================-================================================-============-============================
rc  linux-image-5.4.0-42-generic               5.4.0-42.46~18.04.1                              amd64        Signed kernel image generic
rc  linux-image-5.4.0-67-generic               5.4.0-67.75~18.04.1                              amd64        Signed kernel image generic
rc  linux-image-5.4.0-70-generic               5.4.0-70.78~18.04.1                              amd64        Signed kernel image generic
ii  linux-image-5.4.0-71-generic               5.4.0-71.79~18.04.1                              amd64        Signed kernel image generic
ii  linux-image-5.4.0-72-generic               5.4.0-72.80~18.04.1                              amd64        Signed kernel image generic
ii  linux-image-generic-hwe-18.04              5.4.0.72.80~18.04.65                             amd64        Generic Linux kernel image
```

# 移除旧内核

在上一节的输出中，我们可以确定自己需要移除的内核版本（如上节输出中，可以选择移除 `linux-image-5.4.0-71-generic`）。

使用包管理工具移除内核（以 `linux-image-5.4.0-71-generic` 为示例）：

```bash
sudo apt purge linux-image-5.4.0-71-generic
```
