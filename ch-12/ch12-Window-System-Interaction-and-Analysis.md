# 第12章：WINDOWS系统交互与分析

开发 Microsoft Windows 攻击的方法数不胜数，本章无法涵盖太多。无论在最初还是后期开发中，我们只介绍并研究几个对攻击Windows有用的技术，并非研究所有的。

我们将从三个主题来讨论后面的Microsoft API 文档和安全问题。第一，使用Go的 `syscall` 包，通过执行进程注入和各系统级的别的Window API交互。第二，探索Go的Windows Portable Executable (PE)格式核心包，并编写一个PE文件格式解析器。第三，学习在Go代码中使用C代码。

## Windows API中的OpenProcess()函数

了解了Windows API才能攻击Windows。 通过研究OpenProcess()函数来学习Window API文档，该函数用于获得远端进程的句柄。 `OpenProcess()`的文档在[https://docs.microsoft.com/en-us/windows
/desktop/api/processthreadsapi/nf-processthreadsapi-openprocess/](https://docs.microsoft.com/en-us/windows
/desktop/api/processthreadsapi/nf-processthreadsapi-openprocess/)。 图12-1是该功能对象的详细属性。

![](https://github.com/YYRise/black-hat-go/raw/master/ch-12/images/12-1.png)

图12-1：Windows API中OpenProcess()的结构

在这个特定的实例中，可以看到该对象看起来非常类似于Go中的结构类型。 然而，c++结构中的字段类型并不一定与Go中的类型一致，而且Microsoft数据类型并不总是与Go数据类型匹配。 

Windows中数据类型的定义，请参考[https://docs.microsoft.com/en-us/windows/desktop/WinProg/windows-data-types/](https://docs.microsoft.com/en-us/windows/desktop/WinProg/windows-data-types/)，有助于协调Windows数据类型与Go中的相应数据类型。 表12-1 涵盖了本章后面的进程注入例子中用的类型转换。

表12-1：Windows数据类型和Go数据类型的映射
|Windows数据类型| Go数据类型 |
|:--- | :--- |
| BOOLEAN | byte |
| BOOL | int32 |
| BYTE | byte |
| DWORD | uint32 |
| DWORD32 | uint32 |
| DWORD64 | uint64 |
| WORD | uint16 |
| HANDLE | uintptr (unsigned integer pointer) |
| LPVOID | uintptr |
| SIZE_T | uintptr |
| LPCVOID | uintptr |
| HMODULE | uintptr |
| LPCSTR | uintptr |
| LPDWORD | uintptr |

Go 文档将`uintptr`数据类型定义为“一种大到足以容纳任何指针的位模式的整数类型”。 这是一种特殊的数据类型，稍后在下一节“unsafe.Pointer和uintptr类型”中，讨论 Go的`unsafe`包和类型转换时会看到。 现在，完成对 Windows API 文档的浏览。

接下来应该查看对象的参数； 文档中的Parameters部分有详细介绍。 例如，第一个参数 `dwDesiredAccess`，提供了进程句柄应该拥有的有关访问级别的细节。 然后，Return Value部分定义了系统调用是否成功的预期值(图12-2)。 

![](https://github.com/YYRise/black-hat-go/raw/master/ch-12/images/12-2.png)

图12-2：定义预期的返回值


在接下来的示例代码中使用 `syscall`包时，将利用 `GetLastError`错误消息，尽管这略微偏离标准错误处理(比如if err != nil语法)。 

Windows API文档的最后是Requirements部分，含有如图12-3中的重要细节。 最后一行定义了`dynamic link library` (DLL)，包含可导出的函数(如OpenProcess())，当构建Windows DLL模块的变量声明时，它是必需的。 换句话说，我们不能在不知道合适的Windows DLL模块的情况下，从Go调用相关的Windows API函数。 随着我们进入即将到来的进程注入示例，这一点会变得更清楚。

![](https://github.com/YYRise/black-hat-go/raw/master/ch-12/images/12-3.png)

图12-3：Requirements部分定义了调用API所需的库。

## `unsafe.Pointer`和`uintptr`类型

在处理Go的`syscall`包时，我们肯定需要绕过Go的类型安全保护。 原因是我们需要，例如，建立共享内存结构并在Go和C之间执行类型转换。 