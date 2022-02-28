---
title: 'Windows via C/C++, 5th Edition - Thread Synchronization with Kernel Objects'
date: 2022-02-23 17:14:43
category: 'My Notes'
tags: ['Windows via C/C++', 'Windows']
---

用户模式同步的优点在于它非常的快，如果需要强调线程的性能，那么应该首先考虑使用用户模式线程同步机制。尽管用户模式线程同步机制提供了出色的性能，但它们都有一些局限性（如互锁的函数族仅对单个值进行操作）。实际上，内核对象机制比用户模式机制更通用，唯一的不足是性能不如用户模式机制。使用内核对象机制时，调用线程必须从用户模式切换到内核模式，这种切换开销很大（在 x86 平台上，一个空系统调用大约需要 200 个 CPU 周期）。

几乎可以将所有的内核对象用于同步。对于线程同步而言，这些内核对象都处于 *已示意（Signaled）* 或 *未示意（Nonsignaled）* 状态。状态的切换由 Microsoft 为每个对象创建的规则确定。例如，进程内核对象始终以未示意状态创建。当进程终止时，操作系统会自动地使进程内核对象处于已示意状态。一旦进程内核对象已示意，它就会永远保持这种状态（其状态永远不会变回未示意）。如果希望检查一个进程是否仍在运行，那么仅仅需要调用一个函数以请求操作系统检查进程对应的进程内核对象是否已示意即可。

以下内核对象均可处于已示意或未示意状态：

* 进程（Process）
* 线程（Thread）
* 作业（Job）
* 文件和控制台标准输入/输出/错误流（File and console standard input/output/error streams）
* 事件（Event）
* 可等待定时器（Waitable timer）
* 信号量（Semaphore）
* 互斥量（Mutex）

线程可以将自身置于等待状态，以等待某个内核对象变为已示意。当线程正在等待的对象未示意时，线程是不可调度的；一旦等待的对象已示意，线程就会观察到这一变化从而变为可调度的，并且很快恢复执行。

## 等待函数

*等待函数（Wait Functions）* 使线程自愿地将自己置于等待状态，直到特定的内核对象变为已示意。如果在调用等待函数时内核对象已示意，那么线程不会进入等待状态。

最常用的等待函数是 [`WaitForSingleObject`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitforsingleobject)。如下是一个使用示例：

```cpp
DWORD dw = WaitForSingleObject(hProcess, 5000);
switch (dw) {
case WAIT_OBJECT_0:
    // The process terminated.
    break;

case WAIT_TIMEOUT:
    // The process did not terminate within 5000 milliseconds.
    break;

case WAIT_FAILED:
    // Bad call to function (invalid handle?)
    break;
}
```

[WaitForMultipleObjects](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitformultipleobjects) 与 [`WaitForSingleObject`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitforsingleobject) 类似，但它允许调用线程同时检查多个内核对象的示意状态。如下是一个使用示例：

```cpp
HANDLE h[3];
h[0] = hProcess1;
h[1] = hProcess2;
h[2] = hProcess3;
DWORD dw = WaitForMultipleObjects(3, h, FALSE, 5000);
switch (dw) {
case WAIT_FAILED:
    // Bad call to function (invalid handle?)
    break;

case WAIT_TIMEOUT:
    // None of the objects became signaled within 5000 milliseconds.
    break;

case WAIT_OBJECT_0 + 0:
    // The process identified by h[0] (hProcess1) terminated.
    break;

case WAIT_OBJECT_0 + 1:
    // The process identified by h[1] (hProcess2) terminated.
    break;

case WAIT_OBJECT_0 + 2:
    // The process identified by h[2] (hProcess3) terminated.
    break;
}
```

### 成功等待的副作用

对于某些内核对象，成功地调用 [`WaitForSingleObject`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitforsingleobject) 或 [WaitForMultipleObjects](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitformultipleobjects) 实际上会改变对象的状态。

当使一个对象的状态改变时，称之为 *成功等待的副作用（Successful Wait Side Effect）* 。例如，假设一个线程正在等待一个自动重置事件对象。当事件对象变为已示意时，等待函数会检测到这一点，并可以将 `WAIT_OBJECT_0` 返回给调用线程。然而，在函数返回之前，事件会被设置为未示意状态，这就是成功等待的副作用。

