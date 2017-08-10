---
title: "[C++ Idioms] x1. lookup table"
date: 2017-08-04 21:58:07
categories: [学习笔记,C++ Idioms]
tags:
    - C++
    - Idioms


---
用查表法代替判断。<!--more-->这不属于原出处内容，我也不记得从哪记来的了，这属于优化方法。当判断是bool且值不变的话可以这样，可以加快分支执行速度。
```cpp
if (b) a = c;
else a = d;

static const type lookup_table[] = { d, c };
a = lookup_table[b];

if (b1) a = c;
else if (b2) a = d;
else if (b3) a = e;
else a = f;

static const type lookup_table[] = { f, e, d, d, c, c, c, c };
a = lookup_table[b1 * 4 + b2 * 2 + b3];
```
对于某些情况是可以加快的，但如果需要查表多次才能获得一个结果时，就需要斟酌使用了。  