---
title: "[MMP] AMP"
date: 2017-09-12 14:07:45
categories: [学习笔记, MMP]  
tags: MMP
---
# 概述
C++ Accelerated Massive Parallelism（加速大规模并行计算）。  
可用GPU加速代码的运行。  
Win7以上，Dx11以上支持。

# 使用
头文件为amp.h。  
基本步骤是，构造数据并创建数据映射对象，定义算法，给GPU执行。  

如果需要调试kernel函数，在vs的项目属性的调试-调试器类型处选择仅gpu，cpu和gpu不能同时调试，所以需要这样设置。  
可以在gpu线程窗口查看线程，并行堆栈窗口查看栈帧，其它调试方式跟cpu的差不多。
另外需要注意的是，当不需要带符号整数时，优先用无符号整数，因为无符号整数的模和除法性能比带符号的高。  

构造数据后放入array或array_view。它们几乎具有相同的成员，但基本行为不同。  
array是容器类，在构造时深拷贝数据，array_view是包装类，在kernel访问数据时才复制数据。  

array构造时需要传入迭代器，理论上应该支持所有容器。  
array_view可以用数组、std::array、std::vector存储数据（可以是有data和size成员函数的容器，string应该也行）。  


使用array时，kernel函数修改的是对应的副本，在kernel函数执行完毕后，必须将数据复制回cpu线程上的对象中，默认不可通过它直接访问数据。  
在使用array_view时，构造函数并不会把数据复制到gpu上，如果两个view同时引用同一个数据，则它们引用的是同样的内存空间。如果这样做，则必须同步所有多线程的访问。
使用 array_view 类的主要优点是数据仅在必要的时候移动。


它们之间的异同：

描述         |    array     |        array_view
------------ | ------------ | ------------------------
确定维度     | 编译时       | 编译时
确定大小     | 运行时       | 运行时
形状         | 矩形         | 矩形
数据存储方式 | 数据容器     | 数据包装器
复制方式     | 定义时深拷贝 | kernel函数访问时隐式拷贝
数据检索方式 | 将数组数据复制回cpu线程上的对象 | 直接访问对象或调用synchronize后访问原始容器上的数据


array_view定义为（array类似，略）：
`array_view<类型, 维度> sample(维度大小..., 数据容器);`
类型填const就可以用常量容器传入了。

注意构造参数里是从高维填到低维，如：
```
(z,y,x)
sample[z][y][x]
sample(z,y,x)
```

在进入并行前可以使用**discard_data**, 表示不要把对象数据复制到gpu。  

用parallel\_for\_each函数将数据放入gpu处理，它需要两个参数：计算域和lambda表达式。  
计算域是extent或tiled_extent对象，计算时会为域中每个元素生成一个线程。
lambda表达式是在每个线程上运行的代码，可以用`=`来捕获相关一些变量。表达式需要声明为restrict(amp)或restrict(amp, cpu)，表示会放到gpu执行，在表达式中可以调用其它amp函数。

表达式参数是一个index对象，表示的是数据的位置，它可以用来表示多维下标，其维度与下标的关系为:
定义时：(z,y,x)
使用时：`index<3> idx -> [0]:z [1]:y [2]:x`
extent也类似。

在定义或使用array\_view时，维度不需要跟原始数据相同，可自定义维度。  
parallel\_for\_each在返回时，kernel的运行可以没结束，所以它是异步的
但是通过array\_view访问结果或用synchronize时，会等到运行结束。
可以用array\_view的synchronize\_async获得future对象，然后可以用then来实现任务拼接（配合ppl使用）。

