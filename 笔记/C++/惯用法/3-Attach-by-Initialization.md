---
title: "[C++ Idioms] 3. Attach by Initialization"
date: 2017-07-29 16:29:10
categories: C++ Idioms
tags:
    - C++
    - Idioms
---
在程序启动时用静态对象预先初始化相关内容，由于框架细节不便提供给用户修改，将用户自定义的初始化内容“延迟”到子对象实现，这是一种模板方法的使用例子。<!-- more -->  
注意在使用时如果跨编译单元引用静态对象的内容，结果是不确定的，因为对于不同编译单元静态对象的初始化顺序，标准里并没有明确。   

例
```c++
class Framework
{
public:
	Framework()
	{
		// 向某对象注册
	}
	void Init()
	{
		// 框架初始化代码
		// ...
		DoUserInit();
	}
private:
	virtual void DoUserInit() = 0;
};

class User :
	public Framework
{
	virtual void DoUserInit() override
	{
		// init
	}
};
// 全局对象
User user;
// 这样在程序启动时，可以根据注册的对象调用到用户的初始化代码。

```