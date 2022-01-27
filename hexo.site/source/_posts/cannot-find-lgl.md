---
title: 'cannot find -lGL'
date: 2021-03-25 21:49:05
category: Qt
tags: ['ld', 'Qt5', 'Ubuntu']
---

在 Ubuntu 18.04 LTS 上编译 Qt 库时发生了错误，提示如下：

```
/usr/bin/ld: cannot find -lGL
collect2: error: ld returned 1 exit status
```

这是由于缺少了 [OpenGL](https://en.wikipedia.org/wiki/OpenGL) 库。[Mesa 3D](https://en.wikipedia.org/wiki/Mesa_(computer_graphics)) 是一个开源的 OpenGL 实现，可以通过安装它来解决问题。

在 Ubuntu 上可以执行以下命令来安装：

```bash
sudo apt install libgl1-mesa-dev
```

通过 `locate` 命令，可以看到 GL 库已安装：

```bash
locate libGL.so
/usr/lib/i386-linux-gnu/libGL.so
/usr/lib/i386-linux-gnu/libGL.so.1
/usr/lib/i386-linux-gnu/libGL.so.1.0.0
/usr/lib/x86_64-linux-gnu/libGL.so.1
/usr/lib/x86_64-linux-gnu/libGL.so.1.0.0
```
