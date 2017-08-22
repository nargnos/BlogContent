---
title: "[MMP] Boost线程进程操作相关类库"
date: 2017-08-19 12:13:00
categories: [学习笔记, MMP]  
tags: [MMP,Boost,C++]
---
这里主要写进程间通信和一些boost相关类库，boost的线程相关类库跟stl差不多，重复的部分不记了。<!--more-->关于协程类库以后再记（以前简单看过，现在忘记了）。  

_这个库内容比较多，所以看文档能明白的这里就少记，整体的内容类似于概览（就是告诉有这么个东西，具体去查文档就好了）。_   


# 无锁
lockfree
提供基本的结构如栈、队、循环队等，如果想用链表可以在detail里找到。  
`queue`、`stack` 支持多线程同时读写。  
`spsc_queue` 只支持单生产者单消费者同时处理。   
在pop和push时是不会阻塞的，只会返回操作是否成功。  
注意由于要进行原子操作，其析构和赋值操作必须是极简（default、非虚）的，其类型成员也必须符合这个规则，否则无法使用。    
_可以用指针作为类型，智能指针、function等都不能用做该容器类型。_    
例子在`boost/doc/html/lockfree/examples.html`（这是自带帮助文档的路径）  

# 线程
这里是其它一些扩展（比stl多的那些类）。  
有实用性的类并没有很多，基本stl都包括了。  

注意连接时需要`-lboost_system -lboost_thread`。  

可以使用`thread_attributes`设置调用栈的大小。  

`boost::scoped_thread`、`thread_guard`
一般thread在析构时如果可join就会断言提示，scoped_thread析构时判断是否可join，然后join。
thread_guard也是类似功能，不过需要传入thread

有3种预定模式可以选
`detach`
`join_if_joinable`
`interrupt_and_join_if_joinable`

还有一些join的函数`time_join\try_join`

`thread_group`
线程组，方便管理的东西，内部用list管理。  


`barrier`
运行栅栏，线程同步点。线程运行到这里时等待，直到有指定个线程都等在这就放行，然后重置。  
内部使用条件变量实现。  
gpu编程时会比较常见，一般用来等待数据就绪。  

`latch`
内部使用条件变量实现。需要在构造时传入等待数量。    
跟go的sync.WaitGroup一样的功能。

boost似乎对future相关的类加了一些函数，看官方文档把。  
其它有一些类，文档有写可能会有bug，这里就不记了。  

## 线程池
内部就是`vector<thread>`，一起等待一个同步队列。  

`executor_adaptor`可以把一些执行体统一接口（适配）。  
线程池不能给它sumit工作时间太长的函数，线程还是有限的。  
回调用work存储，这个结构用虚函数组织调用，模板接收不同回调类型（类型擦除，用虚函数调用）。  

```cpp
// 注意cout这里分了多步骤输出，在多线程时会显示错乱（我忘了从哪贴来的例子）
#include <iostream>
#include <boost/thread/thread_pool.hpp>
using namespace std;

int main()
{
	boost::basic_thread_pool p;
	p.submit([]()
	{
		cout << boost::this_thread::get_id() << ":1" << endl;
	});
	p.submit([]()
	{
		cout << boost::this_thread::get_id() << ":2" << endl;
	});
	p.join();
	return 0;
}
```

关于executor还有这样用（）
```cpp
boost::executor_adaptor < boost::executors::basic_thread_pool > ea(4);
boost::executor_adaptor < boost::executors::loop_executor > ea2;
boost::executor_adaptor < boost::executors::serial_executor > ea2(ea1);
boost::executor_adaptor < boost::executors::thread_executor > ea1;
```

这样future的async可以传入池（注意是boost的实现），直接传池也行（这是test里的用法，但是我测试不成功）。  
```cpp
boost::future<int> t1 = boost::async(ea, &f1);
```

# 进程
## 进程管理
```cpp
#include <boost/process.hpp>
```
child  
可以创建进程~~，貌似可以用running检查运行状态，但是测试时发现会有一些问题（运行起来了返回false）~~，可以用wait等待其它进程结束，如果提供的路径不存在会引发异常（可以传入`std::error_code`参数获取，也可以用`ignore_error`参数忽略掉）。  
可以设置一些事件回调什么的，这里略。  
可以在进程里用this_process取得一些相关信息（比如环境变量）。  