例：
```cpp
#include <algorithm>
#include <numeric>
#include <iostream>  
#include <array>

// 主要头文件
#include <amp.h>
// 库
// #include <amp_math.h>
// #include <amp_graphics.h>
// #include <amp_short_vectors.h>

void AmpTest()
{
	using namespace std;

	using DataType = int32_t;
	constexpr int MaxSize = 1024;

	// 构造数据
	array<DataType, MaxSize> data;
	iota(data.begin(), data.end(), 1);

	// 假设算法是这样(不做优化)，对每个元素执行，最后将所有结果求和
	auto algo = [](DataType val)restrict(amp, cpu)
	{
		for (DataType i = 0; i < 1000000; i++)
		{
			val += i;
		}
		return val;
	};

	// cpu
	auto cpuTime = clock();
	DataType cpuRet = std::accumulate(data.begin(), data.end(), 0,
		[=](DataType ret, DataType val) {
		return ret + algo(val);
	});

	cpuTime = clock() - cpuTime;
	cout << "CPU 计算用时： " << cpuTime << " 结果：" << cpuRet << std::endl;


	// gpu
	// 映射数据
	concurrency::array_view<const DataType, 1> dataView(data);

	DataType gpuRet = 0;
	concurrency::array_view<DataType, 1> gpuRetView(1, &gpuRet);

	auto gpuTime = clock();
	// 这个操作是异步的
	concurrency::parallel_for_each(dataView.extent,
		[=](concurrency::index<1> idx) restrict(amp)
	{
		concurrency::atomic_fetch_add(&gpuRetView[0], algo(dataView[idx]));
	});
	gpuRetView.synchronize();

	gpuTime = clock() - gpuTime;
	cout << "GPU 计算用时： " << gpuTime << " 结果：" << gpuRet << std::endl;

	// 验证
	cout << (gpuRet == cpuRet ? "结果相同" : "结果不同") << endl;
}

int main()
{
	AmpTest();
	return 0;
}
```

# parallel\_for\_each 函数
在设备上启动并行计算。它是异步执行的。  
可以传递accelerator\_view参数，让代码在指定的accelerator上运行，如果有array绑定的是不同的accelerator，就会引发异常。
其回调可以传入函数和仿函数，传入仿函数对象时，重写的operator()需要具有restrict(amp)修饰。  

kernel函数必须返回void，由于它没有其它参数，所以其它数据必须在函数对象或lambda中捕获，array对象必须用引用或指针捕获，其它必须用值捕获。

注意传入的extent的维度必须和kernel函数的index参数维度相同。

## restrict关键字
可将限制说明符应用于函数和 lambda 声明。
可使用 restrict(cpu)、restrict(amp)、restrict(cpu, amp) 或 restrict(amp, cpu) 声明的函数调用。 
restrict(A) restrict(B) 形式可以编写为 restrict(A,B)。 

包含 restrict(amp) 子句的函数具有以下限制： 
* 函数只能调用具有 restrict(amp) 子句的函数。 
* 函数必须可内联。 
* 函数只能声明 int、unsigned int、float 和 double 变量，以及只包含这些类型的类和结构。
    也允许使用 bool，但如果您在复合类型中使用它，则它必须是 4 字节对齐的。 
* Lambda 函数无法通过引用捕获，并且无法捕获指针。**但是有两个例外：concurrency::array、concurrency::graphics::texture**，这些对象要按引用捕获。 
* 仅支持引用和单一间接指针作为局部变量、函数参数和返回类型。 
* 不允许使用以下项： 
    * 递归
    * 使用 volatile 关键字声明的变量
    * 虚函数
    * 指向函数的指针
    * 指向成员函数的指针
    * 结构中的指针
    * 指向指针的指针
    * goto 语句
    * Labeled 语句
    * try、catch 或 throw 语句（不允许抛出异常）
    * 全局变量
    * 静态变量。 请改用 tile_static 关键字
    * dynamic_cast 强制转换
    * typeid 运算符
    * asm 声明
    * Varargs 用省略号的声明的变参


# 加速器（Accelerator）
可以使用加速器来指定运行amp代码的设备或仿真器。  
加速器是针对数据并行而优化的硬件功能，它可以连接到PCIe总线(比如GPU)的设备上，或者它可以是主CPU上的扩展指令集。

提供这些参数用于构造：
default_accelerator：默认
cpu_accelerator：用来设置临时数组（高级优化），不能在上面执行amp代码
direct3d_ref：使用cpu软件模拟
direct3d_warp：使用sse在cpu上模拟


