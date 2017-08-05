---
title: "[C++ Idioms] 8. Calling Virtuals During Initialization"
date: 2017-07-30 15:57:09
categories: C++ Idioms
tags:
    - C++
    - Idioms


---
在初始化时调用虚函数。<!--more-->  
有时候初始化的一些操作要推迟到子类实现（模板方法），但是C++不像C#，它在执行基类构造函数时，子类的虚表并没有初始化，所以只能调用到基类自身的虚函数（在某些书中是建议不要使用这类用法的）。

例：
```cpp
class Base
{
public:
	Base()
	{
		Init();
	}
protected:
	virtual void Init()
	{
		cout << "Base"<<endl;
	};
};

class MyClass:
	public Base
{
public:
protected:
	virtual void Init() override
	{
		cout << "MyClass" << endl;
	}
};
// 此时实例化MyClass调用到的Init并不是子类的
```
这时候只能新建一个Create函数或者工厂，在实例化后通过显式调用Init函数实现初始化。  

如果要实现在初始化时调用子类实现的Init，可以用CRTP（很多惯用法都用到这个思想）。  
注意Init函数不能声明为虚函数，因为先初始化父类的关系，在调用时用的并不是子类的虚表。  
还需要注意，在父类调用子类的Init时，要注意子类成员变量是未初始化的，需要酌情用BaseFromMember。  
其实这就是一个CRTP的应用。  

```cpp
class Base
{
public:
	Base() {}
protected:
	//void Init()
	//{
	//	cout << "Base" << endl;
	//};
};
template<typename T>
class InitCaller :
	public Base
{
public:
	InitCaller()
	{
		Self()->Init();
	}
private:
	T* Self()
	{
		return reinterpret_cast<T*>(this);
	}
};

class MyClass :
	public InitCaller<MyClass>
{
public:
	friend InitCaller<MyClass>;
protected:
	void Init()
	{
		cout << "MyClass" << endl;
	}
};
```
