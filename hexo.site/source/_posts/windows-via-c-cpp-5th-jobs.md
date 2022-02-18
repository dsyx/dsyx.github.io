---
title: 'Windows via C/C++, 5th Edition - Jobs'
date: 2022-02-14 09:24:02
category: 'My Notes'
tags: ['Windows via C/C++', 'Windows']
---

Microsoft Windows 提供了[作业内核对象（Job Kernel Object）](https://docs.microsoft.com/en-us/windows/win32/procthread/job-objects)，允许将进程组合在一起，并创建一个“沙盒（sandbox）”来限制进程可以执行的操作。最好将作业对象视为进程的容器。创建包含单个进程的作业也是很有用的，因为您可以对该进程施加各种限制。

如下 `StartRestrictedProcess` 函数将一个进程放置到一个限制进程执行某些操作的能力的作业中：

```cpp
void StartRestrictedProcess() {
    // Check if we are not already associated with a job.
    // If this is the case, there is no way to switch to another job.
    BOOL bInJob = FALSE;
    IsProcessInJob(GetCurrentProcess(), NULL, &bInJob);
    if (bInJob) {
        MessageBox(NULL, TEXT("Process already in a job"), TEXT(""), MB_ICONINFORMATION | MB_OK);
        return;
    }

    // Create a job kernel object.
    HANDLE hjob = CreateJobObject(NULL, TEXT("Wintellect_RestrictedProcessJob"));

    // Place some restrictions on processes in the job.

    // First, set some basic restrictions.
    JOBOBJECT_BASIC_LIMIT_INFORMATION jobli = { 0 };

    // The process always runs in the idle priority class.
    jobli.PriorityClass = IDLE_PRIORITY_CLASS;

    // The job cannot use more than 1 second of CPU time.
    jobli.PerJobUserTimeLimit.QuadPart = 10000; // 1 sec in 100-ns intervals

    // These are the only 2 restrictions I want placed on the job (process).
    jobli.LimitFlags = JOB_OBJECT_LIMIT_PRIORITY_CLASS | JOB_OBJECT_LIMIT_JOB_TIME;
    SetInformationJobObject(hjob, JobObjectBasicLimitInformation, &jobli, sizeof(jobli));

    // Second, set some UI restrictions.
    JOBOBJECT_BASIC_UI_RESTRICTIONS jobuir;
    jobuir.UIRestrictionsClass = JOB_OBJECT_UILIMIT_NONE; // A fancy zero

    // The process can't log off the system.
    jobuir.UIRestrictionsClass |= JOB_OBJECT_UILIMIT_EXITWINDOWS;

    // The process can't access USER objects (such as other windows) in the system.
    jobuir.UIRestrictionsClass |= JOB_OBJECT_UILIMIT_HANDLES;
    
    SetInformationJobObject(hjob, JobObjectBasicUIRestrictions, &jobuir, sizeof(jobuir));

    // Spawn the process that is to be in the job.
    // Note: You must first spawn the process and then place the process in the job.
    //       This means that the process' thread must be initially suspended so that
    //       it can't execute any code outside of the job's restrictions.
    STARTUPINFO si = { sizeof(si) };
    PROCESS_INFORMATION pi;
    TCHAR szCmdLine[8];
    _tcscpy_s(szCmdLine, _countof(szCmdLine), TEXT("CMD"));
    BOOL bResult = CreateProcess(NULL, szCmdLine, NULL, NULL, FALSE, CREATE_SUSPENDED | CREATE_NEW_CONSOLE, NULL, NULL, &si, &pi);

    // Place the process in the job.
    // Note: If this process spawns any children, the children are
    //       automatically part of the same job.
    AssignProcessToJobObject(hjob, pi.hProcess);

    // Now we can allow the child process' thread to execute code.
    ResumeThread(pi.hThread);
    CloseHandle(pi.hThread);

    // Wait for the process to terminate or
    // for all the job's allotted CPU time to be used.
    HANDLE h[2];
    h[0] = pi.hProcess;
    h[1] = hjob;
    DWORD dw = WaitForMultipleObjects(2, h, FALSE, INFINITE);
    switch (dw - WAIT_OBJECT_0) {
    case 0:
        // The process has terminated...
        break;
    case 1:
        // All of the job's allotted CPU time was used...
        break;
    }

    FILETIME CreationTime;
    FILETIME ExitTime;
    FILETIME KernelTime;
    FILETIME UserTime;
    TCHAR szInfo[MAX_PATH];
    GetProcessTimes(pi.hProcess, &CreationTime, &ExitTime, &KernelTime, &UserTime);
    StringCchPrintf(szInfo, _countof(szInfo), TEXT("Kernel = %u | User = %u\n"),
    KernelTime.dwLowDateTime / 10000, UserTime.dwLowDateTime / 10000);
    MessageBox(GetActiveWindow(), szInfo, TEXT("Restricted Process times"), MB_ICONINFORMATION | MB_OK);

    // Clean up properly.
    CloseHandle(pi.hProcess);
    CloseHandle(hjob);
}
```

> 注：默认情况下，当您通过 Windows 资源管理器（Windows Explorer）启动应用程序时，该进程会自动关联到一个专用作业，其名称以“PCA”字符串为前缀。当作业中的进程退出时，资源管理器可能会收到通知。因此，当由 Windows 资源管理器启动的传统应用程序出现故障时，将触发程序兼容性助手（Program Compatibility Assistant）。从 Shell 中启动应用程序则不会发生这种作业关联。

## 对作业的进程设置限制

创建作业后，通常需要设置沙盒（设置限制）以控制作业中的进程可以执行的操作。通过调用 [`SetInformationJobObject`](https://docs.microsoft.com/en-us/windows/win32/api/jobapi2/nf-jobapi2-setinformationjobobject) 可以对作业设置几种不同类型的限制：

* 基本限制和扩展基本限制，可防止作业中的进程独占系统资源。
* 基本 UI 限制，可防止作业中的进程更改用户界面。
* 安全限制，可防止作业中的进程访问安全资源（文件、注册表子项等）。

一旦对作业设置了限制，您可能希望查询这些限制。可以通过调用 [`QueryInformationJobObject`](https://docs.microsoft.com/en-us/windows/win32/api/jobapi2/nf-jobapi2-queryinformationjobobject) 函数轻松地完成此操作。

## 在作业中放置进程

使用 [`AssignProcessToJobObject`](https://docs.microsoft.com/en-us/windows/win32/api/jobapi2/nf-jobapi2-assignprocesstojobobject) 函数可以将进程放置到作业中，一旦进程被放置到作业中，它就不能被转移到其它作业中。可以使用 [`IsProcessInJob`](https://docs.microsoft.com/en-us/windows/win32/api/jobapi/nf-jobapi-isprocessinjob) 函数检查进程是否已经在指定的作业中。

在创建进程后，将进程放置到作业之前，进程并不是作业的一部分，因此它并不会受到作业的限制，进程可以在这段时间内执行开发者预期要限制的事情。可以在创建计划要放置到作业中的进程时，在调用 [`CreateProcess`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessw) 时使用 `CREATE_SUSPENDED` 标志。这将会创建新进程，但不允许该进程执行任何代码。在将进程放置到作业后，可以调用 [`ResumeThread`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-resumethread) 函数使进程中的线程恢复执行。

## 终止作业中的所有进程

要想终止作业中的所有进程，只需调用 [`TerminateJobObject`](https://docs.microsoft.com/en-us/windows/win32/api/jobapi2/nf-jobapi2-terminatejobobject)。这类似于为作业中的每个进程调用 [`TerminateProcess`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-terminateprocess)，并将其退出代码设置为 `uExitCode`。

## 作业通知

通知允许您获知与作业相关的事件。例如，作业中的所有进程何时终止了，或者所有分配的 CPU 时间是否已到期了？作业中何时产生了新进程，或者作业中的进程何时终止了？

如果您关心的是所有分配的 CPU 时间是否已到期，则可以通过调用 [`WaitForSingleObject`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitforsingleobject) 或类似函数来捕获此事件。这是因为一旦用完所有分配的 CPU 时间，Windows 就会终止作业中的所有进程并将作业对象置于已示意（signaled）。随后，您可以通过调用 [`SetInformationJobObject`](https://docs.microsoft.com/en-us/windows/win32/api/jobapi2/nf-jobapi2-setinformationjobobject) 并给予作业更多的 CPU 时间来将作业对象重置回未示意（nonsignaled）状态。

如果需要获得更高级的通知信息（如进程的创建/终止），那么必须创建 I/O 完成端口内核对象（I/O Completion Port Kernel Object），并将作业对象或对象与其关联。然后，必须有一个或多个线程在完成端口上等待作业通知到达，以便可以对其进行处理。

创建 I/O 完成端口后，通过调用 [`SetInformationJobObject`](https://docs.microsoft.com/en-us/windows/win32/api/jobapi2/nf-jobapi2-setinformationjobobject) 将作业与其关联，如下所示：

```cpp
JOBOBJECT_ASSOCIATE_COMPLETION_PORT joacp;
joacp.CompletionKey = 1; // Any value to uniquely identify this job
joacp.CompletionPort = hIOCP; // Handle of completion port that receives notifications
SetInformationJobObject(hJob, JobObjectAssociateCompletionPortInformation, &joacp, sizeof(jaocp));
```

上述代码执行后，系统将监视作业，并在事件发生时将其发布到 I/O 完成端口。线程通常通过调用 [`GetQueuedCompletionStatus`](https://docs.microsoft.com/en-us/windows/win32/api/ioapiset/nf-ioapiset-getqueuedcompletionstatus) 来监视 I/O 完成端口。

最后需要注意的一点是：默认情况下，作业对象被配置为当作业分配的 CPU 时间到期时，作业的所有进程都将自动终止，并且不会发布 `JOB_OBJECT_MSG_END_OF_JOB_TIME` 通知。如果要防止作业对象终止进程，而只是通知您已超过时间，则必须执行如下代码：

```cpp
// Create a JOBOBJECT_END_OF_JOB_TIME_INFORMATION structure and initialize its only member.
JOBOBJECT_END_OF_JOB_TIME_INFORMATION joeojti;
joeojti.EndOfJobTimeAction = JOB_OBJECT_POST_AT_END_OF_JOB;

// Tell the job object what we want it to do when the job time is exceeded.
SetInformationJobObject(hJob, JobObjectEndOfJobTimeInformation, &joeojti, sizeof(joeojti));
```
