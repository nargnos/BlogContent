---
title: "[C++ Idioms] 74. [易] Substitution Failure Is Not An Error"
date: 2017-08-04 16:09:38
categories: [学习笔记,C++ Idioms]
tags:
    - C++
    - Idioms
    - 模板

---
不将匹配失败视为一个错误（SFINAE）。<!--more-->这个算是模板的入门级知识。  
通过构建匹配失败来让编译器选择合适的重载编译。  
如果同时存在就会选择最匹配的。  
匹配失败的情况有：引用不存在（如：enable if）、引用不明确（Member Detector里有相关内容）等。可测试“匹配失败”的地方有：模板参数、函数参数、函数返回值等。  

下面是扩展

-------------
对于用模板选择合适的函数，有两种编写方式：同时存在、一方不编译。
下面写一些例子  
匹配失败的情况：
```cpp
// 调用时用 Func<type>(); 就可以区分调用
// 函数有参数情况类似，不用写明确的模板参数
// 每个方法不必对应使用，它们之间是等效的

// 做模板默认参数，需要写默认值，否则调用时就要传值了
// 后面是什么类型不重要，主要是enable_if引发的匹配失败，所以后面只要给一个默认值就行了
template<typename T, enable_if_t<is_void<T>::value>* = nullptr>
void Func()
{ }
template<typename T, enable_if_t<!is_void<T>::value>* = nullptr>
void Func()
{ }


// 做返回值
template<typename T>
enable_if_t<is_void<T>::value> Func()
{ }
template<typename T>
enable_if_t<!is_void<T>::value> Func()
{ }

// 做函数默认参数
template<typename T>
void Func(enable_if_t<is_void<T>::value>* = nullptr)
{ }
template<typename T>
void Func(enable_if_t<!is_void<T>::value>* = nullptr)
{ }
```

同时存在的情况：
```cpp
template<typename T>
struct Trait;

template<>
struct Trait<int> :
  public integral_constant<int, 0>
{ };
template<>
struct Trait<double> :
  public integral_constant<int, 1>
{
};
template<>
struct Trait<char> :
  public integral_constant<int, 2>
{
};

template<typename T>
void Func()
{
  _Func<Trait<T>::value>();
}
template<int>
void _Func();

// 模板特化重载
template<>
void _Func<0>()
{
}
template<>
void _Func<1>()
{
}
template<>
void _Func<2>()
{
}
```
或者可以用tag区分
```cpp
template<int i>
void Func()
{
  using Type = conditional_t<(i > 2), integral_constant<int, 0>, integral_constant<int, 2>>;
  _Func(Type());
}
void _Func(integral_constant<int, 0>)
{
}
void _Func(integral_constant<int, 2>)
{
}
```