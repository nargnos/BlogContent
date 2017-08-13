---
title: "[C++ Idioms] 35. Hierarchy Generation"
date: 2017-08-01 18:58:12
categories: [学习笔记,C++ Idioms]
tags:
    - C++

    - 模板

---
将多个类组合成新类。<!--more-->  
用组合模式时有时候需要组合多个类，或者需要某个机制能自如的为类增删功能（静态），可以用这个。  

这其实是模板在多继承情况的一种用法，现代C++是可以完整继承模板参数的。原例子为了实现这个所以需要一层层解开参数继承，现在直接用就好。    

原文例子旧了，现在不用这个语法了，现更新为新的例子：
```cpp
template<typename... TClasses>
class Hierarchy:
	public TClasses...
{
};

struct Name
{
	void GetName()
	{}
};
struct Do
{
	void Dance()
	{}
};

template<typename T>
void Func(T&& obj)
{
	obj.GetName();
	obj.Dance();
}

int main(void)
{
	Func(Hierarchy<Name, Do>());
	return 0;
}

```
