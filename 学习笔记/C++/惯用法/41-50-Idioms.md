---
title: "[C++ Idioms] 41-50"
date: 2017-08-1 21:46:00
categories: [学习笔记,C++ Idioms]
tags: [C++, 模板]
description: Iterator Pair、Making New Friends、Metafunction、Move Constructor、Multi-statement Macro、Member Detector、Named Constructor、Named External Argument、Named Loop、Named Parameter


---
# Iterator Pair [易]
迭代器。可以使使用者不必知道容器的实现细节。用得很普遍，不扩展。

---
# Making New Friends [易]
简化模板类友元函数的创建。
就是直接在模板里声明friend函数。原文大概说的就是这意思。

---
# Metafunction
元函数。元编程一般用于编译时选择类型或在编译期计算或构造一些数据；元函数是编写这类算法的主要方式。Stl提供的比较少（好像只有conditional，可以用来应付一些简单的元编程），Boost的MPL库提供了很多用于元编程的类、函数、容器。  

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

---
# Move Constructor [替]
移动构造。原文内容是在没有移动语义时如何实现移动，现在已经有相关支持。  

---
# Multi-statement Macro
写多行宏定义的技巧。
在写多行宏定义时：
```cpp
#define MACRO(X,Y) { statement1; statement2; }
```
这样使用就会无法编译，因为后面多了个分号（其实遵循总写大括号的风格根本不会有这种问题）：
```cpp
if (cond)
   MACRO(10,20); // Compiler error here because of the semi-colon.
else
   statement3;
```
解决方案是用 do{}while(false) 把这多行语句包括起来，这样无论哪种风格都不会出问题。

---
# Member Detector
检查类是否有指定成员。SFINAE的一个应用。  
核心是构建引用歧义使匹配失败。一般是构建成员不存在的方式使匹配失败。  

相关代码和解释如下：
```cpp
template<typename T>
class has_member_MemberName
{
private:
	struct Fallback
	{
		// 声明要测试的成员
		void* MemberName;
	};
	// 如果可以从T继承来同名成员的话，因为同级有两个成员存在
	// 在引用这个成员时会发生引用不明确的错误	
	struct Derived :T, Fallback {};

	// 因为优先选择匹配度更高的，所以当引用明确时会优先选择这个
	// 这里的函数只需要其返回值，不需要函数体（类似declval）
	template<typename U>
	static std::false_type Check(decltype(Derived::MemberName)*);
	// 当引用不明确时会选择这个
	template<typename U>
	static std::true_type Check(U*);
public:
	static constexpr bool value = decltype(Check<Derived>(nullptr))::value;
};

```

上面这个不能用在union或final class的情况，下面是自己的改进版（用成员不存在的方法，未在所有编译器测试，不过好像不能用在私有成员的情况）：
```cpp
template<typename T>
class has_member_MemberName
{
private:
	template<typename U>
	static std::true_type Check(decltype(U::MemberName)*);
	template<typename U>
	static std::false_type Check(U*);

public:
	static constexpr bool value = decltype(Check<T>(nullptr))::value;
};
```

因为使用时需要在类声明里提供成员名字，所以这里用宏定义生成新的类型：
```cpp
#define GENERATE_HAS_MEMBER(name) \
template<typename T>\
class has_member_##name\
{\
private:\
	template<typename U>\
	static std::true_type Check(decltype(U::##name)*);\
	template<typename U>\
	static std::false_type Check(U*);\
public:\
	static constexpr bool value = decltype(Check<T>(nullptr))::value;\
};

GENERATE_HAS_MEMBER(test);
```
对于检查类是否有特定类型声明，也用的是同样的方法，只不过不用decltype而已。  
还可以用来检查函数和其参数（不解释了，看懂上面很容易理解这个）：
```cpp
template<typename T, typename TResult, typename... TArgs>
class HasPolicy
{
	template <typename U, TResult(U::*)(TArgs...)>
	struct Check
	{};

	template <typename U>
	static std::true_type Test(Check<U, &U::FuncName>*);

	template <typename U>
	static std::false_type Test(...);

public:
	static constexpr bool value = decltype(Test<T>(nullptr))::value;
};
```

---
# Named Constructor [易]
命名构造函数，用不同的命名函数生成对象。原文的意思就是工厂。

---
# Named External Argument [缺]


---
# Named Loop
用goto实现 break label/continue label。
源码如下（原文例子）：
```cpp
#define named(blockname) goto blockname; \
                         blockname##_skip: if (0) \
                         blockname:

#define break(blockname) goto blockname##_skip
```
很容易理解。

---
# Named Parameter
命名参数。用在有多个可选参数要设置的情况。  

用链式函数的方式，例：
```cpp
// 简单的例子，有很多不好的地方，只表达出意思，写完整费时间
class MyClass
{
public:
	MyClass& SetA(int val)
	{
		a_ = val;
		return *this;
	}
	MyClass& SetB(char val)
	{
		b_ = val;
		return *this;
	}
private:
	int a_;
	char b_;
};


int main(void)
{
	auto c = MyClass().SetA(1).SetB(2);
	return 0;
}
```
还可以在编写Set函数时返回的是一个代理类，用来详细设置配置信息。
比如 
```cpp
MyType *instance = segment.construct<MyType>
         ("MyType instance")  //name of the object
         (0.0, 0);
MyType *array = segment.construct<MyType>
         ("MyType array")     //name of the object
         [10]                 //number of elements
         (0.0, 0);            //Same two ctor arguments for all objects
```
这里不扩展了，boost里有一些类就是用到这个（interprocess）。
类库中的用法是，在创建对象的同时为对象设置名字。  