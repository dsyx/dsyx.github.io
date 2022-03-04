---
title: 'Windows via C/C++, 5th Edition - Fibers'
date: 2022-03-03 16:55:22
category: 'My Notes'
tags: ['Windows via C/C++', 'Windows']
---

Microsoft 为 Windows 添加了一种由应用程序手动调度的执行单元 —— [纤程（Fiber）](https://docs.microsoft.com/en-us/windows/win32/procthread/fibers)。

Windows 内核中实现了线程，线程是操作系统可见的执行单元，系统将根据 Microsoft 定义的算法调度它们。纤程是在用户模式代码中实现的，因此它们对于系统来说是不可见的，并且应用程序必须定义自己的算法来调度它们。

单个线程可以包含一个或多个纤程。就内核而言，线程是抢占调度的，并且它正在执行代码。线程一次执行一个纤程的代码，具体运行哪个纤程由应用程序定义。

使用纤程时必须先将现有的线程转换为纤程，这可以通过调用 [`ConvertThreadToFiber`](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-convertthreadtofiber) 来完成此操作。此函数会为纤程的执行上下文分配内存，执行上下文由以下元素组成：

* 一个用户定义的值，其初始化为传递给 [`ConvertThreadToFiber`](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-convertthreadtofiber) 的 `pvParam` 参数的值。
* 一个结构化异常处理链的头部。
* 纤程的栈的顶部和底部内存地址（当您将线程转换为纤程时，这也是线程的栈）。
* 各种 CPU 寄存器，包括栈指针、指令指针等。

分配并初始化纤程执行上下文后，执行上下文的地址将与线程相关联，该线程已转换为纤程，并且纤程将在此线程上运行。现在，如果纤程（线程）返回或调用 [`ExitThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-exitthread)，则纤程和线程都会终止。

> 注：[`ConvertThreadToFiber`](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-convertthreadtofiber) 实际上返回纤程执行上下文的内存地址。用户切勿自行读取或写入执行上下文数据。

要创建另一个纤程，线程（当前运行的纤程）应该调用 [`CreateFiber`](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-createfiber) 或 [`CreateFiberEx`](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-createfiberex)：

```cpp
PVOID CreateFiber(
    DWORD dwStackSize,
    PFIBER_START_ROUTINE pfnStartAddress,
    PVOID pvParam);

PVOID CreateFiberEx(
    SIZE_T dwStackCommitSize,
    SIZE_T dwStackReserveSize,
    DWORD dwFlags,
    PFIBER_START_ROUTINE pStartAddress,
    PVOID pvParam);
```

首次调度纤程时，纤程例程将接收到最初传递给 [`CreateFiber`](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-createfiber) 或 [`CreateFiberEx`](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-createfiberex) 的 `pvParam`。

与通过 [`ConvertThreadToFiber`](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-convertthreadtofiber) 获得的纤程不同，通过 [`CreateFiber`](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-createfiber) 或 [`CreateFiberEx`](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-createfiberex) 获得的纤程不会马上执行，因为当前运行的纤程仍在执行，在单个线程上一次只能执行一个纤程。要使新的纤程执行，请调用 [`SwitchToFiber`](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-switchtofiber) 函数。

在内部，[`SwitchToFiber`](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-switchtofiber) 执行以下步骤：

1. 将当前的一些 CPU 寄存器（包括指令指针寄存器和栈指针寄存器）保存在当前运行的纤程的执行上下文中。
2. 将先前保存在即将要运行的纤程的执行上下文中的寄存器加载到 CPU 寄存器中。
3. 将纤程的执行上下文与线程相关联，线程运行指定的纤程。
4. 将线程的指令指针设置为已保存的指令指针。线程（纤程）在此纤程上次执行的位置上继续执行。

[`SwitchToFiber`](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-switchtofiber) 是纤程获得 CPU 时间的唯一方法。您的代码必须在适当的时间点显式地调用 [`SwitchToFiber`](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-switchtofiber) 来控制纤程的调度。

要销毁一个纤程，您可以调用 [`DeleteFiber`](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-deletefiber) 函数。

在销毁所有其它纤程后，由 [`ConvertThreadToFiber`](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-convertthreadtofiber) 获得的纤程可以使用 [`ConvertFiberToThread`](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-convertfibertothread) 函数将自身转换回线程。

如果您需要在单个纤程上存储信息，那么您可以使用 [*FLS（Fiber Local Storage）*](https://docs.microsoft.com/en-us/windows/win32/procthread/fibers#fiber-local-storage) 函数。

为了方便，Microsoft 还提供了几个额外的纤程函数。如果要获取当前运行的纤程的执行上下文的地址，那么您可以调用 [`GetCurrentFiber`](https://docs.microsoft.com/en-us/windows/win32/api/winnt/nf-winnt-getcurrentfiber)。如果要获取当前运行的纤程的纤程数据（在创建纤程时传递的 `pvParam`），那么您可以调用 [`GetFiberData`](https://docs.microsoft.com/en-us/windows/win32/api/winnt/nf-winnt-getfiberdata)。
