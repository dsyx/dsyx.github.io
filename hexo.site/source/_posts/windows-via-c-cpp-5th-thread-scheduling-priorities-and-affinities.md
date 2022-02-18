---
title: 'Windows via C/C++, 5th Edition - Thread Scheduling, Priorities, and Affinities'
date: 2022-02-17 14:42:49
category: 'My Notes'
tags: ['Windows via C/C++', 'Windows', 'C/C++']
---

抢占式操作系统必须使用某种算法来确定线程应该何时调度以及运行多长时间。每个线程都有一个在线程内核对象中维护的上下文结构。此上下文结构反映线程上次执行时线程的 CPU 寄存器的状态。每隔 20 ms 左右，Windows 会查看当前存在的所有线程内核对象，选择一个可调度的线程内核对象，并使用上次保存在线程上下文中的值加载 CPU 的寄存器。此操作称为 *上下文切换（Context Switch）* 。

## 挂起或恢复线程

线程内核对象内部有一个值，该值指示线程的挂起计数。调用 [`CreateProcess`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessw) 或 [`CreateThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createthread) 时，将创建线程内核对象并将其挂起计数初始化为 `1`。这可以防止将线程即刻调度到 CPU，因为初始化线程需要一些时间，不希望系统在线程完全准备就绪之前开始执行线程。

线程完全初始化后，[`CreateProcess`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessw) 或 [`CreateThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createthread) 将检查是否传递了 `CREATE_SUSPENDED` 标志。若是，则函数返回，新线程将保持挂起状态；若否，则函数会将线程的挂起计数递减为 `0`。

调用 [`ResumeThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-resumethread) 可使线程恢复运行，调用 [`SuspendThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-suspendthread) 则可使线程挂起。线程可以被挂起多次，但也必须被恢复同样的次数才能使其可调度。

## 挂起或恢复进程

对于 Windows 来说，不存在挂起或恢复进程的概念，因为进程永远不会调度到 CPU。可以通过挂起进程中的所有线程来达到类似的效果。

## 睡眠

线程可以调用 [`Sleep`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-sleep) 函数来告诉系统它不希望在一定时间内被调度。此函数使线程挂起，直到经过指定的时间。使用此函数时需要注意如下事项：

* 调用 [`Sleep`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-sleep) 允许线程自愿地放弃其剩余的时间片。
* 系统使线程在指定时间内不可调度，但不保证线程在指定时间后会被及时调度。
* 可以调用 [`Sleep`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-sleep) 并传递 `INFINITE` 给参数，以告诉系统永远不要调度线程。
* 可以调用 [`Sleep`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-sleep) 并传递 `0` 给参数，以告诉系统线程自愿地放弃其剩余的时间片，系统可以调度其他线程。如果没有更多具有相同优先级或更高优先级的可调度线程，那么系统可以重新调度该线程。

## 切换到另一线程