[WaitForMultipleObjects](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitformultipleobjects) 以原子方式执行其所有操作（这防止了死锁的情况）。当线程调用 [WaitForMultipleObjects](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitformultipleobjects) 时，该函数可以测试所有对象的示意状态，并将所需的所有副作用作为单个操作执行。例如，两个线程以完全相同的方式调用 [WaitForMultipleObjects](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitformultipleobjects)：

```cpp
HANDLE h[2];
h[0] = hAutoResetEvent1; // Initially nonsignaled
h[1] = hAutoResetEvent2; // Initially nonsignaled
WaitForMultipleObjects(2, h, TRUE, INFINITE);
```

当调用 [WaitForMultipleObjects](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitformultipleobjects) 时，两个事件对象都是未示意的，这将强制两个线程都进入等待状态。随后 `hAutoResetEvent1` 对象变为已示意。两个线程都观察到该事件已示意，但两者都不会唤醒，因为 `hAutoResetEvent2` 对象仍为未示意。由于两个线程都尚未成功等待，因此在 `hAutoResetEvent1` 对象上不会发生任何副作用。

接着，`hAutoResetEvent2` 对象变为已示意。此时，两个线程中的一个会检测到它正在等待的两个对象都已示意，该线程等待成功，两个事件对象都会被设置为非示意状态，并且该线程变为可调度的；另一个线程将继续等待，直到它看到两个事件对象都已示意为止（尽管它最初检测到 `hAutoResetEvent1` 已示意，但现在它将观察到该对象是未示意的）。

> 注：Microsoft 官方表示在多个线程等待单个内核对象的场景下，当对象变成已示意时通过一个公平的算法来选择要唤醒的线程。

## 事件内核对象

在所有内核对象中，事件是最原始的。它们包含一个使用计数、一个指示事件是自动重置（Auto-reset）事件还是手动重置（Manual-reset）事件的布尔值，以及一个指示事件是已示意的还是未示意的的布尔值。

事件表示一个操作已完成。有两种不同类型的事件对象：手动重置事件和自动重置事件。当手动重置事件已示意时，等待该事件的所有线程将变为可调度的；当自动重置事件已示意时，等待该事件的所有线程中只有一个线程变为可调度的。

使用 [`CreateEvent`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-createeventw) 函数可以创建一个事件内核对象。Windows Vista 提供了一个新的函数 [`CreateEventEx`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-createeventexw) 用于创建事件。

其他进程中的线程可以通过使用相同的事件对象名字来调用 [`CreateEvent`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-createeventw) 以访问同一个对象。或者，也可以通过调用 [`OpenEvent`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-openeventw) 来实现类似的效果。

