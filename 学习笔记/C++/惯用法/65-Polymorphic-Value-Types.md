---
title: "[C++ Idioms] 65. [补] Polymorphic Value Types"
date: 2017-08-04 13:24:41
categories: [学习笔记,C++ Idioms]
tags:
    - C++

    - 模板

---
为值类型添加运行时多态支持，并且不使用继承关系。<!--more-->原文没内容，只给了[视频](https://channel9.msdn.com/Events/GoingNative/2013/Inheritance-Is-The-Base-Class-of-Evil)和[PPT](https://github.com/boostcon/cppnow_presentations_2012/blob/master/fri/value_semantics/value_semantics.pdf?raw=true)。这里内容就自己YY。  

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