---
title: "[C++ Idioms] 84. [易] Thin Template"
date: 2017-08-04 17:13:57
categories: [学习笔记,C++ Idioms]
tags:
    - C++
    - Idioms
    - 模板

---
减小模板代码膨胀。<!--more-->一句话：将公共函数或不涉及到模板参数的内容提取到外部或父类中。