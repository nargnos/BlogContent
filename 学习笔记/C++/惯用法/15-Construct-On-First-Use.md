---
title: "[C++ Idioms] 15. [易] Construct On First Use"
date: 2017-07-30 18:03:17
categories: [学习笔记,C++ Idioms]
tags:
    - C++



---
延迟初始化静态变量。<!--more-->  
将静态变量变成函数内静态变量即可。  
在单例中可用到，因为新的标准规定，函数内静态变量是线程安全的（可以在编译器中关掉），所以直接用就好了。  