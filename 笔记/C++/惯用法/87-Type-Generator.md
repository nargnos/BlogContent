---
title:  "[C++ Idioms] 87. [-] Type Generator"
date: 2017-08-04 20:49:37
categories: C++ Idioms
tags:
    - C++
    - Idioms
    - 模板
---
从原有模板类型生成新的类型。<!--more-->一般在偏特化模板时用来简化编程操作。
就是template using，可以设置一些固定参数，这样便可以用新的模板声明去生成类型。没有using可以用template struct，在类型内声明新的类型即可。