```cpp
#include <boost/process.hpp>

using namespace boost::process;

int main()
{
    ipstream pipe_stream;
    child c("gcc --version", std_out > pipe_stream);

    std::string line;

    while (pipe_stream && std::getline(pipe_stream, line) && !line.empty())
        std::cerr << line << std::endl;

    c.wait();
}
```
可以在创建子进程时将输出重定向到管道。  
可以用asio来处理管道的一些io（async_pipe）。  
示例见boost示例，略。  

## 进程间通信
除了上面提到的pipe，还有很多方式可以实现进程间通信。 
Windows一般可以用：剪贴板（Clipboard）、COM、数据拷贝（Data Copy）、消息、DDE、文件映射（File Mapping，共享内存）、邮槽（Mailslots）、管道（Pipes）、远程过程调用（RPC）、Socket等手段来实现（这里不把信号量作为通信手段，因为感觉更多的是同步而不是通信）。  
Linux有文件、管道、信号、共享内存、消息队列、Socket等。  

这里记的是boost的Interprocess库的内容，更多的是共享内存的东西。  
_关于常用的如pipe、socket的笔记不在这记录。系统调用也不在这记录。_  

### 共享内存
```cpp
#include <boost/interprocess/shared_memory_object.hpp>
```
linux连接时可能需要`-lrt`。  
Windows下会创建文件，然后用文件映射的方式共享内存（因为win的共享内存会自动销毁，使用文件映射才能使posix和windows共享内存的生命周期相似，保证可移植性）。  

创建共享内存段。如果已经创建了，会抛异常：
```cpp
shared_memory_object shm_obj  
   (create_only                  //only create  
   ,"shared_memory"              //name  
   ,read_write                   //read-write mode  
   );  
```

打开或创建一个共享内存段：  
```cpp
shared_memory_object shm_obj  
   (open_or_create               //open or create  
   ,"shared_memory"              //name  
   ,read_only                    //read-only mode  
   );  
```
仅打开一个共享内存段。如果不存在，会抛异常：
```cpp
shared_memory_object shm_obj  
   (open_only                    //only open  
   ,"shared_memory"              //name  
   ,read_write                   //read-write mode  
   );  
```
当共享内存对象创建后，默认大小为0，使用truncate设置（分配）大小，

因为共享内存具有内核或文件系统持久化性质，因此用户必须显式销毁它。如果共享内存不存在、文件被打开或文件仍旧被其他进程内存映射，则删除操作可能会失败且返回false：
`shared_memory_object::remove("shared_memory");`
（文档说unix下会调用shm_unlink）

直接取不到指针，需要用mapped_region映射(可以是整个共享内存或是一部分)到某内存位置读取。  
mapped_region可以设置读取权限（跟windows的map差不多）。  
在构造函数里可以传入映射的开始和结束位置，默认是整个段。  
`get_address` 和 `get_size` 可以获得信息。  

```cpp
//Create a shared memory object.
shared_memory_object shm (create_only, "MySharedMemory", read_write);

//Set size
shm.truncate(1000);

//Map the whole shared memory in this process
mapped_region region(shm, read_write);

//Get the address of the region
region.get_address();

//Get the size of the region
region.get_size();
```

可以用guard的方式管理共享内存的创建和销毁，比如：
```cpp
struct shm_remove
{
	shm_remove() { shared_memory_object::remove("MySharedMemory"); }
	~shm_remove(){ shared_memory_object::remove("MySharedMemory"); }
} remover;
```
#### Windows共享内存
`windows_shared_memory`
当没有进程引用共享内存对象时会自己销毁，所以如果创建端在其它进程连接前退出，其它进程连接就会出错。  
创建时必须传入大小，如果要在服务和用户程序间共享内存，名称必须添加前缀"Global\\\\"表示在全局命名空间创建。注意在Session 0之外的会话中，在全局命名空间创建是一种特权操作。  

使用方式跟之前的很像，就只是创建时需要这样：
```cpp
windows_shared_memory shm (create_only, "MySharedMemory", read_write, 1000);
```
如果使用open_only就不需要传入大小。  


#### Linux共享内存
**匿名共享内存**
unix内部用mmap，windows用的是没名字的`windows_shared_memory`对象。  
用 `anonymous_shared_memory` 可以返回一个 mapped_region。  
这样fork后就可以使用mapped_region对象管理。  

**System V 共享内存**
用`xsi_shared_memory`，用法差不多。  
使用`xsi_key`识别不同对象，必须显式删除，不支持写时复制但支持匿名共享内存。  