与往常一样，当不再需要事件内核对象时，应调用 [`CloseHandle`](https://docs.microsoft.com/en-us/windows/win32/api/handleapi/nf-handleapi-closehandle) 函数。

创建事件后，可以直接控制其状态。调用 [`SetEvent`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-setevent) 会将事件更改为已示意状态；而调用 [`ResetEvent`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-resetevent) 则会将事件更改为未示意状态。

Microsoft 为自动重置事件定义了成功等待的副作用规则：当线程成功等待对象时，自动重置事件将自动重置为未示意状态。Microsoft 没有为手动重置事件定义成功等待的副作用。

当多个线程等待同一个事件对象时，若该事件对象是一个手动重置事件，则所有等待线程会被唤醒；若该事件对象是一个自动重置事件，则只有其中一个等待线程会被唤醒。

[`PulseEvent`](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-pulseevent) 使事件变为已示意然后立即变为非示意，就像在调用 [`SetEvent`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-setevent) 后立即调用 [`ResetEvent`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-resetevent) 一样。此函数并不可靠，因此很少会使用它，保留它仅是为了向后兼容。

## 可等待定时器内核对象

可等待定时器是在特定时间或定期间隔发出信号的内核对象，最常用于在特定时间执行某些操作。

要创建可等待定时器，只需简单地调用 [`CreateWaitableTimer`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-createwaitabletimerw)。

进程可以通过调用 [`OpenWaitableTimer`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-openwaitabletimerw) 来获取其现有的可等待定时器的进程相关句柄。

与事件一样，可等待定时器分为手动重置定时器和自动重置定时器。当手动重置定时器已示意时，所有等待定时器的线程将变为可调度的；当自动重置定时器已示意时，只有一个等待线程变为可调度的。

可等待定时器对象始终以未示意状态创建。必须调用 [`SetWaitableTimer`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-setwaitabletimer) 函数以告诉定时器何时应变为已示意的。

[`CancelWaitableTimer`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-cancelwaitabletimer) 函数用于将指定的可等待定时器设置为非活动状态。

### 让可等待定时器对 APC 条目进行排队

Microsoft 允许定时器将一个 APC（Asynchronous Procedure Call）排队到一个线程，该线程在定时器已示意时调用了 [`SetWaitableTimer`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-setwaitabletimer)。

当且仅当调用 [`SetWaitableTimer`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-setwaitabletimer) 的线程处于 *可警示（Alertable）* 状态时，定时器 APC 例程才会在定时器响起时由调用 [`SetWaitableTimer`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-setwaitabletimer) 的同一个线程调用。换句话说，该线程必须在调用 [`SleepEx`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-sleepex)、[`WaitForSingleObjectEx`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitforsingleobjectex)、[`WaitForMultipleObjectsEx`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitformultipleobjectsex)、[`MsgWaitForMultipleObjectsEx`](https://docs.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-msgwaitformultipleobjectsex) 或 [`SignalObjectAndWait`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-signalobjectandwait) 中等待。如果线程未在这些函数之一中等待，那么系统不会将定时器 APC 例程入队。这可以防止线程的 APC 队列因定时器 APC 通知而过载，从而浪费系统内的大量内存。

如果线程在定时器响起时处于可警示的等待状态，那么系统会使线程调用回调例程。只有在处理完所有 APC 条目后，才会从可警示函数中返回。因此，必须确保定时器 APC 例程函数在定时器再次发出信号之前完成执行，以便 APC 条目的入队速度不会快于其处理速度。

以下代码展示了使用定时器和 APC 的正确方法：

```cpp
void SomeFunc() {
    // Create a timer. (It doesn't matter whether it's manual-reset or auto-reset.)
    HANDLE hTimer = CreateWaitableTimer(NULL, TRUE, NULL);

    // Set timer to go off in 5 seconds.
    LARGE_INTEGER li = { 0 };
    SetWaitableTimer(hTimer, &li, 5000, TimerAPCRoutine, NULL, FALSE);

    // Wait in an alertable state for the timer to go off.
    SleepEx(INFINITE, TRUE);

    CloseHandle(hTimer);
}
```

要注意的是，线程不应该等待定时器的句柄，也不应该警示地等待定时器，如下：

```cpp
HANDLE hTimer = CreateWaitableTimer(NULL, FALSE, NULL);
SetWaitableTimer(hTimer, ..., TimerAPCRoutine,...);
WaitForSingleObjectEx(hTimer, INFINITE, TRUE);
```

您不应该编写这样的代码，因为对 [`WaitForSingleObjectEx`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitforsingleobjectex) 的调用实际上等待了定时器两次：警示地等待和等待内核对象句柄。当定时器变为已示意时，等待成功，线程唤醒，这会使线程退出可警示状态，并且不会调用 APC 例程。

## 信号量内核对象

信号量内核对象用于资源计数。除了使用计数外，其还包含两个额外的带符号的 32 位值：最大资源计数和当前资源计数。最大资源计数标识信号量可以控制的最大资源数，当前资源计数指示当前可用的这些资源的数量。

信号量的规则如下：

* 若当前资源计数大于 0，则信号量为已示意。
* 若当前资源计数为 0，则信号量为未示意。
* 系统绝不允许当前资源计数为负数。
* 当前资源计数不能大于最大资源计数。

[`CreateSemaphore`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-createsemaphorew) 和 [`CreateSemaphoreEx`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-createsemaphoreexw) 函数用于创建一个信号量内核对象。[`OpenSemaphore`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-opensemaphorew) 函数用于获取一个现存的信号量的进程相关句柄。

线程通过调用等待函数来获取对资源的访问权限。等待函数会检查信号量的当前资源计数：

* 如果其值大于 0（信号量已示意），那么计数器将递减 1，并且调用线程保持可调度状态。
* 如果其值为 0（信号量未示意），那么系统会将调用线程置于等待状态。当其它线程递增信号量的当前资源计数时，系统会选择等待该信号量的其中一个线程并允许其变为可调度。相应地，系统会递减当前资源计数。

> 注：这些对信号量的测试和设置操作是以原子方式执行的，即当从信号量请求资源时，操作系统会检查该资源是否可用，并在不让其他线程干扰的情况下递减可用资源的计数。只有在资源计数递减后，系统才会允许另一个线程请求访问资源。

线程可以通过调用 [`ReleaseSemaphore`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-releasesemaphore) 来递增信号量的当前资源计数。

## 互斥量内核对象

互斥量内核对象确保线程对单个资源具有互斥访问权限。互斥量对象包含使用计数、线程 ID 和递归计数器。互斥量的行为与临界区段相同，但是互斥量是内核对象，而临界区段是用户模式同步对象。这意味着互斥量比临界区段慢，但也意味着不同进程中的线程可以访问单个互斥量，并且线程可以在等待访问资源时指定超时值。

线程 ID 用于标识系统中当前拥有互斥量的线程，递归计数器指示此线程拥有互斥量的次数。

互斥量的规则如下：

* 若线程 ID 为 0（无效的线程 ID），则互斥量不归任何线程所有并且互斥量为已示意的。
* 若线程 ID 为非零值，则对应的线程拥有互斥量并且互斥量为未示意的。
* 与所有其他内核对象不同，互斥量在操作系统中具有允许它们违反正常规则的特殊代码。

[`CreateMutex`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-createmutexw) 和 [`CreateMutexEx`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-createmutexexw) 函数用于创建一个互斥量。[`OpenMutex`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-openmutexw) 函数用于获取一个现存的互斥量的进程相关句柄。

线程通过调用等待函数来获取对共享资源的访问权限。等待函数会检查互斥量的线程 ID：

* 如果线程 ID 为 0（互斥量已示意），那么线程 ID 设置为调用线程的 ID，递归计数器设置为 1，并且调用线程保持可调度状态。
* 如果线程 ID 不为 0（互斥量未示意），那么调用线程将进入等待状态。当互斥量的线程 ID 设置回 0 时，系统会选择等待该互斥量的其中一个线程，并且将线程 ID 设置为该线程的 ID，将递归计数器设置为 1，允许该线程变为可调度。

> 注：这些对互斥量内核对象的检查和更改是以原子方式执行的。

互斥量在正常的内核对象已示意/未示意规则下有一个特殊的例外。假设一个线程尝试等待一个未示意的互斥量对象。在这种情况下，该线程通常处于一个等待状态。然而，系统会检查尝试获取互斥量的线程是否具有与互斥量对象中记录的线程 ID 相同的线程 ID。如果线程 ID 匹配，那么即使互斥量是未示意的，系统也会允许该线程保持可调度状态。

线程可以通过调用 [`ReleaseMutex`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-releasemutex) 来递减互斥量的递归计数器。当互斥量的递归计数器递减为 0 时，其线程 ID 也会被设置为 0，并且该互斥量会变为已示意的。

### 遗弃问题

与其他内核对象不同，互斥量对象具有“线程所有权”的概念。其他内核对象不会记住哪个线程成功地等待了它，只有互斥量会跟踪这一点。如果拥有互斥量的线程在释放互斥量之前终止，那么系统会认为该互斥量是被遗弃的。

由于系统跟踪所有互斥量内核对象和线程内核对象，因此它确切地知道互斥量何时被放弃。当互斥量被遗弃时，系统会自动将互斥量对象的线程 ID 重置为 0，并将其递归计数器重置为 0。然后，系统会检查当前是否有任何线程正在等待该互斥量。若是，则系统会“公平地”选择一个等待线程。

### 互斥量 vs 临界区段

| Characteristic | Mutex | Critical Section |
| :-- | :-- | :-- |
| Performance | Slow | Fast |
| Can be used across process boundaries | Yes | No |
| Declaration | `HANDLE hmtx;` | `CRITICAL_SECTION cs;` |
| Initialization | `hmtx = CreateMutex (NULL, FALSE, NULL);` | `InitializeCriticalSection(&cs);` |
| Cleanup | `CloseHandle(hmtx);` | `DeleteCriticalSection(&cs);` |
| Infinite wait | `WaitForSingleObject (hmtx, INFINITE);` | `EnterCriticalSection(&cs);` |
| 0 wait | `WaitForSingleObject (hmtx, 0);` | `TryEnterCriticalSection(&cs);` |
| Arbitrary wait | `WaitForSingleObject (hmtx, dwMilliseconds);` | Not possible |
| Release | `ReleaseMutex(hmtx);` | `LeaveCriticalSection(&cs);` |
| Can be waited on with other kernel objects | Yes (use `WaitForMultipleObjects` or similar function) | No |
