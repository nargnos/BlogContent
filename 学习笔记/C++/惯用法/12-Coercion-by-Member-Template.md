---
title: "[C++ Idioms] 12. Coercion by Member Template"
date: 2017-07-30 17:17:12
categories: [学习笔记,C++ Idioms]
tags:
    - C++

    - 模板

---
让模板支持协变和抗变。<!--more-->  
在智能指针中有应用。  

例：
```cpp
template<typename T>
class Template{};

class A{};
class B:
	public A
{};
```
此时`Template<A>`、`Template<B>`是无法互相赋值的，因为它们是不同类型。  
在某些时候需要能在这两个类型中转换（智能指针），就需要这样写operator。  

```cpp
template<typename T>
class Template
{
public:
	template<typename U>
	Template& operator=(const Template<U>&){}
};

class A{};
class B:
	public A
{};
```
这样两个类型便可以互相操作了，只需要在函数里添加自己的实现即可。  
