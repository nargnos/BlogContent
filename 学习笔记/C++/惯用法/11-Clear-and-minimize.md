---
title: "[C++ Idioms] 11. [替] Clear and minimize"
date: 2017-07-30 17:01:39
categories: [学习笔记,C++ Idioms]
tags:
    - C++



---
清除容器并回收内存。<!--more-->  
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
