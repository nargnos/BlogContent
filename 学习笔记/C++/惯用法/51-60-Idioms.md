---
title: "[C++ Idioms] 51-60"
date: 2017-08-02 22:52:42
categories: [学习笔记,C++ Idioms]
tags: [C++, 模板]
description: Named Template Parameters、Nifty Counter、Non-copyable Mixin、Non-member Non-friend Function、Non-throwing swap、Non-Virtual Interface、Nullptr、Object Generator、Object Template、Parameterized Base Class


---
# Named Template Parameters [缺]


---
# Nifty Counter
确保非本地静态对象能够在使用前初始化并在最后才析构。  
设置一个counter，用一个类来管理，跟shared_ptr一样，只不过初始化的时候用placement new的方式。
据原文说cout、cin、cerr、clog使用，但是查看stl没找到（这部分没有源码只有定义）。


---
# Non-copyable Mixin [易]
不可拷贝。boost有类库，就是把拷贝相关删掉或者私有。

---
# Non-member Non-friend Function [缺]


---
# Non-throwing swap [易]
无异常swap。类内部用一个指针管理相关内容，swap时交换指针就行。

---
# Non-Virtual Interface
非虚接口(NVI)。不是指不用虚表的接口，而是指将需要多态实现的一些步骤声明为protected或private的虚函数，并在子类实现（模板方法），而基类对外提供的接口是非虚的。  
一些书里有讨论到这个用法，使不使用看个人喜好。可以视为模板方法的一个实现方式，由于提取了一些公有代码，在某些情况可以减少代码编译的大小。  
```cpp
class Base
{
public:
	// 对外提供这个接口
	void Do()
	{
		// do ...
		Do2();
		// do ...
	}
private:
	virtual void Do2() = 0;
};
class MyClass :public Base
{
	virtual void Do2() override
	{
		// do ...
	}
};
```

---
# Nullptr [替]
空指针。现在已用nullptr替换。
在C中有些地方会用`((void *)0)`，但是赋值的时候会有问题（如`char* p=((void *)0);`）；如果使用0，在重载时便无法区分是数字0还是空指针。

以前没有这个关键字时，为了强调空指针的特性，会用：
```cpp
const // It is a const object...
class nullptr_t 
{
  public:
    template<class T>
    inline operator T*() const // convertible to any type of null non-member pointer...
    { return 0; }

    template<class C, class T>
    inline operator T C::*() const   // or any type of null member pointer...
    { return 0; }

  private:
    void operator&() const;  // Can't take address of nullptr

} nullptr = {};
```

现在可以这样用（这里为空指针参数做了一个重载）：
```cpp
struct MyStruct
{
	void Do(nullptr_t){}
	void Do(int) {}
	void Do(void*) {}
};

int main(void)
{
	MyStruct x;
	x.Do(nullptr);
	x.Do(0);
	x.Do(&x);

	return 0;
}
```

---
# Object Generator
简化对象创建而不需要指定其类型（模板参数）。在某些时候定义类时会设置很多复杂的模板参数，这时候实例化类就要填很多参数来指定其类型，如：
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


---
# Object Template [缺]


---
# Parameterized Base Class
给类添加特性。把一些公有的代码提取出来，并且使用继承模板参数类型的方式封装成类，这样可以给类添加新特性。说不清楚，上代码：  
```cpp
template<typename T>
class GetName :
	public T
{
public:
	const char* Name()
	{
		return typeid(T).name();
	}
};
template<typename T>
class GetSize :
	public T
{
public:
	size_t Size()
	{
		return sizeof(T);
	}
};


class MyClass
{
};

int main(void)
{
	GetName<MyClass> obj;
	auto name = obj.Name();
	GetSize<GetName<MyClass>> obj2;
	obj2.Name();
	obj2.Size();
	return 0;
}

```
这个例子为类添加获取名字、大小的特性，还可以像原文例子那样，给类添加可序列化特性。
使用继承可以保留基类类型，可以向原类型转换。可以在特性类中附加接口继承什么的，这样就可以方便的转换类为其它接口类型。  