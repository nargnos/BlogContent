---
title: "[C++ Idioms] 22. Empty Base Optimization"
date: 2017-07-30 22:21:22
categories: C++ Idioms
tags:
    - C++
    - Idioms
---
优化掉空类/结构体的占用空间。<!--more-->  
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
