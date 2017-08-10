---
title: "[C++ Idioms] 16. Construction Tracker"
date: 2017-07-30 18:14:48
categories: [学习笔记,C++ Idioms]
tags:
    - C++
    - Idioms


---
区分构造函数中产生的同类型异常。<!--more-->  
很多书中建议初始化不要抛出异常，因为有可能无法恢复某些上下文。  
但如果我偏用，且在初始化列表初始化的成员中抛了异常，还抛同样的类型，如何区分是哪个成员在初始化时抛出的？  

在每个成员的初始化时给某个变量赋值标记该成员初始化，然后catch的时候就能获得对应信息。  
这里需要用到一些不常用的语法。构造函数的try catch和逗号表达式做参数传递。   
例：
```cpp
// 原文的例子本身足够好，只做很小的修改
struct A {
	A(char const *) { throw 123; }
};
struct B {
	B(int) { throw 456; }
};
class C {
	enum TrackerType
	{
		None,
		AInit,
		BInit,
		Done = BInit
	};
public:
	// 在构造函数后可以加try catch块，这个特性不太常用
	C(char const* args, TrackerType tracker = TrackerType::None) try :
		// 使用逗号表达式赋值tracker
		a_((tracker = TrackerType::AInit, "abc")),
		b_((tracker = TrackerType::BInit, 123))
	{
		// 确保初始化完毕
		assert(tracker == TrackerType::Done);
	}
	catch (int val)
	{
		// 此时可以根据tracker值做恢复工作，或者再throw
		switch (tracker)
		{
		case C::None:
			break;
		case C::AInit:
			break;
		case C::BInit:
			break;
		default:
			break;
		}
	}
private:
	A a_;
	B b_;
};
```
