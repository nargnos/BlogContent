---
title: "[C++ Idioms] 31-40"
date: 2017-07-30 12:14:17
categories: [学习笔记,C++ Idioms]
tags: [C++, 模板]
description: Final Class、Free Function Allocators、Function Object、Generic Container Idioms、Hierarchy Generation、Include Guard Macro、Inline Guard Macro、Inner Class、Int To Type、Interface Class


---
# Final Class [替]
禁止继承类。 
现在在声明后加final即可。  
之前的方法是将父类构造函数声明为私有，设置要继承的类为友元即可。  

---
# Free Function Allocators [旧]
允许容器使用自定义分配器。 
不理解这个意义何在，因为容器本来就允许自定义分配器啊。  
原文里写因为标准分配器修改底层类型……好像没有啊。  
只能把这个当过过时内容了。  

---
# Function Object [补]
函数对象。
原文没内容，继续YY。这个标题可能指的是伪函数对象，重写operator()后就可以当作函数来用了，比如function，用得十分普遍，不再扩展。

---
# Generic Container Idioms
创建通用容器类。 
在创建通用容器类时，可能类型并没有默认的构造函数，或者类型不支持赋值或者有各种各样的问题。此时可以先分配空间，到添加对象时用`new (&place) Type`（placement new）处理即可。  

stl容器用到这个（部分源码）：  
```cpp
// 前面还有蛮多代码，这里是关键的
template<class _Objty,
	class... _Types>
	void construct(_Objty *_Ptr, _Types&&... _Args)
{	// construct _Objty(_Types...) at _Ptr
	::new ((void *)_Ptr) _Objty(_STD forward<_Types>(_Args)...);
}
```

---
# Hierarchy Generation
将多个类组合成新类。 
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

---
# Include Guard Macro [易]
顾名思义。 
没什么需要扩展的，现在已替换成`#pragma once`，觉得某些编译器依然不支持的或者说某些书不提倡用的请更新你们的环境和把旧书丢掉，请更新你们的知识。  
证据是 <https://en.wikipedia.org/wiki/Pragma_once> 请看编译器支持列表。  

---
# Inline Guard Macro [易]
使用宏定义控制函数的内联。  
就是定义一个inline的别名，用ifdef子类的做开关，然后在函数前用这个别名即可。  
据原文说是用在调试的时候，这样可以快速的切换状态方便调试。  

---
# Inner Class
不使用多继承为类提供多个接口，或在单个抽象中提供同一接口的多个实现。 
其核心是在在其它类继承接口（并在其它类中提供操作自身的方法），然后在自身保存这些类的变量，并提供对相应接口的转换，这样使用时就可以当其它接口来用。  

语言不好描述，看代码：
```cpp
struct IWalk
{
	virtual void Walk() = 0;
};
struct IRun
{
	virtual void Run() = 0;
};
class Person;
// 不一定是内嵌类
class RunImpl :public IRun
{
public:
	explicit RunImpl(Person& p);
	virtual void Run() override;
private:
	Person& p_;
};


class Person
{
public:
	Person() :
		r_(*this),
		w_(*this)
	{
	}
	class WalkImpl :
		public IWalk
	{
	public:
		explicit WalkImpl(Person& p) :p_(p) {}
		virtual void Walk() override
		{
			p_.Do("Walk");
		}
	private:
		Person& p_;
	};
	friend RunImpl;
	operator IRun&()
	{
		return r_;
	}
	operator IWalk&()
	{
		return w_;
	}
private:
	void Do(const string& str)
	{
		cout << str << endl;
	}
	RunImpl r_;
	WalkImpl w_;
};

RunImpl::RunImpl(Person & p) :p_(p)
{
}
void RunImpl::Run()
{
	p_.Do("Run");
}


int main(void)
{
	Person p;
	IRun& r = p;
	r.Run();
	IWalk& w = p;
	w.Walk();
	return 0;
}

```
关于同一对象提供多个同一接口的不同实现，就是多建几个类，用函数返回不同的实现就行了。  

