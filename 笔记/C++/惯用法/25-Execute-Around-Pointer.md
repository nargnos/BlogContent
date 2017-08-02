---
title: "[C++ Idioms] 25. Execute-Around Pointer"
date: 2017-07-31 15:47:35
categories: C++ Idioms
tags:
    - C++
    - Idioms
---
跟踪对象的函数调用。<!--more-->  
使用这个方法可以方便的在调用对象的任何函数前后加入想要执行的代码。  
这是代理模式的一个应用。  

一般形式如下：
```cpp
template<typename T>
class Proxy
{
	Proxy(T* obj):obj_(obj)
	{
		// 自定义执行函数前的代码
	}
	~Proxy()
	{
		// 自定义执行函数后的代码
	}
	T* operator->()
	{
		return obj_;
	}
private:
	T* obj_;
};
template<typename T>
class Logger
{
public:
	Logger(T* obj) :obj_(obj)
	{
	}
	T* operator->()
	{
		return Proxy<T>(obj_);
	}
private:
	T* obj_;
};

```
核心是创建一个代理类，重写operator->，在代理类的ctor、dtor写入自定义代码即可。  
不必担心创建过多代理对象，因为这可以被编译器优化。  
可以扩展重写其它运算符，比如`[]`之类的，还可以用来管理对象的函数调用，设置函数执行权限什么的。  