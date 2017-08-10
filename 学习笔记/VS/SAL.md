---
title: "SAL笔记"
date: 2017-08-05 21:21:41
categories: [学习笔记,VS]
tags: [C++,VS]
---
Microsoft 源代码注释语言 (SAL)   
头文件`sal.h`
使用源代码批注，可以使代码背后的意图更加清晰。 这些注释还可以使用自动化的静态分析工具更准确地分析代码，显著减少误判。  
在代码中使用SAL后，可以在项目属性的代码分析页面勾选**生成时启用代码分析**，即可让编译器帮你检查函数有没有被正确使用。  

----
使用例子如（WinAPI函数定义上带有的那些宏定义就是SAL）:
```cpp
WINBASEAPI
_Ret_maybenull_
HMODULE
WINAPI
LoadLibraryA(
    _In_ LPCSTR lpLibFileName
    );
WINBASEAPI
FARPROC
WINAPI
GetProcAddress(
    _In_ HMODULE hModule,
    _In_ LPCSTR lpProcName
    );
```
比如LoadLibraryA这个定义就可以直接看出，传入的参数不可为空，返回值可能为null。
此时如果这样用：
```cpp
int main(void)
{
	auto dll = LoadLibraryA(0);
	GetProcAddress(dll, 0);
	return 0;
}
```
编译时会警告：
```
“_Param_(1)”可能是“0”: 这不符合函数“LoadLibraryA”的规范。
“_Param_(2)”可能是“0”: 这不符合函数“GetProcAddress”的规范。
```
如果需要通过这个检查，就在调用前后用判断确保值正确即可。


----

SAL 定义了四个基本参数，由用法模式分类。  

类别|批注参数|说明
---|---|---
向调用函数输入|`_In_`|数据传递给被调用函数且被视为只读。
向调用者输出|`_Out_`|调用者为函数提供空间以输出数据。
向调用函数输入并向调用者输出|`_Inout_`|可用数据传入函数也可能会被修改。
向调用者输出指针|`_Outptr_`|类似`_Out_`，但返回的值是一个指针。

上面这些在参数为指针时，值必须非nullptr。
如果参数是可选的就加上`_opt_`后缀，当参数是指针时表示允许nullptr。  


以下是相关定义，基本上看名字就可以知道其功能，所以有一些不写说明，太多。

**函数参数批注**

| 批注 | 说明 |
| ------------- | ------------- |
| `_In_` | 标量，结构，结构指针为类型的输入参数，不会被修改 |
| `_Out_` | 标量，结构，结构指针为类型的输出参数，参数在之前状态不必须有效，但是之后状态必须有效 |
| `_Inout_` | 调用之前之后都必须有效，并假定是不同的值 |
| `_In_z_` | 以 null 结尾的字符串的指针为输入 |
| `_Inout_z_` | 将修改的 null 终止的字符数组的指针。在调用之前和之后都必须有效，但是假定值已经更改。 |
| `_In_reads_(s)`<br>`_In_reads_bytes_(s)` | 一个数组的指针，可被该函数读取。数组大小是 s 元素，该元素绑定有效。<br>_bytes_ 变量以字节为范围而不是元素。  当范围不能表示为元素时，请使用此方法。 |
| `_In_reads_z_(s)` | 指向以 null 终止并具有已知大小的数组的指针。s为元素个数，下同 |
| `_In_reads_or_z_(s)` | 指向以 null 终止或具有已知大小的数组或二者都有的指针。 |
| `_Out_writes_(s)` <br> `_Out_writes_bytes_(s)` | 输出指定长度的数组 |
| `_Out_writes_z_(s)` | 输出指定长度数组并以0结尾 |
| `_Inout_updates_(s)`<br>`_Inout_updates_bytes_(s)` |  |
| `_Inout_updates_z_(s)` |  |
| `_Out_writes_to_(s,c)`<br>`_Out_writes_bytes_to_(s,c)`<br>`_Out_writes_all_(s)`<br>`_Out_writes_bytes_all_(s)` | 写直到c个元素 |
| `_Inout_updates_to_(s,c))`<br>`_Inout_updates_bytes_to_(s,c)` |  |
| `_Inout_updates_all_(s))`<br>`_Inout_updates_bytes_all_(s)` |  |
| `_In_reads_to_ptr_(p)` | 数组的一个指针表达式 类似xxx-123这种 |
| `_In_reads_to_ptr_z_(p)` |  |
| `_Out_writes_to_ptr_(p)` |  |
| `_Out_writes_to_ptr_z_(p)` |  |
| `_Outptr_` | 输出指针，如果是输出引用，把ptr改成ref |
| `_Outptr_opt_` | 可选输出指针 |
| `_Outptr_result_maybenull_` | 参数不能是空的，在点发送状态对位置可以为空。|
| `_Outptr_opt_result_maybenull_` |  |

某些是可以加`_opt_`的，不全记了，用VS时有提示一看就能明白。  

**返回值的批注**
```
_Ret_z_
_Ret_writes_(s)
_Ret_writes_bytes_(s)
_Ret_writes_z_(s)
_Ret_writes_to_(s,c)
_Ret_writes_maybenull_(s)
_Ret_writes_to_maybenull_(s)
_Ret_writes_maybenull_z_(s)
_Ret_maybenull_
_Ret_maybenull_z_
_Ret_null_
_Ret_notnull_
_Ret_writes_bytes_to_
_Ret_writes_bytes_maybenull_
_Ret_writes_bytes_to_maybenull_
```

后面基本上都是定义说明（我不常用），直接帖链接减少无意义内容。


函数参数批注完整内容：
<https://msdn.microsoft.com/zh-cn/library/hh916382(v=vs.120).aspx>


**函数行为批注**
<https://msdn.microsoft.com/zh-cn/library/jj159529.aspx>
`_Check_return_`可以用一下，但是编译时好像没反应

**结构批注**
基本上是一些提示类型大小的
<https://msdn.microsoft.com/zh-cn/library/jj159528.aspx>

**锁定批注**
用来减少并发BUG的
<https://msdn.microsoft.com/zh-cn/library/hh916381.aspx>

**批注的内部函数**
可以在传递参数时用某些函数检查
<https://msdn.microsoft.com/zh-cn/library/jj159527(v=vs.120).aspx>

**例子**
<https://msdn.microsoft.com/zh-cn/library/jj159525.aspx>