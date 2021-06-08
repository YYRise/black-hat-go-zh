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
图12-3：Requirements部分定义了调用API所需的库

## `unsafe.Pointer`和`uintptr`类型

在处理Go的`syscall`包时，肯定需要绕过Go的类型安全保护。 原因是我们需要，例如，建立共享内存结构并在Go和C之间执行类型转换。 本节介绍了操作内存所需要的基础知识，但是也应当进一步研究Go的官方文档。

通过使用Go的`unsafe`包（第9章涉及过的）来绕过Go的安全检查，该包含有绕过Go程序类型安全的操作。Go列出了四条基本的指导方针来帮助我们:
- 任何类型的指针值都能转换成`unsafe.Pointer`。
- `unsafe.Pointer`能够转换成任何类型的指针值。
- `uintptr` 能转换成`unsafe.Pointer`。
- `unsafe.Pointer`能转换成`uinitptr`。

**注意**
*请记住，导入`unsafe`包可能是不可移植的，而且尽管Go通常兼容Go的1版本，但使用`unsafe`包就不能保证了。*

`uintptr`类型允许原生安全类型间的转换或计算，及其他用途。 尽管`uintptr`是整数类型，也广泛的用来表示内存地址。 当与类型安全指针一起使用时，Go的垃圾收集器将在运行时维护相关的引用。

然而，当`unsafe.Pointer`被引入后，情况就会发生变化。 回想下，`uintptr`本质是一个无符号的整数。 如果使用 `unsafe.Pointer` 创建了一个指针，然后赋值给`uintptr`，不能保证Go的垃圾收集器能维护所引用内存地址值的完整性。 图12-4进一步描述了这个问题。

![](https://github.com/YYRise/black-hat-go/raw/master/ch-12/images/12-4.png)
图12-4：使用`unsafe.Pointer`和`uintptr`时的潜在危险指针

图的上半部分是一个引用了Go的类型安全指针的`uintptr`。 因此，在运行时要维护该引用，并进行严格的垃圾收集。 图片的下半部分展示的是引用`unsafe.Pointer`类型的`uintptr`，虽然会被垃圾收集，但Go不保存也不管理任意数据类型的指针。 清单12-1描述了这个问题

``` go
func state() {
	var onload = createEvents("onload")
	var receive = createEvents("receive")
	var success = createEvents("success")
	mapEvents := make(map[string]interface{})
	mapEvents["messageOnload"] = unsafe.Pointer(onload)
	mapEvents["messageReceive"] = unsafe.Pointer(receive) 
	mapEvents["messageSuccess"] = uintptr(unsafe.Pointer(success)) 
	//This line is safe – retains orginal value
	fmt.Println(*(*string)(mapEvents["messageReceive"].(unsafe.Pointer)))
	//This line is unsafe – original value could be garbage collected
	fmt.Println(*(*string)(unsafe.Pointer(mapEvents["messageSuccess"].(uintptr))))
}

func createEvents(s string)| *string {
	return &s
}
```
清单 12-1：将`uintptr`安全地和不安全地与`unsafe.Pointer`一起使用

例如，此代码清单可能是用来创建状态机。 有三个变量，分别通过调用`createEvents()`函数赋值onload、receive和success的指针。创建`map[string]interface{}`。 使用`interface{}`类型是因为可以接收不用类型的数据。 此例中，用来接收`unsafe.Pointer`和`uintptr`类型的值。

此时，很可能已经发现了危险的代码段。 虽然`mapEvents["messageRecieve"]`的值是`unsafe.Pointer`类型，但它仍然保持对`receive`变量的原始引用，并将提供与最初相同的一致输出。 相反，`mapEvents["messageSuccess"]`的值是`uintptr`类型。 这意味着一旦`unsafe.Pointer`值引用被赋值给`uintptr`类型的`success`变量，`success`变量就会被垃圾收集器释放。 此外，`uintptr`只是一种类型，保存内存地址字面意思的整数，而不是对指针的引用。 因此，无法保证预期的输出，因为该值可能不再存在。

有没有一种安全的方式一起使用`uintptr`和`unsafe.Pointer`？ 可以利用`runtime.Keepalive`来做到，`runtime.Keepalive`能够阻止变量被回收。 修改下前面的代码块来看下这个问题（清单12-2）。

``` go
func state() {
	var onload = createEvents("onload")
	var receive = createEvents("receive")
	var success = createEvents("success")
	mapEvents := make(map[string]interface{})
	mapEvents["messageOnload"] = unsafe.Pointer(onload)
	mapEvents["messageReceive"] = unsafe.Pointer(receive) 
	mapEvents["messageSuccess"] = uintptr(unsafe.Pointer(success)) 
	//This line is safe – retains orginal value
	fmt.Println(*(*string)(mapEvents["messageReceive"].(unsafe.Pointer)))
	//This line is unsafe – original value could be garbage collected
	fmt.Println(*(*string)(unsafe.Pointer(mapEvents["messageSuccess"].(uintptr))))

	runtime.KeepAlive(success)
}

func createEvents(s string)| *string {
	return &s
}
```
清单 12-2：使用`runtime.Keepalive`阻止变量被回收

严格来说，我们只添加了一行代码！`runtime .KeepAlive(success)` 这行代码告诉Go在运行时`success`变量维持可访问的，直到明确地释放或运行结束。意思是尽管`success`变量被保存为`uinitptr`，但是不会被垃圾收集，原因是明确地调用了`runtime .KeepAlive()`。

