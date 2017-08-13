---
title: "[C++ Idioms] 89. [替] Type Selection"
date: 2017-08-04 21:23:24
categories: [学习笔记,C++ Idioms]
tags:
    - C++



---
编译时选择类型。<!--more-->之前很多惯用法都有涉及到，这里只上stl源码：
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