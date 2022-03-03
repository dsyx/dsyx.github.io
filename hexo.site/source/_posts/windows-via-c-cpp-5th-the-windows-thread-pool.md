---
title: 'Windows via C/C++, 5th Edition - The Windows Thread Pool'
date: 2022-03-03 09:44:16
category: 'My Notes'
tags: ['Windows via C/C++', 'Windows']
---

每个人对如何管理线程的创建和销毁都有自己不同的看法。Windows 提供了一个 [线程池](https://docs.microsoft.com/en-us/windows/win32/procthread/thread-pools) 机制（围绕 IOCP 构建），使得开发者可以更容易地管理线程的创建和销毁。这个新的通用线程池不一定适用于所有情况，但它通常表现得足够好，并且可以为您节省大量的开发时间。

新的线程池函数允许您：

* 异步地调用函数
* 定时地调用函数
* 当单个内核对象已示意时调用函数
* 当异步 I/O 请求完成时调用函数

> 注：Microsoft 从 Windows 2000 开始就将线程池 API 引入 Windows。在 Windows Vista 中，Microsoft 重构了线程池，并且引入了一组新的线程池 API。

当一个进程初始化时，它没有任何与线程池组件相关的开销。但是，一旦调用了线程池函数，系统就会为进程创建一些内核资源，并且其中一些资源会一直保持到进程终止为止。使用线程池的开销取决于您的使用情况，线程池将代表进程分配一些线程、内核对象和内部数据结构。

## 异步地调用函数

要使用线程池异步地执行函数，首先需要定义一个与 [SimpleCallback](https://docs.microsoft.com/en-us/previous-versions/windows/desktop/legacy/ms686295(v=vs.85)) 原型匹配的函数：

```cpp
VOID NTAPI SimpleCallback(
    PTP_CALLBACK_INSTANCE pInstance,
    PVOID pvContext);
```

然后，向线程池提交请求，让其中一个线程执行该函数。要向线程池提交请求，只需调用 [`TrySubmitThreadpoolCallback`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-trysubmitthreadpoolcallback) 函数。[`TrySubmitThreadpoolCallback`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-trysubmitthreadpoolcallback) 函数会将一个 *工作项（Work Item）* 添加到线程池的队列中（通过调用 [`PostQueuedCompletionStatus`](https://docs.microsoft.com/en-us/windows/win32/fileio/postqueuedcompletionstatus)）。

> 注：您不需要自己调用 [`CreateThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createthread)。[`TrySubmitThreadpoolCallback`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-trysubmitthreadpoolcallback) 函数将自动为进程创建默认线程池，并让线程池中的一个线程调用您的回调函数。

线程池中的线程在处理完请求后不会立即销毁，而是返回到线程池中，以便准备处理其他入队的工作项。线程池不断重用其中的线程，而不是不断创建和销毁线程，这可以显著地提高应用程序的性能，因为创建和销毁线程需要花费大量时间。线程池会根据一个内部算法和应用程序的工作负载来调整自身的一些参数（如线程数）。

### 显式地控制工作项

在某些情况下（如缺少内存或配额限制），对 [`TrySubmitThreadpoolCallback`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-trysubmitthreadpoolcallback) 的调用可能会失败。当多个操作应当协同工作时，这是不可接受的。例如，一个定时器指望一个工作项取消另一个操作。当定时器被设置时，您必须确保被取消的工作项已提交并由线程池处理。但是，当定时器到期时，内存可用性或配额条件可能与创建定时器时不同，并且对 对 [`TrySubmitThreadpoolCallback`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-trysubmitthreadpoolcallback) 的调用可能会失败。在这种情况下，您将在创建定时器的同时创建一个工作项对象，并保留它，直到您明确需要将工作项提交到线程池。

每次调用 [`TrySubmitThreadpoolCallback`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-trysubmitthreadpoolcallback) 时，都会在内部代表您分配一个工作项。如果您计划提交大量工作项，那么您最好将工作项创建一次并多次提交，这样可以提高性能和减少内存消耗。您可以通过使用 [`CreateThreadpoolWork`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-createthreadpoolwork) 函数来创建工作项。当您要向线程池提交请求时，可以调用 [`SubmitThreadpoolWork`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-submitthreadpoolwork) 函数。

如果您有另一个线程想要取消已提交的工作项或挂起自身以等待工作项完成其处理，则可以调用 [`WaitForThreadpoolWorkCallbacks`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-waitforthreadpoolworkcallbacks) 函数。

当您不再需要工作项时，应调用 [`CloseThreadpoolWork`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-closethreadpoolwork) 函数来释放它。

## 定时地调用函数

有时应用程序需要在特定时间执行某些任务。Windows 提供了一个 [可等待定时器内核对象](https://docs.microsoft.com/en-us/windows/win32/sync/waitable-timer-objects) ，该对象可以轻松地获取基于时间的通知。

许多程序员会为应用程序将执行的每个基于时间的任务创建一个可等待定时器对象，这是不必要的，并且浪费系统资源。您可以创建单个可等待定时器，将其设置为下一个预定时间，然后为下一个时间重置定时器，依此类推。然而，完成此操作的代码编写起来十分棘手。幸运的是，您可以让线程池函数为您管理此操作。

若要计划在特定时间执行工作项，首先需要使用 [`TimeoutCallback`](https://docs.microsoft.com/en-us/previous-versions/windows/desktop/legacy/ms686790(v=vs.85)) 原型定义一个回调函数：

```cpp
VOID CALLBACK TimeoutCallback(
    PTP_CALLBACK_INSTANCE pInstance,
    PVOID pvContext,
    PTP_TIMER pTimer);
```

然后，通过调用 [`CreateThreadpoolTimer`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-createthreadpooltimer) 函数来通知线程池何时调用您的函数。

如果要向线程池注册定时器，请调用 [`SetThreadpoolTimer`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-setthreadpooltimer) 函数。您也可以通过调用 [`IsThreadpoolTimerSet`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-isthreadpooltimerset) 来确定是否设置了定时器。

最后，您可以通过调用 [`WaitForThreadpoolTimerCallbacks`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-waitforthreadpooltimercallbacks) 让线程等待定时器完成，也可以通过调用 [`CloseThreadpoolTimer`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-closethreadpooltimer) 函数来释放定时器的内存。

## 当单个内核对象已示意时调用函数

Microsoft 研究发现许多应用程序产生线程只是为了等待一个内核对象变为已示意状态。一旦对象已示意，线程就会将某种通知发布到另一个线程，然后循环返回，等待对象再次示意。

如果您想注册一个要在内核对象已示意时执行的工作项，首先，您编写一个与 [`WaitCallback`](https://docs.microsoft.com/en-us/previous-versions/windows/desktop/legacy/ms687017(v=vs.85)) 原型匹配的函数：

```cpp
VOID CALLBACK WaitCallback(
    PTP_CALLBACK_INSTANCE pInstance,
    PVOID Context,
    PTP_WAIT Wait,
    TP_WAIT_RESULT WaitResult);
```

然后，通过调用 [`CreateThreadpoolWait`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-createthreadpoolwait) 来创建一个线程池等待对象。

当准备就绪时，您需要通过调用 [`SetThreadpoolWait`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-setthreadpoolwait) 函数将内核对象绑定到此线程池等待对象。

在内部，线程池有一个线程调用 [`WaitForMultipleObjects`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitformultipleobjects) 函数，并将已通过 [`SetThreadpoolWait`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-setthreadpoolwait) 函数注册的句柄集传递给它，`bWaitAll` 参数设置为 `FALSE`，以便每当任何句柄已示意时，线程都会唤醒。此外，由于 [`WaitForMultipleObjects`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitformultipleobjects) 不允许将同一句柄多次传递给它，因此应确保不要使用 [`SetThreadpoolWait`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-setthreadpoolwait) 多次注册同一句柄。

一旦线程池线程调用了您的回调函数，相应的等待项将处于非活动状态。这意味着如果您希望在同一内核对象已示意时再次调用回调函数，则需要通过再次调用 [`SetThreadpoolWait`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-setthreadpoolwait) 来重新注册它。

最后，您可以通过调用 [`WaitForThreadpoolWaitCallbacks`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-waitforthreadpoolwaitcallbacks) 让线程等待等待项完成，也可以通过调用 [`CloseThreadpoolWait`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-closethreadpoolwait) 函数来释放等待项的内存。

## 当异步 I/O 请求完成时调用函数

当您希望将 IOCP 与线程池配合使用时，您必须告诉线程池当异步 I/O 操作完成时要调用哪个函数。

首先，您必须编写一个与 [`OverlappedCompletionRoutine`](https://docs.microsoft.com/en-us/previous-versions/windows/desktop/legacy/ms684124(v=vs.85)) 原型匹配的函数：

```cpp
VOID CALLBACK OverlappedCompletionRoutine(
    PTP_CALLBACK_INSTANCE pInstance,
    PVOID pvContext,
    PVOID pOverlapped,
    ULONG IoResult,
    ULONG_PTR NumberOfBytesTransferred,
    PTP_IO pIo);
```

然后，通过调用 [`CreateThreadpoolIo`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-createthreadpoolio) 创建一个线程池 I/O 对象。

准备就绪后，通过调用 [`StartThreadpoolIo`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-startthreadpoolio) 函数将 I/O 项中嵌入的文件/设备与线程池的内部 IOCP 相关联。

要注意的是，在每次调用 [`ReadFile`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-readfile) 和 [`WriteFile`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-writefile) 之前，必须先调用 [`StartThreadpoolIo`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-startthreadpoolio) 。如果在发出 I/O 请求之前未调用 [`StartThreadpoolIo`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-startthreadpoolio) ，则不会调用您的 [`OverlappedCompletionRoutine`](https://docs.microsoft.com/en-us/previous-versions/windows/desktop/legacy/ms684124(v=vs.85)) 回调函数。

如果要在发出 I/O 请求后停止调用您的回调函数，那么可以调用 [`CancelThreadpoolIo`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-cancelthreadpoolio) 函数。

当您使用完文件/设备后，您应调用 [`CloseHandle`](https://docs.microsoft.com/en-us/windows/win32/api/handleapi/nf-handleapi-closehandle) 将其关闭，并调用 [`CloseThreadpoolIo`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-closethreadpoolio) 函数将其与线程池解除关联。

您可以通过调用 [`WaitForThreadpoolIoCallbacks`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-waitforthreadpooliocallbacks) 函数让另一个线程等待未完成的 I/O 请求完成。

## 回调终止操作

线程池使回调方法能够方便地描述在回调函数返回时应执行的一些操作。您的回调函数有一个不透明的 `PTP_CALLBACK_INSTANCE` 类型的 `pInstance` 参数，该参数可以用于调用以下函数之一：

```cpp
VOID LeaveCriticalSectionWhenCallbackReturns(PTP_CALLBACK_INSTANCE pci, PCRITICAL_SECTION pcs);
VOID ReleaseMutexWhenCallbackReturns(PTP_CALLBACK_INSTANCE pci, HANDLE mut);
VOID ReleaseSemaphoreWhenCallbackReturns(PTP_CALLBACK_INSTANCE pci, HANDLE sem, DWORD crel);
VOID SetEventWhenCallbackReturns(PTP_CALLBACK_INSTANCE pci, HANDLE evt);
VOID FreeLibraryWhenCallbackReturns(PTP_CALLBACK_INSTANCE pci, HMODULE mod);
```

对于这些函数，线程池将执行下表指示的终止操作：

| Function | Termination Action |
| :-- | :-- |
| [`LeaveCriticalSectionWhenCallbackReturns`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-leavecriticalsectionwhencallbackreturns) | When the callback returns, the thread pool automatically calls [`LeaveCriticalSection`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-leavecriticalsection), passing the specified `CRITICAL_SECTION` structure. |
| [`ReleaseMutexWhenCallbackReturns`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-releasemutexwhencallbackreturns) | When the callback returns, the thread pool automatically calls [`ReleaseMutex`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-releasemutex), passing the specified `HANDLE`. |
| [`ReleaseSemaphoreWhenCallbackReturns`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-releasesemaphorewhencallbackreturns) | When the callback returns, the thread pool automatically calls [`ReleaseSemaphore`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-releasesemaphore), passing the specified `HANDLE`. |
| [`SetEventWhenCallbackReturns`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-seteventwhencallbackreturns) | When the callback returns, the thread pool automatically calls [`SetEvent`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-setevent), passing the specified `HANDLE`. |
| [`FreeLibraryWhenCallbackReturns`](https://docs.microsoft.com/en-us/windows/win32/api/threadpoolapiset/nf-threadpoolapiset-freelibrarywhencallbackreturns) | When the callback returns, the thread pool automatically calls [`FreeLibrary`](https://docs.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-freelibrary), passing the specified `HMODULE`. |

> 注：对于给定的回调实例，线程池线程只应用一个终止效果。调用的最后一个函数将覆盖前一个函数。

除了这些终止函数之外，还有两个额外的函数适用于回调实例：

```cpp
BOOL CallbackMayRunLong(PTP_CALLBACK_INSTANCE pci);
VOID DisassociateCurrentThreadFromCallback(PTP_CALLBACK_INSTANCE pci);
```
