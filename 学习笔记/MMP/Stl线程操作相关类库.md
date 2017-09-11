---
title: "[MMP] Stl线程操作相关类库"
date: 2017-08-13 17:09:45
categories: [学习笔记, MMP]  
tags: [MMP,STL,C++]
---
stl涉及到多线程的一些内容。<!--more-->这部分有一些内容（atomic）在前面已经提到，重复的这里就不会再记录。  

_花了些时间想怎么整理这部分内容，因为不想弄得跟类库文档一样。所以这里记用法而不记类库介绍。因为这里不是教程。_   
_boost的相关类库跟stl的差不多，比stl多的那些等以后再整理，因为记到了其它文档里等整理到了再单独拿出来。_

这里讨论的是vs中vc带的stl。
因为相关的类库调用到很多编译器内部函数（intrin.h），所以很多都不能直接分析源码，就只记用法吧。  
atomic的一些内容在之前的文档里记了，这里就省略。  

# 线程
~~不用的函数：~~
~~_beginthreadex~~
~~因为C运行库之前不支持多线程，很多函数都往全局变量写，所以使用这个函数可以让这部分数据线程独享，这样就不会出问题了，这个函数以前还有，现在编译环境貌似没了。
现在用的是 std::thread~~

`<thread>`
thread 初始化时便建立线程并执行。在析构时，如果线程未结束，将会引发异常；可用detach将线程和对象分离。
get_id 获取tid。  
native_handle 获取线程句柄，可用来设置关联性。  
thread::hardware_concurrency 获取核心数。

在运行过程中可在线程内部用this_thread管理（sleep、yield）或获得一些相关信息（取tid）。  

用法没什么特别的：
```cpp
using namespace std;
thread t([]() {
	this_thread::yield();
});
t.join();
thread([](int) {
	this_thread::sleep_for(chrono::seconds(1));
}, 123).detach();
thread([](int) {
	this_thread::sleep_until(chrono::system_clock::now() + chrono::seconds(1));
}, 123).join();
```


# 原子操作
`<atomic>`
**atomic**
提供了C、C++两种风格的实现方式，这里用C++的方式。  


```cpp
// 线程安全替代
// if (!flag)
// {
// 	// ...
// 	flag = true;
// }
if (!flag.test_and_set())
{
	// ...
}
```
`test_and_set` 当前是true就返回true，false返回false，并设置当前值为true。  
需要用`ATOMIC_FLAG_INIT`设置初始值，包括flag在内，atomic都需要设置初值，否则就是未初始化类型。

其它情况可以用atomic类，它对很多类型都有特化（包括指针），所以不必担心会出问题。  
load、store、内存序什么的其它文档已记录，这里略。  

exchange和store都可以用原子操作的形式设定值，但是**exchange会返回旧值**。
所以前面的例子可以这样：
```cpp
atomic<bool> flag(false);
if (!flag.exchange(true))
{
	// ...
}
```

cas比较并交换
`compare_exchange_strong`、`compare_exchange_weak` 
```cpp
atomic<int> num(0);
int inOut = 123;
int newVal = 456;
bool ret = num.compare_exchange_strong(inOut, newVal, orderSucc, orderFail);
```
compare_exchange可视为（不过这是原子操作）：
```cpp
if (num == inOut)
{
	num = newVal;
	return true;
}
inOut = num;
return false;
```
当相等时会用orderSucc的排序方式，否则用orderFail。
简单来说就是，返回inOut==num，设置inOut为num旧值，当(inOut==num)==true时，num=newVal。

关于strong、weak，在x86/64 msvc环境**执行路径完全相同**（用的也都是lock cmpxchg）。

>在循环操作中用compare_exchange_weak性能更好，它允许伪失败（比较相同仍返回false）
>对于compare_exchange_weak()函数，当原始值与预期值一致时，存储也可能会不成功。这可能发生在缺少独立“比较-交换”指令的机器上，当处理器不能保证这个操作能够自动的完成——可能是因为线程的操作将指令队列从中间关闭，并且另一个线程安排的指令将会被操作系统所替换(这里线程数多于处理器数量)。这被称为“伪失败”(spurious failure)，因为造成这种情况的原因是时间，而不是变量值。

atomic对整型做了一些特化，可以支持一些算术操作，并允许用户自定义类型，但是貌似类型必须是POD。

# 互斥量
## Mutex
`<mutex>`
**std::mutex**
提供了基本的互斥机制。之前内部好像使用criticalSection实现（vc带的stl，windows），但是现在好像用的是SRWLock，因为锁函数调用时会用到API RtlAcquireSRWLockExclusive。

未lock时unlock会引发错误，不能在其它线程unlock解消占用。
不可递归，不能跨进程使用。
线程退出不会自动unlock。（注意：跟winapi不同，不可递归，线程退出不解消）
unlock后立即lock，其它等待的线程有极大概率不能获取执行机会。

**recursive_mutex**  
递归锁。
线程退出不会清除。
_没有用内核对象的东西，好像是自己弄了一个变量存储递归情况。_
释放互斥量时需要调用与该锁层次深度相同次数的 unlock()。

**timed_mutex**
超时等待锁，超时会返回false。  

**recursive_timed_mutex**	
提供了可被同一线程递归锁定且使用有超时限制的锁的互斥机制

## SharedMutex
`<shared_mutex>`
共享锁。
**shared_mutex**
C++14可用，提供了共享互斥机制。
`lock`和`lock_shared`互斥，`lock_shared`之间可以共存。类似读写锁。

**shared_timed_mutex**
带超时机制的共享锁。

