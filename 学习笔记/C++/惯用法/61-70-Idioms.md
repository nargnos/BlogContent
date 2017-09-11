---
title: "[C++ Idioms] 61-70"
date: 2017-08-03 17:33:05
categories: [学习笔记,C++ Idioms]
tags: [C++, 模板]
description: Pointer To Implementation（Pimpl）、Policy Clone、Policy-based Design、Polymorphic Exception、Polymorphic Value Types、Recursive Type Composition、Requiring or Prohibiting Heap-based Objects、Resource Acquisition Is Initialization、Resource Return、Return Type Resolver


---
# Pointer To Implementation（Pimpl）
将类的实现与其使用者隔离。隔离后的类在独自修改时，不会使其它使用到它的类被重新编译（编译防火墙），还可以对类的使用者隐藏实现。  
这个惯用法有很多文章都会讨论（也给它起了很多名字），但实际看一些代码，其实用得不多（可能是因为要实现调用转发要重新编写一次接口太累了，至少我这样认为）。在速度上是会降一些，但是在可接受范围之内。  

大体上结构是这样的：
```cpp
// 只传达意思，省去了会混淆的内容
class Impl
{
public:
	void Do()
	{
		cout << "DO" << endl;
	}
};

class PImpl
{
public:
	PImpl()
	{
		// 可以直接new，或者用对象池
		impl_ = new Impl();
	}
	~PImpl()
	{
		// 做一些回收工作，这里略
	}
	// 实现接口，实现转发调用，但是如果有很多函数的话就很累了（可以用写脚本生成代码的方式生成）
	void Do()
	{
		impl_->Do();
	}
private:
	Impl* impl_;
};

```
这样在Impl变化的时候，由于其它类型引用的是PImpl，所以不会使其它类被重新编译，在这个类被普遍使用的时候且这个项目很大时可以节约很多编译时间。

---
# Policy Clone
模板策略克隆。让已经绑定到模板的策略类重新绑定到新的类型，在某些不能直接取得类的策略模板原型的时候，通过这个惯用法可以获得新的绑定类型。
如：
```cpp
template<typename T>
struct Policy
{
	static void Print()
	{
		cout << sizeof(T) << endl;
	}
};

template<typename T, typename TPolicy = Policy<T>>
class MyClass
{
public:
	MyClass()
	{
		TPolicy::Print();
		// 假设此时想修改TPolicy的绑定类型（模板参数），是无法修改的，因为不知道它的原始类型
	}
};

```

此时修改成这样便可以解决：
```cpp
struct Policy
{
	static void Print()
	{
		cout << sizeof(T) << endl;
	}
	template<typename U>
	using Rebind = Policy<U>;
};

template<typename T, typename TPolicy = Policy<T>>
class MyClass
{
public:
	MyClass()
	{
		TPolicy::Print();
		// 重绑定
		TPolicy::Rebind<double>::Print();
	}
};
```

---
# Policy-based Design [补]
基于策略设计。原文无内容，这是YY。  
这是模板的策略模式，不同于一般的策略模式实现，这里用模板参数作为策略，用法如下：
```cpp

struct AddPolicy
{
	static int Calc(int val)
	{
		return val + val;
	}
};
struct MultiPolicy
{
	static int Calc(int val)
	{
		return val * val;
	}
};

// 调用时可以传入不同的策略来影响计算结果
template<typename TPolicy = AddPolicy>
int Calc(int val)
{
	int ret = val;
	// 做其它事情
	++ret;
	// 执行类策略
	ret = TPolicy::Calc(ret);
	// 做其它事情
	++ret;
	return ret;
}

template<typename TPolicy = DefaultPolicy>
class MyClass
{
public:
	int DoCalc(int val)
	{
		// 这里借用上面的函数
		return Calc<TPolicy>(val);
	}
};
```
一般策略类的函数可以声明为static，也可以不是静态（在需要保存状态的情况，这时候需要创建相关对象），stl用得很多，比如各种容器可以传入自定义的分配器，这个就是模板策略。

---
# Polymorphic Exception [易]
多态异常。用异常基类捕获异常，需要注意catch用引用捕获，不扩展了。