一个系统可能有多个设备或仿真器，它可能有不同的内存量和不同的共享内存、调试、双精度支持性。
concurrency::accelerator::get_all()得到所有设备或仿真器列表。
可以用concurrency::accelerator::set_default来设置默认的accelerator。  
```cpp
// 可以这样输出
for (auto& i : concurrency::accelerator::get_all())
{
    wcout << "设备： " << i.description << endl;
    wcout << "设备路径： " << i.device_path << endl;
    wcout << "内存（KB）： " << i.dedicated_memory << endl;
    wcout << "是否连接到显示器： " << i.has_display << endl;
    wcout << "默认缓冲区访问权限： " << i.default_cpu_access_type << endl;
    wcout << "是否启用调试： " << i.is_debug << endl;
    wcout << "是否为仿真器： " << i.is_emulated << endl;
    wcout << "是否支持共享内存： " << i.supports_cpu_shared_memory << endl;
    wcout << "是否支持双精度： " << i.supports_double_precision << endl;
    wcout << "是否支持有限双精度： " << i.supports_limited_double_precision << endl;
    wcout << endl;
}
// 可以得到这样的输出
// 设备： 我的主显卡，默认用的是这个
// 设备路径： PCI\xxxxx\xxxxx
// 内存（KB）： 4162432
// 是否连接到显示器： 1
// 默认缓冲区访问权限： 0
// 是否启用调试： 1
// 是否为仿真器： 0
// 是否支持共享内存： 1
// 是否支持双精度： 1
// 是否支持有限双精度： 1
// 
// 设备： cpu带的显卡
// 设备路径： PCI\xxxxx\xxxxx
// 内存（KB）： 115200
// 是否连接到显示器： 0
// 默认缓冲区访问权限： 3
// 是否启用调试： 1
// 是否为仿真器： 0
// 是否支持共享内存： 1
// 是否支持双精度： 1
// 是否支持有限双精度： 1
// 
// 设备： Microsoft Basic Render Driver
// 设备路径： direct3d\warp
// 内存（KB）： 0
// 是否连接到显示器： 0
// 默认缓冲区访问权限： 3
// 是否启用调试： 1
// 是否为仿真器： 1
// 是否支持共享内存： 1
// 是否支持双精度： 1
// 是否支持有限双精度： 1
// 
// 设备： Software Adapter
// 设备路径： direct3d\ref
// 内存（KB）： 0
// 是否连接到显示器： 1
// 默认缓冲区访问权限： 0
// 是否启用调试： 1
// 是否为仿真器： 1
// 是否支持共享内存： 0
// 是否支持双精度： 1
// 是否支持有限双精度： 1
// 
// 设备： CPU accelerator
// 设备路径： cpu
// 内存（KB）： 0
// 是否连接到显示器： 0
// 默认缓冲区访问权限： 3
// 是否启用调试： 0
// 是否为仿真器： 1
// 是否支持共享内存： 1
// 是否支持双精度： 0
// 是否支持有限双精度： 0
```
在选取默认accelerator时，按以下规则依次选取：
1. 如果在调试模式下运行，选择支持调试的
2. 使用`CPPAMP_DEFAULT_ACCELERATOR`设置的accelerator
3. 非模拟设备
4. 有最大可用内存的设备
5. 未连接到显示器的设备

## 共享内存
可以在创建accelerator后设置相关的一些选项，比如让它支持**共享内存**。
选项定义如下：
```cpp
enum access_type
{
    access_type_none = 0,
    access_type_read = (1 << 0),
    access_type_write = (1 << 1),
    access_type_read_write = access_type_read | access_type_write,
    access_type_auto = (1 << 31),
};
```

共享内存可以让accelerator和cpu访问同一位置的内存，可以显著减少复制开销，但是不要同时访问内存，因为会导致未定义行为。
注意AMP默认会自动选择最好的访问权限，设置共享可能会比原先非共享的性能差（带宽或延迟影响），如果共享读写都没问题就默认设置为可读写，否则会以保守的方式设置权限。  

