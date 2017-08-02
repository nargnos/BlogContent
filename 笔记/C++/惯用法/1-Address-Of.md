---
title: "[C++ Idioms] 1. [+] Address Of"
date: 2017-07-29 15:11:44
categories: C++ Idioms
tags:
    - C++
    - Idioms
---
获取对象的真实地址，用在对象重载&的情况。  
STL、Boost使用现成的函数： addressof  <!-- more -->

----------------------
使用情况如下，此时对该类型对象取地址会导致编译失败。  
```c++
class nonaddressable
{
public:
	typedef double useless_type;
private:
	useless_type operator&() const;
};
```

源码如下，**核心就是那句转换**。  
添加const volatile的原因是在转换用const或volatile修饰的引用时，如果不加上这个标识，用`reinterpret_cast`是无法通过编译的（转换操作的特性），所以这里先转为cv引用，再去掉cv标识（因为应该按模板参数捕获到的类型返回，而`reinterpret_cast`是可以添加cv修饰的），然后再取地址转换为对应返回类型指针（此时返回类型如果有带cv修饰会再被加上去）。

```c++
/**
*  @brief Same as C++11 std::addressof
*  @ingroup utilities
*/
template<typename _Tp>
inline _Tp*
__addressof(_Tp& __r) _GLIBCXX_NOEXCEPT
{
	return reinterpret_cast<_Tp*>
		(&const_cast<char&>(reinterpret_cast<const volatile char&>(__r)));
}

/**
*  @brief Returns the actual address of the object or function
*         referenced by r, even in the presence of an overloaded
*         operator&.
*  @param  __r  Reference to an object or function.
*  @return   The actual address.
*/
template<typename _Tp>
inline _Tp*
addressof(_Tp& __r) noexcept
{
	return std::__addressof(__r);
}

```


另一处的源码，添加了对于函数类型的判断，但可从源码看出有builtin的函数可用，修饰也添加了constexpr。
```c++
// TEMPLATE FUNCTION addressof
#ifdef __EDG__ /* TRANSITION, VSO#200707 */
template<class _Ty> inline
_Ty *_Addressof(_Ty& _Val, true_type) _NOEXCEPT
{	// return address of function _Val
	return (_Val);
}

template<class _Ty> inline
_Ty *_Addressof(_Ty& _Val, false_type) _NOEXCEPT
{	// return address of object _Val
	return (reinterpret_cast<_Ty *>(
		&const_cast<char&>(
			reinterpret_cast<const volatile char&>(_Val))));
}

template<class _Ty> inline
_Ty *addressof(_Ty& _Val) _NOEXCEPT
{	// return address of _Val
	return (_Addressof(_Val, is_function<_Ty>()));
}
#else /* __EDG__ */
template<class _Ty> inline
constexpr _Ty *addressof(_Ty& _Val) _NOEXCEPT
{	// return address of _Val
	return (__builtin_addressof(_Val));
}
#endif /* __EDG__ */

```


