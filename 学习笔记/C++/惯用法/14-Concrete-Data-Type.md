---
title: "[C++ Idioms] 14. [易] Concrete Data Type"
date: 2017-07-30 17:57:59
categories: [学习笔记,C++ Idioms]
tags:
    - C++



---
强制让对象只支持栈或者堆分配。<!--more-->  
禁止栈分配就将析构设置为私有，堆是operator new。入门内容不上例子。  