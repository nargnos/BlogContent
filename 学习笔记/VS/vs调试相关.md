---
title: vs调试相关
date: 2017-10-11 13:53:11
categories: [学习笔记,VS]
tags: [C++,VS]
---
主要是设置观察点格式的相关内容。<!--more-->_原始笔记，从MSDN弄来的，可能有格式问题，待调（TODO）。_

断点窗口
------------------------------
**@ 地址断点**
在反汇编窗口中设置的是地址断点  


**@ 函数断点**
可以用断点窗口、调用堆栈窗口添加  
也可以用上下文运算符  
{[函数],[源],[模块] } location  
{[函数],[源],[模块] } variable_name  
{[函数],[源],[模块] } 表达式  

**@ 数据断点**
用断点窗口添加，可以填&变量名  



命令窗口
---------------------
在视图->其它窗口 里

在“命令”模式中，将用等号 (=) 分隔的语句作为比较运算符来计算。例如，如果变量 a 和 b 的值不同，则 >? a = b 返回值 FALSE（假）。但在“即时”模式中，将语句 a=b 作为赋值运算来计算，而不是作为比较运算计算。即，a=b 将变量 a 的值赋值为变量 b 的值。不能在“命令”模式中使用赋值运算。

任务	|解决方案|	示例
-----|------|------------
在“命令”模式中计算表达式。	|表达式以问号 (?) 开始。	|?var
从“命令”模式切换到“即时”模式。|	在窗口中输入 immed，不带大于号 (>)。|	immed
从“即时”模式切换回“命令”模式。|	在窗口中输入 cmd。|	>cmd

命令行中的插入符号 (^) 字符表示紧随其后的字符将按原义而不作为控制字符进行解释。这可用于在参数或开关值（开关名除外）中嵌入直引号 (")、空格、正斜杠、插入符号或其他任何字符。

大部分命令都可以用ui找到，这里比较快捷而已


快速监视
---------------------------
**@ 伪变量**
伪变量是用于在变量窗口或“快速监视”对话框中显示某些信息的术语。 您可以像输入普通变量那样输入伪变量。 但伪变量不是变量，它不与程序中的变量名相对应。


伪变量|功能
-------|----------
$handles|显示应用程序中分配的句柄数。
$vframe|显示当前堆栈帧的地址。
$tid|显示当前线程的线程 ID。
$env|在字符串浏览器显示该环境块。
$cmdline|显示启动程序的命令行字符串。
$pid|显示进程ID.
$ 寄存器名 或 @ 寄存器名|显示寄存器 寄存器名 的内容。通常，只需输入寄存器名便可以显示寄存器的内容。 仅在寄存器名重载变量名时才需要使用此语法。 如果寄存器名与当前范围内的某个变量名同名，则调试器将该名称解释为变量名。 这时就需要使用 $寄存器名 或 @寄存器名。
$clk|以时钟形式显示时间。
$user|显示一个结构，在该结构中含有应用程序运行于的帐户的帐户信息。 出于安全原因，不显示密码信息。
$err|last error的值

在 C# 和 Visual Basic 中，可以使用的伪变量如下表中所示：

伪变量|功能
---|---
$exception|显示最近一个异常的有关信息。 如果没有发生异常，则计算 $exception 将显示错误信息。仅在 Visual C# 中，当“异常助手”处于禁用状态时，如果发生异常，$exception 将被自动添加到“局部变量”窗口中。
$user|显示一个结构，在该结构中含有应用程序运行于的帐户的帐户信息。 出于安全原因，不显示密码信息。

vb的略


**@ 格式说明符**
在监视变量后可以添加格式说明符

说明符| 格式| 原始监视值| 显示的值|
---|---|---|---|
d| 十进制整数| 0x00000066| 102|
o| 无符号的八进制整数| 0x00000066| 000000000146|
x h| 十六进制整数| 102| 0xcccccccc|
X H| 十六进制整数| 102| 0xCCCCCCCC|
c| 单个字符| 0x0065, c| 101 'e'|
s| const char * 字符串| *location*  “hello world”| "hello world"|
sb| const char * 字符串| *location*  “hello world”| hello world|
s8| const char * 字符串| *location*  “hello world”| "hello world"|
s8b| const char * 字符串| *location*  “hello world”| "hello world"|
su| const wchar_t* const char16_t * 字符串| *location*  L”hello world”| L"hello world" u"hello world"|
sub| const wchar_t* const char16_t * 字符串| *location*  L”hello world”| hello world|
bstr| BSTR string| *location*  L”hello world”| L”hello world”|
s32| UTF-32 string| *location*  U”hello world”| U”hello world”|
s32b| UTF-32 string (no quotation marks)| *location*  U”hello world”| hello world|
en| enum| Saturday(6)| 星期六|
hv| 指针类型 - 指示被检查的指针值是数组的堆分配的结果，如 new int[3]。| *location* {*first member*}| *location* {*first member*, *second member*, …}|
na| 取消指向对象的指针的内存地址。| *location* , {member=value…}| {member=value…}|
nd| 仅显示基类信息，忽略派生的类| (Shape*) square 包括基类和派生类信息| 仅显示基类信息|
hr| HRESULT 或 Win32 错误代码。 （调试器自动将 HRESULT 解码，因此这些情况下不需要该说明符。）| S_OK| S_OK|
wc| 窗口类标志| 0x0010| WC_DEFAULTCHAR|
wm| Windows 消息数字| 16| WM_CLOSE|
!| 原始格式，忽略任何数据类型视图自定义项| *customized representation*| 4|

指针的大小说明符作为数组

说明符| 格式| 原始监视 Valuen| 显示的值|
----|---|---|---|
n| 十进制或十六进制整数| pBuffer,32| 将 pBuffer 显示为一个 32 元素的数组。|
exp| 计算结果为一个整数的有效的 C++ 表达式。| pBuffer,bufferSize| 将 pBuffer 显示为 bufferSize 元素的一个数组。|
expand(n)| 计算结果为一个整数的有效的 C++ 表达式。| pBuffer, expand(2)| 显示 pBuffer 的第三个元素|

符号| 格式| 原始监视值| 显示的值|
----|---|---|---|
ma| 64 个 ASCII 字符| 0x0012ffac| 0x0012ffac .4...0...".0W&.......1W&.0.:W..1...."..1.JO&.1.2.."..1...0y....1|
m| 以十六进制表示的 16 个字节，后跟 16 个 ASCII 字符| 0x0012ffac| 0x0012ffac B3 34 CB 00 84 30 94 80 FF 22 8A 30 57 26 00 00 .4...0...".0W&..|
mb| 以十六进制表示的 16 个字节，后跟 16 个 ASCII 字符| 0x0012ffac| 0x0012ffac B3 34 CB 00 84 30 94 80 FF 22 8A 30 57 26 00 00 .4...0...".0W&..|
mw| 8 个字| 0x0012ffac| 0x0012ffac 34B3 00CB 3084 8094 22FF 308A 2657 0000|
md| 4 个双字| 0x0012ffac| 0x0012ffac 00CB34B3 80943084 308A22FF 00002657|
mq| 2 个双字| 0x0012ffac| 0x0012ffac 7ffdf00000000000 5f441a790012fdd4|
mu| 双字节字符 (Unicode)| 0x0012ffac| 0x0012ffac 8478 77f4 ffff ffff 0000 0000 0000 0000|



