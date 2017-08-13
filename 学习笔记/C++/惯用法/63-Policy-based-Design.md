---
title: "[C++ Idioms] 63. [补] Policy-based Design"
date: 2017-08-03 18:08:47
categories: [学习笔记,C++ Idioms]
tags:
    - C++

    - 模板

---
基于策略设计。<!--more-->原文无内容，这是YY。  
这是模板的策略模式，不同于一般的策略模式实现，这里用模板参数作为策略，用法如下：
```cpp

struct AddPolicy
{
	static int Calc(int val)
	{
		return val + val;
	}
};
struct MultiPolicy
{
	static int Calc(int val)
	{
		return val * val;
	}
};

// 调用时可以传入不同的策略来影响计算结果
template<typename TPolicy = AddPolicy>
int Calc(int val)
{
	int ret = val;
	// 做其它事情
	++ret;
	// 执行类策略
	ret = TPolicy::Calc(ret);
	// 做其它事情
	++ret;
	return ret;
}

template<typename TPolicy = DefaultPolicy>
class MyClass
{
public:
	int DoCalc(int val)
	{
		// 这里借用上面的函数
		return Calc<TPolicy>(val);
	}
};
```
一般策略类的函数可以声明为static，也可以不是静态（在需要保存状态的情况，这时候需要创建相关对象），stl用得很多，比如各种容器可以传入自定义的分配器，这个就是模板策略。