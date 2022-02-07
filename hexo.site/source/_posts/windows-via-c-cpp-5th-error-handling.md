---
title: 'Windows via C/C++, 5th Edition - Error Handling'
date: 2022-01-28 10:19:37
category: 'My Notes'
tags: ['Windows via C/C++', 'Windows', 'C/C++']
---

调用 Windows 函数时，它会检查传参的合法性然后执行任务。如果传参非法或执行失败，那么系统将返回一个值以指明原因。下表列出大多数 Windows 函数的返回值的数据类型：

| 数据类型       | 说明  |
| :------------- | :-- |
| `VOID`         | 表明函数不可能发生错误。 |
| `BOOL`         | 失败时返回 0；否则返回非零值。避免测试这种函数的返回值是否为 `TRUE`，最好测试其是否为 `FALSE`。 |
| `HANDLE`       | 失败时通常会返回 `NULL`，某些函数可会返回 `INVALID_HANDLE_VALUE`（定义为 -1）；否则返回 `HANDLE`，其标识一个可操作对象。函数的 Platform SDK 文档会指明返回值的含义。 |
| `PVOID`        | 失败时返回 `NULL`；否则返回 `PVOID`，其标识一个数据块的内存地址。 |
| `LONG`/`DWORD` | 返回计数的函数通常返回一个 `LONG` 或 `DWORD`。如果函数由于某些原因而无法进行计数，则它通常会返回 0 或 -1。调用返回这种数据类型的函数时应该阅读 Platform SDK 文档以确保返回值的含义。 |

Microsoft 编译了一个所有可能的错误码的列表，并且为每个错误码分配了一个 32-bit 的编号，编号的规则如下：

| Bits: | 31-30 | 29 | 28 | 27-16 | 15-0 |
| :-- | :-- | :-- | :-- | :-- | :-- |
| Contents | Severity | Microsoft/customer | Reserved | Facility code | Exception code |
| Meaning  | 0 = Success<br>1 = Informational<br>2 = Warning<br>3 = Error | 0 = Microsoft-defined code<br>1 = customer-defined code | Must be 0 | The first 256 values are reserved by Microsoft | Microsoft/customer-defined code |

如果要创建自定义的错误码，那么根据规定应该将错误码的第 29 位置为 1 以避免与 Microsoft 定义的错误码发生冲突。

当一个 Windows 函数检测到错误时，它会使用一个称为线程本地存储（Thread-local storage）的机制，将相应的错误码与调用的线程关联起来。这种机制使线程间的错误码不会相互影响。当函数返回时，其返回值就会指明是否发生错误，若要确认是什么错误，则可以调用 `GetLastError` 函数以获取与线程关联的错误码。

> 注：由于 `GetLastError` 返回的是线程产生的最后一个错误，并且一些 Windows 函数即使没有发生错误也可能会改写线程关联的错误码。所以 `GetLastError` 应该在调用 Windows 函数并判断其发生错误后调用才有意义。

> 技巧：在 Microsoft Visual Studio 中调试程序时，可以在 Watch 窗口中使用 `@err, hr` 表达式对线程错误码进行观察。Microsoft Visual Studio 还附带一个称为 Error Lookup 的实用工具（Tools > Error Lookup）可以将错误码转换成对应的文本描述。

Windows 提供 [`FormatMessage`](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-formatmessage) 函数，可以将错误码转换成对应的文本描述。
