---
title: "[C++ Idioms] 21. Curiously Recurring Template Pattern"
date: 2017-07-30 22:03:53
categories: C++ Idioms
tags:
    - C++
    - Idioms
---
这就是很多惯用法都用到的CRTP。这是去除虚表的一种方法。<!--more-->  
这是一个很重要的惯用法。其核心就是父类的模板参数是子类类型，这样在父类里就可以强制转换指针为子类类型，而使用子类的函数（隐式接口）。  
在某些情况可以被编译器优化从而获得性能提升。（在现在这种速度比较快的机器下，运行个一亿次会比使用虚表快个几百毫秒，但并不是说这个惯用法没用，用得好速度会有可观的提升）  
其核心结构为：
```cpp
template<typename TDerived>
class Base
{
public:
	void Do()
	{
		auto self = static_cast<TDerived*>(this);
		self->DoImpl();
	}
};

class Derived:
	public Base<Derived>
{
public:
	void DoImpl()
	{
	}
};
```

可以将成员函数声明为静态，配合其它一些技巧使用有奇效。  
WTL源码里有用到，Boost的很多类库也都会用到，比如Asio源码里也有用到（是这个结构的改进版）。  

CRTP由于基类用模板形式，如果有多个子类就会生成多个不同类型的父类，不方便做统一管理；这时候可以加一个中间层Caller用来调用子类的函数，或者用伪虚表的方式来处理。  