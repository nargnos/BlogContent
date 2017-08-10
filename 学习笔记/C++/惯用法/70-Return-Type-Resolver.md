---
title: "[C++ Idioms] 70. Return Type Resolver"
date: 2017-08-04 15:28:04
categories: [学习笔记,C++ Idioms]
tags:
    - C++
    - Idioms
    - 模板

---
推导被初始化或被赋值的对象的类型。<!--more-->又是一个使用operator Type特性的惯用法：
```cpp
class MyClass
{
public:
	template<typename T>
	operator T()
	{
		return T();
	}
};

int main()
{
	int x = MyClass();
	void* y = MyClass();
	return 0;
}
```
将operator Type声明为模板函数可以检测到赋值或者构造的类型。

应用如下：
```cpp
class Range
{
public:
	Range(int begin, int end, int step = 1) :
		begin_(begin),
		end_(end),
		step_(step)
	{
		// 范围检查略
	}
	Range(int end) :
		Range(0, end, 1)
	{
	}
	template<typename TContainer>
	operator TContainer()
	{
		TContainer ret;
		auto inserter = std::back_inserter(ret);
		for (int i = begin_; i < end_; i += step_)
		{
			inserter = i;
		}
		return ret;
	}
private:
	int begin_;
	int end_;
	int step_;
};

int main()
{
	std::vector<int> v = Range(10);
	std::list<char> l = Range(5, 20, 2);
	return 0;
}
```