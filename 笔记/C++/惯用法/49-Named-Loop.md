---
title: "[C++ Idioms] 49. [少] Named Loop"
date: 2017-08-02 21:28:32
categories: C++ Idioms
tags:
    - C++
    - Idioms


---
用goto实现 break label/continue label。<!--more-->
源码如下（原文例子）：
```cpp
#define named(blockname) goto blockname; \
                         blockname##_skip: if (0) \
                         blockname:

#define break(blockname) goto blockname##_skip
```
很容易理解。