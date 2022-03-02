---
title: 'Windows via C/C++, 5th Edition - Synchronous and Asynchronous Device I/O'
date: 2022-02-28 10:28:20
category: 'My Notes'
tags: ['Windows via C/C++', 'Windows']
---

在 Microsoft Windows 应用程序中，线程是用于分割工作的最佳工具。每个线程被分配给一个处理器，这允许多处理器计算机同时执行多个操作，从而提高吞吐量。当线程发出同步设备 I/O 请求时，该线程将暂时挂起，直到设备完成 I/O 请求。这种挂起会降低线程的性能，因为线程无法执行有用的工作（如启动另一个客户端的处理请求）。简而言之，您希望始终保持线程执行有用的工作，并避免它们被阻塞。

为了使线程保持忙碌，您需要让线程就它们将执行的操作相互通信。微软花了数年时间在这一领域进行研究和测试，并开发了一种经过微调的机制来创建这种通信。这种机制称为 *IOCP（I/O Completion Port）* ，其可以帮助您创建高性能、可伸缩的应用程序。使用 IOCP，您可以通过向设备读取和写入数据但不等待设备的响应来使应用程序的线程达到惊人的吞吐量。

## 打开与关闭设备

Windows 的优势之一是它支持数量庞大的设备。本文将设备定义为允许通信的任何东西。下表列出了一些设备及其最常见的用途：

| Device | Most Common Use |
| :-- | :-- |
| File | Persistent storage of arbitrary data |
| Directory | Attribute and file compression settings |
| Logical disk drive | Drive formatting |
| Physical disk drive | Partition table access |
| Serial port | Data transmission over a phone line |
| Parallel port | Data transmission to a printer |
| Mailslot | One-to-many transmission of data, usually over a network to a machine running Windows |
| Named pipe | One-to-one transmission of data, usually over a network to a machine running Windows |
| Anonymous pipe | One-to-one transmission of data on a single machine (never over the network) |
| Socket | Datagram or stream transmission of data, usually over a network to any machine supporting sockets (The machine need not be running Windows.) |
| Console | A text window screen buffer |

线程可以与这些设备进行通信，而无需等待设备响应。Windows 试图尽可能地向软件开发人员隐藏设备差异。也就是说，打开设备后，无论您使用什么设备进行通信，允许您向设备读取和写入数据的 Windows 函数都是相同的。尽管有少数函数可以无视设备差异地用于读取和写入数据，但设备之间肯定是存在差异的。要执行任何类型的 I/O，您必须首先打开所需的设备并获取其句柄。获取设备句柄的方式取决于特定设备。下表列出了各种设备以及打开它们时应调用的函数：

