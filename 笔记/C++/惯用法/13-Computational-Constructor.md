---
title: "[C++ Idioms] 13. [-] Computational Constructor"
date: 2017-07-30 17:51:45
categories: C++ Idioms
tags:
    - C++
    - Idioms
---
让不支持命名返回值优化的编译器支持返回值优化。<!--more-->  
直接把参数准备好，在return那里直接创建就好。不知道这有什么好单独成惯用法的。  
再不济直接把对象引用传到函数里来初始化也差不多。  