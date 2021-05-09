---
title: '解决 Qt5 显示乱码问题'
date: 2021-05-09 16:54:48
tags: ['MSVC', 'Qt5', 'Windows']
---

对于在 Windows 上进行 Qt 开发的小伙伴来说，乱码可能是一个较为头疼的问题。

[Qt for Windows](https://doc.qt.io/qt-5/windows.html) 支持两种编译器：MSVC 和 MinGW-GCC。本文将介绍使用这两种编译器而出现乱码问题的解决办法。如果在 Linux 上也出现了乱码，则可以参考 MinGW-GCC 的做法。

# MinGW-GCC

如果使用的编译器是 MinGW-GCC，一般来说不会遇到乱码问题。出现乱码问题很大可能是因为源文件的编码格式不是 UTF-8（通常是由于代码在 Windows 与 Linux 间共享而造成的）。这时候请检查你自己编写的源文件是否为其他编码格式，若是，则统一改为 UTF-8 编码后再次构建工程。

在 Qt Creator 中可以通过 Tools > Options... > Text Editor > Behavior 中的 File Encodings 下的 Default encoding 更改默认的文本编码格式，建议设置为 UTF-8。

# MSVC

使用 MSVC 编译器遇到乱码问题的可能性会相对高一点。这是因为 MSVC 在文本文件不带 BOM 时会将文件的编码格式假定为当前系统区域设置所对应的编码格式，比如 Windows 中文版的默认编码格式为 GBK。换句话说，如果源文件的编码格式为 UTF-8，那么在 Windows 中文版下 MSVC 会将这些源文件当作 GBK 编码格式处理，这就很容易导致乱码。

> 建议将源文件统一编码为 UTF-8，方便不同平台的共享。

为了解决这个问题，MSVC 提供了编译选项 `/utf-8`，该选项将源文件字符集和执行字符集都指定为 UTF-8。

如果使用 qmake 作为项目构建工具，则可以在 `.pro` 文件中添加：

```
win32:msvc:QMAKE_CXXFLAGS += /utf-8
```

如果使用 CMake 作为项目构建工具，则可以在 `CMakeLists.txt` 文件中添加：

```cmake
if (MSVC)
    add_compile_options(/utf-8)
endif()
```

修改后重新构建项目即可解决乱码问题。
