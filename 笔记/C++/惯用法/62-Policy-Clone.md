---
title: "[C++ Idioms] 62. Policy Clone"
date: 2017-08-03 17:53:08
categories: C++ Idioms
tags:
    - C++
    - Idioms
    - 模板
---
模板策略克隆。<!--more-->让已经绑定到模板的策略类重新绑定到新的类型，在某些不能直接取得类的策略模板原型的时候，通过这个惯用法可以获得新的绑定类型。
如：
```cpp
template<typename T>
struct Policy
{
	static void Print()
	{
		cout << sizeof(T) << endl;
	}
};

template<typename T, typename TPolicy = Policy<T>>
class MyClass
{
public:
	MyClass()
	{
		TPolicy::Print();
		// 假设此时想修改TPolicy的绑定类型（模板参数），是无法修改的，因为不知道它的原始类型
	}
};

```

此时修改成这样便可以解决：
```cpp
struct Policy
{
	static void Print()
	{
		cout << sizeof(T) << endl;
	}
	template<typename U>
	using Rebind = Policy<U>;
};

template<typename T, typename TPolicy = Policy<T>>
class MyClass
{
public:
	MyClass()
	{
		TPolicy::Print();
		// 重绑定
		TPolicy::Rebind<double>::Print();
	}
};
```