---
title: "[C++ Idioms] 68. [易] Resource Acquisition Is Initialization"
date: 2017-08-04 14:31:36
categories: C++ Idioms
tags:
    - C++
    - Idioms


---
资源获取就是初始化（RAII）。<!--more-->使用对象生命周期来控制资源生命周期。主要是使用对象的ctor和dtor，在ctor获取资源（所以叫RAII）， 在对象（可以是栈对象或堆对象）生命周期结束时会自动执行dtor实现释放资源，可以避免泄露和简化编程操作。  
网上相关的内容已经很多了，很容易理解，这里不扩展。