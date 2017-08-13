---
title: "[C++ Idioms] 23. [替] Enable if"
date: 2017-07-31 14:29:49
categories: [学习笔记,C++ Idioms]
tags:
    - C++

    - 模板

---
让编译器根据条件选择合适的函数进行编译。<!--more-->  
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
