---
title: "[C++ Idioms] 21-30"
date: 2017-07-30 12:14:09
categories: [学习笔记,C++ Idioms]
tags: [C++, 模板]
description: Curiously Recurring Template Pattern、Empty Base Optimization、Enable if、Erase Remove、Execute-Around Pointer、Exploding Return Type、Export Guard Macro、Expression Template、Fake Vtable、Fast Pimpl


---
# Curiously Recurring Template Pattern
这就是很多惯用法都用到的CRTP。这是去除虚表的一种方法。 
这是一个很重要的惯用法。其核心就是父类的模板参数是子类类型，这样在父类里就可以强制转换指针为子类类型，而使用子类的函数（隐式接口）。  

在某些情况可以被编译器优化从而获得性能提升。（在现在这种速度比较快的机器下，运行个一亿次会比使用虚表快个几百毫秒，但并不是说这个惯用法没用，用得好速度会有可观的提升）  
其核心结构为：
```cpp
template<typename TDerived>
class Base
{
public:
	void Do()
	{
		auto self = static_cast<TDerived*>(this);
		self->DoImpl();
	}
};

class Derived:
	public Base<Derived>
{
public:
	void DoImpl()
	{
	}
};
```

可以将成员函数声明为静态，配合其它一些技巧使用有奇效。  
WTL源码里有用到，Boost的很多类库也都会用到，比如Asio源码里也有用到（是这个结构的改进版）。  

CRTP由于基类用模板形式，如果有多个子类就会生成多个不同类型的父类，不方便做统一管理；这时候可以加一个中间层Caller用来调用子类的函数，或者用伪虚表的方式来处理。  

---
# Empty Base Optimization [易]
优化掉空类/结构体的占用空间。 
空类型（无成员变量类型）最小大小是1，如果用来做类的成员变量（可能用到这些类型定义的函数），会使类增加一些不必要的大小。  
这个时候可以用private或者protected继承，这样便可以省掉这些占用空间。  
stl和boost有用到一些，不过好像不太好找到。  

unique_ptr里，保存deleter就是用这个方法保存的，内部结构保存指针的同时继承了deleter，因为deleter通常只提供析构方式，是一个无成员变量的类型。  

stl源码：
```cpp
template<class _Ty1,
	class _Ty2,
	bool = is_empty<_Ty1>::value && !is_final<_Ty1>::value>
	class _Compressed_pair final
	: private _Ty1

{
	// ...
};
```
其中的 _Ty1 就是deleter的类型，在使用它的函数时直接调用自身继承而来的函数即可。  

---
# Enable if [替]
让编译器根据条件选择合适的函数进行编译。 
基于SFINAE（Substitution Failure Is Not An Error）原理，在编译器寻找合适的模板时，不会将匹配不上视为一种错误，而会继续找直到没有匹配。  

一般需要配合type trait使用（也可以用其它），主要是构造一系列判断条件，当判断失败时引发匹配失败，这样编译器就可以选择匹配成功的版本进行编译。但是涉及到SFINAE的时候，编译器可能会出现一些BUG，这需要注意不要使用错误语法。   

stl、boost中可以用`enable_if`、`enable_if_t`这两个类。  

用到了模板特化的性质，一般定义如下：
```cpp
template<bool Condition, typename T = void>
struct EnableIf
{};
template<typename T>
struct EnableIf<true, T>
{
	// false没有Type这个定义，目的是使匹配失败
	using Type = T;
};
// 为了方便使用，一般会提供这个
template<bool Condition, typename T = void>
using EnableIfType = typename EnableIf<Condition, T>::Type;

```

Condition是构造的条件，后面是符合条件时导出的类型。  

stl的源码如下：
```cpp
// TEMPLATE CLASS enable_if
template<bool _Test,
	class _Ty = void>
	struct enable_if
{	// type is undefined for assumed !_Test
};

template<class _Ty>
struct enable_if<true, _Ty>
{	// type is _Ty for _Test
	typedef _Ty type;
};

template<bool _Test,
	class _Ty = void>
	using enable_if_t = typename enable_if<_Test, _Ty>::type;
```