| Device | Function Used to Open the Device |
| :-- | :-- |
| File | [`CreateFile`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilew) (`pszName` is pathname or UNC pathname). |
| Directory | [`CreateFile`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilew) (`pszName` is directory name or UNC directory name). Windows allows you to open a directory if you specify the `FILE_FLAG_BACKUP_SEMANTICS` flag in the call to [`CreateFile`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilew). Opening the directory allows you to change the directory's attributes (to normal, hidden, and so on) and its time stamp. |
| Logical disk drive | [`CreateFile`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilew) (`pszName` is `"\\.\x:"`). Windows allows you to open a logical drive if you specify a string in the form of `"\\.\x:"` where `x` is a drive letter. For example, to open drive A, you specify `"\\.\A:"`. Opening a drive allows you to format the drive or determine the media size of the drive. |
| Physical disk drive | [`CreateFile`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilew) (`pszName` is `"\\.\PHYSICALDRIVEx"`). Windows allows you to open a physical drive if you specify a string in the form of `"\\.\PHYSICALDRIVEx"` where `x` is a physical drive number. For example, to read or write to physical sectors on the user's first physical hard disk, you specify `"\\.\PHYSICALDRIVE0"`. Opening a physical drive allows you to access the hard drive's partition tables directly. Opening the physical drive is potentially dangerous; an incorrect write to the drive could make the disk's contents inaccessible by the operating system's file system. |
| Serial port | [`CreateFile`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilew) (`pszName` is `"COMx"`).
| Parallel port | [`CreateFile`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilew) (`pszName` is `"LPTx"`). |
| Mailslot server | [`CreateMailslot`](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-createmailslotw) (`pszName` is `"\\.\mailslot\mailslotname"`). |
| Mailslot client | [`CreateFile`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilew) (`pszName` is `"\\servername\mailslot\mailslotname"`). |
| Named pipe server | [`CreateNamedPipe`](https://docs.microsoft.com/en-us/windows/win32/api/namedpipeapi/nf-namedpipeapi-createnamedpipew) (`pszName` is `"\\.\pipe\pipename"`). |
| Named pipe client | [`CreateFile`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilew) (`pszName` is `"\\servername\pipe\pipename"`). |
| Anonymous pipe | [`CreatePipe`](https://docs.microsoft.com/en-us/windows/win32/api/namedpipeapi/nf-namedpipeapi-createpipe) client and server. |
| Socket | [`socket`](https://docs.microsoft.com/en-us/windows/win32/api/winsock2/nf-winsock2-socket), [`accept`](https://docs.microsoft.com/en-us/windows/win32/api/winsock2/nf-winsock2-accept), or [`AcceptEx`](https://docs.microsoft.com/en-us/windows/win32/api/winsock/nf-winsock-acceptex). |
| Console | [`CreateConsoleScreenBuffer`](https://docs.microsoft.com/en-us/windows/console/createconsolescreenbuffer) or [`GetStdHandle`](https://docs.microsoft.com/en-us/windows/console/getstdhandle). |

这些函数都返回一个标识设备的句柄。可以将返回的句柄传递给各种函数以与设备进行通信。

使用完设备后，必须将其关闭。对于大多数设备，您可以通过调用常见的 [`CloseHandle`](https://docs.microsoft.com/en-us/windows/win32/api/handleapi/nf-handleapi-closehandle) 函数来执行此操作。然而，如果设备是 socket，则必须改为调用 [`closesocket`](https://docs.microsoft.com/en-us/windows/win32/api/winsock2/nf-winsock2-closesocket)。

## 使用文件设备

### 获取文件大小

处理文件时，通常需要获取文件的大小。最简单的方法是调用 [`GetFileSizeEx`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-getfilesizeex)。另一个用于获取文件大小的非常有用的函数是 [`GetCompressedFileSize`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-getcompressedfilesizew)。

[`GetCompressedFileSize`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-getcompressedfilesizew) 返回文件的物理大小，而 [`GetFileSizeEx`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-getfilesizeex) 返回文件的逻辑大小。

### 定位文件指针

调用 [`CreateFile`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilew) 会导致系统创建一个文件内核对象来管理对该文件的操作。文件内核对象的内部是一个文件指针。文件指针指示下一次执行同步读取或写入时的偏移量。最初，文件指针设置为 0，因此如果在调用 [`CreateFile`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilew) 后立即调用 [`ReadFile`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-readfile)，您将从偏移量 0 开始读取文件。如以下代码：

```cpp
BYTE pb[10];
DWORD dwNumBytes;
HANDLE hFile = CreateFile(TEXT("MyFile.dat"), ...); // Pointer set to 0
ReadFile(hFile, pb, 10, &dwNumBytes, NULL);         // Reads bytes 0 - 9
ReadFile(hFile, pb, 10, &dwNumBytes, NULL);         // Reads bytes 10 - 19
```

由于每个文件内核对象都有自己的文件指针，因此打开同一文件两次会得到略有不同的结果：

```cpp
BYTE pb[10];
DWORD dwNumBytes;
HANDLE hFile1 = CreateFile(TEXT("MyFile.dat"), ...); // Pointer set to 0
HANDLE hFile2 = CreateFile(TEXT("MyFile.dat"), ...); // Pointer set to 0
ReadFile(hFile1, pb, 10, &dwNumBytes, NULL);         // Reads bytes 0 - 9
ReadFile(hFile2, pb, 10, &dwNumBytes, NULL);         // Reads bytes 0 - 9
```

如果需要随机访问文件，那么您需要更改与文件内核对象关联的文件指针。可以通过调用 [`SetFilePointerEx`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-setfilepointerex) 来执行此操作。以下是关于 [`SetFilePointerEx`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-setfilepointerex) 的一些细节：

* 将文件的指针设置在文件当前大小之外是合法的。这样做实际上不会增加磁盘上文件的大小，除非您在此位置进行写入或调用 [`SetEndOfFile`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-setendoffile)。
* 将 [`SetFilePointerEx`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-setfilepointerex) 与使用 `FILE_FLAG_NO_BUFFERING` 打开的文件一起使用时，文件指针只能定位在扇区对齐的（Sector-aligned）边界上。
* Windows 不提供 `GetFilePointerEx` 函数，但您可以使用 [`SetFilePointerEx`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-setfilepointerex) 将指针移动 `0` 个字节以实现同样的效果。

### 设置文件的末尾

通常，系统负责在文件关闭时设置文件的未尾。但是，有时您可能希望强制文件变小或变大。这种情况下，您可以调用 [`SetEndOfFile`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-setendoffile)。

此函数将文件的大小截断或扩展为文件对象的文件指针指示的大小。例如，要强制文件的长度为 1024 字节，则可以使用如下方式：

```cpp
HANDLE hFile = CreateFile(...);
LARGE_INTEGER liDistanceToMove;
liDistanceToMove.QuadPart = 1024;
SetFilePointerEx(hFile, liDistanceToMove, NULL, FILE_BEGIN);
SetEndOfFile(hFile);
CloseHandle(hFile);
```

## 执行同步设备 I/O

读取和写入设备的最简单且最常用的函数是 [`ReadFile`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-readfile) 和 [`WriteFile`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-writefile)。

### 将数据刷新到设备

在调用 [`CreateFile`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilew) 函数时，您可以传递相当多的标志来更改系统缓存文件数据的方式。其他一些设备（如 Serial Port、Mailslot 和 Pipe）也会缓存数据。如果要强制系统将缓存的数据写入设备，可以调用 [`FlushFileBuffers`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-flushfilebuffers)。

### 同步 I/O 取消

执行同步 I/O 的函数易于使用，但它们会在发出 I/O 的线程上阻塞，直到请求完成。如果在 GUI 应用程序的 UI 线程上执行同步 I/O，则可能会因此长时间阻塞，此时 UI 将会冻住。

在 Windows Vista 中，Microsoft 增加了一些特性来缓解这个问题。例如，如果 CUI 应用程序由于同步 I/O 而挂起，则用户可以按下 `Ctrl + C` 以重新获得控制权并继续使用控制台。用户不再需要终止控制台进程。此外，新的文件打开/保存对话框允许用户在打开文件花费过长时间时按“取消”按钮。

如果要构建一个响应式应用程序，那么您应该尝试尽可能多地执行异步 I/O 操作。这通常允许您在应用程序中使用很少的线程，从而节省资源（如线程内核对象和栈）。此外，这通常很容易为用户提供在异步启动操作时取消操作的功能。

遗憾的是，某些 Windows API（如 [`CreateFile`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilew)）无法提供异步调用的方法。尽管其中一些方法会在等待时间过长时超时，但最好是调用 API 来强制线程中止等待并取消同步 I/O 操作。在 Windows Vista 中，[`CancelSynchronousIo`](https://docs.microsoft.com/en-us/windows/win32/fileio/cancelsynchronousio-func) 函数允许您取消给定线程的正在挂起的同步 I/O 请求。

> 注：I/O 请求的取消取决于相应系统层的驱动程序的实现。驱动程序可能会不支持取消，在这种情况下，[`CancelSynchronousIo`](https://docs.microsoft.com/en-us/windows/win32/fileio/cancelsynchronousio-func) 始终会返回 `TRUE`，因为此函数已找到标记为已取消的请求。

## 异步设备 I/O 基础

与计算机执行的大多数其他操作相比，设备 I/O 是最慢且最不可预测的操作之一。使用异步设备 I/O 能够更好地利用资源，从而创建更高效的应用程序。

考虑向设备发出异步 I/O 请求的线程，此 I/O 请求传递到设备驱动程序，驱动程序负责实际执行 I/O。当设备驱动程序等待设备响应时，应用程序的线程在等待 I/O 请求完成时不会挂起。相反，此线程继续执行其他有用的任务。在某个时候，设备驱动程序处理完排队的 I/O 请求，并且必须通知应用程序数据已发送、已接收或发生错误。对异步 I/O 请求进行排队是设计高性能、可伸缩应用程序的本质。

要异步访问设备，必须首先调用 [`CreateFile`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilew) 并在 `dwFlagsAndAttributes` 参数中指定 `FILE_FLAG_OVERLAPPED` 标志来打开设备，此标志通知系统您打算异步访问设备。

然后可以使用 [`ReadFile`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-readfile) 和 [`WriteFile`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-writefile) 函数将 I/O 请求排队到设备驱动程序。

### 异步设备 I/O 注意事项

在执行异步 I/O 时，设备驱动程序不一定以 FIFO（First-In First-Out）方式处理排队的 I/O 请求。比如，一个线程执行如下代码，可能会发生先写后读的情况：

```cpp
OVERLAPPED o1 = { 0 };
OVERLAPPED o2 = { 0 };
BYTE bBuffer[100];
ReadFile (hFile, bBuffer, 100, NULL, &o1);
WriteFile(hFile, bBuffer, 100, NULL, &o2);
```

设备驱动程序通常会按乱序执行 I/O 请求（如果这样做有助于提高性能）。例如，为了减少磁头移动和寻道时间，文件系统驱动程序可能会扫描排队的 I/O 请求列表，以查找与硬盘驱动器物理位置上靠近的请求。

当尝试将异步 I/O 请求排队时，设备驱动程序可能会选择同步地处理请求。当您正在读取文件并且系统检查到所需的数据已在系统的缓存中时，可能会发生这种情况。如果数据可用，那么您的 I/O 请求不会排队到设备驱动程序。相反，系统会将数据从缓存复制到缓冲区，并且 I/O 操作完成。

要注意，在 I/O 请求完成之前，不得移动或销毁用于发出异步 I/O 请求的数据缓冲区和 [`OVERLAPPED`](https://docs.microsoft.com/en-us/windows/win32/api/minwinbase/ns-minwinbase-overlapped) 结构。因为将 I/O 请求排队到设备驱动程序时，只是向驱动程序传递数据缓冲区的地址和 [`OVERLAPPED`](https://docs.microsoft.com/en-us/windows/win32/api/minwinbase/ns-minwinbase-overlapped) 结构的地址，并没有拷贝副本。

### 取消排队的设备 I/O 请求

有时，您可能希望在设备驱动程序处理排队的设备 I/O 请求之前取消该请求。Windows 提供了几种方法来执行此操作：

* 可以调用 [`CancelIo`](https://docs.microsoft.com/en-us/windows/win32/fileio/cancelio) 来取消由指定句柄的调用线程排队的所有 I/O 请求（除非该句柄已与 IOCP 关联）。
* 可以通过关闭设备本身的句柄来取消所有排队的 I/O 请求（不管请求是由哪个线程排队的）。
* 当线程终止时，系统会自动取消该线程发出的所有 I/O 请求，但对已与 IOCP 关联的句柄发出的请求除外。
* 如果需要取消在给定文件句柄上提交的单个特定 I/O 请求，可以调用 [`CancelIoEx`](https://docs.microsoft.com/en-us/windows/win32/fileio/cancelioex-func)。

## 接收已完成的 I/O 请求通知

Windows 提供了四种不同的方法用于接收 I/O 完成通知，如下表：

| Technique | Summary |
| :-- | :-- |
| Signaling a device kernel object | Not useful for performing multiple simultaneous I/O requests against a single device. Allows one thread to issue an I/O request and another thread to process it. |
| Signaling an event kernel object | Allows multiple simultaneous I/O requests against a single device. Allows one thread to issue an I/O request and another thread to process it. |
| Using alertable I/O | Allows multiple simultaneous I/O requests against a single device. The thread that issued an I/O request must also process it. |
| Using I/O completion ports | Allows multiple simultaneous I/O requests against a single device. Allows one thread to issue an I/O request and another thread to process it. This technique is highly scalable and has the most flexibility. |

### 示意设备内核对象

一旦线程发出异步 I/O 请求，线程就会继续执行，从而执行有用的工作。最终，线程需要与 I/O 操作的完成同步。

在 Windows 中，设备内核对象可用于线程同步，因此该对象可以处于已示意状态或未示意状态。[`ReadFile`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-readfile) 和 [`WriteFile`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-writefile) 函数在排队 I/O 请求之前将设备内核对象设置为未示意状态。当设备驱动程序完成请求时，驱动程序会将设备内核对象设置为已示意状态。

线程可以通过调用 [`WaitForSingleObject`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitforsingleobject) 或 [`WaitForMultipleObjects`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitformultipleobjects) 来确定异步 I/O 请求是否已完成。下面是一个简单的示例：

```cpp
HANDLE hFile = CreateFile(..., FILE_FLAG_OVERLAPPED, ...);
BYTE bBuffer[100];
OVERLAPPED o = { 0 };
o.Offset = 345;

BOOL bReadDone = ReadFile(hFile, bBuffer, 100, NULL, &o);
DWORD dwError = GetLastError();

if (!bReadDone && (dwError == ERROR_IO_PENDING)) {
    // The I/O is being performed asynchronously; wait for it to complete
    WaitForSingleObject(hFile, INFINITE);
    bReadDone = TRUE;
}

if (bReadDone) {
    // o.Internal contains the I/O error
    // o.InternalHigh contains the number of bytes transferred
    // bBuffer contains the read data
} else {
    // An error occurred; see dwError
}
```

### 示意事件内核对象

[`OVERLAPPED`](https://docs.microsoft.com/en-us/windows/win32/api/minwinbase/ns-minwinbase-overlapped) 结构的 `hEvent` 成员可以用于标识一个事件内核对象，该事件对象必须通过调用 [`CreateEvent`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-createeventw) 来创建。当异步 I/O 请求完成后，设备驱动程序将检查 [`OVERLAPPED`](https://docs.microsoft.com/en-us/windows/win32/api/minwinbase/ns-minwinbase-overlapped) 结构的 `hEvent` 成员是否为 `NULL`。若 `hEvent` 不为 `NULL`，则驱动程序将通过调用 [`SetEvent`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-setevent) 来发出事件信号。

如果要同时执行多个异步设备 I/O 请求，那么必须为每个请求创建一个单独的事件对象，在每个请求的 [`OVERLAPPED`](https://docs.microsoft.com/en-us/windows/win32/api/minwinbase/ns-minwinbase-overlapped) 结构中初始化 `hEvent` 成员，然后调用 [`ReadFile`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-readfile) 或 [`WriteFile`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-writefile)。当需要与 I/O 请求的完成同步时，只需调用 [`WaitForMultipleObjects`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitformultipleobjects)，并传入与未完成的 I/O 请求的 [`OVERLAPPED`](https://docs.microsoft.com/en-us/windows/win32/api/minwinbase/ns-minwinbase-overlapped) 结构关联的事件句柄。使用此方案，可以简单可靠地同时执行多个异步设备 I/O 操作，并使用相同的设备对象。下面是一个简单的示例：

```cpp
HANDLE hFile = CreateFile(..., FILE_FLAG_OVERLAPPED, ...);

BYTE bReadBuffer[10];
OVERLAPPED oRead = { 0 };
oRead.Offset = 0;
oRead.hEvent = CreateEvent(...);
ReadFile(hFile, bReadBuffer, 10, NULL, &oRead);

BYTE bWriteBuffer[10] = { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9 };
OVERLAPPED oWrite = { 0 };
oWrite.Offset = 10;
oWrite.hEvent = CreateEvent(...);
WriteFile(hFile, bWriteBuffer, _countof(bWriteBuffer), NULL, &oWrite);
...

HANDLE h[2];
h[0] = oRead.hEvent;
h[1] = oWrite.hEvent;
DWORD dw = WaitForMultipleObjects(2, h, FALSE, INFINITE);
switch (dw – WAIT_OBJECT_0) {
case 0: // Read completed
    break;
case 1: // Write completed
    break;
}
```

### 可警示 I/O

每当创建线程时，系统也会创建一个与该线程关联的队列，称为 APC（Asynchronous Procedure Call）队列。当发出 I/O 请求时，您可以告诉设备驱动程序将一个条目追加到调用线程的 APC 队列。要将已完成的 I/O 通知排队到线程的 APC 队列，请调用 [`ReadFileEx`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-readfileex) 和 [`WriteFileEx`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-writefileex) 函数。

当您使用 [`ReadFileEx`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-readfileex) 和 [`WriteFileEx`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-writefileex) 发出异步 I/O 请求时，这些函数会将 *完成例程（Completion Routine）* 的地址传递给设备驱动程序。当设备驱动程序完成 I/O 请求后，它会在发出线程的 APC 队列中追加一个条目。此条目包含完成例程的地址和用于启动 I/O 请求的 [`OVERLAPPED`](https://docs.microsoft.com/en-us/windows/win32/api/minwinbase/ns-minwinbase-overlapped) 结构的地址。

> 注：当可警示 I/O 完成时，设备驱动程序不会尝试向事件对象发出信号。

当线程处于可警示状态时，系统将检查其 APC 队列，对于队列中的每个条目，系统都会调用完成例程，并向其传递 I/O 错误码、传输的字节数以及 [`OVERLAPPED`](https://docs.microsoft.com/en-us/windows/win32/api/minwinbase/ns-minwinbase-overlapped) 结构的地址。

现在，让我们看一下系统如何处理异步 I/O 请求。下面的代码将三种不同的异步操作排队：

```cpp
hFile = CreateFile(..., FILE_FLAG_OVERLAPPED, ...);

ReadFileEx(hFile, ...);  // Perform first ReadFileEx
WriteFileEx(hFile, ...); // Perform first WriteFileEx
ReadFileEx(hFile, ...);  // Perform second ReadFileEx

SomeFunc();
```

如果对 `SomeFunc` 的调用需要一些时间来执行，那么系统可能在 `SomeFunc` 返回之前完成这三个操作。虽然线程正在执行 `SomeFunc` 函数，但设备驱动程序会将已完成的 I/O 条目追加到线程的 APC 队列中。APC 队列可能如下所示：

```
first WriteFileEx completed
second ReadFileEx completed
first ReadFileEx completed
```

APC 队列由系统内部维护。系统可以按任何顺序执行排队的 I/O 请求，您最后发出的 I/O 请求可能最先完成。线程的 APC 队列中的每个条目都包含回调函数的地址和传递给该函数的值。

当 I/O 请求完成时，它们只是简单地排队到线程的 APC 队列，回调例程不会立即调用，因为线程可能正忙于执行其他操作，并且无法中断。要处理线程的 APC 队列中的条目，线程必须将自身置于可警示状态。

Windows 提供了六个函数，可以将线程置于可警示状态：

* [`SleepEx`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-sleepex)
* [`WaitForSingleObjectEx`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitforsingleobjectex)
* [`WaitForMultipleObjectsEx`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitformultipleobjectsex)
* [`SignalObjectAndWait`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-signalobjectandwait)
* [`GetQueuedCompletionStatusEx`](https://docs.microsoft.com/en-us/windows/win32/api/ioapiset/nf-ioapiset-getqueuedcompletionstatusex)
* [`MsgWaitForMultipleObjectsEx`](https://docs.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-msgwaitformultipleobjectsex)

当您调用这些函数并将线程置于可警示状态时，系统首先会检查线程的 APC 队列。若队列中至少有一个条目，则系统不会使线程睡眠。相反，系统会从 APC 队列中提取条目，并使线程调用回调例程，直到队列没有条目为止。

在线程的 APC 队列中没有条目时，调用这些函数会使线程挂起。当线程挂起时，如果线程正在等待的内核对象发出信号，或者如果线程队列中出现 APC 条目，那么线程将会唤醒。由于线程处于可警示状态，因此一旦出现 APC 条目，系统就会唤醒线程并清空队列（通过调用回调例程）。然后，这些函数立即返回给调用者 —— 线程不会返回睡眠状态。

#### 可警示 I/O 的优缺点

使用可警示 I/O 方式来执行设备 I/O 有两个可怕的问题：

* 回调函数。可警示 I/O 要求您创建回调函数，这使得编写代码变得更加困难。因为回调函数通常没有足够的上下文信息，因此您最终会在全局变量中放置大量信息。幸运的是，这些全局变量不需要同步，因为调用六个可警示函数之一的线程与执行回调函数的线程相同。
* 线程问题。可警示 I/O 真正的大问题是：发出 I/O 请求的线程也必须处理完成通知。如果一个线程发出多个请求，则该线程必须响应每个请求的完成通知。由于没有负载平衡，因此应用程序无法很好地伸缩。

Windows 提供了 [`QueueUserAPC`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-queueuserapc) 函数，允许您手动将条目排队到线程的 APC 队列。您可以使用 [`QueueUserAPC`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-queueuserapc) 执行极其高效的线程间通信，甚至可以跨进程边界执行。但不幸的是，您只能传递单个值。

[`QueueUserAPC`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-queueuserapc) 还可用于强制线程退出等待状态。假设您有一个调用了 [`WaitForSingleObjectEx`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitforsingleobjectex) 的线程，其等待内核对象发出信号。在线程等待时，用户希望终止应用程序。您希望线程可以干净地销毁自己，这时候可以利用 [`QueueUserAPC`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-queueuserapc) 来唤醒线程然后让其销毁自己。以下是一个示例：

```cpp
// The APC callback function has nothing to do
VOID WINAPI APCFunc(ULONG_PTR dwParam) {
    // Nothing to do in here
}

UINT WINAPI ThreadFunc(PVOID pvParam) {
    HANDLE hEvent = (HANDLE) pvParam; // Handle is passed to this thread

    // Wait in an alertable state so that we can be forced to exit cleanly
    DWORD dw = WaitForSingleObjectEx(hEvent, INFINITE, TRUE);
    if (dw == WAIT_OBJECT_0) {
        // Object became signaled
    }
    if (dw == WAIT_IO_COMPLETION) {
        // QueueUserAPC forced us out of a wait state
        return(0); // Thread dies cleanly
    }

    ...

    return(0);
}

void main() {
    HANDLE hEvent = CreateEvent(...);
    HANDLE hThread = (HANDLE) _beginthreadex(NULL, 0, ThreadFunc, (PVOID) hEvent, 0, NULL);

    ...

    // Force the secondary thread to exit cleanly
    QueueUserAPC(APCFunc, hThread, NULL);
    WaitForSingleObject(hThread, INFINITE);
    CloseHandle(hThread);
    CloseHandle(hEvent);
}
```

### IOCP

Windows 被设计为一个安全、可靠的操作系统，运行为数千名用户提供服务的应用程序。从历史上看，您已经能够通过遵循以下两个模型之一来构建服务应用程序：

* 串行模型 —— 单个线程等待客户端发出请求。当请求传入时，线程将唤醒并处理客户端的请求。
* 并发模型 —— 单个线程等待客户端请求，然后创建一个新线程来处理该请求。当新线程处理客户端的请求时，原始线程将循环返回并等待另一个客户端请求。处理客户端请求的线程在完成处理时终止。

串行模型的问题在于它不能很好地处理多个同时的请求。如果两个客户端同时发出请求，则一次只能处理一个请求，第二个请求必须等待第一个请求完成处理。使用串行模型设计的服务无法利用多处理器计算机。

由于串行模型的限制，并发模型非常受欢迎。在并发模型中，为每个客户端请求创建一个线程来处理。优点是等待传入请求的线程几乎没有什么工作要做。大多数情况下，此线程处于睡眠状态。当客户端请求传入时，此线程将唤醒，并创建一个新线程来处理该请求，然后等待另一个客户端请求。由于每个客户端请求都有自己的线程，因此服务器应用程序可以很好地伸缩，并且可以轻松地利用多处理器计算机。

Windows 团队注意到并发模型应用程序性能并没有达到预期的水平。处理大量同时发生的客户端请求意味着会有大量线程在系统中同时运行。由于所有这些线程都是可运行的，Windows 内核将会花费大量时间在线程间的上下文切换上。为了使 Windows 成为一个优秀的服务器环境，Microsoft 设计了 IOCP 内核对象来解决这个问题。

#### 创建 IOCP

[IOCP](https://docs.microsoft.com/en-us/windows/win32/fileio/i-o-completion-ports) 背后的理论是：并发运行的线程数必须有一个上限，即 500 个并发请求不能允许存在 500 个可运行线程。如果一台机器有两个 CPU，那么有两个以上的可运行线程实际上是没有意义的。一旦可运行线程数超过可用的 CPU，系统就必须花时间来执行线程上下文切换，这会浪费宝贵的 CPU 周期，这就是并发模型的潜在缺陷。

并发模型的另一个缺陷是为每个请求创建一个新线程。与创建进程相比，创建线程的开销很小，但这种开销也是不可忽略的。如果在应用程序初始化时创建一个线程池并且使其中的线程一直存在，则可以提高服务应用程序的性能。IOCP 被设计为与线程池一起配合使用。

IOCP 可能是最复杂的内核对象。若要创建 IOCP，请调用 [`CreateIoCompletionPort`](https://docs.microsoft.com/en-us/windows/win32/api/ioapiset/nf-ioapiset-createiocompletionport)。

此函数执行了两个任务：创建一个 IOCP；将设备与 IOCP 相关联。为了简化这个函数，可以使用如下两个函数来独立地完成单个任务：

```cpp
HANDLE CreateNewCompletionPort(DWORD dwNumberOfConcurrentThreads) {
    return(CreateIoCompletionPort(INVALID_HANDLE_VALUE, NULL, 0, dwNumberOfConcurrentThreads));
}

BOOL AssociateDeviceWithCompletionPort(
    HANDLE hCompletionPort, HANDLE hDevice, DWORD dwCompletionKey) {

    HANDLE h = CreateIoCompletionPort(hDevice, hCompletionPort, dwCompletionKey, 0);
    return(h == hCompletionPort);
}
```

创建 IOCP 时，内核实际上会创建五种不同的数据结构，如下图所示：

![The internal workings of an I/O completion port](/images/the-internal-workings-of-an-iocp.svg)

第一个数据结构是一个设备列表（Device List），指示与 IOCP 关联的一个或多个设备。每次将设备关联到 IOCP 时，系统都会将传递给 [`CreateIoCompletionPort`](https://docs.microsoft.com/en-us/windows/win32/api/ioapiset/nf-ioapiset-createiocompletionport) 函数的设备句柄和完成键等信息追加到 IOCP 的设备列表。

第二个数据结构是一个 I/O 完成队列（I/O Completion Queue）。当设备的异步 I/O 请求完成时，系统将检查该设备是否与 IOCP 关联，如果是，系统就会将已完成的 I/O 请求条目追加到 IOCP 的 I/O 完成队列的末尾。此队列中的每个条目都指示已传输的字节数、设备与 IOCP 关联时设置的完成键、指向 I/O 请求的 [`OVERLAPPED`](https://docs.microsoft.com/en-us/windows/win32/api/minwinbase/ns-minwinbase-overlapped) 结构的指针以及错误码。

#### 围绕 IOCP 进行架构设计

当服务应用程序初始化时，它应通过调用 [`CreateIoCompletionPort`](https://docs.microsoft.com/en-us/windows/win32/api/ioapiset/nf-ioapiset-createiocompletionport) 函数来创建 IOCP。然后，它应创建一个线程池来处理客户端请求。

线程池中的所有线程都应执行相同的函数。通常，线程函数执行某种初始化，然后进入一个循环，当服务进程指示要停止时，该循环应终止。在循环内部，线程应该调用 [`GetQueuedCompletionStatus`](https://docs.microsoft.com/en-us/windows/win32/api/ioapiset/nf-ioapiset-getqueuedcompletionstatus) 函数以将自身置于睡眠状态并等待设备 I/O 请求完成。 

与 IOCP 关联的第三个数据结构是一个等待线程队列（Waiting Thread Queue）。当线程池中的线程调用 [`GetQueuedCompletionStatus`](https://docs.microsoft.com/en-us/windows/win32/api/ioapiset/nf-ioapiset-getqueuedcompletionstatus) 时，调用线程的 ID 将入队到此等待线程队列中，从而使 IOCP 内核对象始终知道哪些线程当前正在等待处理已完成的 I/O 请求。当一个条目出现在 IOCP 的 I/O 完成队列中时，IOCP 将唤醒等待线程队列中的一个线程。

确定 [`GetQueuedCompletionStatus`](https://docs.microsoft.com/en-us/windows/win32/api/ioapiset/nf-ioapiset-getqueuedcompletionstatus) 返回的原因有些困难。下面的代码演示了执行此操作的正确方法：

```cpp
DWORD dwNumBytes;
ULONG_PTR CompletionKey;
OVERLAPPED* pOverlapped;

// hIOCP is initialized somewhere else in the program
BOOL bOk = GetQueuedCompletionStatus(hIOCP, &dwNumBytes, &CompletionKey, &pOverlapped, 1000);
DWORD dwError = GetLastError();

if (bOk) {
    // Process a successfully completed I/O request
} else {
    if (pOverlapped != NULL) {
        // Process a failed completed I/O request
        // dwError contains the reason for failure
    } else {
        if (dwError == WAIT_TIMEOUT) {
            // Time-out while waiting for completed I/O entry
        } else {
            // Bad call to GetQueuedCompletionStatus
            // dwError contains the reason for the bad call
        }
    }
}
```

条目将以先进先出的方式从 I/O 完成队列中删除。但是，调用 [`GetQueuedCompletionStatus`](https://docs.microsoft.com/en-us/windows/win32/api/ioapiset/nf-ioapiset-getqueuedcompletionstatus) 的线程将以后进先出的方式唤醒。这样做的原因是为了提高性能。因为如果 I/O 请求完成的速度比线程处理 I/O 完成条目的速度要慢时，单个线程就足以处理这些 I/O 完成条目，系统只需保持同一个线程一直唤醒，而其它线程可以继续睡眠。系统甚至可以将这些未调度线程的内存资源交换到磁盘，并从处理器缓存中刷新。

在 Windows Vista 中，如果希望不断地提交大量 I/O 请求，而不是增加等待 IOCP 的线程数（这将导致上下文切换的开销增加），那么可以通过调用 [`GetQueuedCompletionStatusEx`](https://docs.microsoft.com/en-us/windows/win32/api/ioapiset/nf-ioapiset-getqueuedcompletionstatusex) 函数同时检索多个 I/O 请求的结果。

> 注：
> 
> 当您向与 IOCP 关联的设备发出异步 I/O 请求时，Windows 会将结果排队到 IOCP。即使异步请求是同步执行的，也是如此，这时可能会略微地降低性能，因为必须将完成的请求信息放在 IOCP 中，并且线程必须从 IOCP 中提取它。
> 
> 若要略微地提高性能，可以通过调用 [`SetFileCompletionNotificationModes`](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-setfilecompletionnotificationmodes) 函数来告诉 Windows 不要将同步执行的异步请求排队到与设备关联的 IOCP。
> 
> 非常注重性能的程序员可能还需要考虑使用 [`SetFileIoOverlappedRange`](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-setfileiooverlappedrange) 函数。

#### IOCP 如何管理线程池

在创建 IOCP 时，需要指定最大并发值。当已完成的 I/O 条目入队时，IOCP 需要唤醒正在等待的线程。然而，IOCP 唤醒的线程数不会超过最大并发值。因此，假设您指定了最大并发值为二，那么当四个 I/O 请求完成并且有四个线程正在等待对 [`GetQueuedCompletionStatus`](https://docs.microsoft.com/en-us/windows/win32/api/ioapiset/nf-ioapiset-getqueuedcompletionstatus) 的调用时，IOCP 也只允许两个线程唤醒。

您应该会注意到上述假设中，四个线程中有两个似乎是多余的，它们可能永远不会被唤醒。

然而 IOCP 是非常智能的。当 IOCP 唤醒线程时，它会将线程的 ID 放置到释放线程列表（Released Thread List）。这允许 IOCP 记住它唤醒了哪些线程，并监视这些线程的执行。如果已释放的线程调用了任何将线程置于等待状态的函数，则 IOCP 会检测到这一点，并通过将线程的 ID 从释放线程列表移动到暂停线程列表（Paused Thread List）来更新其内部数据结构。

IOCP 的目标是在释放线程列表中保持最大并发值所允许的条目数。如果一个已释放的线程因任何原因进入等待状态，则释放线程列表将收缩，并且 IOCP 将释放另一个等待线程。如果一个已暂停的线程被唤醒，那么它将离开暂停线程列表并重新进入释放线程列表。这意味着释放线程列表中现在可以包含多于最大并发值所允许的条目数。

> 注：
> 
> 一旦线程调用 [`GetQueuedCompletionStatus`](https://docs.microsoft.com/en-us/windows/win32/api/ioapiset/nf-ioapiset-getqueuedcompletionstatus)，该线程就会被“分配”到指定的 IOCP。系统假定所有已分配的线程都代表该 IOCP 执行工作。仅当正在运行的已分配线程数小于 IOCP 的最大并发值时，IOCP 才会从线程池中唤醒线程。
> 
> 可以通过以下三种方式之一打破 线程/IOCP 分配：
> * 线程退出。
> * 线程调用 [`GetQueuedCompletionStatus`](https://docs.microsoft.com/en-us/windows/win32/api/ioapiset/nf-ioapiset-getqueuedcompletionstatus)，并传递其他 IOCP 的句柄。
> * 销毁线程当前分配到的 IOCP。

#### 线程池中的线程数

考虑两个问题。首先，当服务应用程序初始化时，您希望创建一组最少的线程，这样就不必定期创建和销毁线程。请记住，创建和销毁线程会消耗 CPU 时间，因此最好尽可能减少此过程。其次，您希望设置最大线程数，因为创建太多线程会浪费系统资源（如 RAM）。

您可能希望尝试不同数量的线程。大多数服务（包括 Microsoft Internet Information Services）使用启发式（Heuristic）算法来管理线程池。您可以创建以下变量来管理线程池：

```cpp
LONG g_nThreadsMin;  // Minimum number of threads in pool
LONG g_nThreadsMax;  // Maximum number of threads in pool
LONG g_nThreadsCrnt; // Current number of threads in pool
LONG g_nThreadsBusy; // Number of busy threads in pool
```

当应用程序初始化时，您可以创建 `g_nThreadsMin` 数量的线程，所有这些线程都执行相同的线程池函数。以下伪代码展示了此线程函数的大概轮廓：

```cpp
DWORD WINAPI ThreadPoolFunc(PVOID pv) {
    // Thread is entering pool
    InterlockedIncrement(&g_nThreadsCrnt);
    InterlockedIncrement(&g_nThreadsBusy);

    for (BOOL bStayInPool = TRUE; bStayInPool;) {
        // Thread stops executing and waits for something to do
        InterlockedDecrement(&m_nThreadsBusy);
        BOOL bOk = GetQueuedCompletionStatus(...);
        DWORD dwIOError = GetLastError();

        // Thread has something to do, so it's busy
        int nThreadsBusy = InterlockedIncrement(&m_nThreadsBusy);

        // Should we add another thread to the pool?
        if (nThreadsBusy == m_nThreadsCrnt) { // All threads are busy
            if (nThreadsBusy < m_nThreadsMax) { // The pool isn't full
                if (GetCPUUsage() < 75) { // CPU usage is below 75%
                    // Add thread to pool
                    CloseHandle(chBEGINTHREADEX(...));
                }
            }
        }

        if (!bOk && (dwIOError == WAIT_TIMEOUT)) { // Thread timed out
            // There isn't much for the server to do, and this thread
            // can die even if it still has outstanding I/O requests
            bStayInPool = FALSE;
        }

        if (bOk || (po != NULL)) {
            // Thread woke to process something; process it
            ...

            if (GetCPUUsage() > 90) { // CPU usage is above 90%
                if (g_nThreadsCrnt > g_nThreadsMin)) { // Pool above min
                    bStayInPool = FALSE; // Remove thread from pool
                }
            }
        }
    }

    // Thread is leaving pool
    InterlockedDecrement(&g_nThreadsBusy);
    InterlockedDecrement(&g_nThreadsCurrent);
    return(0);
}
```

#### 模拟已完成的 I/O 请求

IOCP 不一定要与设备 I/O 一起使用，它还可以用于线程间通信。在 [可警示 I/O 的优缺点](#可警示-i-o-的优缺点) 中提到的 [`QueueUserAPC`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-queueuserapc) 函数允许一个线程将 APC 条目发布到另一个线程。IOCP 也具有类似的函数 [`PostQueuedCompletionStatus`](https://docs.microsoft.com/en-us/windows/win32/fileio/postqueuedcompletionstatus)。

[`PostQueuedCompletionStatus`](https://docs.microsoft.com/en-us/windows/win32/fileio/postqueuedcompletionstatus) 函数非常有用，它为您提供了一种与线程池中的所有线程进行通信的方法。例如，当用户终止服务应用程序时，您希望所有线程都干净地退出。但是，如果线程正在等待 IOCP，并且没有 I/O 请求传入，那么线程将无法唤醒。通过为线程池中的每个线程调用一次 [`PostQueuedCompletionStatus`](https://docs.microsoft.com/en-us/windows/win32/fileio/postqueuedcompletionstatus)，可以使这些线程唤醒，被唤醒的线程应当检查从 [`GetQueuedCompletionStatus`](https://docs.microsoft.com/en-us/windows/win32/api/ioapiset/nf-ioapiset-getqueuedcompletionstatus) 返回的值以获知服务应用程序是否正在终止，并相应地清理和退出。

在 Windows Vista 中，当您调用 [`CloseHandle`](https://docs.microsoft.com/en-us/windows/win32/api/handleapi/nf-handleapi-closehandle) 关闭 IOCP 时，所有因调用 [`GetQueuedCompletionStatus`](https://docs.microsoft.com/en-us/windows/win32/api/ioapiset/nf-ioapiset-getqueuedcompletionstatus) 而正在等待的线程都将唤醒，[`GetQueuedCompletionStatus`](https://docs.microsoft.com/en-us/windows/win32/api/ioapiset/nf-ioapiset-getqueuedcompletionstatus) 函数将返回 `FALSE`。这些线程随后调用 [`GetLastError`](https://docs.microsoft.com/en-us/windows/win32/api/errhandlingapi/nf-errhandlingapi-getlasterror) 将返回 `ERROR_INVALID_HANDLE`。可以利用这个信息来干净地销毁线程。
