---
title: "[C++ Idioms] 38. Inner Class"
date: 2017-08-01 19:22:32
categories: [学习笔记,C++ Idioms]
tags:
    - C++



---
不使用多继承为类提供多个接口，或在单个抽象中提供同一接口的多个实现。<!--more-->  
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
