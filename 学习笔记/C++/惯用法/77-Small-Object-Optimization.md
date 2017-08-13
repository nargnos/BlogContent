---
title: "[C++ Idioms] 77. [补] Small Object Optimization"
date: 2017-08-04 16:24:36
categories: [学习笔记,C++ Idioms]
tags:
    - C++



---
优化类型擦除的效率。<!--more-->原文没内容，YY。原文只给了一句提示，在function中实现类型擦除，但是比堆分配更快。  
曾经简单的看过function代码，它好像是这么做的（或者是看string代码记混了）：在类内部存储有一个等于指针大小的数组，如果设置的类型大于这个数组，就new类型并在数组里存储这个类型的指针，否则就直接用placement new在这个数组里存储这个类型。  
这样当对象是小对象时就不用new了，提升了速度。