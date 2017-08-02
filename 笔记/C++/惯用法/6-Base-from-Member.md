---
title: "[C++ Idioms] 6. Base from Member"
date: 2017-07-29 22:28:44
categories: C++ Idioms
tags:
    - C++
    - Idioms
    - 模板
---
用派生类的成员初始化基类的成员。<!--more-->  
当基类初始化时需要某个参数，但是那个参数是子类成员，这时候因为C++的初始化顺序是基类在子类前初始化，如果将子类成员当作参数传给基类，在基类使用时将会导致错误。  

情况如下：
```cpp
class Base
{
public:
	Base(stringstream& s)
	{
		s << "text";
	}
};

class MyClass :
	public Base
{
public:
	MyClass() :
		Base(var_)
	{
		// 此时因为var_未初始化，在Base使用时会导致出错
	}
private:
	stringstream var_;
};

```
  
这时候可以用多重继承解决，把基类需要的变量另外放到另一个类中，然后在基类之前继承这个类，这样这个变量就会在基类之前初始化完毕，基类就能使用了。  
这个方法可以扩展为，在基类初始化前先用其它一些参数初始化子类成员再给基类使用。  

例：
```cpp

template<typename T, int UniqueID = 0>
class BaseFromMember
{
protected:
	explicit BaseFromMember() {}
	T member_;
};

class Base
{
public:
	Base(stringstream& s)
	{
		s << "text";
	}
};

class MyClass :
	public BaseFromMember<stringstream>,
	public Base
{
public:
	MyClass() :
		Base(BaseFromMember<stringstream>::member_)
	{
	}
};
```
注意实现这个类时一般带一个UniqueID，因为当有多个同类型的成员时，如果不在模板做区分，将无法取得正确的成员。  

Boost可以用类库base_from_member，其源码跟例子的很像，就不贴了。

