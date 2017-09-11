---
title: "[C++ Idioms] 71-80"
date: 2017-08-04 15:56:36
categories: [学习笔记,C++ Idioms]
tags: [C++, 模板]
description: Runtime Static Initialization Order Idioms、Safe Bool、Scope Guard、Substitution Failure Is Not An Error、Shortening Long Template Names、Shrink to fit、Small Object Optimization、Smart Pointer、Storage Class Tracker、Tag Dispatching

---
# Runtime Static Initialization Order Idioms [易]
控制静态对象的初始化顺序。以前提过了（Construct On First Use、Nifty Counter Idiom），这里不重复了。

---
# Safe Bool [易]
为类提供布尔测试。代码：
```cpp
struct Testable
{
    explicit operator bool() const {
          return false;
    }
};
```

---
# Scope Guard [易]
为RAII提供一些灵活性。用RAII时如果只想在出错或发生异常时释放资源，可以在类里添加一个flag，dtor根据flag选择是否释放，在正常退出时设置flag即可。

---
# Substitution Failure Is Not An Error [易]
不将匹配失败视为一个错误（SFINAE）。这个算是模板的入门级知识。  
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

---
# Shortening Long Template Names [补]
缩短模板名。原文没内容，估计是重新定义一个template struct导出该模板或者用template using重新命名之类的。

---
# Shrink to fit [易]
将容器的内存占用缩小到跟元素个数一样的大小。
Clear and minimize已经写了，不再扩展。

---
# Small Object Optimization [补]
优化类型擦除的效率。原文没内容，YY。原文只给了一句提示，在function中实现类型擦除，但是比堆分配更快。  
曾经简单的看过function代码，它好像是这么做的（或者是看string代码记混了）：在类内部存储有一个等于指针大小的数组，如果设置的类型大于这个数组，就new类型并在数组里存储这个类型的指针，否则就直接用placement new在这个数组里存储这个类型。  
这样当对象是小对象时就不用new了，提升了速度。

---
# Smart Pointer [易]
智能指针。不需要扩展。

---
# Storage Class Tracker [缺]


---
# Tag Dispatching [补]
类似int to type。
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