创建好accelerator后，就用default\_view创建一个accelerator\_view对象，accelerator\_view表示accelerator上的虚拟设备抽象。  
使用例子如下：
```cpp
using namespace std;
using namespace concurrency;

std::vector<int> data(5);

iota(data.begin(), data.end(), 1);

accelerator acc(accelerator::default_accelerator);

if (!acc.supports_cpu_shared_memory)
{
    std::cout << "不支持共享内存" << std::endl;
    return;
}

acc.set_default_cpu_access_type(access_type_read_write);

concurrency::accelerator_view accView = acc.default_view;

concurrency::array<int, 1> a(5, data.begin(), data.end(),
    accView, concurrency::access_type_read_write);

concurrency::parallel_for_each(
    a.extent,
    [=, &a](concurrency::index<1> idx) restrict(amp)
{
    a[idx] = a[idx] * 10;
});

for (int i = 0; i < 5; i++)
{
    cout << a[i] << endl;
}
// 如果不用共享内存就需要先将数据复制到本地再访问
data = a;
for (int i = 0; i < 5; i++)
{
    cout << data[i] << endl;
}
```


# 平铺/分组（Tiles）
可以使用平铺来加速代码。它将线程划分为相等的矩形子集或图块，如果使用合适的平铺大小和算法，就可以使代码获得更多加速。  
一般用在算法中需要访问多个内存位置的时候，在运行时先把相关内存都载入，再在算法中使用，这样就不需要读array/array_view造成性能损失。  

需要的基本组件是：
* tile_static 变量
    tile_static关键字用于声明可在所有平铺线程中访问的变量（注意是**该平铺分组的所有线程**，不是全局所有线程）。此变量的生存期在执行到达声明点时开始，在内核函数返回时结束。访问tile\_static变量比访问全局内存（array、array\_view）快得多，一般用法是将数据从全局内存一次性复制到该变量中，然后就可以多次访问它。
    该关键字具有这些限制：
    * 只能在具有restrict(amp)修饰符的函数中使用
    * 不能用来修饰指针或引用变量
    * 不能具有初始值，并且不会调用默认构造和析构
    * 未初始化的tile_static变量的值是不确定的
    * 如果在非平铺的调用中使用，其行为是不确定的
* tile_barrier::wait 函数
    调用时将挂起当前线程，直到同一平铺的所有线程都运行到这里，然后再放行。一般用来同步tile_static变量的访问。
* 全局和本地索引
    可以访问相对于全局内存的索引和平铺的索引。
* tiled\_extent类 和 tiled\_index类
    在平铺时parallel\_for\_each使用的是tiled\_extent对象，kernel函数的参数是tiled\_index类型。


在使用平铺时，需要指定固定大小的1、2、3维度，另外还具有这些限制：
* 平铺区域内维度大小不能超过1024，比如：
    * 3D：D0 x D1 x D2 ≤ 1024；并且 D0 ≤ 64
    * 2D：D0 x D1 ≤ 1024
    * 1D：D0 ≤ 1024
* 作为提交给parallel\_for\_each的第一个参数，其平铺网络必须是可分割的。

在kernel函数里用变量时，其数据会被保存在当前线程的寄存器中。每个gpu线程都有自己的寄存器，但是不能访问其它线程的寄存器。其速度和大小可以类比cpu的寄存器。  
在使用tile\_static时，数据存储在tile\_static共享内存中，只能在平铺模式中使用。
注意tile\_static是分组可访问，会有竟态条件出现。  

