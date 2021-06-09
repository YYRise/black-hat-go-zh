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

`uintptr`类型允许原生安全类型间的转换或计算，及其他用途。 尽管`uintptr`是整数类型，也广泛的用来表示内存地址。 当与类型安全指针一起使用时，Go的GC将在运行时维护相关的引用。

然而，当`unsafe.Pointer`被引入后，情况就会发生变化。 回想下，`uintptr`本质是一个无符号的整数。 如果使用 `unsafe.Pointer` 创建了一个指针，然后赋值给`uintptr`，不能保证Go的GC能维护所引用内存地址值的完整性。 图12-4进一步描述了这个问题。

![](https://github.com/YYRise/black-hat-go/raw/master/ch-12/images/12-4.png)
图12-4：使用`unsafe.Pointer`和`uintptr`时的潜在危险指针

图的上半部分是一个引用了Go的类型安全指针的`uintptr`。 因此，在运行时要维护该引用，并进行严格的GC。 图片的下半部分展示的是引用`unsafe.Pointer`类型的`uintptr`，虽然会被GC，但Go不保存也不管理任意数据类型的指针。 清单12-1描述了这个问题

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

此时，很可能已经发现了危险的代码段。 虽然`mapEvents["messageRecieve"]`的值是`unsafe.Pointer`类型，但它仍然保持对`receive`变量的原始引用，并将提供与最初相同的一致输出。 相反，`mapEvents["messageSuccess"]`的值是`uintptr`类型。 这意味着一旦`unsafe.Pointer`值引用被赋值给`uintptr`类型的`success`变量，`success`变量就会被GC释放。 此外，`uintptr`只是一种类型，保存内存地址字面意思的整数，而不是对指针的引用。 因此，无法保证预期的输出，因为该值可能不再存在。

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

严格来说，我们只添加了一行代码！`runtime .KeepAlive(success)` 这行代码告诉Go在运行时`success`变量维持可访问的，直到明确地释放或运行结束。意思是尽管`success`变量被保存为`uinitptr`，但是不会被GC，原因是明确地调用了`runtime .KeepAlive()`。

注意，Go中的`syscall`包广泛地使用了`unsafe.Pointer()`，虽然某些函数，如`syscall9()`，例外地有类型安全，但并非所有函数都用的。 此外，当做破解项目时，肯定会遇到需要以不安全的方式操作堆或堆栈内存的情况。


## 使用`syscall`包执行进程注入

通常，我们需要把代码注入到进程中。可能是因为我们想要获得对系统（shell）的远程命令行访问，甚至在源代码不可用时调试运行中的程序。 理解进程注入机制将会帮助我们执行更有趣的任务，例如加载内存中的恶意软件或钩子函数。 无论哪种方式，本节将演示如何使用Go与Microsoft Windows api交互，来执行进程注入。 我们将把存储在磁盘上的有效负载注入到已存在的进程内存中。整个过程如图12-5所示。

![](https://github.com/YYRise/black-hat-go/raw/master/ch-12/images/12-5.png)
图12-5：进程注入原理

第1步，使用Windows的`OpenProcess()`函数建立进程句柄，以及所需的进程访问权限。 这是进程级交互所需要的，无论处理本地进程还是远端进程。

在第2步的Windows`VirtualAllocEx()`函数中使用获取到的进程句柄，该函数在远端进程中申请虚拟内存。这是字节级的代码（如shellcode或DLL）加载到远程内存中所必需的。

第3步，使用Windows的`WriteProcessMemory()`函数将字节级的代码加载到内存中。 注入进程到这一步，作为攻击者的我们要决定如何使用我们的shellcode或DLL。 当想要搞懂运行中的程序时，这也是需要注入调试代码的地方。

最后，第4步，使用Windows的`CreateRemoteThread()`函数调用本地导出的Windows DLL函数，如位于`Kernel32.dll`中的`LoadLibraryA()`，这样我们就可以使用`WriteProcessMemory()`来执行之前植入进程中的代码。

我们刚才描述的四个步骤只是一个基本的流程注入示例。 我们将在整个进程注入示例中定义一些额外的文件和函数，这里没有必要介绍这些文件和函数，但在遇到它们时再详细介绍。

### 定义Windows DLL和赋值变量
清单12-3中的第一步是创建`winmods`文件。（所有代码都在[https://github.com/blackhat-go/bhg/](https://github.com/blackhat-go/bhg/)。）该文件定义了本地的 Windows DLL，用于维护导出系统级的API，Go的`syscall`包调用这些API。 `winmods`文件中有比我们示例项目所需要的更多的Windows DLL模块引用的声明和分配，但我们仍会记录下，以便在更高级的注入代码中使用。

```go 
import "syscall"

var (
	ModKernel32 = syscall.NewLazyDLL("kernel32.dll")
	modUser32 = syscall.NewLazyDLL("user32.dll")
	modAdvapi32 = syscall.NewLazyDLL("Advapi32.dll")

	ProcOpenProcessToken 		= modAdvapi32.NewProc("GetProcessToken")
	ProcLookupPrivilegeValueW 	= modAdvapi32.NewProc("LookupPrivilegeValueW")
	ProcLookupPrivilegeNameW 	= modAdvapi32.NewProc("LookupPrivilegeNameW")
	ProcAdjustTokenPrivileges 	= modAdvapi32.NewProc("AdjustTokenPrivileges")
	ProcGetAsyncKeyState 		= modUser32.NewProc("GetAsyncKeyState")
	ProcVirtualAlloc 			= ModKernel32.NewProc("VirtualAlloc")
	ProcCreateThread 			= ModKernel32.NewProc("CreateThread")
	ProcWaitForSingleObject 	= ModKernel32.NewProc("WaitForSingleObject")
	ProcVirtualAllocEx 			= ModKernel32.NewProc("VirtualAllocEx")
	ProcVirtualFreeEx 			= ModKernel32.NewProc("VirtualFreeEx")
	ProcCreateRemoteThread 		= ModKernel32.NewProc("CreateRemoteThread")
	ProcGetLastError 			= ModKernel32.NewProc("GetLastError")
	ProcWriteProcessMemory 		= ModKernel32.NewProc("WriteProcessMemory")
	ProcOpenProcess 			= ModKernel32.NewProc("OpenProcess")
	ProcGetCurrentProcess 		= ModKernel32.NewProc("GetCurrentProcess")
	ProcIsDebuggerPresent 		= ModKernel32.NewProc("IsDebuggerPresent")
	ProcGetProcAddress 			= ModKernel32.NewProc("GetProcAddress")
	ProcCloseHandle 			= ModKernel32.NewProc("CloseHandle")
	ProcGetExitCodeThread 		= ModKernel32.NewProc("GetExitCodeThread")
)
```
清单12-3：winmods文件(/ch-12/procInjector/winsys/winmods.go)

使用`NewLazyDLL()`加载`Kernel32` DLL。 `Kernel32`管理了很多Windows内部进程的功能，像地址，句柄，内存申请等等。（值得注意的是，从Go版本1.12.2开始，可以使用一些新函数：`LoadLibraryEx()`和`NewLazySystemDLL()`来更好的加载DLL，并防止系统DLL被劫持攻击。）