系统提供了一个名为 [`SwitchToThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-switchtothread) 的函数，该函数允许另一个可调度线程运行。

调用此函数时，系统将检查是否存在饥饿线程。若否，则 [`SwitchToThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-switchtothread) 会立即返回；若是，则 [`SwitchToThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-switchtothread) 将调度该线程。允许饥饿线程运行一个时间片，然后系统调度器将照常运行。

调用 [`SwitchToThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-switchtothread) 类似于调用 [`Sleep`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-sleep) 并传递 `0`。不同之处在于，[`SwitchToThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-switchtothread) 允许执行优先级较低的线程。

### 在超线程 CPU 上切换到另一线程

超线程（Hyper-Threaded）CPU 具有多个“逻辑（logical）”CPU，每个都可以运行一个线程。每个线程都有自己的架构状态（寄存器组），但所有线程共享主执行资源（如 CPU cache）。当一个线程暂停（缓存未命中、分支错误预测、等待上一条指令的结果等）时，CPU 会自动执行另一个线程，这是在没有操作系统干预的情况下发生的。

在超线程 CPU 上执行旋转循环（spin loops）时，需要强制当前线程暂停以便其他线程可以访问芯片的资源。x86 架构支持 PAUSE 汇编语言指令。PAUSE 指令可确保避免内存顺序冲突，从而提高性能。在 x86 上，PAUSE 指令等效于 REP NOP 指令，这使得它与不支持超线程的早期的 IA-32 CPU 兼容。PAUSE 会导致有限延迟（在某些 CPU 上为 0）。在 Win32 API 中，x86 PAUSE 指令是通过调用 WinNT.h 中定义的 [`YieldProcessor`](https://docs.microsoft.com/en-us/windows/win32/api/winnt/nf-winnt-yieldprocessor) 宏发出的。

## 线程的执行时间

有时希望计算线程执行特定任务所需的时间。许多人所做的是编写类似于以下内容的代码，利用 [`GetTickCount64`](https://docs.microsoft.com/en-us/windows/win32/api/sysinfoapi/nf-sysinfoapi-gettickcount64) 函数：

```cpp
// Get the current time (start time).
ULONGLONG qwStartTime = GetTickCount64();

// Perform complex algorithm here.

// Subtract start time from current time to get duration.
ULONGLONG qwElapsedTime = GetTickCount64() - qwStartTime;
```

此代码做出一个简单的假设：它不会被中断。但是，在抢占式操作系统中永远不会知道线程何时会被调度。可以利用操作系统提供的 [`GetThreadTimes`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-getthreadtimes) 函数来准确地实现这个任务：

```cpp
__int64 FileTimeToQuadWord(PFILETIME pft) {
    return(Int64ShllMod32(pft->dwHighDateTime, 32) | pft->dwLowDateTime);
}

void PerformLongOperation() {
    FILETIME ftKernelTimeStart, ftKernelTimeEnd;
    FILETIME ftUserTimeStart, ftUserTimeEnd;
    FILETIME ftDummy;
    __int64 qwKernelTimeElapsed, qwUserTimeElapsed, qwTotalTimeElapsed;

    // Get starting times.
    GetThreadTimes(GetCurrentThread(), &ftDummy, &ftDummy, &ftKernelTimeStart, &ftUserTimeStart);

    // Perform complex algorithm here.

    // Get ending times.
    GetThreadTimes(GetCurrentThread(), &ftDummy, &ftDummy, &ftKernelTimeEnd, &ftUserTimeEnd);

    // Get the elapsed kernel and user times by converting the start and end times from FILETIMEs to quad words,
    // and then subtract the start times from the end times.
    qwKernelTimeElapsed = FileTimeToQuadWord(&ftKernelTimeEnd) - FileTimeToQuadWord(&ftKernelTimeStart);
    qwUserTimeElapsed = FileTimeToQuadWord(&ftUserTimeEnd) - FileTimeToQuadWord(&ftUserTimeStart);

    // Get total time duration by adding the kernel and user times.
    qwTotalTimeElapsed = qwKernelTimeElapsed + qwUserTimeElapsed;
    
    // The total elapsed time is in qwTotalTimeElapsed.
}
```

[`GetProcessTimes`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-getprocesstimes) 返回应用于指定进程中所有线程（甚至是已终止的线程）的时间。

对于高分辨率的分析，Windows 提供了以下函数：

```cpp
BOOL QueryPerformanceFrequency(LARGE_INTEGER* pliFrequency);
BOOL QueryPerformanceCounter(LARGE_INTEGER* pliCount);
```

## CONTEXT

[`CONTEXT`](https://docs.microsoft.com/en-us/windows/win32/api/winnt/ns-winnt-context) 结构允许系统记住线程的状态，以便线程可以在下次运行时从中断的位置继续。

Windows 允许查看线程内核对象的内部，并获取其当前的 CPU 寄存器组。为此，只需调用 [`GetThreadContext`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-getthreadcontext) 函数。

应该在调用 [`GetThreadContext`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-getthreadcontext) 之前调用 [`SuspendThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-suspendthread)。否则，可能线程可能会被调度，并且线程的上下文可能与返回的内容不同。线程实际上有两个上下文：用户模式和内核模式。[`GetThreadContext`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-getthreadcontext) 只能返回线程的用户模式上下文。如果调用 [`SuspendThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-suspendthread) 来停止某个线程，但该线程当前正以内核模式执行，那么即使 [`SuspendThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-suspendthread) 实际上尚未挂起该线程，其用户模式上下文也是稳定的。

Windows 允许更改 `CONTEXT` 结构中的成员，然后通过调用 [`SetThreadContext`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-setthreadcontext) 将新的寄存器值放回线程内核对象中。同样地，要更改上下文的线程应先挂起，否则结果将不可预测。

## 线程优先级

