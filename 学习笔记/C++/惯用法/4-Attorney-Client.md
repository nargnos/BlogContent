---
title: "[C++ Idioms] 4. Attorney Client"
date: 2017-07-29 16:53:21
categories: [学习笔记,C++ Idioms]
tags:
    - C++
    - Idioms
    - 模板

---
友元代理，限制友元的访问权限。<!-- more -->  
C++的friend给的权限过大，这个可以限制其权限。  
或者可以在某个类需要设置多个友元对象时，可以简化设置操作。  
其核心是**用中间层来传递友元关系**，同时控制访问权限。  
Boost有例子，iterator_facade的相关操作就用到了这个惯用法。  

例：
```cpp
// 比如有一个类希望其友元只能访问varA
class Friend
{
public:
	friend class MyClass;
private:
	int varA_;
	int varB_;
};
// 但是此时这个友元类“越界”访问了，原始类对此无能为力
class MyClass
{
public:
	void MyFriend(Friend& f)
	{
		f.varA_ = 0;
		f.varB_ = 0;
	}
};
```

通过这个惯用法，加入中间层：
```cpp
class Friend
{
public:
	friend class Attorney;
private:
	int varA_;
	int varB_;
};
// 使用这个类传递友元关系
class Attorney
{
private:
	// 如果想在多个类中共享，可直接设置函数为public，然后取消设置这个类的友元
	friend class MyClass;
	static void SetVarA(Friend& f, int val)
	{
		f.varA_ = val;
	}
	static int GetVarA(Friend& f)
	{
		return f.varA_;
	}
	// 或者更直接的
	static int& VarA(Friend& f)
	{
		return f.varA_;
	}
};

class MyClass
{
public:
	void MyFriend(Friend& f)
	{
		Attorney::SetVarA(f, 0);
		auto varA = Attorney::GetVarA(f);
		auto varARef = Attorney::VarA(f);
		// 无法访问varB
		// f.varB_ = 0;
	}
};
class MyClass2
{
public:
	void MyFriend(Friend& f)
	{
		// 此时由于MyClass2不是Attorney的友元，便不具有访问权限
		// Attorney::SetVarA(f, 0);
		// auto varA = Attorney::GetVarA(f);
		// auto varARef = Attorney::VarA(f);
	}
};

```

在用CRTP（后面会说到）的时候，不方便设置父类友元或者父类的一些操作是由其它类配合完成的，这样在使用到的时候就不能单纯设置友元。这时候传递友元就需要一些技巧：  
声明一个第三方类，子类设置其为友元，第三方类里声明模板函数（这样就可以接收所有子类），执行的是子类的一些私有函数（类似桥接），然后父类调用这个第三方类并传入子类指针（static_cast），就可以调用到子类的私有成员。  

Boost源码中iterator_facade的使用方式就类似这样。  

例：
```cpp
// 假设crtp参数构成比较复杂
template<typename TDerived, int...>
class Base
{
public:
	void DoSomething()
	{
		auto self = reinterpret_cast<TDerived*>(this);
		// 需要访问子类的私有函数
		// 但此时不设置Base为友元是无法访问这些函数的
		// self->DoA();
		// self->DoB();
	}
};
// 第一层继承，负责根据TChild类型填充Base的参数
template<typename TChild>
class Derived :
	// 这里参数只是例子
	public Base<TChild, 1, 2, 3>
{
	// 此时在这里声明友元可以让父类访问Derived
	// 但是无法让父类访问下一级派生TChild的私有成员
	// friend Base<TChild, 1, 2, 3>;

	// 假设Base还调用到这个类的其它一些函数，这里略
};

class MyClass :
	public Derived<MyClass>
{
public:
	// friend Base<MyClass, 1, 2, 3>;
	// 可以在这一步写这个friend让Base访问到对应函数
	// 但是此时MyClass的编写者就要了解是什么类（Base）需要调用
	// 可能整个继承链的每一层都有一些函数要调用到自身的私有成员，这样就需要写很多friend
	// 此时还需要了解这个类需要哪些参数及这些参数如何生成，这样如果派生多了会很麻烦
	// 也就是说，会导致继承者需要了解父类的实现细节
	
	// 此时无法通过使用 friend Base; 让使用各种模板参数的Base都具有友元效果	
	// 此时就只能使这些函数变public？
private:
	void DoA() {}
	void DoB() {}
};
```

此时添加一个第三方类做友元传递。  

```cpp
class Access
{
public:
	template<typename T>
	static void CallDoA(T* ptr)
	{
		ptr->DoA();
	}
	template<typename T>
	static void CallDoB(T* ptr)
	{
		ptr->DoB();
	}
};

template<typename TDerived, int...>
class Base
{
public:
	void DoSomething()
	{
		auto self = reinterpret_cast<TDerived*>(this);
		// 访问子类的私有函数
		Access::CallDoA(self);
		Access::CallDoB(self);
	}
};

template<typename TChild>
class Derived :
	public Base<TChild, 1, 2, 3>
{};

class MyClass :
	public Derived<MyClass>
{
public:
	// 此时只需要设置Access为友元，Access中用隐式接口调用本类私有对象
	// 此时继承者不需要了解父类的任何实现细节
	friend Access;
private:
	void DoA() {}
	void DoB() {}
};

```
