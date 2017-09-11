---
title: "[C++ Idioms] 81-90"
date: 2017-08-05 12:14:53
categories: [学习笔记,C++ Idioms]
tags: [C++, 模板]
description: Temporary Base Class、Temporary Proxy、The result_of technique、Thin Template、Traits、Type Erasure、Type Generator、Type Safe Enum、Type Selection、Virtual Constructor

---
# Temporary Base Class
降低创建临时对象的成本。这个惯用法并不能消除或减少临时对象创建，只是将创建的成本降低。  
比如在创建某个算术类型时，需要重载其运算符，这类函数往往返回的是原本的类型，这个过程中会产生大量的临时对象消耗性能。此时可以识别出某些可以优化的运算节点实现降低临时对象创建成本。比如在做二元运算时，可以返回一个中间类型，用于存储运算结果，之后跟原类型的计算都可以附加在这个中间类型上，直到赋值给返回值时再转换成原类型，这个过程中就省掉了创建原类型临时对象的消耗。具体代码看原文，太长了不贴，这里只记录想法。  
或者可以用模板元编程来做，具体看ratio类。

---
# Temporary Proxy
观察跟踪operator []的行为。这个在之前的惯用法有提到一点，就是重写operator []返回一个代理对象，代理重写=（用于设置值）、operator Type（用于获取值），这样便可以跟踪其行为。
stl的bitset类用到，其它的哪些用没注意。

---
# The result_of technique [替]
获取函数返回值。可以用result_of，stl源码如下：
```cpp
#define _RESULT_OF(CALL_OPT, X1, X2) \
template<class _Fty, \
	class... _Args> \
	struct result_of<_Fty CALL_OPT (_Args...)> \
		: _Result_of<void, _Fty, _Args...> \
	{	/* template to determine result of call operation */ \
	};

_NON_MEMBER_CALL(_RESULT_OF, , )
#undef _RESULT_OF
```
这里为方便定义各种参数压栈顺序，定义`_RESULT_OF`后在`_NON_MEMBER_CALL`设置stdcall什么的。
因为其定义是用类型作为参数，所以在使用时只能填类型而不能用变量，跟decltype是不同的。

---
# Thin Template [易]
减小模板代码膨胀。一句话：将公共函数或不涉及到模板参数的内容提取到外部或父类中。

---
# Traits
类型萃取。原文没内容，但实在没什么可写的，网上有很多文章，不想写跟别人重复的东西。  
trait目的是获取类的特性，基本的类都在type_traits，boost的可能会多一些，有一些是直接用模板编写，有一些需要借助编译器内部函数实现。（相关定义和介绍以后贴在另一个地方。）  
借助编译器内部函数实现的基本上类库都有了，下面用一些简单例子举例用模板实现的。
想不出复杂的情况（懒得想），用这个简单的情况表达这个惯用法的意思。  
```cpp
// 假设ID是类的一些特有属性
struct A
{
	// 这里声明了ID，但实际并不一定所有的类都会声明
	// 当不声明时可以通过另一个类来定义关联
	static constexpr int ID = 0;
};
struct B {};
struct C {};

// 这里是默认，不需要时可只留声明
template<typename T>
struct Trait
{
	static constexpr int ID = -1;
};
// 特化
template<>
struct Trait<A>
{
	static constexpr int ID = A::ID;
};
template<>
struct Trait<B>
{
	static constexpr int ID = 1;
};

int main()
{
	// 实际用时是这样，还有很多用法，可以返回类型或判断等，
	// type_trait类库声明了很多用来判断类型的trait，不过一般都需要编译器内部函数
	// 一般配合模板编程使用
	auto a = Trait<A>::ID;
	auto b = Trait<B>::ID;
	auto c = Trait<C>::ID;
	return 0;
}
```
其它的一些用法在之前的惯用法中有提到一些，这里不列了。


---
# Type Erasure
类型擦除。就是用一个模板接收其类型然后用统一接口调用，这样就可以实现“擦除”类型，文字描述不太清楚，用之前的一些惯用法写一个（不用虚函数）（我不常用boost的any，不知道它的实现是怎样的，这里写的用法不一定对）：
```cpp

class TypeBase
{
public:
	using FuncPtr = void(*)(TypeBase*);
	TypeBase(FuncPtr func) :
		func_(func)
	{
	}
	void Func()
	{
		func_(this);
	}
private:
	FuncPtr func_;
};


template<typename T>
class Type :
	public TypeBase
{
public:
	explicit Type(T&& obj) :
		TypeBase(&FuncImpl),
		obj_(std::forward<T>(obj))
	{
	}
private:
	static void FuncImpl(TypeBase* ptr)
	{
		auto self = static_cast<Type<T>*>(ptr);
		Visit(self->obj_);
	}
	T obj_;
};


class Any
{
public:
	template<typename T>
	Any(T&& obj) :
		obj_(MakeObj(std::forward<T>(obj)))
	{
	}
	Any() {}
	void Visit()
	{
		assert(obj_);
		obj_->Func();
	}
	template<typename T>
	Any& operator=(T&& obj)
	{
		if (!std::is_same<T, Any>::value)
		{
			obj_ = MakeObj(std::forward<T>(obj));
		}

		return *this;
	}

private:
	template<typename T>
	std::unique_ptr<TypeBase> MakeObj(T&& obj)
	{
		return std::make_unique<Type<T>>(std::forward<T>(obj));
	}
	std::unique_ptr<TypeBase> obj_;
};
template<typename T>
void Visit(T&& val)
{
	cout << val << endl;
}

int main()
{
	Any any;
	any = 1 + 2 + 3;
	any.Visit();
	any = "hello world";
	any.Visit();
	any = [](std::ostream& out) ->std::ostream& {out << "end" << endl; return out; };
	any.Visit();
	return 0;
}
```

---
# Type Generator [易]
从原有模板类型生成新的类型。一般在偏特化模板时用来简化编程操作。
就是template using，可以设置一些固定参数，这样便可以用新的模板声明去生成类型。没有using可以用template struct，在类型内声明新的类型即可。

---
# Type Safe Enum [替]
类型安全的enum。现在已经可以用enum class。

---
# Type Selection [替]
编译时选择类型。之前很多惯用法都有涉及到，这里只上stl源码：
```cpp
// TEMPLATE CLASS conditional
template<bool _Test,
	class _Ty1,
	class _Ty2>
	struct conditional
{	// type is _Ty2 for assumed !_Test
	typedef _Ty2 type;
};

template<class _Ty1,
	class _Ty2>
	struct conditional<true, _Ty1, _Ty2>
{	// type is _Ty1 for _Test
	typedef _Ty1 type;
};
```

---
# Virtual Constructor [易]
在创建类型时不需要知道它的具体类型。原型模式，不扩展。
