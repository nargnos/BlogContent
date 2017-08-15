---
title: "[C++ Idioms] 80. [补] Tag Dispatching"
date: 2017-08-04 16:34:11
categories: [学习笔记,C++ Idioms]
tags:
    - C++



---
类似int to type。<!--more-->
现在的原文没内容，但是以前记了一些笔记是这个标题的（跟int to type一起的），不是完全能保证是这个的内容。
在某些情况用if判断的时候，可能有一个分支是无法过编译的，这时可以用这个技巧把选择的值具象化成类型，用参数重载匹配来做。
使用的原理是SFINAE。

将值具化成类，传入dummy参数，实现对不同值的重载，比如`true_type` `false_type`
```cpp
template <int I>
struct Int2Type
{
  enum { value = I };
};

template <class T, unsigned int N>
class Array : public std::array <T, N>
{
   enum AlgoType { NOOP, INSERTION_SORT, QUICK_SORT };
   static const int algo = (N==0) ? NOOP : 
                           (N==1) ? NOOP :
			   (N<50) ? INSERTION_SORT : QUICK_SORT;
   void sort (Int2Type<NOOP>) { std::cout << "NOOP\n"; }
   void sort (Int2Type<INSERTION_SORT>) { std::cout << "INSERTION_SORT\n"; }
   void sort (Int2Type<QUICK_SORT>) { std::cout << "QUICK_SORT\n"; }
 public:
   void sort()
   {
     sort (Int2Type<algo>());
   }
};
int main(void)
{
  Array<int, 1> a;
  a.sort(); // No-op!
  Array<int, 400> b;
  b.sort(); // Quick sort  
}
```
将不同的值具化为类型可以实现不同值重载
还可以这样
```
template <int I>
struct Int2Type
{
  enum { value = I };
  typedef int value_type;
  typedef Int2Type<I> type;
  typedef Int2Type<I+1> next;
  typedef Int2Type<I-1> previous;
};
```


稍微扩展一下，上面说得不太清楚，比如：
```cpp
struct A
{
	static int Calc(int val) { return 2 * val; }
};

struct B
{
	static int Calc2(int val) { return 2 + val; }
};
class MyClass
{
public:
	template<typename T>
	int Do(int val)
	{
		if (std::is_same<T, A>::value)
		{
			return T::Calc(val); // error
		}
		return T::Calc2(val); // error
	}
};
int main(void)
{
	MyClass m;
	m.Do<A>(2);
	return 0;
}
```
这样写是无法过编译的，因为即使用不到后面的内容（类中的第二个return）但它仍属于编译时要检查的东西，此时可以把类改成这样，让编译器选择合适的版本：
```cpp
class MyClass
{
public:
	template<typename T>
	int Do(int val)
	{
		return Calc<T>(val);
	}
private:
	template<typename T>
	std::enable_if_t<std::is_same<T, A>::value, int> Calc(int val)
	{
		return T::Calc(val);
	}
	
	template<typename T>
	std::enable_if_t<std::is_same<T, B>::value, int> Calc(int val)
	{
		return T::Calc2(val);
	}
};
```

简单点就是：
假设有一个函数：
```cpp
void func(bool val)
{
	if (val)
	{
		// do xxx
	}
	else
	{
		// do yyy
	}
}
```
在某些情况xxx、yyy不能同时被编译，或者想把这两个操作分开，这时候可以选择用这个方法：
```cpp
void func(true_type)
{
	// do xxx
}
void func(false_type)
{
	// do yyy
}
// 调用时
void x()
{
	func(true_type{});
}
```