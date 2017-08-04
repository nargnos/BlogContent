---
title: "[C++ Idioms] 74. [-] Substitution Failure Is Not An Error"
date: 2017-08-04 16:09:38
categories: C++ Idioms
tags:
    - C++
    - Idioms
    - 模板
---
不将匹配失败视为一个错误（SFINAE）。<!--more-->这个算是模板的入门级知识。通过构建匹配失败来让编译器选择合适的重载编译。匹配失败的情况有：引用不存在（如：enable if）、引用不明确（Member Detector里有相关内容）等。可测试“匹配失败”的地方有：模板参数、函数参数、函数返回值等。这里不举例了，前面很多惯用法都有。