---
title: "[MMP] Windows相关API"
date: 2017-08-13 16:29:19
categories: [学习笔记, MMP]  
tags: [MMP,TODO]
---
这里只是记录一部分（之前笔记记的（从其它地方复制来的，做了一些修改），整理放在这），不全。现在新（指的是xp以上）系统里有线程池的API，而且添加了很多有用的函数，可以去找官方文档。  <!--more-->
MMP这部分剩下还有将近2000行的笔记要整理，就先不整这里了，因为一般不直接用WinAPI。  
TODO

**@ CreateThread**

创建线程
```
WINBASEAPI
_Ret_maybenull_
HANDLE
WINAPI
CreateThread(
    _In_opt_ LPSECURITY_ATTRIBUTES lpThreadAttributes,
    _In_ SIZE_T dwStackSize,
    _In_ LPTHREAD_START_ROUTINE lpStartAddress,
    _In_opt_ __drv_aliasesMem LPVOID lpParameter,
    _In_ DWORD dwCreationFlags,
    _Out_opt_ LPDWORD lpThreadId
    );
```

函数说明：
@lpThreadAttributes 线程内核对象的安全属性，一般传入NULL表示使用默认设置。
@dwStackSize 线程栈空间大小。传入0表示使用默认大小（1MB）。
@lpStartAddress 新线程所执行的线程函数地址，多个线程可以使用同一个函数地址。
@lpParameter 传给线程函数的参数。
@dwCreationFlags 指定额外的标志来控制线程的创建，为0表示线程创建之后立即就可以进行调度，如果为CREATE_SUSPENDED则表示线程创建后暂停运行，这样它就无法调度，直到调用ResumeThread()。
@lpThreadId 返回线程的ID号，传入NULL表示不需要返回该线程ID号。

@return：成功返回新线程的句柄，失败返回NULL。 


**@ WaitForSingleObject \ WaitForMultipleObjects**

等待函数 – 使线程进入等待状态，直到指定的内核对象被触发
```
WINBASEAPI
DWORD
WINAPI
WaitForSingleObject(
    _In_ HANDLE hHandle,
    _In_ DWORD dwMilliseconds
    );
```

@hHandle 要等待的内核对象。
@dwMilliseconds 最长等待的时间，以毫秒为单位，如传入5000就表示5秒，传入0就立即返回，传入INFINITE表示无限等待。
因为线程的句柄在线程运行时是未触发的，线程结束运行，句柄处于触发状态。所以可以用WaitForSingleObject()来等待一个线程结束运行。

@return：在指定的时间内对象被触发，函数返回WAIT_OBJECT_0。超过最长等待时间对象仍未被触发返回WAIT_TIMEOUT。传入参数有错误将返回WAIT_FAILED




**@ Interlock**
增减操作

```
LONG __cdecl InterlockedIncrement(LONG volatile* Addend);
LONG __cdecl InterlockedDecrement(LONG volatile* Addend);
```

返回变量执行增减操作之后的值。返回运算后的值，注意！加个负数就是减。
```
LONG __cdec InterlockedExchangeAdd(LONG volatile* Addend, LONGValue);
```

赋值操作

```
LONG __cdecl InterlockedExchange(LONG volatile* Target, LONGValue);
```
Value就是新值，函数会返回原先的值。



**@ CriticalSection**
线程退出时不会自动解消，不能跨进程使用
进入和离开可以在不同线程使用，可以从其它线程解消掉占用（貌似不推荐）
可递归

void InitializeCriticalSection(LPCRITICAL_SECTION lpCriticalSection);
定义临界区变量后必须先初始化
void DeleteCriticalSection(LPCRITICAL_SECTION lpCriticalSection);
用完之后记得销毁
void EnterCriticalSection(LPCRITICAL_SECTIONlpCriticalSection);
系统保证各线程互斥的进入临界区域。
void LeaveCriticalSection(LPCRITICAL_SECTIONlpCriticalSection);
离开关临界区域

临界区有自旋锁的功能，适当使用可以提高性能
BOOL InitializeCriticalSectionAndSpinCount
  LPCRITICAL_SECTION lpCriticalSection,
  DWORD dwSpinCount);
函数说明：旋转次数一般设置为4000。
DWORD SetCriticalSectionSpinCount(
  LPCRITICAL_SECTION lpCriticalSection,
  DWORD dwSpinCount);


**@ Event**
触发(set状态)表示允许运行
事件也不会自动解消，因为逻辑问题不需要
CreateEvent
函数功能：创建事件
HANDLE CreateEvent(
 LPSECURITY_ATTRIBUTES lpEventAttributes,
 BOOL bManualReset,
 BOOL bInitialState,
 LPCTSTR lpName
);

函数说明：
第一个参数表示安全控制，一般直接传入NULL。
第二个参数确定事件是手动置位还是自动置位，传入TRUE表示手动置位，传入FALSE表示自动置位。自动回复成未触发
第三个参数表示事件的初始状态，传入**TRUE表示已触发**。
第四个参数表示事件的名称，传入NULL表示匿名事件。

