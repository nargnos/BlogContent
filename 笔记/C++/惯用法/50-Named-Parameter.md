---
title: "[C++ Idioms] 50. Named Parameter"
date: 2017-08-02 22:45:26
categories: C++ Idioms
tags:
    - C++
    - Idioms


---
命名参数。<!--more-->用在有多个可选参数要设置的情况。  

用链式函数的方式，例：
```cpp
// 简单的例子，有很多不好的地方，只表达出意思，写完整费时间
class MyClass
{
public:
	MyClass& SetA(int val)
	{
		a_ = val;
		return *this;
	}
	MyClass& SetB(char val)
	{
		b_ = val;
		return *this;
	}
private:
	int a_;
	char b_;
};


int main(void)
{
	auto c = MyClass().SetA(1).SetB(2);
	return 0;
}
```
还可以在编写Set函数时返回的是一个代理类，用来详细设置配置信息。
比如 
```cpp
MyType *instance = segment.construct<MyType>
         ("MyType instance")  //name of the object
         (0.0, 0);
MyType *array = segment.construct<MyType>
         ("MyType array")     //name of the object
         [10]                 //number of elements
         (0.0, 0);            //Same two ctor arguments for all objects
```
这里不扩展了，boost里有一些类就是用到这个.