# 锁操作、算法
## Guard
`<mutex>`
**lock_guard**
实现了严格的基于范围（Scope-based）的互斥所有权（Mutex ownership）的封装。
raii，创建时调用lock，析构时调用unlock。不提供成员函数（依靠ctor、dtor管理）。
具有同时管理多个mtx的能力（锁定时也可以直接用std::lock函数）。

使用例子（锁类型可替换，下同）：
```cpp
mutex mtx;
recursive_mutex r_mtx;
// 也可以只管理一个
lock_guard<mutex, recursive_mutex> guard(mtx, r_mtx);
```


**unique_lock**
实现了可移动（Movable）的互斥所有权的封装。
限制锁的所有权（一次只能管理一个锁）。提供一系列的成员函数。

```cpp
timed_mutex t_mtx;
unique_lock<timed_mutex> ul(t_mtx);
```

**shared_lock** C++14
`<shared_mutex>`
实现了可移动的共享互斥所有权的封装。
在创建时会调用`lock_shared`，析构时调用`unlock_shared`。
只能接受共享锁类型。

```cpp
shared_mutex s_mtx;
shared_lock<shared_mutex> sl(s_mtx);
```


## 参数
用来设置锁的策略，用tag的形式设置。

**adopt_lock**
创建时不锁，并表明之前已经锁过了。  
```cpp
mutex mtx;
mtx.lock();
lock_guard<mutex> guard(mtx, adopt_lock);
```

**defer_lock**
lock_guard不能用。
unique_lock中创建时不锁，并且之前也没锁过。

**try_to_lock**
lock_guard不能用。
unique_lock中创建时试锁，如果成功会在析构时解锁，不成功就不处理。  

## 常用锁算法
try_lock: 尝试获得互斥锁的所有权
lock: 加锁指定的互斥锁，如果不能则阻塞（Block）
std::lock: 可以一次锁住多个对象，不会有副作用（如死锁什么的）

**单次调用**
once_flag: 用于确保 `call_once` 仅请求调用指定函数一次的辅助对象
call_once: 仅请求调用指定函数**一次**，即使被多条线程调用，注意这个函数是阻塞的，会保证执行（或曾经执行）过才放行

# 条件变量

`<condition_variable>`
`condition_variable`在wait时只支持`unique_lock<mutex>`，用`condition_variable_any`可支持更多参数。  
当wait时线程会释放锁并挂起（这个挂起好像并不是等待传入的锁而引起的挂起），唤醒时获取锁，并检查条件，符合将会继续运行（或者未设置条件时），否则继续阻塞。  
在wait之前，线程必须获取要等待的锁，否则无法唤醒。  
notify会唤醒（一个或所有）等待的线程，如果调用时没有线程再等待，则忽略（先notify再wait是无效的）掉。  
如果有多个线程等待，唤醒一个时不知道是否会惊群，没看到有相关记录。
有超时机制。

可以设置条件，实际执行的是下面的代码（唤醒时检查，不符合就继续阻塞）：
注意在wait调用时也会检查一次，为true就不进入阻塞步骤。  
```cpp
// 在wait时设置的条件返回true时继续执行，false阻塞
while(!Pred())
    wait(Lck);
```
可以这个用来实现信号量。  
注意检查和等待不是原子的，在调用notify唤醒线程后再设置条件为true，此时会有一定几率通过验证，这是调用速度决定的。（所以最好先设置条件再唤醒）  
多个条件变量可以共享一个互斥量。  
在Windows下，wait用的是SleepConditionVariableSRW，不用SleepConditionVariableCS，所以mutex的实现现在用的是SRWLock。



# 异步操作
_其实stl的异步操作类库并没有ppl的好用，但是在其它环境好像没有ppl。_
## future
`<future>`

**future**
可用get阻塞线程，当future有值时就会唤醒阻塞的线程，也可以用wait，还提供了超时机制可用。  
get只能取值一次，用get取值多次会发生错误；如果要取多次就用share获取`share_future`对象，这个对象也可以用wait、get等函数阻塞线程。  
一般这个类型不直接创建（因为是无效的），需要由其它Providers(提供者)创建。  
在取值前可以用valid()来判断是否有效（有没有取过值）。

注意：这个类型是可以传递任何类型的，也就是说可以传递容器、函数甚至future自身。

**promise**  (提供者)
保存一个值用于异步获取，表示将来可以用它取到值。取值前需要先生成future对象，给它赋值后，从之前用它生成的对象可以取到值。  
```cpp
promise<int> pm;
auto fu = pm.get_future();
// 给其它线程设置值
thread t([pm_ = move(pm)]() mutable {pm_.set_value(123); });
cout << fu.get() << endl;
```

可以用`set_value_at_thread_exit`设置在promise对象析构时给future设置指定值（但是此时便不能在中途设置值）。  
时未设置值，会从future处引发异常，也可以用`set_exception`、`set_exception_at_thread_exit`直接引发异常。  

**packaged_task**  (提供者)  
打包一个函数来存储它的返回值，以用于异步获取。  
类似promise，但是它需要传入函数，在调用`packaged_task`时会把函数的返回值传递给future。  
```cpp
packaged_task<int(int)> task([](int val) {return val*val; });
auto fu = task.get_future();
// 可以给线程池执行
thread(move(task), 10).detach();
cout << fu.get() << endl;
```
因为这是一个伪函数，稍修改便可以作为task用来提交给线程池使用。  

**async**  (函数提供者)  
异步运行（可能会运行在一个新的线程中）一个函数，且返回一个将会保存结果的 future 对象。  
可设置调用参数deferred，这会在获取值时调用函数，否则在调用async时会创建线程调用函数。  

# 最后
可能有漏记的，发现了再补充。  