每个线程都分配有一个优先级编号，范围从 0（最低）到 31（最高）。系统会根据优先级从高到低地检查可调度线程，这意味着相对较低优先级的线程可能永远不会被调度，如系统中一直存在优先级为 31 的可调度线程，则优先级为 30 的可调度线程永远不会被调度，这种情况称为饥饿（Starvation）。在多处理器机器上发生饥饿的可能性会低得多，因为在这种机器上，优先级为 31 的可调度线程和优先级为 30 的可调度线程可能同时运行，系统会始终尝试使 CPU 保持繁忙状态，仅当没有可调度线程时，CPU 才会空闲。

优先级较高的线程始终会抢占优先级较低的线程，而不管较低优先级的线程正在执行什么。例如，如果优先级为 5 的线程正在运行，并且系统确定了优先级为 6 的线程已准备好运行，则系统会立即挂起优先级较低的线程（即使它处于其时间片的中间），并将 CPU 分配给优先级较高的线程（获得一个完整的时间片）。

当系统启动时，它会创建一个名为 *零页线程（Zero Page Thread）* 的特殊线程。此线程优先级为 0，并且是整个系统中唯一以优先级 0 运行的线程。在没有其他线程需要执行工作时，零页线程负责将系统中所有空闲 RAM 页清零。

## 优先级的抽象视图

Windows API 在系统的调度器上公开了一个抽象层，这个抽象层会根据正在运行的系统的版本来“解释”请求的参数。

在设计应用程序时，应考虑可能与应用程序一起运行的其他应用程序。然后，应该根据应用程序中线程的响应速度选择优先级类别。

Windows 支持六种优先级类别：空闲（Idle）、低于正常（Below Normal）、正常（Normal）、高于正常（Above Normal）、高（High）和实时（Real-time）。正常是最常见的优先级类别，99% 的应用程序都应该使用它：

| Priority Class | Description |
| :-- | :-- |
| Real-time | The threads in this process must respond immediately to events to execute time-critical tasks. Threads in this process also preempt operating system components. Use this priority class with extreme caution. |
| High | The threads in this process must respond immediately to events to execute time-critical tasks. The Task Manager runs at this class so that a user can kill runaway processes. |
| Above normal | The threads in this process run between the normal and high-priority classes. |
| Normal | The threads in this process have no special scheduling needs. |
| Below normal | The threads in this process run between the normal and idle priority classes. |
| Idle | The threads in this process run when the system is otherwise idle. This process is typically used by screen savers or background utility and statistics-gathering software. |

空闲优先级类别非常适合在系统几乎不执行任何操作时运行的应用程序。只有在绝对必要时，才应使用高优先级类别。如果可能，应避免使用实时优先级类别。实时优先级可能会干扰操作系统任务，因为大多数操作系统线程以较低的优先级执行。

> 注：进程不能运行在实时优先级类别，除非用户具有“提高计划优先级（Increase Scheduling Priority）”权限。默认情况下，任何指定为管理员或超级用户的用户都具有此权限。

选择优先级类别后，应该停止考虑应用程序与其他应用程序的关连，而只需专注于应用程序中的线程即可。

Windows 支持七个相对线程优先级：空闲（Idle）、最低（Lowest）、低于正常（Below Normal）、正常（Normal）、高于正常（Above Normal）、最高（Highest）和时间关键（Time-critical）。这些优先级相对于进程的优先级类别。同样，大多数线程使用正常的线程优先级：

| Relative Thread Priority | Description |
| :-- | :-- |
| Time-critical | Thread runs at 31 for the real-time priority class and at 15 for all other priority classes. |
| Highest | Thread runs two levels above normal. |
| Above normal | Thread runs one level above normal. |
| Normal | Thread runs normally for the process' priority class. |
| Below normal | Thread runs one level below normal. |
| Lowest | Thread runs two levels below normal. |
| Idle | Thread runs at 16 for the real-time priority class and at 1 for all other priority classes. |

进程优先级类别与相对线程优先级共同决定线程的优先级。应用开发者不应该使用具体数值来为线程赋予优先级，而应该通过设置进程优先级类别和相对线程优先级来让系统决定如何映射到优先级级别：

