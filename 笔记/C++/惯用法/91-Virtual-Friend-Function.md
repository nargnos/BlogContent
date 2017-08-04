---
title: "[C++ Idioms] 91. [-] Virtual Friend Function"
date: 2017-08-04 21:28:54
categories: C++ Idioms
tags:
    - C++
    - Idioms
---
让子类能够实用父类的友元函数。<!--more-->友元函数参数是父类，内容用模板方法的方式编写，这样便可以在子类使用父类的友元函数。