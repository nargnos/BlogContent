---
title: "[C++ Idioms] 34. [少] Generic Container Idioms"
date: 2017-08-01 18:11:38
categories: [学习笔记,C++ Idioms]
tags:
    - C++
    - Idioms


---
创建通用容器类。<!--more-->  
在创建通用容器类时，可能类型并没有默认的构造函数，或者类型不支持赋值或者有各种各样的问题。此时可以先分配空间，到添加对象时用`new (&place) Type`（placement new）处理即可。  

stl容器用到这个（部分源码）：  
```cpp
// 前面还有蛮多代码，这里是关键的
template<class _Objty,
	class... _Types>
	void construct(_Objty *_Ptr, _Types&&... _Args)
{	// construct _Objty(_Types...) at _Ptr
	::new ((void *)_Ptr) _Objty(_STD forward<_Types>(_Args)...);
}
```