<table>
<thead>
  <tr>
    <th rowspan="2" style="text-align:center">Relative Thread Priority</th>
    <th colspan="6" style="text-align:center">Process Priority Class</th>
  </tr>
  <tr>
    <th style="text-align:center">Idle</th>
    <th style="text-align:center">Below Normal</th>
    <th style="text-align:center">Normal</th>
    <th style="text-align:center">Above Normal</th>
    <th style="text-align:center">High</th>
    <th style="text-align:center">Real-Time</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td>Time-critical</td>
    <td style="text-align:center">15</td>
    <td style="text-align:center">15</td>
    <td style="text-align:center">15</td>
    <td style="text-align:center">15</td>
    <td style="text-align:center">15</td>
    <td style="text-align:center">31</td>
  </tr>
  <tr>
    <td>Highest</td>
    <td style="text-align:center">6</td>
    <td style="text-align:center">8</td>
    <td style="text-align:center">10</td>
    <td style="text-align:center">12</td>
    <td style="text-align:center">15</td>
    <td style="text-align:center">26</td>
  </tr>
  <tr>
    <td>Above normal</td>
    <td style="text-align:center">5</td>
    <td style="text-align:center">7</td>
    <td style="text-align:center">9</td>
    <td style="text-align:center">11</td>
    <td style="text-align:center">14</td>
    <td style="text-align:center">25</td>
  </tr>
  <tr>
    <td>Normal</td>
    <td style="text-align:center">4</td>
    <td style="text-align:center">6</td>
    <td style="text-align:center">8</td>
    <td style="text-align:center">10</td>
    <td style="text-align:center">13</td>
    <td style="text-align:center">24</td>
  </tr>
  <tr>
    <td>Below normal</td>
    <td style="text-align:center">3</td>
    <td style="text-align:center">5</td>
    <td style="text-align:center">7</td>
    <td style="text-align:center">9</td>
    <td style="text-align:center">12</td>
    <td style="text-align:center">23</td>
  </tr>
  <tr>
    <td>Lowest</td>
    <td style="text-align:center">2</td>
    <td style="text-align:center">4</td>
    <td style="text-align:center">6</td>
    <td style="text-align:center">8</td>
    <td style="text-align:center">11</td>
    <td style="text-align:center">22</td>
  </tr>
  <tr>
    <td>Idle</td>
    <td style="text-align:center">1</td>
    <td style="text-align:center">1</td>
    <td style="text-align:center">1</td>
    <td style="text-align:center">1</td>
    <td style="text-align:center">1</td>
    <td style="text-align:center">16</td>
  </tr>
</tbody>
</table>

> 注：上表的映射值在不同版本的 Windows 上可能存在差异。该表没有展示线程映射到优先级 0 的任何方式，这是因为 0 优先级是为零页线程保留的，并且系统不允许任何其他线程的优先级为 0。此外，无法获得以下优先级别：17、18、19、20、21、27、28、29 或 30，这些是为在内核模式下运行的设备驱动程序准备的。

## 编程优先级

当调用 [`CreateProcess`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessw) 时，可以在 `fdwCreate` 参数中传递所需的优先级类别：

| Priority Class | Symbolic Identifiers |
| :-- | :-- |
| Real-time | `REALTIME_PRIORITY_CLASS` |
| High | `HIGH_PRIORITY_CLASS` |
| Above normal | `ABOVE_NORMAL_PRIORITY_CLASS` |
| Normal | `NORMAL_PRIORITY_CLASS` |
| Below normal | `BELOW_NORMAL_PRIORITY_CLASS` |
| Idle | `IDLE_PRIORITY_CLASS` |

通过调用 [`SetPriorityClass`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-setpriorityclass) 可以更改指定进程的优先级类别。通过调用 [`GetPriorityClass`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-getpriorityclass) 可以检索指定进程的优先级类别。

使用命令行 Shell 调用程序时，程序的起始优先级为正常。使用 `START` 命令调用程序时可以指定一个起始优先级，如 `START /LOW CALC.EXE`。

首次创建线程时，其相对线程优先级始终设置为正常。使用以下函数可以设置和获取线程的相对优先级：

```cpp
BOOL SetThreadPriority(HANDLE hThread, int nPriority);
int GetThreadPriority(HANDLE hThread);
```

### 动态提升线程优先级

系统通过将线程的相对优先级与线程的进程的优先级类别相结合来确定线程的优先级，这称为线程的基本优先级（Base Priority Level）。有时，系统会提升线程的优先级，这通常是为了响应某些 I/O 事件（如窗口消息或磁盘读取）。

