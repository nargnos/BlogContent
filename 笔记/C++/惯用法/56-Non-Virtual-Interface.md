---
title: "[C++ Idioms] 56. Non-Virtual Interface"
date: 2017-08-03 12:29:25
categories: C++ Idioms
tags:
    - C++
    - Idioms


---
非虚接口(NVI)。<!--more-->不是指不用虚表的接口，而是指将需要多态实现的一些步骤声明为protected或private的虚函数，并在子类实现（模板方法），而基类对外提供的接口是非虚的。  
一些书里有讨论到这个用法，使不使用看个人喜好。可以视为模板方法的一个实现方式，由于提取了一些公有代码，在某些情况可以减少代码编译的大小。  
```cpp
class Base
{
public:
	// 对外提供这个接口
	void Do()
	{
		// do ...
		Do2();
		// do ...
	}
private:
	virtual void Do2() = 0;
};
class MyClass :public Base
{
	virtual void Do2() override
	{
		// do ...
	}
};
```