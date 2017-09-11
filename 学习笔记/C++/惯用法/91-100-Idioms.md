---
title: "[C++ Idioms] 91-100"
date: 2017-08-05 15:32:25
categories: [学习笔记,C++ Idioms]
tags:
    - C++
description: Virtual Friend Function、Lookup Table

---
_原文没那么多内容，记笔记时只到91，这里附加一些其它内容_

# Virtual Friend Function [易]
让子类能够使用父类的友元函数。友元函数参数是父类，内容用模板方法的方式编写，这样便可以在子类使用父类的友元函数。

---
# Lookup Table
用查表法代替判断。这不属于原出处内容，我也不记得从哪记来的了，这属于优化方法。当判断是bool且值不变的话可以这样，可以加快分支执行速度。
```cpp
if (b) a = c;
else a = d;

static const type lookup_table[] = { d, c };
a = lookup_table[b];

if (b1) a = c;
else if (b2) a = d;
else if (b3) a = e;
else a = f;

static const type lookup_table[] = { f, e, d, d, c, c, c, c };
a = lookup_table[b1 * 4 + b2 * 2 + b3];
```
对于某些情况是可以加快的，但如果需要查表多次才能获得一个结果时，就需要斟酌使用了。  

---
# do-while-break
有do-while的语言都可以用，一般在这种情况用：
```cpp
do
{
    if (condition)
    {
        break;
    }
    // ...
    if (condition)
    {
        break;
    }
    // ...
    // ...
} while (false);
// ...
```
当需要多行检查时，如果不满足某个条件就执行错误处理，可以用这个形式写。
当不满足时可以直接break到while后，省去了goto的使用。
不过个人不太提倡这种用法，出现这种情况时，应当在if失败后return错误标识（或throw），在外部执行相关错误处理，而不是混到一起（因为这样做外部不知道这个函数是否被成功执行）。