使用时构造出选择条件让编译器选择。  
```cpp
// 例子，不止双项，还可以构造出多项选择
template<typename T>
EnableIfType<is_pod<T>::value> Func(T&)
{}
template<typename T>
EnableIfType<!is_pod<T>::value> Func(T&)
{}
template<typename T>
EnableIfType<(sizeof(T) > sizeof(int))> Func2(const T&)
{}
template<typename T>
EnableIfType<(sizeof(T) == sizeof(int))> Func2(const T&)
{}
template<typename T>
EnableIfType<(sizeof(T) < sizeof(int))> Func2(const T&)
{}

// stl一样
// 不一定要把条件涉及到的都写完整，最终匹配失败会引发编译失败，这在某些情况是有用的
template<int Flag>
enable_if_t<(Flag == 1), int> Func3()
{}
template<int Flag>
enable_if_t<(Flag == 2), char> Func3()
{}
// 可用在参数中
template<typename T>
void Func4(T val, enable_if_t<is_arithmetic<T>::value>* = nullptr)
{}
template<typename T>
void Func4(T val, enable_if_t<!is_arithmetic<T>::value>* = nullptr)
{}
// 可用在模板参数中，这里一般用于单项区分
template<typename T, typename = enable_if_t<is_arithmetic<T>::value>>
void Func5(T val)
{}
```

常见错误用法：
```cpp
// 在使用这个结构时，这个语法表示的是，同时在同一上下文声明同样特征的重载
// 会引发编译失败，因为enable_if_t在匹配之前已经求值了
template<typename T>
struct MyStruct
{
	enable_if_t<is_pod<T>::value> Func() {} // ERROR
	enable_if_t<!is_pod<T>::value> Func() {} // ERROR
};
```

---
# Erase Remove [易]
从容器中消除元素。 
std::remove 不会把数据移除，它只是把要移除的内容放到容器末尾。因为它接收的参数是迭代器，并不知道如何将数据从容器中删除。  
所以在删除vector的内容时，用remove处理后要用erase真正删除元素。  
在很多书里都提到这个，不再扩展。  

---
# Execute-Around Pointer
跟踪对象的函数调用。 
使用这个方法可以方便的在调用对象的任何函数前后加入想要执行的代码。  
这是代理模式的一个应用。  

一般形式如下：
```cpp
template<typename T>
class Proxy
{
	Proxy(T* obj):obj_(obj)
	{
		// 自定义执行函数前的代码
	}
	~Proxy()
	{
		// 自定义执行函数后的代码
	}
	T* operator->()
	{
		return obj_;
	}
private:
	T* obj_;
};
template<typename T>
class Logger
{
public:
	Logger(T* obj) :obj_(obj)
	{
	}
	T* operator->()
	{
		return Proxy<T>(obj_);
	}
private:
	T* obj_;
};

```
核心是创建一个代理类，重写operator->，在代理类的ctor、dtor写入自定义代码即可。  
不必担心创建过多代理对象，因为这可以被编译器优化。  
可以扩展重写其它运算符，比如`[]`之类的，还可以用来管理对象的函数调用，设置函数执行权限什么的。  

---
# Exploding Return Type [易]
用返回值返回异常或错误代码。 
这个在原文中没内容。查了一下原文附带的文档，指的可能是创建一个复合对象（union），可带错误码和返回值，通过判断是否发生错误，获取返回值或错误码。  
个人觉得并不实用，因为实现这个复合对象是要花时间的（而且会引入复杂度），可以改为多返回值（用tuple），这样使用更加方便。  

---
# Export Guard Macro [缺]

---
# Expression Template
在C++中创建一个域特定的嵌入式语言（DSEL）。
可以对表达式延迟求值，支持函数式编程等。在没有lambda时可以用这个方法实现。    

对于延迟求值或者函数式编程，现在可以用lambda表达式解决（用高阶函数）。  
例：
```cpp
// 某些旧的编译器可能无法编译这个
auto Func()
{
	return [](auto x, auto y)
	{
		auto val = x + y;
		return val*val;
	};
}

int main(void)
{
	auto func = Func();
	auto ret = func(2, 3);
	cout << ret << endl;
	return 0;
}
```

如果用类来实现的话比较麻烦（没能完全掌握），全模板的话还比较简单，但是用模板生成后面带参运行的表达式树就不太好弄了。现在直接用lambda表达式就好，不需要搞这么多东西。  
感兴趣的可以去看Boost的Peonix库。  

_尝试编了点简单的例子，原文的例子用的方法不太好，所以不按原文的例子；这是生成伪函数然后延迟求值。_   

