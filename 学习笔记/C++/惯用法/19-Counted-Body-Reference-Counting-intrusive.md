---
title: "[C++ Idioms] 19. [易] Counted Body/Reference Counting (intrusive)"
date: 2017-07-30 19:28:04
categories: [学习笔记,C++ Idioms]
tags:
    - C++



---
享元模式的一个应用。<!--more-->  
给某个共享资源加counter，为0时析构掉，直接用个智能指针就行。   
“享元”已经能概括这个惯用法了，不再扩展。  
