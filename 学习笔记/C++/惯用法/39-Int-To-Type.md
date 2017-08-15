---
title: "[C++ Idioms] 39. [替] Int To Type"
date: 2017-08-01 21:13:07
categories: [学习笔记,C++ Idioms]
tags:
    - C++

    - 模板

---
将具体数字变为类型。<!--more-->可用integral_constant实现，这是模板元编程的一个类。  
比较常用的有`true_type`、`false_type`。一般配合模板来用，可用在函数参数中，使函数对不同参数的值产生重载。这样可以跳过if判断，让编译器直接优化代码。  
还可以放在模板参数、返回类型里。相关的类还有integer_sequence（介绍在其它文档里），它存的是数列。  

```cpp
class MyClass
{
public:
	void Do()
	{
		Func(integral_constant<int, I>());
	}
private:
	void Func(integral_constant<int, 0>)
	{
	}
	void Func(integral_constant<int, 1>)
	{
	}
};
template<typename T>
class MyClass
{
public:
	void Do()
	{
		Func(integral_constant<bool, is_pod<T>::value>());
	}
private:
	void Func(true_type)
	{
	}
	void Func(false_type)
	{
	}
};
```

它的实现很简单，这里直接贴stl代码：
```cpp
// TEMPLATE CLASS integral_constant
template<class _Ty,
	_Ty _Val>
	struct integral_constant
{	// convenient template for integral constant types
	static constexpr _Ty value = _Val;

	typedef _Ty value_type;
	typedef integral_constant<_Ty, _Val> type;

	constexpr operator value_type() const _NOEXCEPT
	{	// return stored value
		return (value);
	}

	constexpr value_type operator()() const _NOEXCEPT
	{	// return stored value
		return (value);
	}
};
```
在constexpr关键字没出现前使用enum定义value枚举。现在新的代码都改了。   

