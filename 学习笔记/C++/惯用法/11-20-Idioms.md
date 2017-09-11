---
title: "[C++ Idioms] 11-20"
date: 2017-07-30 12:06:17
categories: [学习笔记,C++ Idioms]
tags: [C++, 模板]
description: Clear and minimize、Coercion by Member Template、Computational Constructor、Concrete Data Type、Construct On First Use、Construction Tracker、Copy and swap、Copy on write、Counted Body/Reference Counting (intrusive)、Covariant Return Types

---
# Clear and minimize [替]
清除容器并回收内存。 
有一些容器会分配大于内容的内存，在clear时内存还会占用（capacity），所以此时要清掉这部分占用的空间。  
现代C++直接用`shrink_to_fit`处理，要清空就先执行clear即可。  

下面是旧用法：
```cpp
vector<int> vec(0x100);
vec.clear();
vec.swap(vector<int>());
vector<int>().swap(vec); // 也可
auto size = vec.capacity();
```

如果仅仅想把多占用的空间释放，而不想清除容器：
```cpp
vector<int> vec(0x100);	
vec.swap(vector<int>(vec));
vector<int>(vec).swap(vec); // 也可
auto size = vec.capacity();
```
临时对象的位置是无所谓的，不过印象中好像是不能用临时对象做参数的，所以才加了一些不必要的例子，难道新版改了？  

---
# Coercion by Member Template
让模板支持协变和抗变。 
在智能指针中有应用。  

例：
```cpp
template<typename T>
class Template{};

class A{};
class B:
	public A
{};
```
此时`Template<A>`、`Template<B>`是无法互相赋值的，因为它们是不同类型。  
在某些时候需要能在这两个类型中转换（智能指针），就需要这样写operator。  

```cpp
template<typename T>
class Template
{
public:
	template<typename U>
	Template& operator=(const Template<U>&){}
};

class A{};
class B:
	public A
{};
```
这样两个类型便可以互相操作了，只需要在函数里添加自己的实现即可。  

---
# Computational Constructor [易]
让不支持命名返回值优化的编译器支持返回值优化。 
直接把参数准备好，在return那里直接创建就好。不知道这有什么好单独成惯用法的。  
再不济直接把对象引用传到函数里来初始化也差不多。  

---
# Concrete Data Type [易]
强制让对象只支持栈或者堆分配。 
禁止栈分配就将析构设置为私有，堆是operator new。入门内容不上例子。  

---
# Construct On First Use [易]
延迟初始化静态变量。 
将静态变量变成函数内静态变量即可。  
在单例中可用到，因为新的标准规定，函数内静态变量是线程安全的（可以在编译器中关掉），所以直接用就好了。 

---
# Construction Tracker
区分构造函数中产生的同类型异常。 
很多书中建议初始化不要抛出异常，因为有可能无法恢复某些上下文。  
但如果我偏用，且在初始化列表初始化的成员中抛了异常，还抛同样的类型，如何区分是哪个成员在初始化时抛出的？  

在每个成员的初始化时给某个变量赋值标记该成员初始化，然后catch的时候就能获得对应信息。  
这里需要用到一些不常用的语法。构造函数的try catch和逗号表达式做参数传递。   
例：
```cpp
// 原文的例子本身足够好，只做很小的修改
struct A {
	A(char const *) { throw 123; }
};
struct B {
	B(int) { throw 456; }
};
class C {
	enum TrackerType
	{
		None,
		AInit,
		BInit,
		Done = BInit
	};
public:
	// 在构造函数后可以加try catch块，这个特性不太常用
	C(char const* args, TrackerType tracker = TrackerType::None) try :
		// 使用逗号表达式赋值tracker
		a_((tracker = TrackerType::AInit, "abc")),
		b_((tracker = TrackerType::BInit, 123))
	{
		// 确保初始化完毕
		assert(tracker == TrackerType::Done);
	}
	catch (int val)
	{
		// 此时可以根据tracker值做恢复工作，或者再throw
		switch (tracker)
		{
		case C::None:
			break;
		case C::AInit:
			break;
		case C::BInit:
			break;
		default:
			break;
		}
	}
private:
	A a_;
	B b_;
};
```

---
# Copy and swap [易]
创建赋值操作的异常安全实现。 
在重写 operator= 时，如果已经有异常安全的swap操作，直接用复制构造的临时对象+swap自身来达到赋值效果。  
stl的一些结构中会用到，比较普遍，不上例子了。  

---
# Copy on write [易]
写时复制。 
复制构造时保存数据指针，等到写入时再复制。很多地方会用到，比如fork、内存映射（可以设置写时复制标识）等。  

_据说stl的string里有实现这个，但是找了一遍没见到，调试时也没发现有写时复制的迹象，也跟踪了数据指针，发现都是新复制的，是不是漏看了什么……_  

关于捕获写操作，只要重写operator `->`、`.`就可以，写两个版本，一个返回const，一个没const，这样在写入时编译器会选择没const的版本，这样就可以捕获到写操作，数组的话重写`[]`就可以了。  
可以在重写operator时返回代理对象，这样更好编写一些。  

---
# Counted Body/Reference Counting (intrusive) [易]
享元模式的一个应用。 
给某个共享资源加counter，为0时析构掉，直接用个智能指针就行。   
“享元”已经能概括这个惯用法了，不再扩展。  

---
# Covariant Return Types [易]
返回值协变。 
C++支持返回值协变，不知道以前的C++版本支不支持。总之，这个惯用法是用到这个特性，  
在做原型的时候，clone函数从基类继承而来，到子类实现时可以把返回值修改成子类类型。  
比较简单，不做扩展。  
