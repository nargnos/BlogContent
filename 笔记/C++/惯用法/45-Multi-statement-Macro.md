---
title: "[C++ Idioms] 45. [-] Multi-statement Macro"
date: 2017-08-02 15:13:19
categories: C++ Idioms
tags:
    - C++
    - Idioms
---
写多行宏定义的技巧。<!--more-->
在写多行宏定义时：
```cpp
#define MACRO(X,Y) { statement1; statement2; }
```
这样使用就会无法编译，因为后面多了个分号（其实遵循总写大括号的风格根本不会有这种问题）：
```cpp
if (cond)
   MACRO(10,20); // Compiler error here because of the semi-colon.
else
   statement3;
```
解决方案是用 do{}while(false) 把这多行语句包括起来，这样无论哪种风格都不会出问题。