例：
```cpp
// 篇幅较长就省掉了一些内容
// 因为是例子，所以复杂化实现，其实几行代码可以解决
template<typename T>
struct Expression
{
	template<T Val>
	struct ConstantExpression
	{
		constexpr T operator()()
		{
			return Val;
		}
	};
	class VariableExpression
	{
	public:
		VariableExpression(T val) :val_(val) {}
		T operator()()
		{
			return val_;
		}
	private:
		T val_;
	};


	template<typename TL, typename TR>
	class AddOp
	{
	public:
		AddOp(TL l, TR r) :
			l_(l), r_(r) {}
		T operator()()
		{
			return l_() + r_();
		}
	private:
		TL l_;
		TR r_;
	};

	template<typename TL, typename TR>
	class MultiOp
	{
	public:
		MultiOp(TL l, TR r) :
			l_(l), r_(r) {}
		T operator()()
		{
			return l_() * r_();
		}
	private:
		TL l_;
		TR r_;
	};
	// 这个用法可能某些编译器无法编译
	// x^2 + 2xy + y^2
	static auto Func(T xVal, T yVal)
	{
		// 把结构弄得复杂点做例子
		using E = Expression<int>;
		// x*y x*x
		using VarMult = E::MultiOp<E::VariableExpression, E::VariableExpression>;
		// 2*x*y
		using VarX2 = E::MultiOp<E::ConstantExpression<2>, VarMult>;
		// x*x + 2*x*y
		using Xp2AVarX2 = E::AddOp<VarMult, VarX2>;
		// x^2 + 2xy + y^2
		using Final = E::AddOp<Xp2AVarX2, VarMult>;

		E::VariableExpression x(xVal);
		E::VariableExpression y(yVal);

		VarMult xp2(x, x);
		VarMult yp2(y, y);
		VarMult xy(x, y);

		VarX2 xy2(E::ConstantExpression<2>(), xy);
		Xp2AVarX2 res0(xp2, xy2);
		return Final(res0, yp2);
	}
};


int main(void)
{
	using E = Expression<int>;
	auto result = E::Func(2, 3);

	auto ret = result();
	cout << ret << endl;
	return 0;
}
```

---
# Fake Vtable [补]
伪虚表。 
这个惯用法原文是空的没有内容，那我自己根据名字YY一个。  
这是看一些类库的源代码总结出来的。  
nginx源码里面有用到，就是声明一个结构体，保存“对象”要用到的函数指针，在需要重写（或替换）函数时，直接把相应的函数指针替换掉，这样相当于用C实现了虚表机制。  

在Asio的源码里有C++方法实现的伪虚表机制（用来存Ops），结构如下：
```cpp
class Base
{
public:
	using DoSth = int(*)(Base* self, int arg);
	explicit Base(DoSth doSth) :
		doSth_(doSth)
	{
	}
	int Do(int arg)
	{
		return doSth_(this, arg);
	}
	// 为了保证正确析构只能这样，否则自己实现太麻烦了
	virtual ~Base() = default;
private:
	DoSth doSth_;
};

class MyClass :
	public Base
{
public:
	// 这里用来支持再继承
	MyClass(DoSth doSth, int val) :
		Base(doSth),
		val_(val)
	{
	}
	explicit MyClass(int val) :
		MyClass(&MyDoSth, val)
	{
	}
protected:
	int val_;
private:
	static int MyDoSth(Base* ptr, int arg)
	{
		auto self = static_cast<MyClass*>(ptr);
		return self->val_*arg;
	}
};
class MyClass2 :
	public MyClass
{
public:
	MyClass2(int val) :
		MyClass(&MyDoSth, val)
	{
	}

private:
	static int MyDoSth(Base* ptr, int arg)
	{
		auto self = static_cast<MyClass2*>(ptr);
		return self->val_ + arg;
	}
};

int main(void)
{
	MyClass c(10);
	Base* b = &c;
	auto ret = b->Do(20);
	cout << ret << endl;

	MyClass2 c2(10);
	b = &c2;
	ret = b->Do(20);
	cout << ret << endl;

	return 0;
}
```
在类内部维护函数指针，从构造函数传入，有重写就替换掉，这样便实现了虚表机制。  
在某些情况是可以被编译器优化的，跟用虚表的速度差不了多少（根据寻址关系来看会比虚表快一点，但是实际测试是差不多的，因为现在的机器太快了）。这个惯用法可以用来处理CRTP的基类需要模板参数的问题。  
对于某些需要用function的情况，如果用模板方式存储回调，也可以用这个方式提取公共父类统一处理。这个惯用法对于一些有虚表洁癖的人真是福音啊。  

---
# Fast Pimpl [补]
加快Pimpl的速度。 
这个在原文也是没内容的，自己YY一个。因为用字典序的关系，这个排得靠前，所以相关内容其实在Pimpl里。  
在使用Pimpl的时候，new会有性能上的问题，解决方法是在impl中定义静态内存池，重写new和delete使用内存池分配，这样实例化时可以加快，又保留了Pimpl的特性。  