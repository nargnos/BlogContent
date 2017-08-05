---
title: "[C++ Idioms] 43. Metafunction"
date: 2017-08-02 13:10:05
categories: C++ Idioms
tags:
    - C++
    - Idioms
    - 模板

---
元函数。<!--more-->元编程一般用于编译时选择类型或在编译期计算或构造一些数据；元函数是编写这类算法的主要方式。Stl提供的比较少（好像只有conditional，可以用来应付一些简单的元编程），Boost的MPL库提供了很多用于元编程的类、函数、容器。  

只会一些简单的元编程，**现在会的技术东西不足以解读MPL库**，很大一个原因是不知道宏定义怎么和模板配合在一起，而且因为无法调试，所以比较难分析，待之后有能力再补充。

这是简单的使用例子：
```cpp
struct value_printer
{
	template< typename U > void operator()(U x)
	{
		std::cout << x << endl;
	}
};

int main()
{
	using hello = mpl::string<'Hel', 'lo!'>;
	cout << mpl::c_str<hello>::value << endl;
	using vec = mpl::vector_c<int, 1, 9, 2, 8, 3, 7, 4, 6, 5>;
	using res = mpl::sort<vec>::type;
	mpl::for_each<res>(value_printer());
}
```