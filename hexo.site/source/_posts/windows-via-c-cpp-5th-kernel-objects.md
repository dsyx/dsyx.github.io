---
title: 'Windows via C/C++, 5th Edition - Kernel Objects'
date: 2022-02-07 10:40:27
category: 'My Notes'
tags: ['Windows via C/C++', 'Windows']
---

# 什么是内核对象？

Windows 内核对象（Kernel Object）是一个包含资源维护信息的结构化内存块。系统创建并维护若干类型的[内核对象](https://docs.microsoft.com/en-us/windows/win32/sysinfo/kernel-objects)，如访问令牌对象（access token object）。因为内核对象的数据结构只能被内核访问，所以用户程序只能通过调用 Windows 函数来创建和维护这些对象。当调用了会创建内核对象的函数时，函数会返回一个句柄（handle）以标识所创建的对象。

句柄是进程相关的，因此直接将句柄值传递给其它进程是没有意义的。但是可以通过“[跨进程边界共享内核对象（Sharing Kernel Objects Across Process Boundaries）](#跨进程边界共享内核对象)”来实现进程间的内核对象共享。

## 使用计数

内核对象是属于内核的而不是进程的，这意味着内核对象的生命周期不一定与进程的生命周期一致。内核通过每个对象中包含的一个使用计数（usage count）来获知有多少进程正在使用它。

使用计数是内核对象最通用的数据成员之一。当对象首次被创建时，其使用计数会被设置为 1；然后，当另一个进程访问它时，使用计数就递增 1。当一个进程终止时，内核会自动递减该进程仍然打开的所有内核对象的使用计数。如果内核对象的使用计数为 0，那么内核就会销毁该对象。

## 安全

可以使用安全描述符（security descriptor）来保护内核对象。安全描述符描述了对象的属主和对象的访问权限。安全描述符通常用于编写服务器应用。

大多数用于创建内核对象的函数有一个指向 `SECURITY_ATTRIBUTES` 结构的指针参数，如 `CreateFileMapping` 函数：

```cpp
HANDLE CreateFileMapping(
    HANDLE hFile,
    PSECURITY_ATTRIBUTES psa,
    DWORD flProtect,
    DWORD dwMaximumSizeHigh,
    DWORD dwMaximumSizeLow,
    PCTSTR pszName);
```

大多数应用会简单地向该参数传递 `NULL` 以使用默认的安全。

# 进程的内核对象句柄表

当一个进程被初始化时，系统会为其分配一个句柄表。这个表只用于内核对象，而不会用于用户对象或 GDI 对象。没有文档介绍如何处理和管理这个表，但其大体上如下：

| Index | Pointer to Kernel Object Memory Block | Access Mask (DWORD of Flag Bits) | Flags |
| :-- | :-- | :-- | :-- |
| 1 | 0x???????? | 0x???????? | 0x???????? |
| 2 | 0x???????? | 0x???????? | 0x???????? |
| ... | ... | ... | ... |

## 创建内核对象

当进程首次被初始化时，它的句柄表是空的。然后，当进程中的线程调用创建内核对象的函数（如 `CreateFileMapping`）时，内核就会为该对象分配一个内存块并初始化它。接着，内核会对进程的句柄表进行扫描以找出一个空项，然后记录下对象的相关信息。

用于创建内核对象的函数都会返回与进程相关的句柄，这些句柄可以被同一进程中运行的任一或所有线程使用。

当调用一个接受内核对象句柄作为参数的函数时，应当传递一个由 **Create\*** 函数返回的值。如果传递了一个无效的句柄，那么该函数会返回失败，并且 `GetLastError` 返回 6（`ERROR_INVALID_HANDLE`）。如果调用函数以创建内核对象但调用失败了，那么返回的句柄值通常是 0（`NULL`），这是由于系统内存不足或者遇到了安全问题；少数函数在失败时返回的句柄值是 -1（`INVALID_HANDLE_VALUE`）。因此，当查看会创建内核对象的函数返回值时，必须格外小心。

## 关闭内核对象

可以通过调用 [`CloseHandle`](https://docs.microsoft.com/en-us/windows/win32/api/handleapi/nf-handleapi-closehandle) 来关闭一个已打开的内核对象句柄。

该函数首先会检查调用进程的句柄表，以确保句柄值的合法性。如果该句柄是有效的，那么系统会递减该内核对象的使用计数（若使用计数为 0，则从内存中销毁该内核对象）；如果句柄是无效的，那么 `CloseHandle` 会返回 `FALSE` 并且 `GetLastError` 会返回 `ERROR_INVALID_HANDLE`（若进程处于调试状态下，则系统会抛出异常 0xC0000008 “An invalid handle was specified”）。

`CloseHandle` 返回前会清除进程句柄表中相应的项，因此传递给 `CloseHandle` 的句柄将在进程中失效，后续上下文中不应该再使用它。

# 跨进程边界共享内核对象

有三种允许进程共享内核对象的机制：
* 对象句柄继承（Object Handle Inheritance）
* 命名对象（Naming Object）
* 复制对象句柄（Duplicating Object Handle）

## 对象句柄继承

仅当进程间存在父子关系时才能使用对象句柄继承。在这种场景下，父进程有一个或多个可用的内核对象，父进程决定产生一个子进程并赋予子进程访问父进程的内核对象。为此，父进程必须执行如下步骤。

首先，当父进程创建内核对象时，父进程必须向系统指示它希望该对象的句柄是可继承的。为了创建可继承的句柄，父进程必须分配并初始化 `SECURITY_ATTRIBUTES` 结构，并将该结构的地址传递给特定的 **Create** 函数。下面的代码创建一个互斥对象（mutex object），并得到一个可继承的句柄：

```cpp
SECURITY_ATTRIBUTES sa;
sa.nLength = sizeof(sa);
sa.lpSecurityDescriptor = NULL;
sa.bInheritHandle = TRUE; // Make the returned handle inheritable.

HANDLE hMutex = CreateMutex(&sa, FALSE, NULL);
```

下一步，父进程调用 `CreateProcess` 函数来产生子进程，其中传递 `TRUE` 给 `bInheritHandles` 参数：

```cpp
BOOL CreateProcess(
    PCTSTR pszApplicationName,
    PTSTR pszCommandLine,
    PSECURITY_ATTRIBUTES psaProcess,
    PSECURITY_ATTRIBUTES psaThread,
    BOOL bInheritHandles,
    DWORD dwCreationFlags,
    PVOID pvEnvironment,
    PCTSTR pszCurrentDirectory,
    LPSTARTUPINFO pStartupInfo,
    PPROCESS_INFORMATION pProcessInformation);
```

由于 `bInheritHandles` 参数为 `TRUE`，所以系统在创建子进程时会为其初始化一个空的进程句柄表，并且遍历父进程的进程句柄表以找出那些可继承的句柄项，然后将这些句柄项复制到子进程的进程句柄表的相同位置（这意味着这些句柄值在父子进程中是相同的）。此外，系统还会递增这些内核对象的使用计数。之后，父子进程对这些句柄的维护是独立的，即父进程关闭句柄不会影响到子进程。

> 注：对象句柄继承仅在生成子进程时应用。若父进程要使用可继承的句柄创建任何新的内核对象，则已在运行的子进程将不会继承这些新句柄。

对象句柄继承有一个奇怪的特征：子进程不知道它继承了什么句柄。这通常需要一些方法（如命令行参数、进程间通信、环境变量等）来告知子进程它所继承的句柄的值。

### 更改句柄的标志

有时，父进程可能需要产生多个子进程，但只希望其中几个子进程继承对象句柄。这种情况下可以使用 `SetHandleInformation` 函数来改变内核对象句柄的继承标志：

```cpp
BOOL SetHandleInformation(
    HANDLE hObject,
    DWORD dwMask,
    DWORD dwFlags);
```

`hObject` 标识一个有效的句柄，`dwMask` 告知函数哪些标志需要更改，`dwFlags` 指示需要将标志设置为什么。

当前，每个句柄都有两个关联的标志，可以通过或运算来同时操作两个标志：

```cpp
#define HANDLE_FLAG_INHERIT             0x00000001
#define HANDLE_FLAG_PROTECT_FROM_CLOSE  0x00000002
```

若要打开继承标志：

```cpp
SetHandleInformation(hObj, HANDLE_FLAG_INHERIT, HANDLE_FLAG_INHERIT);
```

若要关闭继承标志：

```cpp
SetHandleInformation(hObj, HANDLE_FLAG_INHERIT, 0);
```

`HANDLE_FLAG_PROTECT_FROM_CLOSE` 标志告知系统此句柄不应允许被关闭。关闭打开了该标志的句柄会失败或发生异常（调试状态下）。

## 命名对象

大多数内核对象可以被命名。如下函数可以创建命名的互斥量：

```cpp
HANDLE CreateMutex(
    PSECURITY_ATTRIBUTES psa,
    BOOL bInitialOwner,
    PCTSTR pszName);
```

大多数的内核对象创建函数具有一个通用的最后参数 `pszName`。如果此参数为 `NULL`，那么系统将创建匿名的内核对象，否则创建指定名字的对象。命名的内核对象可以通过名字来共享。`pszName` 接受一个零结尾（zero-terminated）的字符串，其最大长度为 `MAX_PATH`。

> 注：Microsoft 没有提供内核对象的命名指南，所有内核对象共享单个命名空间，因此需要注意赋予给内核对象的名字是否已存在。

通过命名来共享对象的方式如下，假设进程 A 启动并调用如下函数：

```cpp
HANDLE hMutexProcessA = CreateMutex(NULL, FALSE, TEXT("JeffMutex"));
```

随后，假设进程 B 启动并调用如下函数：

```cpp
HANDLE hMutexProcessB = CreateMutex(NULL, FALSE, TEXT("JeffMutex"));
```

当进程 B 调用 `CreateMutex` 时，系统会先检查名字为“JeffMutex”的内核对象是否已存在，然后会检查对象的类型。由于 B 尝试创建互斥量，并且已存在的名字为“JeffMutex”的对象也是互斥量，因此系统会进行安全检查以查看调用方是否具有对该对象的完全访问权限。若有，则系统将在 B 的句柄表中找出一个空条目，并将该条目初始化为指向现有的内核对象。若对象类型不匹配，或者调用方被拒绝访问，则 `CreateMutex` 将失败。

当 B 调用的 `CreateMutex` 成功返回时，实际上并没有创建一个互斥量，而是简单地引用了系统中已存在的同名互斥量。因此“JeffMutex”的使用计数将递增。此外，与对象句柄继承不一样，B 中的句柄值不必与 A 中的相同。

可以使用 **Open\*** 函数来替代 **Create\*** 函数以引用已存在的内核对象，如 `OpenMutex`。与对应的 **Create\*** 函数一样，**Open\*** 函数也有一个通用的最后参数 `pszName`。不同的是，如果指定名字的内核对象不存在时，**Create\*** 函数会创建它，而 **Open\*** 函数只会简单地返回失败。

> 贴士：可以通过创建一个 GUID 并将其字符串表示作为对象名字来确保对象的唯一性。

> 贴士：可以利用对象名字的唯一性来防止应用程序运行多个实例，如以下代码：
> ```cpp
> int WINAPI _tWinMain(HINSTANCE hInstExe, HINSTANCE, PTSTR pszCmdLine, int nCmdShow) {
>     HANDLE h = CreateMutex(NULL, FALSE, TEXT("{FA531CC1-0497-11d3-A180-00105A276C3E}"));
>     if (GetLastError() == ERROR_ALREADY_EXISTS) {
>         // There is already an instance of this application running.
>         // Close the object and immediately return.
>         CloseHandle(h);
>         return(0);
>     }
> 
>     // This is the first instance of this application running.
>     ...
>     // Before exiting, close the object.
>     CloseHandle(h);
>     return(0);
> }
> ```

### 终端服务命名空间

终端服务（Terminal Services）会使上述假设的情景发生一些变化。运行终端服务的机器具有多个为内核对象准备的命名空间：一个全局命名空间（通常被服务所使用），用于所有客户端会话都可以访问的内核对象；每个客户端会话都拥有自己的命名空间。

如果希望获知进程正在运行在哪个终端服务会话上，那么可以使用 `ProcessIdToSessionId` 函数。示例代码如下：

```cpp
DWORD processID = GetCurrentProcessId();
DWORD sessionID;
if (ProcessIdToSessionId(processID, &sessionID)) {
    tprintf(TEXT("Process '%u' runs in Terminal Services session '%u'"), processID, sessionID);
} else {
    // ProcessIdToSessionId might fail if you don't have enough rights
    // to access the process for which you pass the ID as parameter.
    // Notice that it is not the case here because we're using our own process ID.
    tprintf(TEXT("Unable to get Terminal Services session ID for process '%u'"), processID);
}
```

服务的命名内核对象始终位于全局命名空间中。默认情况下，在终端服务中，应用程序的命名内核对象位于会话的命名空间中。但是，可以通过在名字前面加上“Global\”前缀来强制命名对象进入全局命名空间，如下所示：

```cpp
HANDLE h = CreateEvent(NULL, FALSE, FALSE, TEXT("Global\\MyName"));
```

也可以通过在名字前面加上“Local\”前缀来显式声明希望内核对象位于当前会话的命名空间中，如下所示：

```cpp
HANDLE h = CreateEvent(NULL, FALSE, FALSE, TEXT("Local\\MyName"));
```

Microsoft 将 *Global* 和 *Local* 视为保留关键字，除非强制使用特定的命名空间，否则不应在对象名称中使用这些关键字。Microsoft 还认为 *Session* 是一个保留关键字。

> 注：所有这些保留关键字都区分大小写。

## 复制对象句柄

使用 `DuplicateHandle` 函数可以复制对象句柄：

```cpp
BOOL DuplicateHandle(
    HANDLE hSourceProcessHandle,
    HANDLE hSourceHandle,
    HANDLE hTargetProcessHandle,
    PHANDLE phTargetHandle,
    DWORD dwDesiredAccess,
    BOOL bInheritHandle,
    DWORD dwOptions);
```

此函数获取一个进程的句柄表中的条目，并将该条目复制到另一个进程的句柄表中。

`hSourceProcessHandle` 和 `hTargetProcessHandle` 必须是进程内核对象句柄，并且必须与调用 `DuplicateHandle` 函数的进程相关联。

`hSourceHandle` 可以是任何类型的内核对象句柄，并且其必须与 `hSourceProcessHandle` 所标识的进程相关联。

`phTargetHandle` 是一个 `HANDLE` 变量的地址，该变量用于接收复制后与 `hTargetProcessHandle` 所标识的进程相关联的句柄值。

最后的三个参数用于指示目标进程的内核对象句柄项使用的访问掩码和继承标志。`dwOptions` 可以是 0 或 `DUPLICATE_SAME_ACCESS` 与 `DUPLICATE_CLOSE_SOURCE` 的任意组合。指定 `DUPLICATE_SAME_ACCESS` 会使目标句柄的访问掩码与源句柄的一致，并且使 `DuplicateHandle` 忽略 `dwDesiredAccess` 参数。指定 `DUPLICATE_CLOSE_SOURCE` 会起到在源进程中关闭句柄的效果，源进程可以轻松地将内核对象移交给目标进程，内核对象的使用计数不受影响。

复制对象句柄也存在与对象句柄继承一样的奇怪特征：目标进程不会收到任何关于有新内核对象可用的通知。由于复制对象句柄发生在目标进程运行之后，因此这通常需要使用进程间通信来告知目标进程新可用内核对象的句柄值。