### 内存映射文件
可以将文件的一部分与进程地址空间相关联，而不需要完全载入整个文件（因为文件可能会大于整个进程的地址空间）。  
文件映射可以用于进程间通信，当进程写入时在另一个进程可以看到修改，但是因为系统必须将文件内容与存储器内容同步，所以不比共享内存快。  
创建步骤为：先创建（或选择）要映射的文件，然后将文件和进程地址空间关联。  
在关联时可以设置对应内存的权限（读写执行）。  

```cpp
#include <boost/interprocess/file_mapping.hpp>
file_mapping file(path,read_write);
mapped_region region(file, read_write);
```
可以设置映射范围，如果不填表示映射整个文件，如果没写大小，将会从偏移位置映射到文件尾。  
在使用flush更新时，填入的参数也用一样的方式表示范围。  
注意flush时操作的是映射的范围（表示的是映射范围的offset），对于自身`mapped_region`未映射的部分则不处理。  

用 file_mapping::remove(const char *) 删除。  

#### offset_ptr
不同程序中，文件的映射位置（返回的地址）不一定相同（可以在映射时指定映射地址，但不一定会成功，注意指定地址时，偏移量需要为页大小的整数倍）。
这会使这部分内存内的指针失效，但是相互之间的偏移量不变，这时候就需要用偏移指针来处理数据。  
也是由于这个原因，不要在这部分内存中使用引用（在其它进程会失效），也不要用虚函数，因为虚表里有指针。  
另外需要注意类静态成员的使用，因为在其它进程里可能会修改它。  

```cpp
#include <boost/interprocess/offset_ptr.hpp>
```

只要是同一个共享块，即使在不同映射地址或不同进程都可以正确使用。  
注意，这里将偏移量1定义为空指针（原因见文档），所以不能指向自身偏移1字节的地址。  

其它一些内容在`boost/doc/html/interprocess/interprocess_smart_ptr.html`  

### 同步、通信机制
如果只能共享内存，但是多进程之间不同步，这个共享是没有意义的。  
boost提供以下一些同步机制。  
基本上跟线程用的差不多，分成匿名和命名的，命名要在在构造时传入对象名，匿名的需要用一些方式让多进程共享（比如通过共享内存传递）。

#### Mutex
```cpp
// 这个对象是可以通过共享内存传递的（因为用自旋锁）
#include <boost/interprocess/sync/interprocess_mutex.hpp>
#include <boost/interprocess/sync/interprocess_recursive_mutex.hpp>
// 这个在windows下内部似乎用的是文件的形式
#include <boost/interprocess/sync/named_mutex.hpp>
#include <boost/interprocess/sync/named_recursive_mutex.hpp>
// 相关工具
#include <boost/interprocess/sync/scoped_lock.hpp>
#include <boost/interprocess/sync/interprocess_sharable_mutex.hpp>
#include <boost/interprocess/sync/named_sharable_mutex.hpp>
#include <boost/interprocess/sync/interprocess_upgradable_mutex.hpp>
#include <boost/interprocess/sync/sharable_lock.hpp>
#include <boost/interprocess/sync/upgradable_lock.hpp>
```


#### Condition Variable
```cpp
#include <boost/interprocess/sync/interprocess_condition.hpp>
#include <boost/interprocess/sync/interprocess_condition_any.hpp>
#include <boost/interprocess/sync/named_condition.hpp>
#include <boost/interprocess/sync/named_condition_any.hpp>
```
同样也是分匿名命名，没什么特别，因为有官方文档，内容就略。  
注意这些类型用的锁对应前面的匿名命名互斥量。  

#### Semaphore
```cpp
#include <boost/interprocess/sync/interprocess_semaphore.hpp>
#include <boost/interprocess/sync/named_semaphore.hpp>
```

#### File Lock
有两种方式：系统维护锁定列表，进程可以获取，但是也可以忽略掉这个锁定从而修改文件，需要进程之间合作来实现同步访问；系统强制检查读写请求，验证执行操作。
由于可移植性原因，这里使用第一种方式实现。  
```cpp
#include <boost/interprocess/sync/file_lock.hpp>
```

#### Message Queue
消息队列。线程可以添加删除消息，消息具有优先级。  
线程有3种方式使用：
* 阻塞：如果发送时满或接收时空，线程将会阻塞到队列可用。  
* 尝试：如果发生前面阻塞的情况时，会返回错误（并允许继续执行其它代码）。  
* 定时：为阻塞设置超时时间。  

消息队列只在进程间复制原始字节，这表示如果要发送对象，这个对象必须是可序列化的。  