OpenEvent
函数功能：根据名称获得一个事件句柄。
HANDLE OpenEvent(
 DWORD dwDesiredAccess,
 BOOL bInheritHandle,
 LPCTSTRlpName     //名称
);

函数说明：
第一个参数表示访问权限，对事件一般传入EVENT_ALL_ACCESS。详细解释可以查看MSDN文档。
第二个参数表示事件句柄继承性，一般传入TRUE即可。
第三个参数表示名称，不同进程中的各线程可以通过名称来确保它们访问同一个事件。

SetEvent
函数功能：触发事件、Signals the event.
BOOL SetEvent(HANDLE hEvent);
函数说明：每次触发后，必有一个或多个处于等待状态下的线程变成可调度状态。

ResetEvent
函数功能：将事件设为末触发
BOOL ResetEvent(HANDLE Event);

由于事件是内核对象，因此使用CloseHandle()就可以完成清理与销毁了。




**@ WinApi Mutex**
可递归，windows互斥量在线程退出时会解消，保证跨进程使用安全
不能在其它线程通过调用unlock解消掉mutex的占用



CreateMutex
函数功能：创建互斥量（注意与事件Event的创建函数对比）
函数原型：
HANDLE CreateMutex(
  LPSECURITY_ATTRIBUTES lpMutexAttributes,
  BOOL bInitialOwner,     
  LPCTSTR lpName
);

函数说明：
第一个参数表示安全控制，一般直接传入NULL。
第二个参数用来确定互斥量的初始拥有者。是否创建时触发
第三个参数用来设置互斥量的名称，在多个进程中的线程就是通过名称来确保它们访问的是同一个互斥量。
@return：
成功返回一个表示互斥量的句柄，失败返回NULL。

HANDLE OpenMutex(
 DWORD dwDesiredAccess,
 BOOL bInheritHandle,
 LPCTSTR lpName     //名称
);

函数说明：
第一个参数表示访问权限，对互斥量一般传入MUTEX_ALL_ACCESS。详细解释可以查看MSDN文档。
第二个参数表示互斥量句柄继承性，一般传入TRUE即可。
第三个参数表示名称。某一个进程中的线程创建互斥量后，其它进程中的线程就可以通过这个函数来找到这个互斥量。
@return：
成功返回一个表示互斥量的句柄，失败返回NULL。

BOOL ReleaseMutex (HANDLEhMutex)
函数说明：
访问互斥资源前应该要调用等待函数，结束访问时就要调用ReleaseMutex()来表示自己已经结束访问，其它线程可以开始访问了。



**@ Semaphore**
信号量不会自动解消
CreateSemaphore
函数功能：创建信号量
函数原型：
HANDLE CreateSemaphore(
  LPSECURITY_ATTRIBUTES lpSemaphoreAttributes,
  LONG lInitialCount,
  LONG lMaximumCount,
  LPCTSTR lpName
);
第一个参数表示安全控制，一般直接传入NULL。
第二个参数表示初始资源数量。
第三个参数表示最大并发数量。
第四个参数表示信号量的名称，传入NULL表示匿名信号量。

OpenSemaphore
打开信号量
函数原型：
HANDLE OpenSemaphore(
  DWORD dwDesiredAccess,
  BOOL bInheritHandle,
  LPCTSTR lpName
);

函数说明：
第一个参数表示访问权限，对一般传入SEMAPHORE_ALL_ACCESS。详细解释可以查看MSDN文档。
第二个参数表示信号量句柄继承性，一般传入TRUE即可。
第三个参数表示名称，不同进程中的各线程可以通过名称来确保它们访问同一个信号量。


ReleaseSemaphore
递增信号量的当前资源计数
BOOL ReleaseSemaphore(
  HANDLE hSemaphore,
  LONG lReleaseCount,  
  LPLONG lpPreviousCount 
);

函数说明：
第一个参数是信号量的句柄。
第二个参数表示增加个数，必须大于0且不超过最大资源数量。
第三个参数可以用来传出先前的资源计数，设为NULL表示不需要传出。



**@ SRWLock**
读写锁，有新的slim版本
InitializeSRWLock
函数功能：初始化读写锁
函数原型：VOID InitializeSRWLock(PSRWLOCK SRWLock);
函数说明：初始化（没有删除或销毁SRWLOCK的函数，系统会自动清理）

AcquireSRWLockExclusive
函数功能：写入者线程申请写资源。
函数原型：VOID AcquireSRWLockExclusive(PSRWLOCK SRWLock);

ReleaseSRWLockExclusive
函数功能：写入者线程写资源完毕，释放对资源的占用。
函数原型：VOID ReleaseSRWLockExclusive(PSRWLOCK SRWLock);

AcquireSRWLockShared
函数功能：读取者线程申请读资源。
函数原型：VOID AcquireSRWLockShared(PSRWLOCK SRWLock);

ReleaseSRWLockShared
函数功能：读取者线程结束读取资源，释放对资源的占用。
函数原型：VOID ReleaseSRWLockShared(PSRWLOCK SRWLock);