---
# Polymorphic Value Types [补]
为值类型添加运行时多态支持，并且不使用继承关系。原文没内容，只给了[视频](https://channel9.msdn.com/Events/GoingNative/2013/Inheritance-Is-The-Base-Class-of-Evil)和[PPT](https://github.com/boostcon/cppnow_presentations_2012/blob/master/fri/value_semantics/value_semantics.pdf?raw=true)。这里内容就自己YY。  

先来说视频内容。先放代码：
```cpp
void draw(const string& x, ostream& out, size_t position)
{
	out << string(position, ' ') << x << endl;
}
void draw(const int& x, ostream& out, size_t position)
{
	out << string(position, ' ') << x << endl;
}

class object_t {
public:
	template <typename T>
	object_t(const T& x) :
		object_(make_shared<model<T>>(x))
	{
	}
	friend void draw(const object_t& x, ostream& out, size_t position)
	{
		x.object_->draw_(out, position);
	}
private:
	struct concept_t {
		virtual ~concept_t() = default;
		virtual void draw_(ostream&, size_t) const = 0;
	};
	template <typename T>
	struct model : concept_t {
		model(const T& x) : data_(x) { }
		void draw_(ostream& out, size_t position) const
		{
			draw(data_, out, position);
		}
		T data_;
	};
	shared_ptr<const concept_t> object_;
};
using document_t = vector<object_t>;
void draw(const document_t& x, ostream& out, size_t position) {
	out << string(position, ' ') << "<document>" << endl;
	for (auto& e : x) draw(e, out, position + 2);
	out << string(position, ' ') << "</document>" << endl;
}
using history_t = vector<document_t>;
void commit(history_t& x)
{
	assert(x.size());
	x.push_back(x.back());
}
void undo(history_t& x)
{
	assert(x.size()); x.pop_back();
}
document_t& current(history_t& x)
{
	assert(x.size()); return x.back();
}
class my_class_t {    /* ... */ };
void draw(const my_class_t&, ostream& out, size_t position)
{
	out << string(position, ' ') << "my_class_t" << endl;
}

int main() {

	history_t h(1);
	current(h).emplace_back(0);
	current(h).emplace_back(string("Hello!"));
	draw(current(h), cout, 0);
	cout << "--------------------------" << endl;
	commit(h);
	current(h).emplace_back(current(h));
	current(h).emplace_back(my_class_t());
	current(h)[1] = string("World");
	draw(current(h), cout, 0);
	cout << "--------------------------" << endl;
	undo(h);
	draw(current(h), cout, 0);
}
```
在编写photoshop的历史记录功能时，需要记录每个操作步骤，此时如果将每个步骤的结果图片都存储那是不可能的（内存占用会暴涨，而且还有多个图层），所以这里用命令模式设计，只记录操作命令；然后他们提出了一种技巧，可以在保留历史记录的同时可对之前的记录进行选择性重放或修改（历史记录画笔功能），修改后还能返回之前未修改的记录；视频中的主要内容就是讨论这个和如何优化在复制历史命令时产生的复制构造消耗性能的问题。
上面代码就是解决方案，`document_t`可以看作是文档的历史记录（`vector<command>`，保存了各操作命令），`history_t`是文档历史记录的历史记录（`vector<document_t>`），在完成某个操作时，会使用`commit`复制一份`document_t`，其后的操作都在新历史记录版本上操作，此时可以对复制版本做任何修改而不影响到之前的记录，在撤销时直接把当前历史丢掉，就得到了旧版本历史。对于解决复制拷贝的问题，用`shared_ptr`存储命令就行了（享元），复制时只复制相应指针，修改历史记录时只是把对应记录的指针换成指向其它命令对象。  

那么这个视频内容跟这个惯用法有什么关系呢？原文和PPT后带了一个类库[链接](http://stlab.adobe.com/group__poly__related.html)，研究了一下，它可能是指，在类型擦除后仍可对这个类型执行相对应的重载函数，其核心结构可能是`model`里的`draw_`，看了一下poly源码，似乎也有点这个意思，不过例子太少不太会用这个类库，所以先放下，以后技术长进再回来研究。

---
# Recursive Type Composition [缺]
编译时递归类型定义。原文无内容，附带的链接是一篇论文，但是我无法查看原文（貌似注册蛮麻烦的）。  
本来想YY一些，但是看了论文的摘要，感觉说的不是我想YY的东西，所以……此篇无内容。

---
# Requiring or Prohibiting Heap-based Objects [易]
让对象只能在堆或栈中创建。以前写过了，这里应该是原文内容重复了。

---
# Resource Acquisition Is Initialization [易]
资源获取就是初始化（RAII）。使用对象生命周期来控制资源生命周期。主要是使用对象的ctor和dtor，在ctor获取资源（所以叫RAII）， 在对象（可以是栈对象或堆对象）生命周期结束时会自动执行dtor实现释放资源，可以避免泄露和简化编程操作。  
网上相关的内容已经很多了，很容易理解，这里不扩展。

---
# Resource Return [易]
在工厂函数返回时明确返回资源的所有权特征。在工厂返回对象时有时候会返回指针，此时可能会不知道是由工厂管理对象的生存周期还是此时转交给用户管理，这样会造成对象所有权不明确，此时用智能指针返回就行了（这也提醒我们在编写工厂的时候不要只是返回一个对象指针，因为使用者不知道该由谁管理其生存期，除非写注释）。

---
# Return Type Resolver
推导被初始化或被赋值的对象的类型。又是一个使用operator Type特性的惯用法：
```cpp
class MyClass
{
public:
	template<typename T>
	operator T()
	{
		return T();
	}
};

int main()
{
	int x = MyClass();
	void* y = MyClass();
	return 0;
}
```
将operator Type声明为模板函数可以检测到赋值或者构造的类型。

应用如下：
```cpp
class Range
{
public:
	Range(int begin, int end, int step = 1) :
		begin_(begin),
		end_(end),
		step_(step)
	{
		// 范围检查略
	}
	Range(int end) :
		Range(0, end, 1)
	{
	}
	template<typename TContainer>
	operator TContainer()
	{
		TContainer ret;
		auto inserter = std::back_inserter(ret);
		for (int i = begin_; i < end_; i += step_)
		{
			inserter = i;
		}
		return ret;
	}
private:
	int begin_;
	int end_;
	int step_;
};

int main()
{
	std::vector<int> v = Range(10);
	std::list<char> l = Range(5, 20, 2);
	return 0;
}
```