---
title: "[C++ Idioms] 17. [易] Copy and swap"
date: 2017-07-30 18:39:17
categories: C++ Idioms
tags:
    - C++
    - Idioms


---
创建赋值操作的异常安全实现。<!--more-->  
在重写 operator= 时，如果已经有异常安全的swap操作，直接用复制构造的临时对象+swap自身来达到赋值效果。  
stl的一些结构中会用到，比较普遍，不上例子了。  