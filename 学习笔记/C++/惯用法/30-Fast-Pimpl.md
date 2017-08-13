---
title: "[C++ Idioms] 30. [补] Fast Pimpl"
date: 2017-08-01 14:41:20
categories: [学习笔记,C++ Idioms]
tags:
    - C++



---
加快Pimpl的速度。<!--more-->  
这个在原文也是没内容的，自己YY一个。因为用字典序的关系，这个排得靠前，所以相关内容其实在Pimpl里。  
在使用Pimpl的时候，new会有性能上的问题，解决方法是在impl中定义静态内存池，重写new和delete使用内存池分配，这样实例化时可以加快，又保留了Pimpl的特性。  
