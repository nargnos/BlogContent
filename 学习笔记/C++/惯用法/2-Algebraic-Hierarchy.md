---
title: "[C++ Idioms] 2. [易] Algebraic Hierarchy"
date: 2017-07-29 15:46:16
categories: [学习笔记,C++ Idioms]
tags:
    - C++
    - Idioms


---
目的是隐藏对象操作细节，可以通过一些简单的接口调用。  <!-- more --> 
比如`complex`复数库（高中内容i^2=-1的复数），**重载运算符**使之能通过这些运算符完成一些比较复杂的操作。  
```c++
complex<int> i(0, 1);
auto res = i * i; // res == complex<int>(-1)
```
其它的一些库多多少少都有一些应用，比如stream、string等。  
