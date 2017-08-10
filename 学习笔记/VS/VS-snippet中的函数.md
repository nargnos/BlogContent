---
title: VS snippet中的函数
date: 2017-08-05 21:14:58
categories: [学习笔记,VS]
tags: VS
---
编写snippet中会用到的一些函数。<!--more-->

功能|说明|可用的语言
---|---|---
GenerateSwitchCases( EnumerationLiteral )|为 EnumerationLiteral 参数指定的枚举成员生成一个 switch 语句和一组 case 语句。<br> EnumerationLiteral 参数必须是对枚举文本或枚举类型的引用。|Visual C#
ClassName()|返回包含已插入代码段的类的名称。|Visual C#
SimpleTypeName( TypeName )|在已调用该代码段的上下文中将 TypeName 参数缩减为其最简单的形式。|Visual C#，貌似c++不能用
