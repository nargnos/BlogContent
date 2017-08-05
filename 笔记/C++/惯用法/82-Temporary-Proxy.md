---
title: "[C++ Idioms] 82. [少] Temporary Proxy"
date: 2017-08-04 16:48:44
categories: C++ Idioms
tags:
    - C++
    - Idioms


---
观察跟踪operator []的行为。<!--more-->这个在之前的惯用法有提到一点，就是重写operator []返回一个代理对象，代理重写=（用于设置值）、operator Type（用于获取值），这样便可以跟踪其行为。
stl的bitset类用到，其它的哪些用没注意。