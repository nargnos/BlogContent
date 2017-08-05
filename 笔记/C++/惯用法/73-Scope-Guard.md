---
title: "[C++ Idioms] 73. [易] Scope Guard"
date: 2017-08-04 16:04:14
categories: C++ Idioms
tags:
    - C++
    - Idioms


---
为RAII提供一些灵活性。<!--more-->用RAII时如果只想在出错或发生异常时释放资源，可以在类里添加一个flag，dtor根据flag选择是否释放，在正常退出时设置flag即可。