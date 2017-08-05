---
title: "[C++ Idioms] 55. [易] Non-throwing swap"
date: 2017-08-02 23:11:13
categories: C++ Idioms
tags:
    - C++
    - Idioms


---
无异常swap。<!--more-->类内部用一个指针管理相关内容，swap时交换指针就行。