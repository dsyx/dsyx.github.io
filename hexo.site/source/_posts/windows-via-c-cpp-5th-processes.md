---
title: 'Windows via C/C++, 5th Edition - Processes'
date: 2022-02-09 14:46:03
category: 'My Notes'
tags: ['Windows via C/C++', 'Windows', 'C/C++']
---

进程（Process）通常被定义为正在运行的程序实例，由两部分组成：

* 一个被操作系统用来管理进程的内核对象。这个内核对象也是系统用来存放关于进程的统计信息的地方。
* 一个包含所有可执行文件（executable）或 DLL（动态链接库，Dynamic-Link Library）模块的代码和数据的地址空间。它还包含了动态内存分配，如线程栈和堆的分配。

进程是惰性的。对于要完成任何事情的进程，它必须具有在其上下文（context）中运行的线程（thread），线程负责执行进程地址空间中的代码。单个进程可能包含多个线程，这些线程在进程的地址空间中“同时地（simultaneously）”执行代码。为此，每个线程都有自己的一组 CPU 寄存器（register）和栈（stack）。每个进程至少有一个线程。创建进程时，系统会自动创建其第一个线程，称为*主线程（primary thread）*。此线程可以创建其他线程，而这些线程又可以创建更多线程。如果在进程的地址空间中没有执行代码的线程，则该进程没有理由继续存在，并且系统将自动销毁进程及其地址空间。

操作系统会为每个线程安排一些 CPU 时间，通过轮询（round-robin）方式为线程提供时间片（称为*quantum*），让人产生所有线程同时运行的错觉。若机器具有多个 CPU，则操作系统用于在 CPU 上对线程进行负载平衡的算法会很复杂。Microsoft Windows 可以同时在每个 CPU 上安排不同的线程，以便多个线程真正地同时运行。Windows 内核负责处理此类系统上线程的所有管理和调度。

# Windows 应用程序的起点

Windows 支持两种应用类型：GUI（Graphical User Interface）应用和 CUI（Console User Interface）应用。使用 Microsoft Visual Studio 创建应用工程时，集成环境会设置各种链接器开关，以便链接器在生成的可执行文件中嵌入正确类型的子系统。GUI 应用的链接器开关是 `/SUBSYSTEM:WINDOWS`；CUI 应用的链接器开关是 `/SUBSYSTEM:CONSOLE`。当用户运行应用程序时，操作系统的加载器会查看可执行映像的标头（header）并获取子系统值。

Windows 应用程序必须具有一个在应用程序开始运行时被调用的入口点函数，具体取决于应用的类型和是否使用 Unicode：

> 注：操作系统实际上并不调用您编写的入口点函数。相反，它调用由 C/C++ 运行时实现的 C/C++ 运行时启动函数，并在链接时设置 `-entry:` 命令行选项。

```cpp
int WINAPI _tWinMain(
    HINSTANCE hInstanceExe,
    HINSTANCE,
    PTSTR pszCmdLine,
    int nCmdShow);

int _tmain(
    int argc,
    TCHAR *argv[],
    TCHAR *envp[]);
```

| Application Type | Entry Point | Startup Function Embedded in Your Executable |
| :-- | :-- | :-- |
| GUI application that wants ANSI characters and strings | _tWinMain (WinMain) | WinMainCRTStartup |
| GUI application that wants Unicode characters and strings | _tWinMain (wWinMain) | wWinMainCRTStartup |
| CUI application that wants ANSI characters and strings | _tmain (Main) | mainCRTStartup |
| CUI application that wants Unicode characters and strings | _tmain (Wmain) | wmainCRTStartup |

链接器负责在链接可执行文件时选择正确的 C/C++ 运行时启动函数。如果指定了 `/SUBSYSTEM:WINDOWS` 链接器开关，则链接器希望找到 `WinMain` 或 `wWinMain` 函数。如果这两个函数都不存在，那么链接器将返回“未解析的外部符号（unresolved external symbol）”错误；否则，它会调用 `WinMainCRTStartup` 或 `wWinMainCRTStartup` 函数。同样，如果指定了 `/SUBSYSTEM:CONSOLE` 链接器开关，则链接器希望找到 `main` 或 `wmain` 函数，并调用 `mainCRTStartup` 或 `wmainCRTStartup` 函数。

