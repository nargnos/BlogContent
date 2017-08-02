---
title: "[C++ Idioms] 52. [-] Nifty Counter"
date: 2017-08-02 23:03:38
categories: C++ Idioms
tags:
    - C++
    - Idioms
---
确保非本地静态对象能够在使用前初始化并在最后才析构。<!--more-->  
设置一个counter，用一个类来管理，跟shared_ptr一样，只不过初始化的时候用placement new的方式。
据原文说cout、cin、cerr、clog使用，但是查看stl没找到（这部分没有源码只有定义）。