例如，在高优先级类别进程中有一个正常线程优先级的线程，其基本优先级为 13。如果用户按下某个键，则系统会将 `WM_KEYDOWN` 消息放到线程的队列中。由于消息已出现在线程的队列中，因此线程是可调度的。此外，键盘设备驱动程序可以告诉系统暂时提高线程的级别，因此，线程可能会提升 2，并且当前优先级为 15。

线程会被调度一个时间片（优先级为 15）。一旦该时间片到期，系统就会将线程的优先级从 15 降至 14，以进行下一个时间片。线程的第三个时间片会以优先级 13 执行。线程的其他时间片都将在优先级 13（线程的基本优先级）下执行。

> 注：线程的当前优先级永远不会低于线程的基本优先级。

系统仅提升基本优先级介于 1 和 15 之间的线程。实际上，这就是为什么此范围被称为动态优先级范围的原因。此外，系统永远不会将线程提升到实时范围（高于 15）。由于实时范围内的线程大多数执行操作系统功能，因此对强制实施上限可防止应用程序干扰操作系统。此外，系统从不动态提升实时范围（16 到 31）中的线程。

系统的动态提升可能会对线程的性能产生影响，因此 Microsoft 添加了以下两个函数，以允许禁用系统对线程优先级的动态提升：

```cpp
BOOL SetProcessPriorityBoost(HANDLE hProcess, BOOL bDisablePriorityBoost);
BOOL SetThreadPriorityBoost(HANDLE hThread, BOOL bDisablePriorityBoost);
```

[`SetProcessPriorityBoost`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-setprocesspriorityboost) 告诉系统为进程内的所有线程启用或禁用优先级提升；[`SetThreadPriorityBoost`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-setthreadpriorityboost) 允许为单个线程启用或禁用优先级提升。这两个函数具有对应项，允许您确定是启用还是禁用优先级提升：

```cpp
BOOL GetProcessPriorityBoost(HANDLE hProcess, PBOOL pbDisablePriorityBoost);
BOOL GetThreadPriorityBoost(HANDLE hThread, PBOOL pbDisablePriorityBoost);
```

另一种情况也会导致系统动态提升线程的优先级。比如，一个优先级为 4 的线程已准备好运行，但无法运行，因为优先级为 8 的线程是可持续调度的。在这种情况下，优先级 4 线程渴望 CPU 时间。当系统检测到某个线程在大约三到四秒内 CPU 时间不足时，它会动态地将饥饿线程的优先级提升为 15，并允许该线程运行两倍的时间量。当两倍时间量到期时，该线程会马上回到其基本优先级。

### 为前台进程调整调度器

当用户使用进程的窗口时，该进程称为前台进程（Foreground Process），所有其他进程都是后台进程（Background Process）。用户肯定更希望他或她使用的进程比后台进程的行为响应更快。为了提高前台进程的响应能力，Windows 调整了前台进程中线程的调度算法。系统为前台进程线程提供的时间量比它们通常接收的时间量大。仅当前台进程属于正常优先级类别时，才会执行此调整。如果它属于任何其他优先级，则不执行任何调整。

### 安排 I/O 请求优先级

设置线程优先级会影响线程如何调度 CPU 资源。然而，线程还会执行 I/O 请求以从磁盘文件中读取和写入数据。如果低优先级线程获得 CPU 时间，它可以在很短的时间内简单地将数百或数千个 I/O 请求入队。由于 I/O 请求通常需要时间来处理，因此低优先级线程可能会通过挂起高优先级线程来显著影响系统的响应能力，从而阻止它们完成工作。因此，您可以看到计算机在执行长时间运行的低优先级服务（如磁盘碎片整理程序、病毒扫描程序、内容索引器等）时响应速度变慢。

从 Windows Vista 开始，线程可以在发出 I/O 请求时指定优先级。可以通过调用 [`SetThreadPriority`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-setthreadpriority) 并传递 `THREAD_MODE_BACKGROUND_BEGIN` 来告诉 Windows，该线程应该发出低优先级的 I/O 请求。请注意，这也会降低线程的 CPU 调度优先级。可以通过调用 [`SetThreadPriority`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-setthreadpriority) 并传递 `THREAD_ MODE_BACKGROUND_END` 将线程返回到发出正常优先级 I/O 请求（以及正常的 CPU 调度优先级）。

