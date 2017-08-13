---
title: "[C++ Idioms] 46. Member Detector"
date: 2017-08-02 18:46:03
categories: [学习笔记,C++ Idioms]
tags:
    - C++

    - 模板

---
检查类是否有指定成员。<!--more-->SFINAE的一个应用。  
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