```cpp
#include <boost/interprocess/ipc/message_queue.hpp>
using boost::interprocess;
//Create a message_queue. If the queue
//exists throws an exception
message_queue mq
   (create_only         //only create
   ,"message_queue"     //name
   ,100                 //max message number
   ,100                 //max message size
   );

//Creates or opens a message_queue. If the queue
//does not exist creates it, otherwise opens it.
//Message number and size are ignored if the queue
//is opened
message_queue mq
   (open_or_create      //open or create
   ,"message_queue"     //name
   ,100                 //max message number
   ,100                 //max message size
   );   

//Opens a message_queue. If the queue
//does not exist throws an exception.
message_queue mq
   (open_only           //only open
   ,"message_queue"     //name
   );
```
创建时需要设置最大消息数和消息最大大小。  
使用send发送时需要附带消息大小和优先级信息。  
使用receive接收时需要传入buffer，接收到的是原始字节，需要反序列化。  
可用message_queue::remove删除消息队列对象。  

### 托管共享内存段
可以把管理共享内存的一些操作交由类库来做，可以实现动态分配内存、将对象与名称关联等。  
一般用`managed_shared_memory\wmanaged_shared_memory`，它们用的是偏移指针，如果想用固定指针时，就用`fixed_managed_shared_memory\wfixed_managed_shared_memory`。  

#### Managed Shared Memory 
```cpp
#include <boost/interprocess/managed_shared_memory.hpp>
```
创建方式同前面的其它对象的创建。销毁用`shared_memory_object::remove`。  

**通用操作**
这些操作在下面其它的类型中也是通用的。
基本上都是一些内存池会有的操作。  

`get_address`  
获得基础内存块的地址
`get_free_memory`  
可以获取剩余大小  
`allocate`  
分配内存，判断是否allocate成功可以用这个函数判断前后大小来获得
`allocate_aligned`  
分配对齐内存  
`allocate_many`  
一次分配一连串的内存  
`deallocate`  
回收  
`get_handle_from_address`  
可以获得分配的地址的handle，然后传给其它程序使用

`remove_shared_memory_on_destroy name("")`
可以在析构时销毁共享块
`grow`
可以增长大小，非线程安全，这是在当没有其它拥有相同资源的进程存在时调用的
`shrink_to_fit`
非线程安全

`construct<Type>`
可以在共享块中创建对象，对象数可以省掉
`construct<Type>("对象名")[对象数](对象构造参数)`
还可以传入数组作为构建对象数组时的参数
`construct<Type>("对象名")[对象数](&构造参数1数组,&构造参数2数组)`

之后查找对象时使用
`find<Type>(name)`
返回类型为
`std::pair<Type*, managed_shared_memory::size_type>`

可以用`find_or_construct`获得或创建对象
用`destroy<Type>(name)` 销毁

当执行某函数时不希望在函数执行期间内存被其它进程修改（指的是创建销毁对象），可以使用`atomic_func`，注意这个函数仅阻止其他进程创建、搜索和销毁对象。  

`allocation_command`
可以用来调整缓冲区，实现前后扩展收缩。  

可以使用`open_copy_on_write`做构造参数，这样便具有写时复制特性。  


如果希望用windows的共享内存机制，就用
```cpp
#include <boost/interprocess/managed_windows_shared_memory.hpp>
```
Xsi（System V）可用
```cpp
#include <boost/interprocess/managed_xsi_shared_memory.hpp>
```

#### Managed Mapped File
可以用`managed_mapped_file`来管理文件映射。  
```cpp
#include <boost/interprocess/managed_mapped_file.hpp>
```
可以创建一个文件并映射，提供的函数跟`managed_shared_memory`差不多，可以指定路径，但是似乎会检查文件格式是否符合自身定义的格式，如果不符合会抛异常。  
使用`file_mapping::remove`销毁，也可以用删除文件的方式删除。  

#### Managed External Buffer
```cpp
#include <boost/interprocess/managed_external_buffer.hpp>
```
提供了跟前面相同的接口，可以对自己定义的内存位置使用，创建了相关对象后，可以直接通过网络或其它方式传递给其它进程，因为对象结构相同，其它进程可直接使用，省掉了序列化反序列化的操作。  
比如可以定义成allocator，这样就可以传送比如vector这样的对象给其它进程。  

#### Managed Heap Memory
类似上面，只是内存由类帮忙管理。
```cpp
#include <boost/interprocess/managed_heap_memory.hpp>
```
## 衍生结构
基于托管共享内存段衍生的一些结构。  