可以声明一个总接口，然后用map存储，这样便可以做动态接口扩展。这是对这个惯用法的举一反三。在某个类需要多个接口实现并且这些接口经常变化时可以使用。    
```cpp
struct Interface
{
	virtual ~Interface() = default;
};
struct IWalk :public Interface
{
	virtual void Walk() = 0;
};
struct IRun :public Interface
{
	virtual void Run() = 0;
};
class Person;
class RunImpl :public IRun
{
public:
	explicit RunImpl(Person& p);
	virtual void Run() override;
private:
	Person& p_;
};
class WalkImpl :
	public IWalk
{
public:
	explicit WalkImpl(Person& p);
	virtual void Walk() override;
private:
	Person& p_;
};

// 只提供一些主要的代码，只为了表达我的意思，
// 没有做完备的检查，所以某些情况可能有BUG
class Person
{
public:	
	// 也可以用其它方式返回
	template<typename T>
	operator T*()
	{
		auto name = typeid(T).name();
		auto& ret = map_[name];		
		return dynamic_cast<T*>(ret.get());
	}
	// 也可以提供删除接口的方法，这里略，要不代码太长了
	template<typename TInterface, typename TImpl>
	void AddInterface()
	{
		static_assert(is_assignable<TInterface, TImpl>::value, "");
		// 不一定用类型名，可用很多其它东西做key
		auto name = typeid(TInterface).name();
		map_[name] = make_unique<TImpl>(*this);
	}

	// 为了实现方便，这函数用public
	void Do(const string& str)
	{
		cout << str << endl;
	}
private:
	unordered_map<string, unique_ptr<Interface>> map_;
};

RunImpl::RunImpl(Person & p) :p_(p)
{
}
void RunImpl::Run()
{
	p_.Do("Run");
}

WalkImpl::WalkImpl(Person & p) :p_(p) {}

void WalkImpl::Walk()
{
	p_.Do("Walk");
}

int main(void)
{
	Person p;
	// 这样动态添加接口，类就可以向对应接口转换
	p.AddInterface<IRun, RunImpl>();
	IRun* r = p;
	r->Run();
	// 此时未添加接口
	IWalk* w = p;
	assert(w == nullptr);

	// 添加接口
	p.AddInterface<IWalk, WalkImpl>();
	w = p;
	assert(w != nullptr);
	w->Walk();
	return 0;
}

```

---
# Int To Type [替]
将具体数字变为类型。可用integral_constant实现，这是模板元编程的一个类。  
比较常用的有`true_type`、`false_type`。一般配合模板来用，可用在函数参数中，使函数对不同参数的值产生重载。这样可以跳过if判断，让编译器直接优化代码。  
还可以放在模板参数、返回类型里。相关的类还有integer_sequence（介绍在其它文档里），它存的是数列。  

```cpp
class MyClass
{
public:
	void Do()
	{
		Func(integral_constant<int, I>());
	}
private:
	void Func(integral_constant<int, 0>)
	{
	}
	void Func(integral_constant<int, 1>)
	{
	}
};
template<typename T>
class MyClass
{
public:
	void Do()
	{
		Func(integral_constant<bool, is_pod<T>::value>());
	}
private:
	void Func(true_type)
	{
	}
	void Func(false_type)
	{
	}
};
```

它的实现很简单，这里直接贴stl代码：
```cpp
// TEMPLATE CLASS integral_constant
template<class _Ty,
	_Ty _Val>
	struct integral_constant
{	// convenient template for integral constant types
	static constexpr _Ty value = _Val;

	typedef _Ty value_type;
	typedef integral_constant<_Ty, _Val> type;

	constexpr operator value_type() const _NOEXCEPT
	{	// return stored value
		return (value);
	}

	constexpr value_type operator()() const _NOEXCEPT
	{	// return stored value
		return (value);
	}
};
```
在constexpr关键字没出现前使用enum定义value枚举。现在新的代码都改了。   

---
# Interface Class [易]
将类当作接口使用。这个用得很普遍，不再扩展。