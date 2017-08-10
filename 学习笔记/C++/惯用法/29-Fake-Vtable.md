---
title: "[C++ Idioms] 29. [补] Fake Vtable"
date: 2017-08-01 13:51:09
categories: [学习笔记,C++ Idioms]
tags:
    - C++
    - Idioms


---
伪虚表。<!--more-->  
这个惯用法原文是空的没有内容，那我自己根据名字YY一个。  
这是看一些类库的源代码总结出来的。  
nginx源码里面有用到，就是声明一个结构体，保存“对象”要用到的函数指针，在需要重写（或替换）函数时，直接把相应的函数指针替换掉，这样相当于用C实现了虚表机制。  

在Asio的源码里有C++方法实现的伪虚表机制（用来存Ops），结构如下：
```cpp
class Base
{
public:
	using DoSth = int(*)(Base* self, int arg);
	explicit Base(DoSth doSth) :
		doSth_(doSth)
	{
	}
	int Do(int arg)
	{
		return doSth_(this, arg);
	}
	// 为了保证正确析构只能这样，否则自己实现太麻烦了
	virtual ~Base() = default;
private:
	DoSth doSth_;
};

class MyClass :
	public Base
{
public:
	// 这里用来支持再继承
	MyClass(DoSth doSth, int val) :
		Base(doSth),
		val_(val)
	{
	}
	explicit MyClass(int val) :
		MyClass(&MyDoSth, val)
	{
	}
protected:
	int val_;
private:
	static int MyDoSth(Base* ptr, int arg)
	{
		auto self = static_cast<MyClass*>(ptr);
		return self->val_*arg;
	}
};
class MyClass2 :
	public MyClass
{
public:
	MyClass2(int val) :
		MyClass(&MyDoSth, val)
	{
	}

private:
	static int MyDoSth(Base* ptr, int arg)
	{
		auto self = static_cast<MyClass2*>(ptr);
		return self->val_ + arg;
	}
};

int main(void)
{
	MyClass c(10);
	Base* b = &c;
	auto ret = b->Do(20);
	cout << ret << endl;

	MyClass2 c2(10);
	b = &c2;
	ret = b->Do(20);
	cout << ret << endl;

	return 0;
}
```
在类内部维护函数指针，从构造函数传入，有重写就替换掉，这样便实现了虚表机制。  
在某些情况是可以被编译器优化的，跟用虚表的速度差不了多少（根据寻址关系来看会比虚表快一点，但是实际测试是差不多的，因为现在的机器太快了）。这个惯用法可以用来处理CRTP的基类需要模板参数的问题。  
对于某些需要用function的情况，如果用模板方式存储回调，也可以用这个方式提取公共父类统一处理。这个惯用法对于一些有虚表洁癖的人真是福音啊。  