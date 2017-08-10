---
title: "[C++ Idioms] 85. Traits"
date: 2017-08-04 17:24:29
categories: [学习笔记,C++ Idioms]
tags:
    - C++
    - Idioms
    - 模板

---
类型萃取。<!--more-->原文没内容，但实在没什么可写的，网上有很多文章，不想写跟别人重复的东西。  
trait目的是获取类的特性，基本的类都在type_traits，boost的可能会多一些，有一些是直接用模板编写，有一些需要借助编译器内部函数实现。（相关定义和介绍以后贴在另一个地方。）  
借助编译器内部函数实现的基本上类库都有了，下面用一些简单例子举例用模板实现的。
想不出复杂的情况（懒得想），用这个简单的情况表达这个惯用法的意思。  
```cpp
// 假设ID是类的一些特有属性
struct A
{
	// 这里声明了ID，但实际并不一定所有的类都会声明
	// 当不声明时可以通过另一个类来定义关联
	static constexpr int ID = 0;
};
struct B {};
struct C {};

// 这里是默认，不需要时可只留声明
template<typename T>
struct Trait
{
	static constexpr int ID = -1;
};
// 特化
template<>
struct Trait<A>
{
	static constexpr int ID = A::ID;
};
template<>
struct Trait<B>
{
	static constexpr int ID = 1;
};

int main()
{
	// 实际用时是这样，还有很多用法，可以返回类型或判断等，
	// type_trait类库声明了很多用来判断类型的trait，不过一般都需要编译器内部函数
	// 一般配合模板编程使用
	auto a = Trait<A>::ID;
	auto b = Trait<B>::ID;
	auto c = Trait<C>::ID;
	return 0;
}
```
其它的一些用法在之前的惯用法中有提到一些，这里不列了。