> 注：如果未指定 `/SUBSYSTEM` 链接器开关，那么链接器会检查代码中是否存在四个函数（`WinMain`、`wWinMain`、`main`、`wmain`）之一，并以此来推断应该将哪个子系统及 C/C++ 启动函数嵌入到可执行文件中。

所有 C/C++ 运行时启动函数基本上都执行相同的操作，不同之处在于它们是处理 ANSI 还是 Unicode 字符串，以及在初始化 C 运行时库后调用哪个入口点函数：

* 检索指向新进程的完整命令行的指针。
* 检索指向新进程环境变量的指针。
* 初始化 C/C++ 运行时的全局变量。
* 初始化 C 运行时内存分配函数（`malloc` 和 `calloc`）和其他低级输入/输出例程使用的堆。
* 调用所有全局的和静态的 C++ 类对象的构造函数。

在初始化这些之后，C/C++ 启动函数会调用应用的入口点函数。如果你编写了一个 `_tWinMain` 函数并且启用了 `_UNICODE`，则调用如下所示：

```cpp
GetStartupInfo(&StartupInfo);
int nMainRetVal = wWinMain((HINSTANCE)&_ImageBase, NULL, pszCommandLineUnicode,
    (StartupInfo.dwFlags & STARTF_USESHOWWINDOW)
        ? StartupInfo.wShowWindow : SW_SHOWDEFAULT);
```

> 注：`_ImageBase` 是链接器定义的伪变量，它展示可执行文件映射到应用程序内存的位置。

如果你编写了一个 `_tmain` 函数并且启用了 `_UNICODE`，则调用如下所示：

```cpp
int nMainRetVal = wmain(argc, argv, envp);
```

当入口点函数返回时，启动函数会使用返回值 `nMainRetVal` 调用 C 运行时 `exit` 函数，`exit` 函数会执行如下操作：

* 调用任何通过调用 `_onexit` 函数注册的函数。
* 调用所有全局的和静态的 C++ 类对象的析构函数。
* 在 `DEBUG` 构建中，如果已设置 `_CRTDBG_LEAK_CHECK_DF` 标志，则 C/C++ 运行时内存管理中的泄漏将通过调用 `_CrtDumpMemoryLeaks` 函数列出。
* 调用操作系统的 `ExitProcess` 函数，并将 `nMainRetVal` 传递给它。这会导致操作系统终止进程并设置其退出码。

## 进程实例句柄

加载到进程地址空间中的每个可执行文件或 DLL 文件都分配有一个唯一的实例句柄。可执行文件的实例将作为 `(w)WinMain` 的第一个参数 `hInstanceExe`。加载资源的调用通常需要句柄的值，如要从可执行文件的映像中加载图标资源，则需要调用以下函数：

```cpp
HICON LoadIcon(
    HINSTANCE hInstance,
    PCTSTR pszIcon);
```

某些函数需要 `HMODULE` 类型的参数，如 `GetModuleFileName` 函数：

```cpp
DWORD GetModuleFileName(
    HMODULE hInstModule,
    PTSTR pszPath,
    DWORD cchPath);
```

> 注：事实上，`HMODULE` 和 `HINSTANCE` 是一样的东西。如果函数的文档指示需要 `HMODULE`，那么可以传递一个 `HINSTANCE`，反之亦然。定义出两个数据类型是因为在 16 位 Windows 中，它们所标识的东西是不同的。

