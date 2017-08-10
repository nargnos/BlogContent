---
title:  "[C++ Idioms] 31. [替] Final Class"
date: 2017-08-01 17:56:25
categories: [学习笔记,C++ Idioms]
tags:
    - C++
    - Idioms


---
禁止继承类。<!--more-->  
现在在声明后加final即可。  
之前的方法是将父类构造函数声明为私有，设置要继承的类为友元即可。  