### 分配器
不要交换两个具有不同分配器（包括不同分配目标）的容器，其行为是不确定的，但是错误是肯定的。（原因见文档）  
```cpp
#include <boost/interprocess/allocators/allocator.hpp>
```
这里的allocator是适配器。  

在分配小内存或者连续对象时，会有不同的分配策略，所以类库提供`node_allocator/ private_node_allocator/cached_node_allocator`分配方式。  

**node_allocator**
```cpp
#include <boost/interprocess/allocators/node_allocator.hpp>
```
初始化时它会在段中搜索自身是否存在，不存在就创建一个，保证单例，总节点共享。  
线程安全。
具体看文档，这里略。  

**private_node_allocator**
```cpp
#include <boost/interprocess/allocators/private_node_allocator.hpp>
```
这个跟上面一样，但是每个对象有自己的隔离存储池，这样在销毁时会比较快。  
非线程安全。  

**cached_node_allocator**
```cpp
#include <boost/interprocess/allocators/cached_node_allocator.hpp>
```

从相同的段分配节点，但是会私下缓存其中一些，这样以后分配就没有同步的开销，当缓存满时会返还一些给池。  
分配和释放不是线程安全的。（文档是这么写的，我前面的理解可能有误，具体看文档。）  


例子：
```cpp
typedef allocator<int, managed_shared_memory::segment_manager>  ShmemAllocator;
```

### 池
类似分配器，也分成了几种不同策略的池。它们可以在销毁对象时回收内存。  
```cpp
#include <boost/interprocess/allocators/adaptive_pool.hpp>
#include <boost/interprocess/allocators/private_adaptive_pool.hpp>
#include <boost/interprocess/allocators/cached_adaptive_pool.hpp>
```

### 容器和其它结构
其实就是重新using了标准容器，分配器是需要自己传入的。  
直接用stl容器可能也一样（没试过）。  
```cpp
#include <boost/interprocess/containers/vector.hpp>
#include <boost/interprocess/containers/deque.hpp>
#include <boost/interprocess/containers/list.hpp>
#include <boost/interprocess/containers/slist.hpp>
#include <boost/interprocess/containers/set.hpp>
#include <boost/interprocess/containers/map.hpp>
#include <boost/interprocess/containers/flat_set.hpp>
#include <boost/interprocess/containers/flat_map.hpp>
#include <boost/interprocess/containers/string.hpp>
// ... 其它见文档
```
使用方法
```cpp
#include <boost/interprocess/containers/vector.hpp>
#include <boost/interprocess/allocators/allocator.hpp>
#include <boost/interprocess/managed_shared_memory.hpp>

int main ()
{
   using namespace boost::interprocess;
   //Remove shared memory on construction and destruction
   struct shm_remove
   {
      shm_remove() { shared_memory_object::remove("MySharedMemory"); }
      ~shm_remove(){ shared_memory_object::remove("MySharedMemory"); }
   } remover;

   //A managed shared memory where we can construct objects
   //associated with a c-string
   managed_shared_memory segment(create_only,
                                 "MySharedMemory",  //segment name
                                 65536);

   //Alias an STL-like allocator of ints that allocates ints from the segment
   typedef allocator<int, managed_shared_memory::segment_manager>
      ShmemAllocator;

   //Alias a vector that uses the previous STL-like allocator
   typedef vector<int, ShmemAllocator> MyVector;

   int initVal[]        = {0, 1, 2, 3, 4, 5, 6 };
   const int *begVal    = initVal;
   const int *endVal    = initVal + sizeof(initVal)/sizeof(initVal[0]);

   //Initialize the STL-like allocator
   const ShmemAllocator alloc_inst (segment.get_segment_manager());
   // 这样在扩展大小的时候就可以使用设置的allocator分配元素，
   // 这个分配器初始化时需要填入共享块的manager（get_segment_manager获得）
   // 然后再用共享块创建vector对象,需要传入alloc对象,其它进程用find获取对象即可

   //Construct the vector in the shared memory segment with the STL-like allocator
   //from a range of iterators
   MyVector *myvector =
      segment.construct<MyVector>
         ("MyVector")/*object name*/
         (begVal     /*first ctor parameter*/,
         endVal     /*second ctor parameter*/,
         alloc_inst /*third ctor parameter*/);

   //Use vector as your want
   std::sort(myvector->rbegin(), myvector->rend());
   // . . .
   //When done, destroy and delete vector from the segment
   segment.destroy<MyVector>("MyVector");
   return 0;
}
```

## 其它
有一些关于内存分配算法的内容和一些自定义相关类库的内容见帮助文档
