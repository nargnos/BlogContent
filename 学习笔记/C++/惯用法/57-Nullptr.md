---
title:  "[C++ Idioms] 57. [替] Nullptr"
date: 2017-08-03 12:41:19
categories: [学习笔记,C++ Idioms]
tags:
    - C++
    - Idioms


---
空指针。<!--more-->现在已用nullptr替换。
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