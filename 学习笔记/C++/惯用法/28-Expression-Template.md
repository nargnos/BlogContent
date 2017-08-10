---
title: "[C++ Idioms] 28. Expression Template"
date: 2017-07-31 18:40:24
categories: [学习笔记,C++ Idioms]
tags:
    - C++
    - Idioms
    - 模板

---
在C++中创建一个域特定的嵌入式语言（DSEL）。<!--more-->
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



