---
title: 'Windows via C/C++, 5th Edition - Thread Basics'
date: 2022-02-14 16:30:44
category: 'My Notes'
tags: ['Windows via C/C++', 'Windows']
---

线程（Thread）描述了进程（Process）中的执行路径，由两部分组成：

* 一个被操作系统用来管理线程的[内核对象](https://dsyx.github.io/2022/02/07/windows-via-c-cpp-5th-kernel-objects/)。这个内核对象也是系统用来存放关于线程的统计信息的地方。
* 一个用来维护线程执行代码时所需的所有函数参数和局部变量的线程栈（Thread Stack）。

[进程](https://dsyx.github.io/2022/02/09/windows-via-c-cpp-5th-processes/)是惰性的，它从不执行任何操作，仅仅是线程的容器。线程总是在某个进程的上下文（context）中创建，并在该进程中度过它们的整个生命周期。这实际上意味着线程在其进程的地址空间内执行代码并操作数据，因此如果有两个或多个线程在单个进程的上下文中运行，则这些线程共享单个地址空间。线程可以执行相同的代码并操作相同的数据。线程还可以共享内核对象句柄，因为[句柄表](https://dsyx.github.io/2022/02/07/windows-via-c-cpp-5th-kernel-objects/#%E8%BF%9B%E7%A8%8B%E7%9A%84%E5%86%85%E6%A0%B8%E5%AF%B9%E8%B1%A1%E5%8F%A5%E6%9F%84%E8%A1%A8)存在于每个进程，而不是每个线程。

> 注：创建进程时，系统会自动创建其第一个线程，称为*主线程（Primary Thread）*。

进程所使用的系统资源比线程多得多，其原因是地址空间。为进程创建虚拟地址空间需要大量的系统资源。系统中会进行大量记录保存，这需要大量的内存。此外，由于 `.exe` 和 `.dll` 文件被加载到地址空间中，因此还需要文件资源。实际上，线程只有一个内核对象和一个栈，几乎不涉及记录保存，因此只需要很少的内存。

## 创建线程

每个线程都必须有一个入口点（entry-point）函数以开始执行，如下：

```cpp
DWORD WINAPI ThreadFunc(PVOID pvParam) {
    DWORD dwResult = 0;
    ...
    return(dwResult);
}
```

线程函数可以执行任何所需的任务。当线程函数返回时，线程停止运行，其栈内存被释放，线程内核对象（Thread Kernel Object）的使用计数递减。

调用 [`CreateThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createthread) 函数可以创建一个新的线程：

```cpp
HANDLE CreateThread(
    PSECURITY_ATTRIBUTES psa,
    DWORD cbStackSize,
    PTHREAD_START_ROUTINE pfnStartAddr,
    PVOID pvParam,
    DWORD dwCreateFlags,
    PDWORD pdwThreadID);
```

调用 [`CreateThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createthread) 时，系统会创建一个线程内核对象。线程内核对象不是线程本身，而是操作系统用于管理线程的小型数据结构。这与进程和进程内核对象相互关联的方式相同。系统从进程的地址空间中分配内存以供线程的栈使用。新线程在与创建线程的线程在相同的进程上下文中运行。因此，新线程可以访问进程的所有内核对象句柄、进程中的所有内存以及同一进程中所有其他线程的栈。这使得单个进程中的多个线程非常容易相互通信。

> 注：[`CreateThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createthread) 函数是创建线程的 Windows 函数。但是，如果您正在编写 C/C++ 代码，则永远不应调用 [`CreateThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createthread)。相反，您应该使用 Microsoft C++ 运行时库函数 [`_beginthreadex`](https://docs.microsoft.com/en-us/cpp/c-runtime-library/reference/beginthread-beginthreadex)。

## 终止线程

线程可以通过四种方式终止：

* 从线程函数返回。（强烈建议这样做）
* 线程通过调用 [`ExitThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-exitthread) 函数来终止自身。（避免使用此方法）
* 同一进程或另一个进程中的线程调用 [`TerminateThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-terminatethread) 函数。（避免使用此方法）
* 包含线程的进程终止。（避免使用此方法）

当线程终止时，将发生以下操作：

* 线程拥有的所有 User 对象句柄都将被释放。在 Windows 中，大多数对象都归包含创建对象的线程的进程所有。但是，线程拥有两种 User 对象：窗口（window）和钩子（hook）。当线程终止时，系统会自动销毁任何窗口，并卸载由线程创建或安装的任何钩子。仅当进程终止时，才会销毁其他对象。
* 线程的退代码从 `STILL_ACTIVE` 更改为传递给 [`ExitThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-exitthread) 或 [`TerminateThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-terminatethread) 的退出码。
* 线程内核对象的状态变为已示意（signaled）。
* 如果该线程是进程中的最后一个活动线程，那么系统也会认为该进程已终止。
* 线程内核对象的使用计数减少 1。

可以调用 [`GetExitCodeThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-getexitcodethread) 来检查指定的线程是否已终止以及终止时的退出码。

### 从线程函数返回

应将线程设计为仅在从线程函数返回时终止，这是保证正确清理所有线程资源的唯一方法。

从线程函数返回可确保以下内容：

* 在线程函数中创建的所有 C++ 对象都将通过其析构函数正确销毁。
* 系统将正确释放线程栈使用的内存。
* 系统会将线程的退出码（保留在线程内核对象中）设置为线程函数的返回值。
* 系统将递减线程内核对象的使用计数。

### 线程通过调用 ExitThread 函数来终止自身

可以通过让线程调用 [`ExitThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-exitthread) 来强制线程终止。此函数终止线程，并使操作系统清理线程使用的所有操作系统资源。但是，您的 C/C++ 资源（如 C++ 类对象）可能不会被销毁，参考[进程中的一个线程调用 ExitProcess 函数](https://dsyx.github.io/2022/02/09/windows-via-c-cpp-5th-processes/#%E8%BF%9B%E7%A8%8B%E4%B8%AD%E7%9A%84%E4%B8%80%E4%B8%AA%E7%BA%BF%E7%A8%8B%E8%B0%83%E7%94%A8-exitprocess-%E5%87%BD%E6%95%B0)。

> 注：[`ExitThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-exitthread) 函数是终止线程的 Windows 函数。如果您正在编写 C/C++ 代码，则永远不应调用 [`ExitThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-exitthread)。相反，您应该使用 Microsoft C++ 运行时库函数 [`_endthreadex`](https://docs.microsoft.com/en-us/cpp/c-runtime-library/reference/endthread-endthreadex)。

### 同一进程或另一个进程中的线程调用 TerminateThread 函数

与 [`ExitThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-exitthread) 不同，[`TerminateThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-terminatethread) 可以终止任何线程。

> 注：[`TerminateThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-terminatethread) 函数是异步的。因此，如果需要确定线程已终止，那么可能需要调用 [`WaitForSingleObject`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitforsingleobject) 或类似的函数。

设计良好的应用程序不应该使用此函数，因为被终止的线程不会收到它正在被终止的通知。线程无法正确地进行清理，并且无法防止自身被终止。

> 注：当线程通过返回或调用 [`ExitThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-exitthread) 而终止时，线程的栈将被销毁。但是，如果使用 [`TerminateThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-terminatethread)，则在拥有该线程的进程终止之前，系统不会销毁线程的栈。微软特意如此地实现了 [`TerminateThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-terminatethread)。如果其他仍在执行的线程要引用被强制终止的线程的栈上的值，则这些其他线程将引发访问冲突。通过将已终止线程的栈保留在内存中，其他线程可以继续正常执行。
> 
> 此外，DLL 通常会在线程终止时收到通知。但是，如果使用 [`TerminateThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-terminatethread) 强行终止线程，则 DLL 不会收到此通知，这可能会阻止正确的清理。

### 包含线程的进程终止

[`ExitProcess`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-exitprocess) 和 [`TerminateProcess`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-terminateprocess) 函数也会终止线程。当整个进程终止时，进程使用的所有资源都将被清理。这两个函数会导致进程中的剩余线程被强制终止，就好像为每个剩余线程调用了 [`TerminateThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-terminatethread) 一样。因此，如果应用程序中有多个线程同时运行，则应该显式地处理每个线程在主线程返回之前如何停止。

## 线程的一些内部细节

![How a thread is created and initialized](/images/how-a-thread-is-created-and-initialized.svg)

调用 [`CreateThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createthread) 会导致系统创建一个线程内核对象。此对象的初始使用计数为 `2`（在线程停止运行并且从 [`CreateThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createthread) 返回的句柄被关闭之前，不会销毁线程内核对象）。线程内核对象的其他属性也被初始化：挂起计数设置为 `1`、退出码设置为 `STILL_ACTIVE`、对象设置为非示意（nonsignaled）状态。

创建内核对象后，系统将为线程栈分配内存（此内存是从进程的地址空间分配的，因为线程没有自己的地址空间）。然后，系统将两个值写入新线程栈的上端（线程栈始终从高内存地址向低内存地址增进）。首先写入栈的值是传递给 [`CreateThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createthread) 的 `pvParam` 参数的值，紧接着是 `pfnStartAddr` 的值。

每个线程都有自己的一组 CPU 寄存器，称为线程的*上下文*。上下文反映了线程上次执行时线程的 CPU 寄存器的状态。线程的 CPU 寄存器组保存在 `CONTEXT` 结构（在 WinNT.h 头文件中定义）中。`CONTEXT` 结构本身包含在线程内核对象中。

IP（Instruction Pointer）寄存器和 SP（Stack Pointer）寄存器是线程上下文中两个最重要的寄存器。线程始终在进程的上下文中运行，因此这两个地址标识的都是所属进程的地址空间中的内存。初始化线程内核对象时，`CONTEXT` 结构的 SP 寄存器将设置为 `pfnStartAddr` 在线程栈上所处位置的地址。IP 寄存器设置为名为 `RtlUserThreadStart` 的未记录函数的地址，该函数由 NTDLL.dll 模块导出，通常执行如下操作：

```cpp
VOID RtlUserThreadStart(PTHREAD_START_ROUTINE pfnStartAddr, PVOID pvParam) {
    __try {
        ExitThread((pfnStartAddr)(pvParam));
    }

    __except(UnhandledExceptionFilter(GetExceptionInformation())) {
        ExitProcess(GetExceptionCode());
    }
    // NOTE: We never get here.
}
```

线程完全初始化后，系统将检查调用 [`CreateThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createthread) 时是否传递了 `CREATE_SUSPENDED` 标志。如果未传递此标志，系统会将线程的挂起计数递减为 `0`，并且线程可以调度到处理器。然后，系统使用上次保存在线程上下文中的值加载实际的 CPU 寄存器。之后，线程可以执行代码并操作其进程地址空间中的数据。

由于新线程的 IP 设置为 `RtlUserThreadStart`，因此该函数实际上是线程开始执行的位置。`RtlUserThreadStart` 的原型使人认为该函数接收两个参数，且暗示该函数是从另一个函数调用的，但事实并非如此。新线程只是刚刚出现并在此处开始执行。`RtlUserThreadStart` 可以访问这两个参数，它们是有效的，因为操作系统将值显式写入线程的栈上（这是参数传递给函数的通常方式）。需要注意的是，某些 CPU 架构使用 CPU 寄存器而不是栈来传递参数，对于这些架构，系统会在允许线程执行 `RtlUserThreadStart` 函数之前，正确地初始化合适的寄存器。

当新线程执行 `RtlUserThreadStart` 函数时，会发生以下事情：

* 围绕线程函数设置了 SEH（Structured Exception Handling）框架，以便在线程执行时引发的任何异常都由系统进行一些默认处理。
* 系统调用线程函数，并将传递给 [`CreateThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createthread) 函数的 `pvParam` 参数传递给它。
* 当线程函数返回时，`RtlUserThreadStart` 会调用 [`ExitThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-exitthread)，并将线程函数的返回值传递给它。线程内核对象的使用计数递减，线程停止执行。
* 如果线程引发未处理的异常，则由 `RtlUserThreadStart` 函数设置的 SEH 框架将处理该异常。通常这意味着向用户显示一个消息框，并且当用户关闭该消息框时，`RtlUserThreadStart` 将调用 [`ExitProcess`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-exitprocess) 以终止整个进程，而不仅仅是有问题的线程。

## C/C++ 运行时库注意事项

有四个 native C/C++ 运行时库和两个 managed world of Microsoft .NET 随 Visual Studio 一起提供。请注意，这些库都支持多线程开发：不再有专门设计用于仅面向单线程开发的 C/C++ 库。

| Library Name | Description |
| :-- | :-- |
| LibCMt.lib | Statically linked release version of the library. |
| LibCMtD.lib | Statically linked debug version of the library. |
| MSVCRt.lib | Import library for dynamically linking the release version of the MSVCR80.dll library. (This is the default library when you create a new project.) |
| MSVCRtD.lib | Import library for dynamically linking the debug version of the MSVCR80D.dll library. |
| MSVCMRt.lib | Import library used for mixed managed/native code. |
| MSVCURt.lib | Import library compiled as 100-percent pure MSIL code. |

通过项目的 **属性（Properties） > C/C++ > 代码生成（Code Generation） > 运行时库（Runtime Library）** 可以配置项目所链接到的 C/C++ 运行时库。

由于标准 C 运行时库是在 1970 年左右发明的，该库的发明者没有考虑将 C 运行时库与多线程应用程序配合使用的问题。如果需要创建线程并且希望使用 C/C++ 运行时库，那么应该使用 [`_beginthread`](https://docs.microsoft.com/en-us/cpp/c-runtime-library/reference/beginthread-beginthreadex) 而不是 [`CreateThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createthread)。因为 [`CreateThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createthread) 没有为 C/C++ 运行时库执行一些保障线程安全的处理。

> 参考：[Microsoft Documentation - Windows/Apps/Win32/Desktop Technologies/System Services/Processes and Threads](https://docs.microsoft.com/en-us/windows/win32/procthread/processes-and-threads)。

## 获取自身的标识

当线程执行时，它们经常希望调用更改其执行环境的 Windows 函数。例如，线程可能想要更改其优先级（Priority）或其进程的优先级。由于线程更改其（或其进程）环境是很常见的，因此 Windows 提供了一些函数，使线程可以简单地引用其进程内核对象或自己的线程内核对象：

```cpp
HANDLE GetCurrentProcess();
HANDLE GetCurrentThread();
```

这两个函数都将返回调用线程的进程/线程内核对象的伪句柄（Pseudo Handle）。这些函数不会在调用进程的句柄表中创建新句柄。此外，调用这些函数不会影响进程/线程内核对象的使用计数。如果调用 [`CloseHandle`](https://docs.microsoft.com/en-us/windows/win32/api/handleapi/nf-handleapi-closehandle) 并传递伪句柄，那么 [`CloseHandle`](https://docs.microsoft.com/en-us/windows/win32/api/handleapi/nf-handleapi-closehandle) 会简单地返回 `FALSE`。

当调用需要进程或线程句柄的 Windows 函数时，可以传递伪句柄。例如，线程可以通过调用 [`GetProcessTimes`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-getprocesstimes) 来查询其进程的时间使用情况：

```cpp
FILETIME ftCreationTime, ftExitTime, ftKernelTime, ftUserTime;
GetProcessTimes(GetCurrentProcess(), &ftCreationTime, &ftExitTime, &ftKernelTime, &ftUserTime);
```

一些 Windows 函数允许通过进程/线程的唯一系统范围（Unique Systemwide）ID 来标识特定进程或线程。以下函数允许线程查询其进程的唯一 ID 或其自己的唯一 ID：

```cpp
DWORD GetCurrentProcessId();
DWORD GetCurrentThreadId();
```

### 将伪句柄转换为实句柄

有时可能需要获取线程的实句柄（Real Handle），而不是伪句柄。“实”的意思是明确地标识唯一线程的句柄。比如如下代码：

```cpp
DWORD WINAPI ParentThread(PVOID pvParam) {
    HANDLE hThreadParent = GetCurrentThread();
    CreateThread(NULL, 0, ChildThread, (PVOID) hThreadParent, 0, NULL);
    // Function continues...
}

DWORD WINAPI ChildThread(PVOID pvParam) {
    HANDLE hThreadParent = (HANDLE) pvParam;
    FILETIME ftCreationTime, ftExitTime, ftKernelTime, ftUserTime;
    GetThreadTimes(hThreadParent, &ftCreationTime, &ftExitTime, &ftKernelTime, &ftUserTime);
    // Function continues...
}
```

预期的想法是让父线程将标识父线程的线程句柄传递给子线程。然而父线程传递的是伪句柄，而不是实句柄。当子线程开始执行时，它会将伪句柄传递给 [`GetThreadTimes`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-getthreadtimes) 函数，这会导致子线程获得自己的 CPU 时间，而不是父线程的 CPU 时间，这是因为线程伪句柄是当前线程的句柄，即伪句柄指代的是正在调用函数的线程。

要使代码符合预期想法，必须将伪句柄转换为实句柄。可以利用 [`DuplicateHandle`](https://docs.microsoft.com/en-us/windows/win32/api/handleapi/nf-handleapi-duplicatehandle) 函数来完成这件事：

```cpp
DWORD WINAPI ParentThread(PVOID pvParam) {
    HANDLE hThreadParent;

    DuplicateHandle(
        GetCurrentProcess(),    // Handle of process that thread
                                // pseudohandle is relative to

        GetCurrentThread(),     // Parent thread's pseudohandle
        GetCurrentProcess(),    // Handle of process that the new, real,
                                // thread handle is relative to

        &hThreadParent,         // Will receive the new, real, handle
                                // identifying the parent thread
        0,                      // Ignored due to DUPLICATE_SAME_ACCESS
        FALSE,                  // New thread handle is not inheritable
        DUPLICATE_SAME_ACCESS); // New thread handle has same
                                // access as pseudohandle

    CreateThread(NULL, 0, ChildThread, (PVOID) hThreadParent, 0, NULL);
    // Function continues...
}

DWORD WINAPI ChildThread(PVOID pvParam) {
    HANDLE hThreadParent = (HANDLE) pvParam;
    FILETIME ftCreationTime, ftExitTime, ftKernelTime, ftUserTime;
    GetThreadTimes(hThreadParent, &ftCreationTime, &ftExitTime, &ftKernelTime, &ftUserTime);
    CloseHandle(hThreadParent);
    // Function continues...
}
```
