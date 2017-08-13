---
title: "[C++ Idioms] 60. Parameterized Base Class"
date: 2017-08-03 15:11:25
categories: [学习笔记,C++ Idioms]
tags:
    - C++

    - 模板

---
给类添加特性。<!--more-->把一些公有的代码提取出来，并且使用继承模板参数类型的方式封装成类，这样可以给类添加新特性。说不清楚，上代码：  
```cpp
template<typename T>
class GetName :
	public T
{
public:
	const char* Name()
	{
		return typeid(T).name();
	}
};
template<typename T>
class GetSize :
	public T
{
public:
	size_t Size()
	{
		return sizeof(T);
	}
};


class MyClass
{
};

int main(void)
{
	GetName<MyClass> obj;
	auto name = obj.Name();
	GetSize<GetName<MyClass>> obj2;
	obj2.Name();
	obj2.Size();
	return 0;
}

```
这个例子为类添加获取名字、大小的特性，还可以像原文例子那样，给类添加可序列化特性。
使用继承可以保留基类类型，可以向原类型转换。可以在特性类中附加接口继承什么的，这样就可以方便的转换类为其它接口类型。  

