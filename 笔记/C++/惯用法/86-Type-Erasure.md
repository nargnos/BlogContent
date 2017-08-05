---
title: "[C++ Idioms] 86. Type Erasure"
date: 2017-08-04 19:37:58
categories: C++ Idioms
tags:
    - C++
    - Idioms
    - 模板

---
类型擦除。<!--more-->就是用一个模板接收其类型然后用统一接口调用，这样就可以实现“擦除”类型，文字描述不太清楚，用之前的一些惯用法写一个（不用虚函数）（我不常用boost的any，不知道它的实现是怎样的，这里写的用法不一定对）：
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