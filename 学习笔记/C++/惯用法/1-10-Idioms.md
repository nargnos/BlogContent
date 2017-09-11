---
title: "[C++ Idioms] 1-10"
date: 2017-07-30 11:07:09
categories: [学习笔记,C++ Idioms]
tags: [C++, 模板]
description: Address Of、Algebraic Hierarchy、Attorney Client、Barton-Nackman trick、Base from Member、Mutant、Calling Virtuals During Initialization、Capability Query、Checked delete


---
_之前分开放文件太多，现十个一组合并。日期什么的就乱改一个（保证文档先后顺序）吧。_


# Address Of [替]
获取对象的真实地址，用在对象重载&的情况。  
STL、Boost使用现成的函数： addressof。  

-------
在如下情况，对该类型对象取地址会导致编译失败，此时可以用addressof获取地址。  
```c++
class nonaddressable
{
public:
	typedef double useless_type;
private:
	useless_type operator&() const;
};
```

addressof的stl源码如下，**核心就是那句转换**。  
添加const volatile的原因是在转换用const或volatile修饰的引用时，如果不加上这个标识，用`reinterpret_cast`是无法通过编译的（转换操作的特性）。  
所以这里先转为cv引用，再去掉cv标识（因为应该按模板参数捕获到的类型返回，而`reinterpret_cast`是可以添加cv修饰的），然后再取地址转换为对应返回类型指针（此时返回类型如果有带cv修饰会再被加上去）。

```c++
/**
*  @brief Same as C++11 std::addressof
*  @ingroup utilities
*/
template<typename _Tp>
inline _Tp*
__addressof(_Tp& __r) _GLIBCXX_NOEXCEPT
{
	return reinterpret_cast<_Tp*>
		(&const_cast<char&>(reinterpret_cast<const volatile char&>(__r)));
}

/**
*  @brief Returns the actual address of the object or function
*         referenced by r, even in the presence of an overloaded
*         operator&.
*  @param  __r  Reference to an object or function.
*  @return   The actual address.
*/
template<typename _Tp>
inline _Tp*
addressof(_Tp& __r) noexcept
{
	return std::__addressof(__r);
}

```


另一处的源码，添加了对于函数类型的判断，但可从源码看出有builtin的函数可用，修饰也添加了constexpr。
```c++
// TEMPLATE FUNCTION addressof
#ifdef __EDG__ /* TRANSITION, VSO#200707 */
template<class _Ty> inline
_Ty *_Addressof(_Ty& _Val, true_type) _NOEXCEPT
{	// return address of function _Val
	return (_Val);
}

template<class _Ty> inline
_Ty *_Addressof(_Ty& _Val, false_type) _NOEXCEPT
{	// return address of object _Val
	return (reinterpret_cast<_Ty *>(
		&const_cast<char&>(
			reinterpret_cast<const volatile char&>(_Val))));
}

template<class _Ty> inline
_Ty *addressof(_Ty& _Val) _NOEXCEPT
{	// return address of _Val
	return (_Addressof(_Val, is_function<_Ty>()));
}
#else /* __EDG__ */
template<class _Ty> inline
constexpr _Ty *addressof(_Ty& _Val) _NOEXCEPT
{	// return address of _Val
	return (__builtin_addressof(_Val));
}
#endif /* __EDG__ */

```

---
# Algebraic Hierarchy [易]
目的是隐藏对象操作细节，可以通过一些简单的接口调用。   
比如`complex`复数库（高中内容i^2=-1的复数），**重载运算符**使之能通过这些运算符完成一些比较复杂的操作。  
```c++
complex<int> i(0, 1);
auto res = i * i; // res == complex<int>(-1)
```
其它的一些库多多少少都有一些应用，比如stream、string等。  

---
# Attach by Initialization
当框架需要在程序启动时初始化，由于框架细节不便提供给用户修改，就将用户自定义的初始化内容“延迟”到子对象实现（模板方法）。    
注意在使用时如果跨编译单元引用静态对象的内容，结果是不确定的，因为对于不同编译单元静态对象的初始化顺序，标准里并没有明确。   

例：
```c++
class Framework
{
public:
	Framework()
	{
		// 向某对象注册
	}
	void Init()
	{
		// 框架初始化代码
		// ...
		DoUserInit();
	}
private:
	virtual void DoUserInit() = 0;
};

class User :
	public Framework
{
	virtual void DoUserInit() override
	{
		// init
	}
};
// 全局对象
User user;
// 这样在程序启动时，可以根据注册的对象调用到用户的初始化代码。

```

