---
title: "[C++ Idioms] 72. [易] Safe Bool"
date: 2017-08-04 15:58:54
categories: [学习笔记,C++ Idioms]
tags:
    - C++



---
为类提供布尔测试。<!--more-->代码：
```cpp
struct Testable
{
    explicit operator bool() const {
          return false;
    }
};
```