<br/>
**索引关系**  
平铺后，可取到的索引如下(注意顺序是z/y/x)：  
![](https://i-msdn.sec.s-msft.com/dynimg/IC660452.jpeg)  
![](https://i-msdn.sec.s-msft.com/dynimg/IC588202.jpeg)  

tiled_extent: 图块坐标转换
global: 全局每个维度坐标
tile: 分块的坐标
local: 分块内元素的坐标

对于index
index[x]: 所在维度的坐标
index: 总体的坐标

## 内存屏障
如果tile\_static 变量需要填充再使用，可用tile\_barrier::wait 函数来等待填充完毕。  
有两种必须同步的内存访问：全局内存访问、tile\_static内存访问。array仅分配全局内存，array\_view可引用全局内存或tile\_static内存。

有3种等待方式（这里可能涉及到了内存序的内容），每一个方式都是等待分块内的其它线程到达（不保证全局）：
* wait/wait\_with\_all\_memory\_fence: 并确保全局和tile_static内存可见
* wait\_with\_global\_memory\_fence: 确保全局内存可见
* wait\_with\_tile\_static\_memory\_fence: 确保tile_static内存可见

例：
```cpp
using namespace std;
using namespace concurrency;
static constexpr int Y = 9;
static constexpr int X = 8;

// y=9 x=8
std::array<uint32_t, Y * X> data;
iota(data.begin(), data.end(), 0);
array_view<uint32_t, 2> dataView(Y, X, data);


// 假设分块大小为 y3 * x2，需要计算块内平均值
static constexpr int TY = 3;
static constexpr int TX = 2;

std::array<uint32_t, Y / TY * X / TX> ret;
array_view<uint32_t, 2> retView(Y / TY, X / TX, ret);
retView.discard_data();

parallel_for_each(dataView.extent.tile<TY, TX>(),
    [=](tiled_index<TY, TX> idx) restrict(amp)
{
    // 组内共有变量
    tile_static uint32_t tileAvgData;
    bool isFirst = idx.local[0] == 0 && idx.local[1] == 0;

    if (isFirst)
    {
        tileAvgData = 0;
    }
    idx.barrier.wait_with_tile_static_memory_fence();

    concurrency::atomic_fetch_add(&tileAvgData, dataView[idx]);

    idx.barrier.wait_with_tile_static_memory_fence();

    if (isFirst)
    {
        retView[idx.tile] = tileAvgData / (TX*TY);
    }
});
retView.synchronize();

// 输出，请无视这个算法，它不是重点
int idxX = X / TX;
for (auto& i : ret)
{
    cout << i << "\t";
    --idxX;
    if (idxX == 0)
    {
        cout << endl;
        idxX = X / TX;
    }
}


// 输出如下
// 8       10      12      14
// 32      34      36      38
// 56      58      60      62
```

# 数学库
头文件为amp_math.h。包括两个数学库。  
Concurrency::precise\_math 命名空间中的双精度库支持双精度函数。  它还支持单精度功能，尽管硬件仍需要支持双精度。  
Concurrency::fast\_math 命名空间中的快速数学库包含另一组数学函数。
只支持 float 操作数的这些函数，执行速度更快，但精确度不如双精度数学库中的函数。
基本上cmath的函数都有，用法相同，不做过多介绍。  

# 图形库
头文件amp_graphics.h。  
包括为加速的图形编程而设计的图形库。此库仅用于支持本地图形功能的设备。
函数在 Concurrency::graphics 命名空间中。

图形库的关键组件有：  
* texture 类：可以使用纹理类创建内存或文件纹理。
    纹理类似于数组，因为它们包含数据，而它们在分配和复制构造方面类似于标准模板库 (STL) 中的容器。
    texture 类的模板参数是元素类型与维度。可以是 1、2 或 3维。  
    元素类型是Short Vector库中定义的类型。
* writeonly\_texture\_view 类：为任何纹理提供只写访问权。
* Short Vector库：定义一个基于 int、uint、float、double、norm或 unorm 的长度为 2、3 和 4 的短矢量类型集。

因为我用不到，这部分内容没看，相关内容在[链接1](https://msdn.microsoft.com/zh-cn/library/hh913015.aspx)、[链接2](https://msdn.microsoft.com/en-us/library/hh913015.aspx)、[链接3](https://msdn.microsoft.com/en-us/library/hh708799.aspx)。  

# 最后
整体感觉比较简单，一小时内可以学会（比cuda简单多了），并且amp可以跟ppl相配合，可以很方便的开发高性能程序，只不过只支持Windows。  
另外上面提到设置临时数组的高级优化方法，我没有在msdn找到相关内容。  
调试时用的并行可视化工具，据说在分析菜单里，我在vs里没看到（找遍了好像只有性能分析工具），以后用到了再更新文档。  