---
title: 'Windows via C/C++, 5th Edition - Thread Synchronization in User Mode'
date: 2022-02-18 16:01:09
category: 'My Notes'
tags: ['Windows via C/C++', 'Windows']
---

线程需要如下两种情况下进行同步：

* 当有多个线程访问共享资源而不使资源被破坏时。
* 当一个线程需要通知一个或多个其他线程特定任务已完成时。

## 原子访问

线程同步的很大一部分与原子访问（Atomic Access）相关，即线程访问一个资源并保证没有其他线程同时访问同一资源的能力。

Windows 提供了[互锁（Interlocked）函数族](https://docs.microsoft.com/en-us/windows/win32/sync/synchronization-functions#interlocked-functions)，这些函数允许以原子操作方式对一个值进行操作，如 [`InterlockedExchangeAdd`](https://docs.microsoft.com/en-us/windows/win32/api/winnt/nf-winnt-interlockedexchangeadd) 函数。

互锁函数执行速度是非常快的，通常执行只需要很少的（通常小于 50 个）CPU 周期，并且不会从用户模式切换到内核模式（通常需要超过 1000 个 CPU 周期）。

[`InterlockedExchange`](https://docs.microsoft.com/en-us/windows/win32/api/winnt/nf-winnt-interlockedexchange) 和 [`InterlockedExchangePointer`](https://docs.microsoft.com/en-us/windows/win32/api/winnt/nf-winnt-interlockedexchangepointer) 以原子方式将第一个指针参数所指向位置的当前值替换为第二个参数的值，并返回第一个指针参数所指向位置的原始值。[`InterlockedExchange`](https://docs.microsoft.com/en-us/windows/win32/api/winnt/nf-winnt-interlockedexchange) 在实现自旋锁（Spinlock）时非常有用：

```cpp
// Global variable indicating whether a shared resource is in use or not
BOOL g_fResourceInUse = FALSE; ...
void Func1() {
    // Wait to access the resource.
    while (InterlockedExchange(&g_fResourceInUse, TRUE) == TRUE)
        Sleep(0);

    // Access the resource.
    ...

    // We no longer need to access the resource.
    InterlockedExchange(&g_fResourceInUse, FALSE);
}
```

`while` 循环反复旋转，将 `g_fResourceInUse` 中的值更改为 `TRUE`，并检查其原先的值是否为 `TRUE`。如果该值为 `FALSE`，则表示资源不是“使用中”，但调用线程只是将其设置为“使用中”并退出循环；如果该值为 `TRUE`，则表示资源正由另一个线程“使用中”，并且 `while` 循环继续旋转。

如果另一个线程执行类似的代码，它将在其 `while` 循环中旋转，直到 `g_fResourceInUse` 变回 `FALSE`。函数末尾调用了 [`InterlockedExchange`](https://docs.microsoft.com/en-us/windows/win32/api/winnt/nf-winnt-interlockedexchange) 以将 `g_fResourceInUse` 设置回 `FALSE`。

使用此技术时必须格外小心，因为自旋锁会浪费 CPU 时间。CPU 必须不断比较两个值，直到一个值因另一个线程“神奇地”更改。此代码还假定使用自旋锁的所有线程都在同一优先级运行。

此外，还应确保锁变量和锁保护的数据在不同的缓存行中维护。如果锁变量和数据共享同一缓存行，则使用该资源的 CPU 将与任何尝试访问该资源的 CPU 争用，这会损害性能。

应避免在单 CPU 的机器上使用自旋锁。如果一个线程正在旋转，那么它会浪费宝贵的 CPU 时间，还妨碍了其他线程更改该值。在前面展示的 `while` 循环中使用 [`Sleep`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-sleep) 可在一定程度上改善这种情况。如果您使用 [`Sleep`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-sleep)，那么您可能希望睡眠一个随机的时间，并且每次访问资源的请求被拒绝时，您可能希望进一步增加睡眠时间。这可以防止线程简单地浪费 CPU 时间。根据具体情况，最好是完全删除对 [`Sleep`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-sleep) 的调用；或者将其替换为对 [`SwitchToThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-switchtothread) 的调用。

自旋锁假定始终能在短时间内访问受保护的资源，这使得它比旋转然后切换到内核模式并等待更有效。许多开发者会旋转一定次数（如 4000 次），如果仍然拒绝访问资源，那么就将线程切换到内核模式并等待（不消耗 CPU 时间），直到资源变为可用。

自旋锁在多 CPU 的机器上很有用，因为一个线程可以在一个 CPU 上旋转，而另一个线程在另一个 CPU 上运行。即便如此，也必须小心地使用自旋锁。

## 缓存行

如果要构建在多处理器计算机上运行的高性能应用程序，那么必须了解 CPU 缓存行（Cache Lines）。当 CPU 从内存中读取一个字节时，它不仅会获取单个字节，还会提取足够的字节来填充缓存行。缓存行由 32 个字节（对于较旧的 CPU）、64 个字节甚至 128 个字节（取决于 CPU）组成，并且它们始终分别地在 32 字节、64 字节或 128 字节边界上对齐。缓存行的存在是为了提高性能。通常，应用程序会操作一组相邻的字节。如果这些字节在缓存中，则 CPU 不必访问内存总线。

但是，在多处理器环境中，缓存行会使得内存更新变得更加困难，如以下示例所示：

1. CPU1 读取一个字节，导致该字节及其相邻字节被读入 CPU1 的缓存行。
2. CPU2 读取相同的字节，这导致步骤 1 中的相同字节被读入 CPU2 的缓存行。
3. CPU1 更改内存中的字节，导致该字节被写入到 CPU1 的缓存行。但该信息尚未被写入到 RAM。
4. CPU2 再次读取相同的字节。由于此字节已位于 CPU2 的缓存行中，因此它不必访问内存。但是 CPU2 不会在内存中看到该字节的新值。

这种情况将是灾难性的。当然，芯片设计人员很清楚这个问题，并在设计 CPU 时处理这个问题。具体而言，当 CPU 更改缓存行中的字节时，计算机中的其他 CPU 会意识到这一点，并且其缓存行将失效。因此，在刚刚展示的示例中，当 CPU1 更改字节的值时，CPU2 的缓存将失效。在步骤 4 中，CPU1 必须将其缓存刷新到 RAM，而 CPU2 必须再次访问内存以重新填充其缓存行。缓存行可以帮助提高性能，但它们也可能对多处理器计算机造成不利影响。

这一切意味着，您应该将应用程序的数据分组到缓存行中（大小块和缓存行边界）。目的是确保不同的 CPU 访问由至少一个缓存行边界分隔的不同的内存地址。此外，您应该将只读数据（或不经常读取的数据）与读写数据分开。并且，您应该将大概率会同时访问的数据片段组合在一起。

下面是一个设计不佳的数据结构示例：

```cpp
struct CUSTINFO {
    DWORD dwCustomerID;       // Mostly read-only
    int nBalanceDue;          // Read-write
    wchar_t szName[100];      // Mostly read-only
    FILETIME ftLastOrderDate; // Read-write
};
```

确定 CPU 的缓存行大小的最简单方法是调用 Win32 的 [`GetLogicalProcessorInformation`](https://docs.microsoft.com/en-us/windows/win32/api/sysinfoapi/nf-sysinfoapi-getlogicalprocessorinformation) 函数。此函数返回 `SYSTEM_LOGICAL_PROCESSOR_INFORMATION` 结构的数组。您可以检查结构的 `Cache` 字段，该字段引用了一个 `CACHE_DESCRIPTOR` 结构，其中包含指示 CPU 缓存行大小的 `LineSize` 字段。获得此信息后，可以使用 C/C++ 编译器的 [`__declspec(align(#))`](https://docs.microsoft.com/en-us/cpp/cpp/align-cpp) 指令来控制字段对齐。以下是此数据结构的改进版本：

```cpp
#define CACHE_ALIGN 64

// Force each structure to be in a different cache line.
struct __declspec(align(CACHE_ALIGN)) CUSTINFO {
    DWORD dwCustomerID;  // Mostly read-only
    wchar_t szName[100]; // Mostly read-only

    // Force the following members to be in a different cache line.
    __declspec(align(CACHE_ALIGN))
    int nBalanceDue;          // Read-write
    FILETIME ftLastOrderDate; // Read-write
};
```

> 注：最好始终由单个线程访问数据（函数参数和局部变量是确保这一点的最简单方法），或者始终由单个 CPU 访问数据（使用线程亲和性）。这样的话，可以完全避免缓存行问题。

## 高级线程同步

当线程想要访问共享资源或收到某些“特殊事件”的通知时，该线程必须调用操作系统函数，并向其传递参数以指示线程正在等待什么。如果操作系统检测到资源可用或发生了特殊事件，则该函数将返回，并且线程保持可调度状态。如果资源不可用或特殊事件尚未发生，系统会将线程置于等待状态，从而使线程不可调度，这可以防止线程浪费任何 CPU 时间。当线程正在等待时，系统将为线程充当代理。系统会记住线程想要什么，并在资源变为可用时（线程的执行与特殊事件同步）自动地使线程退出等待状态。

事实上，大多数线程几乎总是处于等待状态。当系统检测到所有线程处于等待状态几分钟后，系统的电源管理就会开始起效。

### 要避免的技术

如果没有同步对象并且操作系统没有监视特殊事件的能力，线程将被迫使用如下的技术将自己与特殊事件同步。然而，如果操作系统具有对线程同步的内置支持，则切勿使用此技术：

```cpp
volatile BOOL g_fFinishedCalculation = FALSE;

int WINAPI _tWinMain(...) {
    CreateThread(..., RecalcFunc, ...);
    ...
    // Wait for the recalculation to complete.
    while (!g_fFinishedCalculation)
        ;
    ...
}

DWORD WINAPI RecalcFunc(PVOID pvParam) {
    // Perform the recalculation.
    ...
    g_fFinishedCalculation = TRUE;

    return(0);
}
```

在此技术中，一个线程通过连续轮询由多个线程共享或可访问的变量的状态，将自身与另一个线程中任务的完成同步。

> 注：此处需要使用 `volatile` 类型限定符。这告诉编译器，变量可以被应用程序本身之外的东西（如操作系统、硬件或并发执行的线程）修改。

## 临界区段

一个 *临界区段（Critical Section）* 是代码的一个小部分，其需要对某些共享资源进行独占访问。这是一种让数行代码“原子”操作资源的方法。“原子”意味着在一个线程离开临界区段之前，其他线程不能进入该临界区段。

以下是一段有问题的代码，演示了不使用临界区段会发生什么情况：

```cpp
const int COUNT = 1000;
int g_nSum = 0;

DWORD WINAPI FirstThread(PVOID pvParam) {
    g_nSum = 0;
    for (int n = 1; n <= COUNT; n++) {
        g_nSum += n;
    }
    return(g_nSum);
}

DWORD WINAPI SecondThread(PVOID pvParam) {
    g_nSum = 0;
    for (int n = 1; n <= COUNT; n++) {
        g_nSum += n;
    }
    return(g_nSum);
}
```

由于两个线程都访问共享变量（`g_nSum`），因此如果两个线程同时执行（可能在不同的 CPU 上），则每个线程都会在另一个线程的后面修改 `g_nSum`，从而导致不可预知的结果。

现在使用临界区段更正代码：

```cpp
const int COUNT = 10;
int g_nSum = 0;
CRITICAL_SECTION g_cs;

DWORD WINAPI FirstThread(PVOID pvParam) {
    EnterCriticalSection(&g_cs);
    g_nSum = 0;
    for (int n = 1; n <= COUNT; n++) {
        g_nSum += n;
    }
    LeaveCriticalSection(&g_cs);
    return(g_nSum);
}

DWORD WINAPI SecondThread(PVOID pvParam) {
    EnterCriticalSection(&g_cs);
    g_nSum = 0;
    for (int n = 1; n <= COUNT; n++) {
        g_nSum += n;
    }
    LeaveCriticalSection(&g_cs);
    return(g_nSum);
}
```

首先分配了一个 `CRITICAL_SECTION` 数据结构 `g_cs`，然后将任何涉及共享资源（`g_nSum`）的代码包装在对 [`EnterCriticalSection`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-entercriticalsection) 和 [`LeaveCriticalSection`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-leavecriticalsection) 的调用中。请注意，在对 [`EnterCriticalSection`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-entercriticalsection) 和 [`LeaveCriticalSection`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-leavecriticalsection) 的所有调用中都传递了 `g_cs` 的地址。

### 临界区段与自旋锁

当一个线程尝试进入另一个线程拥有的临界区段时，调用线程将立即进入等待状态。这意味着线程必须从用户模式切换到内核模式（大约 1000 个 CPU 周期），这种切换非常昂贵。在多处理器计算机上，当前拥有资源的线程可能会在其他处理器上执行，并可能很快放弃对资源的控制。拥有该资源的线程可能会在其他线程完成到内核模式的切换之前释放资源。这种情况情况下会浪费大量 CPU 时间。

为了提高临界区段的性能，Microsoft 已将自旋锁合并到其中。因此，当调用 [`EnterCriticalSection`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-entercriticalsection) 时，它会使用自旋锁循环，尝试多次获取资源。仅当所有尝试都失败时，线程才会切换到内核模式以进入等待状态。

要将自旋锁与临界区段一起使用，您应该调用 [`InitializeCriticalSectionAndSpinCount`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-initializecriticalsectionandspincount) 函数来初始化临界区段。

### 临界区段与错误处理

对 [`InitializeCriticalSection`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-initializecriticalsection) 函数的调用可能会失败（可能性很小）。微软在最初设计该函数时并没有真正考虑过这一点，这就是为什么该函数被原型化为返回 `VOID` 的原因。该函数可能会失败，因为它会分配一个内存块，以便系统可以具有一些内部调试信息。如果此内存分配失败，则会引发 `STATUS_NO_MEMORY` 异常。您可以使用结构化异常处理将其捕获到代码中。

## SRWLock

SRWLock（Slim Reader-Writer Lock）与简单的临界区段具有相同的用途：保护单个资源免受不同线程的访问。但是，与临界区段不同，SRWLock 允许您区分只想读取资源值的线程（读者）和尝试更新资源值的其他线程（写者）。所有读者应该可以同时访问共享资源，因为只读取资源的值不会有数据损坏的风险。当写者想要更新资源时，就需要同步，此时访问应该是独占的（不允许任何其他线程访问资源）。

要使用 SRWLock，首先需要分配一个 `SRWLOCK` 结构并使用 [`InitializeSRWLock`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-initializesrwlock) 函数初始化它。

初始化 SRWLock 后，写者可以尝试通过调用 [`AcquireSRWLockExclusive`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-acquiresrwlockexclusive) 来获取对资源的独占访问权限。更新资源后，通过调用 [`ReleaseSRWLockExclusive`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-releasesrwlockexclusive) 来释放锁。

对于读者，则使用以下两个函数来获取资源访问的权限：

```cpp
VOID AcquireSRWLockShared(PSRWLOCK SRWLock);
VOID ReleaseSRWLockShared(PSRWLOCK SRWLock);
```

## 条件变量

在一些需要同步的情景中，线程必须以原子方式释放资源上的锁并阻塞，直到满足某个条件为止。使用 *条件变量（Condition Variables）* 可以简化这些场景所需的代码，这可以通过 [`SleepConditionVariableCS`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-sleepconditionvariablecs) 或 [`SleepConditionVariableSRW`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-sleepconditionvariablesrw) 函数来实现。
