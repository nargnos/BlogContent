---
title: "[C++ Idioms] 58. Object Generator"
date: 2017-08-03 14:39:24
categories: [学习笔记,C++ Idioms]
tags:
    - C++

    - 模板

---
简化对象创建而不需要指定其类型（模板参数）。<!--more-->在某些时候定义类时会设置很多复杂的模板参数，这时候实例化类就要填很多参数来指定其类型，如：
```cpp
template<typename T>
class MyClass
{
public:
	MyClass(T val) {}
	void operator()()
	{
		obj_();
	}
private:
	T obj_;
};

void Func()
{
	// 这时候要填写详细的参数类型
	auto obj = new MyClass<void()>(&Func);
}
```

如果此时创建一个函数用来识别参数并实例化类，这样创建该对象就会很方便：
```cpp
template<typename T>
MyClass<T>* Create(T val)
{
	return new MyClass<T>(val);
}
void Func2()
{
	auto obj = Create(&Func);
}
```

stl里很多类都会用，其目的就是为了方便生成对象。
