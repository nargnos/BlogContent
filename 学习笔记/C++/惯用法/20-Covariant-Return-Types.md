---
title:  "[C++ Idioms] 20. [易] Covariant Return Types"
date: 2017-07-30 21:59:32
categories: [学习笔记,C++ Idioms]
tags:
    - C++



---
返回值协变。<!--more-->  
C++支持返回值协变，不知道以前的C++版本支不支持。总之，这个惯用法是用到这个特性，  
在做原型的时候，clone函数从基类继承而来，到子类实现时可以把返回值修改成子类类型。  
比较简单，不做扩展。  