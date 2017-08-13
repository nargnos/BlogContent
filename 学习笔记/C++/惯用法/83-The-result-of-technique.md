---
title: "[C++ Idioms] 83. [替] The result_of technique"
date: 2017-08-04 17:02:35
categories: [学习笔记,C++ Idioms]
tags:
    - C++

    - 模板

---
获取函数返回值。<!--more-->可以用result_of，stl源码如下：
```cpp
#define _RESULT_OF(CALL_OPT, X1, X2) \
template<class _Fty, \
	class... _Args> \
	struct result_of<_Fty CALL_OPT (_Args...)> \
		: _Result_of<void, _Fty, _Args...> \
	{	/* template to determine result of call operation */ \
	};

_NON_MEMBER_CALL(_RESULT_OF, , )
#undef _RESULT_OF
```
这里为方便定义各种参数压栈顺序，定义`_RESULT_OF`后在`_NON_MEMBER_CALL`设置stdcall什么的。
因为其定义是用类型作为参数，所以在使用时只能填类型而不能用变量，跟decltype是不同的。