---
title: "[C++ Idioms] 64. [易] Polymorphic Exception"
date: 2017-08-03 19:54:10
categories: C++ Idioms
tags:
    - C++
    - Idioms


---
多态异常。<!--more-->用异常基类捕获异常，需要注意catch用引用捕获，不扩展了。