`(w)WinMain` 的 `hInstanceExe` 参数的实际值是系统将可执行文件的映像加载到进程地址空间中的内存基址。加载可执行文件映像的基址由链接器确定，可以使用 Microsoft 链接器的 [`/BASE:address`](https://docs.microsoft.com/en-us/cpp/build/reference/base-base-address) 链接器开关更改应用程序加载到的基址。

可以使用 [`GetModuleHandle`](https://docs.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-getmodulehandlew) 函数来检索进程已加载的指定模块。

## 进程的前一个实例句柄

C/C++ 运行时启动代码始终将 `NULL` 传递给 `(w)WinMain` 的 `hPrevInstance` 参数。此参数在 16 位 Windows 中使用，保留此参数仅仅是为了简化 16 位 Windows 应用程序的移植。请切勿在代码中引用此参数。

## 进程的命令行

创建新进程时，将传递该进程的命令行。命令行至少包含用于创建新进程的可执行文件的名称（第一个标记）。当 C 运行时的启动代码开始执行 GUI 应用程序时，它会通过调用 [`GetCommandLine`](https://docs.microsoft.com/en-us/windows/win32/api/processenv/nf-processenv-getcommandlinew) Windows 函数来检索进程的完整命令行，它会跳过可执行文件的名称，并将指向命令行其余部分的指针传递给 `WinMain` 的 `pszCmdLine` 参数。

`ShellAPI.h` 中声明了 [`CommandLineToArgvW`](https://docs.microsoft.com/en-us/windows/win32/api/shellapi/nf-shellapi-commandlinetoargvw) 函数，可用于辅助命令行标记的提取。

## 进程的环境变量

每个进程都有一个与之关联的环境块。环境块是在进程的地址空间内分配的内存块，其中包含一组具有以下外观的字符串：

```
=::=::\ ...
VarName1=VarValue1\0
VarName2=VarValue2\0
VarName3=VarValue3\0 ...
VarNameX=VarValueX\0
\0
```

每个字符串的第一部分是环境变量的名称，后跟一个等号，等号后是要分配给变量的值。

> 注：除了第一个 `=::=::\ ...` 字符串之外，块中的其他一些字符串可能以 `=` 字符开头。在这种情况下，这些字符串不会用作环境变量。

有两种访问环境块的方法。第一种方法是通过调用 [`GetEnvironmentStrings`](https://docs.microsoft.com/en-us/windows/win32/api/processenv/nf-processenv-getenvironmentstrings) 函数来检索完整的环境块。第二种方法是通过 [`main`](https://docs.microsoft.com/en-us/cpp/c-language/main-function-and-program-execution) 入口点收到的 `TCHAR* env[]` 参数（仅适用于 CUI 应用程序）。

当用户登录到 Windows 时，系统将创建 shell 进程，并将一组环境字符串与其关联。系统通过检查注册表（registry）中的两个键（key）来获取环境字符串的初始集。第一个键包含适用于系统的所有环境变量的列表：`HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\Environment`；第二个键包含适用于当前登录用户的所有环境变量的列表：`HKEY_CURRENT_USER\Environment`。用户可以在 **控制面板（Control Panel） > 系统（System） > 高级系统设置（Advanced System Settings） > 环境变量（Environment Variables）** 中维护这些环境变量。

> 注：只有具有管理员权限的用户才能更改 **系统变量（System Variables）** 列表中包含的变量。

应用程序可以使用各种注册表函数来修改这些注册表项。但是，要使更改对所有应用程序生效，用户必须注销，然后重新登录。某些应用程序（如资源管理器、任务管理器和控制面板）可以在其主窗口收到 `WM_SETTINGCHANGE` 消息时使用新的注册表项更新其环境块。如果应用程序更新了注册表项，并希望让感兴趣的应用程序更新其环境块，则可以进行以下调用：

```cpp
SendMessage(HWND_BROADCAST, WM_SETTINGCHANGE, 0, (LPARAM) TEXT("Environment"));
```

通常，子进程会继承父进程的环境变量。父进程可以在创建子进程时决定子进程的环境变量。由于子进程继承的环境变量是拷贝所得而不是引用所得的，所以后续父、子进程对环境变量的修改不会相互影响。

如果要使用环境变量，则应用程序可以利用一些函数：

* [`GetEnvironmentVariable`](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-getenvironmentvariable) 函数用于确定环境变量的存在和值。
* [`ExpandEnvironmentStrings`](https://docs.microsoft.com/en-us/windows/win32/api/processenv/nf-processenv-expandenvironmentstringsw) 函数用于解析包含可替换字符串的内容，由一对百分号包裹的部分指示为可替换字符串，如 `%USERPROFILE%\Documents`，`%USERPROFILE%` 将替换为环境变量 `USERPROFILE` 的值。
* [`SetEnvironmentVariable`](https://docs.microsoft.com/en-us/windows/win32/api/processenv/nf-processenv-setenvironmentvariablew) 函数用于添加、删除或修改环境变量的值。

## 进程的亲和性

通常，进程中的线程可以在主机中的任何 CPU 上执行。但是，也可以强制进程的线程在可用 CPU 的子集上运行，这称为处理器亲和性（processor affinity）。子进程会继承其父进程的亲和性。

## 进程的错误模式

每个进程有一组相关联的标志，这些标志告诉系统进程应如何响应严重错误，包括磁盘介质故障、未处理的异常、文件查找故障和数据未对齐等。进程可以通过调用 [`SetErrorMode`](https://docs.microsoft.com/en-us/windows/win32/api/errhandlingapi/nf-errhandlingapi-seterrormode) 函数来告诉系统如何处理这些每个错误。

默认情况下，子进程继承其父进程的错误模式标志。即如果进程打开了 `SEM_NOGPFAULTERRORBOX` 标志，然后产生了一个子进程，那么子进程也将打开此标志。但是，子进程不会收到有关此问题的通知，并且子进程可能尚未编写如何处理 GP 故障。如果子进程的某个线程发生了 GP 故障，则子进程可能会在不通知用户的情况下终止。父进程可以通过在调用 [`CreateProcess`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessw) 时指定 `CREATE_DEFAULT_ERROR_MODE` 标志来防止子进程继承其错误模式。

## 进程的当前驱动器和目录

系统在内部跟踪进程的当前驱动器（Drive）和目录（Directory）。这些信息是基于每个进程进行维护的，因此在进程中更改当前驱动器或目录会更改进程中所有线程的此信息。当进程调用需要路径的函数时，若没有提供完整的路径，则会基于当前驱动器和目录进行查找。

可以通过调用以下两个函数来获取和设置进程的当前驱动器和目录：

* [`GetCurrentDirectory`](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-getcurrentdirectory)
* [`SetCurrentDirectory`](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-setcurrentdirectory)

### 进程的当前目录

系统会跟踪进程的当前驱动器和目录，但不会跟踪每个驱动器的当前目录。但是，有一些操作系统支持处理多个驱动器的当前目录。此支持通过进程的环境字符串提供。例如，一个进程可以有两个环境变量，如下所示：

```
=C:=C:\Utility\Bin
=D:=D:\Program Files
```

这些变量指示进程的驱动器 C 的当前目录是 `\Utility\Bin`，而其驱动器 D 的当前目录是 `\Program Files`。

如果调用一个函数，并传递一个驱动器限定名称（如：`D:readme.txt`）以指示驱动器不是当前驱动器，那么系统将在进程的环境块中查找与指定驱动器号关联的变量。如果变量存在，则系统将使用变量的值作为当前目录；如果变量不存在，则系统会假定当前目录为驱动器的根目录。

> 注：可以使用 C 运行时函数 [`_chdir`](https://docs.microsoft.com/en-us/cpp/c-runtime-library/reference/chdir-wchdir) 替代 [`SetCurrentDirectory`](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-setcurrentdirectory) 来改变当前目录。与 [`SetCurrentDirectory`](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-setcurrentdirectory) 不同，[`_chdir`](https://docs.microsoft.com/en-us/cpp/c-runtime-library/reference/chdir-wchdir) 还会添加或修改环境变量，因此会保留不同驱动器的当前目录。

可以通过调用 [`GetFullPathName`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-getfullpathnamew) 来获取其当前目录，如获取驱动器 C 的当前目录：

```cpp
TCHAR szCurDir[MAX_PATH];
DWORD cchLength = GetFullPathName(TEXT("C:"), MAX_PATH, szCurDir, NULL);
```

## 系统版本

有时候应用程序需要确定用户正在运行哪个版本的 Windows，可以使用如下函数获取相关信息：

* [`GetVersion`](https://docs.microsoft.com/en-us/windows/win32/api/sysinfoapi/nf-sysinfoapi-getversion)，最开始是为 16 位 Windows 设计的。
* [`GetVersionEx`](https://docs.microsoft.com/en-us/windows/win32/api/sysinfoapi/nf-sysinfoapi-getversionexw)，[`GetVersion`](https://docs.microsoft.com/en-us/windows/win32/api/sysinfoapi/nf-sysinfoapi-getversion) 的增强版。
* [`VerifyVersionInfo`](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-verifyversioninfow)，从 Windows Vista 开始提供的。

> 注：最新的方法应该查看 [Microsoft Documentation - Windows/Apps/Win32/Desktop Technologies/System Services/Windows System Information](https://docs.microsoft.com/en-us/windows/win32/sysinfo/windows-system-information)。

# 创建进程

使用 [`CreateProcess`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessw) 函数可以创建进程：

```cpp
BOOL CreateProcess(
    PCTSTR pszApplicationName,
    PTSTR pszCommandLine,
    PSECURITY_ATTRIBUTES psaProcess,
    PSECURITY_ATTRIBUTES psaThread,
    BOOL bInheritHandles,
    DWORD fdwCreate,
    PVOID pvEnvironment,
    PCTSTR pszCurDir,
    PSTARTUPINFO psiStartInfo,
    PPROCESS_INFORMATION ppiProcInfo);
```

调用 [`CreateProcess`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessw) 函数时，系统会创建一个进程内核对象（Process Kernel Object），并将其使用计数初始化为 1。进程内核对象不是进程本身，而是操作系统用于管理进程的小型数据结构。系统会为新进程创建一个虚拟地址空间，并将可执行文件的代码和数据以及任何所需的 DLL 加载到进程的地址空间中。

然后，系统会为新进程的主线程创建一个线程内核对象（Thread Kernel Object），并将其使用计数初始化为 1。与进程内核对象一样，线程内核对象是操作系统用于管理线程的小型数据结构。主线程会从执行 C/C++ 运行时启动代码开始。如果系统成功地创建新进程和主线程，则 [`CreateProcess`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessw) 将返回 `TRUE`。

> 注：[`CreateProcess`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessw) 会在进程完全初始化之前返回 `TRUE`。这意味着操作系统加载器尚未尝试查找所有必需的 DLL。如果找不到 DLL 或无法正确初始化，则进程将终止。由于 [`CreateProcess`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessw) 返回 `TRUE`，因此父进程不会知道任何初始化的问题。

# 终止进程

进程可以通过四种方式终止：

* 从主线程的入口点函数返回。（强烈建议这样做）
* 进程中的一个线程调用 [`ExitProcess`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-exitprocess) 函数。（避免使用此方法）
* 另一个进程中的线程调用 [`TerminateProcess`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-terminateprocess) 函数。（避免使用此方法）
* 进程中的所有线程自行终止。（很少会发生）

当一个进程终止时，以下动作会被执行：

1. 进程中任何剩余的线程会被终止。
2. 释放进程分配的所有 User 和 GDI 对象，并关闭所有内核对象。
3. 进程的退出码从 `STILL_ACTIVE` 更改为传递给 `ExitProcess` 或 `TerminateProcess` 的退出码。
4. 进程内核对象的状态变为已示意（signaled）。
5. 进程内核对象的使用计数递减 1。

> 注：进程终止后，其相关的进程内核对象不一定会被销毁，因为可能有其它进程正在使用它。如 A 进程持有已打开的 B 进程相关的进程内核对象句柄，A 进程可以使用 [`GetExitCodeProcess`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-getexitcodeprocess) 函数来获取 B 进程的终止状态，若 B 未终止，则通过输出参数 `pdwExitCode` 返回 `STILL_ACTIVE`；否则返回 B 的退出码。

## 从主线程的入口点函数返回

应将应用程序设计为仅在从主线程的入口点函数返回时终止，这是保证正确清理所有主线程资源的唯一方法。

从主线程的入口点函数返回可确保以下内容：

* 此线程创建的任何 C++ 对象都将使用其析构函数正确销毁。
* 系统将正确释放线程栈使用的内存。
* 系统会将进程的退出码（保存在进程内核对象中）设置为入口点函数的返回值。
* 系统将递减进程内核对象的使用计数。

## 进程中的一个线程调用 ExitProcess 函数

当进程中的一个线程调用 [`ExitProcess`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-exitprocess) 时，进程将终止，操作系统保证该进程的所有进程或线程的系统资源会得到很好的清理。

然而显式地调用 [`ExitProcess`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-exitprocess) 可能会导致 C/C++ 运行时无法正确地清理，如下代码所示：

```cpp
#include <windows.h>
#include <stdio.h>

class CSomeObj {
public:
    CSomeObj()  { printf("Constructor\r\n"); }
    ~CSomeObj() { printf("Destructor\r\n"); }
};

CSomeObj g_GlobalObj;

void main () {
    CSomeObj LocalObj;
    ExitProcess(0); // This shouldn't be here

    // At the end of this function, the compiler automatically added
    // the code necessary to call LocalObj's destructor.
    // ExitProcess prevents it from executing.
}
```

执行上述代码只会看到：

```
Constructor
Constructor
```

C++ 对象未被正确销毁！这是因为 [`ExitProcess`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-exitprocess) 会强制进程马上终止，导致 C/C++ 运行时没有机会进行清理。

## 另一个进程中的线程调用 TerminateProcess 函数

与 [`ExitProcess`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-exitprocess) 相同，调用 [`TerminateProcess`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-terminateprocess) 也会终止进程。不同的是，任何线程都可以调用 [`TerminateProcess`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-terminateprocess) 来终止另一个进程或自身的进程。

仅当无法使用其他方法强制进程退出时，才应使用 [`TerminateProcess`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-terminateprocess)。被终止的进程不会收到关于它正在终止的通知（这意味着应用程序可能无法正确地进行清理），也无法防止自身被终止（正常安全机制除外）。例如，进程可能无法将其内存中包含的数据刷写到磁盘，但其使用的系统资源（如打开的文件对象）会被完全地清理。

> 注：[`TerminateProcess`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-terminateprocess) 函数是异步的。因此，如果需要确定进程已终止，那么可能需要调用 [`WaitForSingleObject`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitforsingleobject) 或类似的函数。

## 进程中的所有线程自行终止

如果进程中的所有线程自行终止（可能因为它们都调用 [`ExitThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-exitthread) 或它们都被 [`TerminateThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-terminatethread) 终止），那么进程的退出码将被设置为与终止运行的最后一个线程相同的退出码。

# 子进程

在设计应用程序时，可能会遇到希望请求执行另一个任务，但同时希望原本任务继续执行的情况。其中一种解决方法是创建一个新的进程，让新进程执行另一个任务。

如果要创建新进程，让它执行一些任务并等待结果，则可以使用类似于如下的代码：

```cpp
PROCESS_INFORMATION pi;
DWORD dwExitCode;

// Spawn the child process.
BOOL fSuccess = CreateProcess(..., &pi);
if (fSuccess) {
    // Close the thread handle as soon as it is no longer needed!
    CloseHandle(pi.hThread);

    // Suspend our execution until the child has terminated.
    WaitForSingleObject(pi.hProcess, INFINITE);

    // The child process terminated; get its exit code.
    GetExitCodeProcess(pi.hProcess, &dwExitCode);

    // Close the process handle as soon as it is no longer needed.
    CloseHandle(pi.hProcess);
}
```

大多数情况下，应用程序会将另一个进程作为分离的进程（Detached Process）启动。这意味着在创建并执行进程后，父进程不再需要与新进程进行交互。若要放弃与子进程的所有关联，则父进程必须通过调用 [`CloseHandle`](https://docs.microsoft.com/en-us/windows/win32/api/handleapi/nf-handleapi-closehandle) 来关闭其拥有的新进程及其主线程的句柄。如下所示：

```cpp
PROCESS_INFORMATION pi;

// Spawn the child process.
BOOL fSuccess = CreateProcess(..., &pi);
if (fSuccess) {
    // Allow the system to destroy the process & thread kernel
    // objects as soon as the child process terminates.
    CloseHandle(pi.hThread);
    CloseHandle(pi.hProcess);
}
```
