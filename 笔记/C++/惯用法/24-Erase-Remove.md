---
title: "[C++ Idioms] 24. [易] Erase Remove"
date: 2017-07-31 15:29:59
categories: C++ Idioms
tags:
    - C++
    - Idioms


---
从容器中消除元素。<!--more-->  
std::remove 不会把数据移除，它只是把要移除的内容放到容器末尾。因为它接收的参数是迭代器，并不知道如何将数据从容器中删除。  
所以在删除vector的内容时，用remove处理后要用erase真正删除元素。  
在很多书里都提到这个，不再扩展。  