---
# Attorney Client
友元代理，限制友元的访问权限。  
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

---
# Barton-Nackman trick [旧]
已过时  
具体查看原链接<https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Barton-Nackman_trick>  
当时提出时不能声明函数模板，现在已经没有这问题。  

---
# Base from Member
用派生类的成员初始化基类的成员。  
基类初始化的参数需要由子类提供时，因为C++的初始化顺序是基类在子类前初始化， 在传入子类成员时，如果此时子类成员需要初始化而基类在构造函数中使用到它，就会引发错误。  

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
  
可以用多重继承解决这个问题。把基类需要的变量另外放到另一个类中，然后在基类之前继承这个类，这样这个变量就会在基类之前初始化完毕，基类就能使用了。   
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
注意实现这个类时一般带一个UniqueID，因为当有多个同类型的成员时，如果不在模板做区分，将无法取得正确的成员（引用歧义）。  

Boost可以用类库`base_from_member`，其源码跟例子的很像，就不贴了。

---
# Mutant
转置结构的顺序而不通过物理内存修改。  
不修改内存左右翻转二叉树用的就是这个。就是重新定义一个结构，让左右变量定义交换；在转换时直接替换类型定义即可。  
好像Boost有专门的库实现这个，Bimap库也有用到这个方法。有一定局限性，但是在某些情况使用可以有极大的性能提升。  
很容易理解，不上例子了。 

---
# Calling Virtuals During Initialization
在初始化时调用虚函数。  
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

---
# Capability Query
在运行时检查对象是否支持某接口。  
dynamic_cast的应用，用if判断就好了，入门内容。  

---
# Checked delete
安全的delete对象。  

不安全原因：
```cpp
class Object;
void DestoryObject(Object* obj)
{
	delete obj;
}

class Object
{
public:
	~Object()
	{
        // 此时如果用DestoryObject析构对象，是无法执行到这里的
		cout << "dtor" << endl;
	}
};
```
在使用delete之前，如果对象未完整声明，会调用到默认的析构函数而跳过之后声明的析构函数。  

而如果在析构前加了一层模板函数：
```cpp
class Object;
template<typename T>
void Destory(T* ptr)
{
	delete ptr;
}
void DestoryObject(Object* obj)
{
	Destory(obj);
}
class Object
{
public:
	~Object()
	{
		cout << "dtor" << endl;
	}
};
```

这时候是可以调用到自定义析构的。  
在智能指针实现里就用到的这个方式（析构时）。比如STL里的源码：  
```cpp
// TEMPLATE CLASS default_delete
template<class _Ty>
struct default_delete
{	// default deleter for unique_ptr
	constexpr default_delete() _NOEXCEPT = default;

	template<class _Ty2,
		class = typename enable_if<is_convertible<_Ty2 *, _Ty *>::value,
		void>::type>
		default_delete(const default_delete<_Ty2>&) _NOEXCEPT
	{	// construct from another default_delete
	}

	void operator()(_Ty *_Ptr) const _NOEXCEPT
	{	// delete a pointer
		static_assert(0 < sizeof(_Ty),
			"can't delete an incomplete type");
		delete _Ptr;
	}
};
```
此时static_assert是可以通过的，但有时候用智能指针时，后文没有把类型定义完整（有可能是未引用这个类型的头文件而只做了前置声明），还是会无法编译（即使没有自定义析构），因为它用了那句静态断言。  
sizeof无法获取未定义类型的大小，此时会阻止编译（有些编译器会返回0，这样就会被断言阻断）。而已定义的类型返回总是大于0，所以会通过断言。  
所以在delete前加上断言检查就可以安全的delete对象。  

在Boost里提供的checked_delete函数就是用来干这事的。  
不过它阻止编译的方法不是用静态断言，而是用：
```cpp
template<class T> inline void checked_delete(T * x)
{
    // intentionally complex - simplification causes regressions
    typedef char type_must_be_complete[ sizeof(T)? 1: -1 ];
    (void) sizeof(type_must_be_complete);
    delete x;
}
```
如果sizeof可以求值，那么它在类型未定义时创建一个大小为-1的数组，这个语句会引起编译失败。（可能为了兼容性问题或者历史原因，没有用static_assert，但是新版stl里已经在用了）  
