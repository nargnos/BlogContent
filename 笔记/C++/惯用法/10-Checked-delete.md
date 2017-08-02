---
title:  "[C++ Idioms] 10. Checked delete"
date: 2017-07-30 16:29:26
categories: C++ Idioms
tags:
    - C++
    - Idioms
---
安全的delete对象。<!--more-->  

不安全原因：
```cpp
class Object;
void DestoryObject(Object* obj)
{
	delete obj;
}

class Object
{
public:
	~Object()
	{
        // 此时如果用DestoryObject析构对象，是无法执行到这里的
		cout << "dtor" << endl;
	}
};
```
在使用delete之前，如果对象未完整声明，会调用到默认的delete过程而跳过之后声明的析构函数。  

而如果在析构前加了一层模板函数：
```cpp
class Object;
template<typename T>
void Destory(T* ptr)
{
	delete ptr;
}
void DestoryObject(Object* obj)
{
	Destory(obj);
}
class Object
{
public:
	~Object()
	{
		cout << "dtor" << endl;
	}
};
```

这时候是可以调用到自定义析构的。  
在智能指针实现里就用到的这个方式。比如STL里的源码：  
```cpp
// TEMPLATE CLASS default_delete
template<class _Ty>
struct default_delete
{	// default deleter for unique_ptr
	constexpr default_delete() _NOEXCEPT = default;

	template<class _Ty2,
		class = typename enable_if<is_convertible<_Ty2 *, _Ty *>::value,
		void>::type>
		default_delete(const default_delete<_Ty2>&) _NOEXCEPT
	{	// construct from another default_delete
	}

	void operator()(_Ty *_Ptr) const _NOEXCEPT
	{	// delete a pointer
		static_assert(0 < sizeof(_Ty),
			"can't delete an incomplete type");
		delete _Ptr;
	}
};
```
此时static_assert是可以通过的，但有时候用智能指针时，后文没有把类型定义完整，还是会无法编译（即使没有自定义析构），因为它用了那句静态断言。sizeof无法获取未定义类型的大小，此时会阻止编译（有些编译器会返回0，这样就会被断言阻断）。而已定义的类型返回总是大于0，所以会通过断言。  
所以在delete前加上断言检查就可以安全的delete对象。  

在Boost里提供的checked_delete函数就是用来干这事的。  
不过它阻止编译的方法不是用静态断言，而是用：
```cpp
template<class T> inline void checked_delete(T * x)
{
    // intentionally complex - simplification causes regressions
    typedef char type_must_be_complete[ sizeof(T)? 1: -1 ];
    (void) sizeof(type_must_be_complete);
    delete x;
}
```
如果sizeof可以求值，那么它在类型未定义时创建一个大小为-1的数组，这个语句会引起编译失败。（可能为了兼容性问题或者历史原因，没有用static_assert，但是新版stl里已经在用了）  

