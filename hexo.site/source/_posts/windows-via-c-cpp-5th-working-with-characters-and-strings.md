---
title: 'Windows via C/C++, 5th Edition - Working with Characters and Strings'
date: 2022-01-28 14:01:18
category: 'My Notes'
tags: ['Windows via C/C++', 'Windows', 'C/C++']
---

在 Windows Vista 中，每个 Unicode 字符都使用 UTF-16（Unicode Transformation Format）进行编码。UTF-16 将每个字符编码为 16 位，当 16 位不足以表示所有字符时，其将使用代理（surrogate），代理是一种使用 32 位来表示单个字符的方法。

从 Windows NT 起，所有 Windows 版本都是使用 Unicode 从头开始构建的。也就是说，用于创建窗口、显示文本、执行字符串操作等的所有核心函数都需要 Unicode 字符串。如果你通过传递 ANSI 字符串来调用任何 Windows 函数，那么函数会首先将该字符串转换为 Unicode，然后将 Unicode 字符串传递给操作系统。如果希望从函数返回 ANSI 字符串，则系统会在返回到应用程序之前将 Unicode 字符串转换为 ANSI 字符串。所有这些转换都是在您看不见的情况下发生的。当然，系统执行所有这些字符串转换需要时间和内存开销。

当 Windows 公开那些需要字符串作为参数的函数时，它通常会提供两个版本：以大写 W 结尾的接受 Unicode 字符串；以大写 A 结尾的接受 ANSI 字符串，如 [`CreateWindowExW`](https://docs.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-createwindowexw) 和 [`CreateWindowExA`](https://docs.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-createwindowexa)。然而我们通常调用的是无后缀的函数（如 `CreateWindowEx`），这使我们可以通过宏定义 `UNICODE` 来控制编译时使用那个版本的函数。

> 注：使用 Visual Studio 创建新项目时，默认定义了 `UNICODE`。这意味着默认使用 W 版本函数。

> 注：在 Windows Vista 下，A 版本函数只是一个简单的翻译层，它会分配内存以将 ANSI 字符串转换为 Unicode 字符串，然后调用 W 版本函数，当 W 版本函数返回后，A 版本函数会释放掉它分配的内存，最后将结果返回给你。因此应用程序需要更多内存并且运行速度相对较慢。从一开始就使用 Unicode 开发应用程序，可以使应用程序更有效地执行。此外，Windows 的这些翻译函数中存在一些众所周知的 bug，因此避免使用它们也可以消除一些潜在的 bug。如果你要创建动态链接库（DLL）给其它开发者使用，那么建议你将函数导出为两个版本：W 版本和 A 版本，其中 A 版本仅作为一个简单的翻译层。这样可以与 Windows 函数的风格保持一致。

Windows API 中的某些函数，如 `WinExec` 和 `OpenFile` 是为了保持对 16-bit Windows 程序的兼容，新的应用开发应该避免使用它们。

与 Windows 函数一样，C 运行时库提供了一组函数来操作 ANSI 字符和字符串（如 [`strlen`](https://docs.microsoft.com/en-us/cpp/c-runtime-library/reference/strlen-wcslen-mbslen-mbslen-l-mbstrlen-mbstrlen-l?view=msvc-170)），另一组函数来操作 Unicode 字符和字符串（如 [`wcslen`](https://docs.microsoft.com/en-us/cpp/c-runtime-library/reference/strlen-wcslen-mbslen-mbslen-l-mbstrlen-mbstrlen-l?view=msvc-170)）。但是，与 Windows 函数不同，这些函数的 ANSI 版本不会将字符串转换为 Unicode，然后在内部调用函数的 Unicode 版本。

任何修改字符串的函数都会显露出潜在的危险：如果目标字符串缓冲区不够大，无法包含生成的字符串，则会发生内存损坏。下面是一个示例：

```cpp
// The following puts 4 characters in a
// 3-character buffer, resulting in memory corruption
WCHAR szBuffer[3] = L"";
wcscpy(szBuffer, L"abc"); // The terminating 0 is a character too!
```

`strcpy` 和 `wcscpy` 函数（以及大多数其他字符串操作函数）的问题在于，它们不接受指定缓冲区最大大小的参数，因此，该函数不知道它正在损坏内存。Microsoft 现在提供了一组新函数，这些函数取代了 C 运行时库提供的不安全的字符串操作函数（如 `wcscat`）。若要编写安全的代码，不应再使用任何熟悉的修改字符串的 C 运行时函数。相反，您应该利用 Microsoft 的 `StrSafe.h` 文件定义的安全的字符串函数。

对于许多打开文本文件并对其进行处理的应用程序（如编译器），如果在打开文件后，应用程序可以确定文本文件是否包含 ANSI 字符或 Unicode 字符，这将非常方便。由AdvApi32导出并在中声明的IsTextUnicode函数.dll可以帮助进行以下区分：

由 `AdvApi32.dll` 导出并由 `WinBase.h` 声明的 [`IsTextUnicode`](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-istextunicode) 函数用于确定缓冲区是否可能包含某种形式的 Unicode 文本。该函数使用一系列统计和确定性方法来猜测缓冲区的内容。由于这不是一门精确的科学，因此该函数可能会返回不正确的结果。