如果希望进程中的所有线程发出低优先级 I/O 请求并具有低 CPU 调度，则可以调用 [`SetPriorityClass`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-setpriorityclass) 并传递 `PROCESS_MODE_BACKGROUND_BEGIN`。

在更精细的粒度下，正常优先级线程可以对特定文件执行后台优先级 I/O，如以下代码片段所示：

```cpp
FILE_IO_PRIORITY_HINT_INFO phi;
phi.PriorityHint = IoPriorityHintLow;
SetFileInformationByHandle(hFile, FileIoPriorityHintInfo, &phi, sizeof(PriorityHint));
```

[`SetFileInformationByHandle`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-setfileinformationbyhandle) 设置的优先级会覆盖 [`SetPriorityClass`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-setpriorityclass) 或 [`SetThreadPriority`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-setthreadpriority) 设置的优先级。

## 亲和性

默认情况下，Windows Vista 在将线程分配给处理器时使用 *软亲和性（Soft Affinity）* 。这意味着，如果所有其他因素相同，它将尝试在线程上次运行的那个处理器上运行线程。让线程保留在单个处理器上有助于重用仍在处理器内存缓存中的数据。

有一种称为 [NUMA（Non-Uniform Memory Access）](https://en.wikipedia.org/wiki/Non-uniform_memory_access) 的计算机架构，其中机器由多个板组成。每个板都有独自的 CPU 和独自的内存库。当 CPU 访问其自身板上的内存时，NUMA 系统的性能最佳。如果 CPU 需要接触另一块板上的内存，则会对性能造成巨大影响。在这种场景下，希望同一进程的线程都能在同一板上的 CPU 上运行。为了适应这种计算机架构，Windows Vista 允许设置进程和线程亲和性，即可以控制哪些 CPU 可以运行哪些线程，这称为 *硬亲和性（Hard Affinity）*。

系统在启动时会确定计算机中有多少 CPU 可用，应用程序可以通过调用 [`GetSystemInfo`](https://docs.microsoft.com/en-us/windows/win32/api/sysinfoapi/nf-sysinfoapi-getsysteminfo) 来查询计算机上的 CPU 数量。默认情况下，可以将任何线程调度到这些 CPU 中的任何一个。要将单个进程中的线程限制在可用 CPU 的子集上运行，可以调用 [`SetProcessAffinityMask`](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-setprocessaffinitymask)。

> 注：子进程会继承进程亲和性。此外，还可以使用作业内核对象来设置一组进程的亲和性。

[`GetProcessAffinityMask`](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-getprocessaffinitymask) 会返回进程的亲和性掩码。

可以通过调用 [`SetThreadAffinityMask`](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-setthreadaffinitymask) 为单个线程设置亲和性掩码。

有时，将线程强制到特定的 CPU 并不是一个好主意。要为线程设置理想的 CPU，请调用 [`SetThreadIdealProcessor`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-setthreadidealprocessor)。

还可以在可执行文件的标头中设置处理器亲和性，可以使用类似于如下代码来利用 ImageHlp.h 中声明的函数：

```cpp
// Load the EXE into memory.
PLOADED_IMAGE pLoadedImage = ImageLoad(szExeName, NULL);

// Get the current load configuration information for the EXE.
IMAGE_LOAD_CONFIG_DIRECTORY ilcd;
GetImageConfigInformation(pLoadedImage, &ilcd);

// Change the processor affinity mask.
ilcd.ProcessAffinityMask = 0x00000003; // I desire CPUs 0 and 1

// Save the new load configuration information.
SetImageConfigInformation(pLoadedImage, &ilcd);

// Unload the EXE from memory
ImageUnload(pLoadedImage);
```

当 Windows Vista 在 x86 计算机上启动时，可以限制系统将使用的 CPU 数量。在引导周期中，系统会检查 BCD（Boot Configuration Data），该数据存储用于替换旧的 boot.ini 文本文件，并提供计算机硬件和固件的更高级别的抽象。

BCD 的编程化配置是通过 WMI（Windows Management Instrumentation）完成的，也可以通过图形用户界面访问一些最常见的参数。要限制 Windows 使用的 CPU 数量，您需要在 **控制面板（Control Panel） > 管理工具（Administrative Tools） > 系统配置（System Configuration） > 引导（Boot） > 高级选项（Advanced） > 处理器个数（Number Of Processors）** 中填写所需的数目。
