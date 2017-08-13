---
title: "[C++ Idioms] 37. [易] Inline Guard Macro"
date: 2017-08-01 19:18:13
categories: [学习笔记,C++ Idioms]
tags:
    - C++



---
使用宏定义控制函数的内联。  
就是定义一个inline的别名，用ifdef子类的做开关，然后在函数前用这个别名即可。  
据原文说是用在调试的时候，这样可以快速的切换状态方便调试。  