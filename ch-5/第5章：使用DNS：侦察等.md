## 摘要

*Domain Name System (DNS)* 定位网络域名并解析其IP地址。对于攻击者是非常有用的武器，因为组织通常允许协议从受限制的网络中流出，并且常常不能充分监视其使用。这需要一点知识，但是精明的攻击者几乎可以在攻击链的每一步利用这些问题，包括侦察，命令和控制（C2），甚至数据过滤。在本章中，学习使用Go和第三方包如何写一个自己的工程来执行这些功能。

首先解析主机名和IP地址，来显示能够枚举出的很多的DNS记录类型。然后使用前面几章中介绍的模式来构建一个大规模并发的子域猜测工具。最后，学会如何实现DNS服务和代理，然后使用DNS隧道来建立一个限制网络之外的C2通道!

## 实现DNS客户端

开发这个复杂程序前，先让我们来熟悉一些客户端的操作。Go内置的 net` 包提供了很多功能，并支持大多数（如果不是全部）的记录类型。内置包的优点是简单易懂的API。例如 `LookupAddr(addr string)` 返回给定IP地址的主机名列表。劣势是不能指定目标服务，相反，内置包使用操作系统上配置的解析器。另一个缺点是不能详细即检查结果。

使用Miek Gieben编写的名为Go DNS的第三方包来规避这些缺点，这是我们的首选DNS包，因为它是高度模块化，并经过良好的测试。使用下面的命令来安装：

```shell script
$ go get github.com/miekg/dns
```

安装好之后，就可以跟着接下来的示例代码操作了。首先执行记录查找，以便为主机名解析IP地址。

### 检索**A Records**

让我们先从查找 *fully qualified domain name (FQDN)* 开始，其指定了主机在DNS层次结构中的确切位置。然后再尝试使用 DNS记录类型的  *A record* ，将FQDN解析为IP地址。使用  *A records* 将域名绑定到IP地址。清单5-1 是一个查找的例子：

```go
package main

import "github.com/miekg/dns"

func main() {
	var msg dns.Msg
	fqdn := dns.Fqdn("stacktitan.com")
	msg.SetQuestion(fqdn, dns.TypeA)
	dns.Exchange(&msg, "8.8.8.8:53")
}
```

清单 5-1:检索A Records (https://github.com/blackhat-go/bhg/ch-5/get_a/main.go/)

先new一个`Msg`，然后调用 `fqdn(string)` 将域转换为可以与DNS服务器交换的 FQDN。下一步，调用 `SetQuestion(string, uint16)` 来修改 `MSG` 的内部状态，该函数使用 `TypeA` 表示专门查找 *A record* 。（`TypeA` 被定义为常量。在该包的文档中查看其他支持的类型。）最后，调用 `Exchange(*Msg, string)` 将消息发送到所提供的服务器地址，即本例中由谷歌操作的DNS服务器。

正如您可能知道的，这段代码不是很有用。尽管向 DNS 服务器发送了查询请求 *A record*，但是没有处理结果；也就是没有对结果做任何有意义的事情。在用Go编程之前，先来看看DNS返回的结果是什么样子，以便能够对协议和不同的查询类型有更深入的了解。

执行清单 5-1 之前，先运行像 Wireshark 或 tcpdump 抓包工具来查看流量。下面是在LINUX上使用tcpdump的示例：

```shell script
$ sudo tcpdump -i eth0 -n udp port 53
```

在另一个终端，像下面这样编译并执行程序：

```shell script
$ go run main.go
```

执行代码后，在抓包的输出中应该会看到一个通过 8.8.8.8.53 的UDP连接。应该也会看到下面这样的DNS协议详情：

```shell script
$ sudo tcpdump -i eth0 -n udp port 53
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on ens33, link-type EN10MB (Ethernet), capture size 262144 bytes 
23:55:16.523741 IP 192.168.7.51.53307 > 8.8.8.8.53: 25147+ A? stacktitan.com. (32) 23:55:16.650905 IP 8.8.8.8.53 > 192.168.7.51.53307: 25147 1/0/0 A 104.131.56.170 (48)
```

需要对该输出的几行进一步解释。首先，当请求 DNS *A record* 时，使用UDP创建了一个从 192.168.7.51 到 8.8.8.8.53的查询。从Google的 8.8.8.8 DNS服务器响应返回，其含有解析后的IP地址，为104.131.56.170 。

使用像 tcpdump这样的抓包工具，能将域名 *stacktitan.com* 解析成 IP地址。现在看看如何用Go来提取这些信息。

### 使用**Msg**结构体处理应答

`Exchange(*Msg, string)` 返回 `(*Msg, error)类型的值 。返回 `error` 类型是Go 的惯用法，但为什么它会返回*Msg呢？为了清楚起见，查看源码中该结构的定义：

```go
type Msg struct {
	MsgHdr
	Compress bool       `json:"-"` // If true, the message will be compressed when converted to wire format.
	Question []Question // Holds the RR(s) of the question section.
	Answer   []RR       // Holds the RR(s) of the answer section.
	Ns       []RR       // Holds the RR(s) of the authority section.
	Extra    []RR       // Holds the RR(s) of the additional section.
}
```

正如所见，`Msg` 中包含查询和结果。这将所有DNS查询及其结果合并到一个统一的结构中。`Msg` 类型还有几个方法来方便地处理数据。例如，`SetQuestion()` 方法方便地修改 `Question` 切片。可以使用 `append()` 直接修改该切片，然后取得结果。` RR` 类型的 `Answer` 切片存有查询的结果。清单 5-2 展示了如何处理应答：

```go
package main

import (
	"fmt"

	"github.com/miekg/dns"
)

func main() {
	var msg dns.Msg
	fqdn := dns.Fqdn("stacktitan.com")
	msg.SetQuestion(fqdn, dns.TypeA)
	in, err := dns.Exchange(&msg, "8.8.8.8:53")
	if err != nil {
		panic(err)
	}
	if len(in.Answer) < 1 {
		fmt.Println("No records")
		return
	}
	for _, answer := range in.Answer {
		if a, ok := answer.(*dns.A); ok {
			fmt.Println(a.A)
		}
	}
}
```

清单 5-2: 处理 DNS 结果 (https://github.com/blackhat-go/bhg/ch-5/get_all_a/main.go/)

例子中先保存 `Exchange` 的返回值，然后检查 `error` ，有错误的话调用 `panic()` 来退出程序。 `panic()` 函数能快速查看调用堆栈和发现出错的地方。下一步，验证  `Answer` 切片的长度是否小于1，如果不是的话，表明没有记录，然后立即退出，毕竟，当域名无法解析时，会有合法的实例。

` RR` 是仅有两个方法的接口，并且都不能访问存储在 answer 中的IP地址。要想访问IP地址的话，需要执行类型断言来创建数据的实例作为所需的类型。

首先，遍历 answer。接下来，执行类型断言来确保处理的是 ` *dns.A` 类型。执行断言时会返回两个值：该断言类型的数据和一个表示断言是否成功的 `bool` 值。判断断言成功后打印出  `a.A` 中的IP地址。尽管 `net.IP` 类型没有实现 ` String()` 方法，但仍然可以轻松的打印。

花点时间研究下代码，修改下 DNS 查询，查找其他的记录。类型断言可能不熟悉，但它与其他语言中的类型转换类似。

### 枚举子域名

既然您已经知道如何使用Go作为DNS客户端，那么就可以创建有用的工具了。在本部分，创建子域猜测程序。猜测目标的子域名和其他DNS记录是侦察的基础步骤，因为知道的子域越多，就可以尝试越多的攻击。为程序提供一个候选单词列表(一个字典文件)，用于猜测子域。

使用DNS，发送请求的速度与操作系统处理包数据的速度一样快。虽然语言和运行时不会成为瓶颈，但是目标服务器会。控制程序的并发就变得重要了，就像前几章那样。

首先在 **GOPATH**下创建  *subdomain_guesser* 文件夹，然后创建 *main.go* 文件。下一步，当开始写一个新工具时，必须确定该程序需要哪些参数。该子域猜测程序需要几个参数，包括目标域，用来猜测子域的文件，使用的目标DNS服务器，启动的工作线程数。Go使用 `flag` 包来解析命令行参数。虽然在所有示例代码中没有使用 `flag` 包，但是在本例中，我们选择使用它来演示更健壮、更优雅的参数解析。参数解析代码如清单 5-3 所示。

```go
package main

import (
  "flag"
)

func main() {
  var (
    flDomain      = flag.String("domain", "", "The domain to perform guessing against.")
    flWordlist    = flag.String("wordlist", "", "The wordlist to use for guessing.")
    flWorkerCount = flag.Int("c", 100, "The amount of workers to use.")
    flServerAddr  = flag.String("server", "8.8.8.8:53", "The DNS server to use.")
  )
	flag.Parse()
}
```

清单 5-3: 构建子域猜测 (https://github.com/blackhat-go/bhg/ch-5/subdomain_guesser/main.go/)

首先，声明 `flDomain` 变量的代码行接收 `String` 类型的参数，并声明空字符串为默认值，该变量将被解析为 `domain` 选项。下一行代码声明了 `flWorkerCount` 变量。需要使用 `Integer` 类型的值作为 `c` 命令行选项。本例中设置为 100 个默认线程。但是这个值可能太保守了，可以在测试时随意增加该值。最后，调用 `flag.Parse()` 将输入的命令行参数值解析到相应的变量中。

> 注意：可能已经注意到这个示例违反了Unix条例，因为示例中定义了非可选的参数。请随意使用 *os.Args*。我们只是发现 `flag` 包更简单、更快。

如果编译该程序的话应该会出现未使用的变量的错误。在调用`flag.Parse()`这行代码的后面添加下面的代码。该代码将变量打印到 stdout，确认下输入的 `-domain` 和 `-wordlist` ：

```go
if *flDomain == "" || *flWordlist == "" { 
  fmt.Println("-domain and -wordlist are required") 
  os.Exit(1)
}
fmt.Println(*flWorkerCount, *flServerAddr)
```

让工具报告可解析的名称及它们各自的IP地址，需要创建 `struct` 类型来存储这些信息。在 `main()` 函数中的定义如下：

```go
type result struct {
    IPAddress string
    Hostname string
}
```

使用该工具查询两个主要的类型—A 和 CNAME。在各自的函数中分别查询。让函数尽可能的短小，并只做一件事是个不错的想法。这种开发风格让您在将来编写更小的测试。

### 查询 **A** 和 **CNAME** 记录

创建了两个函数执行查询：一个是 **A** 记录，另一个是 **CNAME** 记录。这两个函数的第一个参数都是 `FQDN`，第二个参数都是 `DNS` 服务器地址。每个函数也都返回一个字符串类型的切片和错误。将清单 5-3 中函数添加到代码中。这些函数都应该定义在 `main()`函数外。

```go
func lookupA(fqdn, serverAddr string) ([]string, error) {
	var m dns.Msg
	var ips []string
	m.SetQuestion(dns.Fqdn(fqdn), dns.TypeA)
	in, err := dns.Exchange(&m, serverAddr)
	if err != nil {
		return ips, err
	}
	if len(in.Answer) < 1 {
		return ips, errors.New("no answer")
	}
	for _, answer := range in.Answer {
		if a, ok := answer.(*dns.A); ok {
			ips = append(ips, a.A.String())
		}
	}
	return ips, nil
}

func lookupCNAME(fqdn, serverAddr string) ([]string, error) {
	var m dns.Msg
	var fqdns []string
	m.SetQuestion(dns.Fqdn(fqdn), dns.TypeCNAME)
	in, err := dns.Exchange(&m, serverAddr)
	if err != nil {
		return fqdns, err
	}
	if len(in.Answer) < 1 {
		return fqdns, errors.New("no answer")
	}
	for _, answer := range in.Answer {
		if c, ok := answer.(*dns.CNAME); ok {
			fqdns = append(fqdns, c.Target)
		}
	}
	return fqdns, nil
}
```

这段代码应该很熟悉了，因为与本章第一节中编写的代码几乎相同。第一个函数 `lookupA` 返回 IP地址的数组，第二个函数 `lookupCNAME` 返回主机名数组。

*CNAME* ，即 *canonical name* ，记录一个FQDN指向另一个FQDN，后者作为第一个FQDN的别名。例如，假设example.com组织的所有者希望通过使用WordPress托管服务来托管一个WordPress站点。该服务可能有数百个IP地址来平衡所有用户的站点，因此提供单个站点的IP地址是不可行的。WordPress主机服务可以提供一个规范名称(CNAME)， example.com的所有者可以参考它。因此 *www .example.com*  可能有一个指向 *someserver.hostingcompany.org* 的CNAME，反过来这个CNAME又有一个指向IP地址的A记录。这允许 *example.com* 的所有者将其站点托管在没有IP信息的服务器上。

通常这意味着需要跟着 **CNAMES** 的轨迹，以最终得到有效的 **A** 记录。之所以说 *轨迹* 是因为 **CNAMES** 链可以是无穷无尽的。在 `main()` 外添加下面的函数，看下如何使用 ** CNAMES** 的轨迹来追踪到有效的 **A** 记录。

```go
func lookup(fqdn, serverAddr string) []result {
	var results []result
	var cfqdn = fqdn // Don't modify the original.
	for {
		cnames, err := lookupCNAME(cfqdn, serverAddr)
		if err == nil && len(cnames) > 0 {
			cfqdn = cnames[0]
			continue // We have to process the next CNAME.
		}
		ips, err := lookupA(cfqdn, serverAddr)
		if err != nil {
			break // There are no A records for this hostname.
		}
		for _, ip := range ips {
			results = append(results, result{IPAddress: ip, Hostname: fqdn})
		}
		break // We have processed all the results.
	}
	return results
}
```

首先，定义存储结果的切片。下一步，复制传入的第一个参数 `FQDN` ，这样不但不会丢失猜测的原始 `FQDN` ，而且还可以用来第一次查询。之后启动无限循环，尝试解析 **FQDN** 的 **CNAME** 。如果没有发生错误，且至少返回一个 **CNAME** ，设置 `cfqdn` 为返回的 **CNAME** ，使用 `continue` 退回到循环的开始。跟随 ** CNAMES** 的轨迹，直到失败。失败则表明到达链尾，然后就可以查找 **A** 记录；但是如果有错误，则表明在查找记录时某些东西出错了，然后早点退出循环。如果找到有效的 **A** 记录，将每个IP地址追加到待返回的 `results` 切片中，然后退出循环。最后，将 `results` 返回给调用者。

与名称解析相关的逻辑看起来很合理。但是没有考虑性能。为方便并发将示例改为 goroutine 执行。

### 传参给工作函数

创建 `goroutine` 池用于将任务传送给执行任务单元的 *worker function*。通过使用 `channel` 协调分配任务和收集结果。在第2章中的构建并发的端口扫描有过类似做法。

继续扩展清单 5-3的代码。首先，在 `main()` 函数外创建 `worker()` 函数。该函数的参数为三个  channel：一个 channel 用于发送是否关闭的信号，一个域名 channel 用于接收工作，一个channel 用于发送结果。该函数还需要一个最终的字符串参数来指定要使用的DNS服务器。`worker()` 函数的代码如下：

```go
type empty struct{}

func worker(tracker chan empty, fqdns chan string, gather chan []result, serverAddr string) {
	for fqdn := range fqdns {
		results := lookup(fqdn, serverAddr)
		if len(results) > 0 {
			gather <- results
		}
	}
	var e empty
	tracker <- e
}
```



引入 `worker()` 函数前，先定义一个 empty 类型用于追踪 worker 完成。empty 是个没有字段的 `struct` ；使用空的 struct 是因为其字节数为0，在使用时几乎没有影响和开销。然后，在 `worker()` 函数内，循环遍历用于传递FQDN的域名 channel 。之后收集 lookup() 函数的结果，并且检查确保至少有一个结果发给 `gather` channel，该channel将结果累积到 main() 中。channel被关闭后循环会退出，把 `empty` 结构体发送给 `tracker` channel ，通知调用者所有的工作的完成了。 将空 `struct` 发送给 `tracker` channel 是重要的最后一步。如果没有这一步的话会有竞争条件，因为调用者可能在 ` gather` channel 收到结果之前就退出了。

至此，所有的前提准备都创建好了，回到 `main()` 中继续完成清单 5-3中的程序。定义几个变量来保存结果，定义传递给 `worker` 的channel。把下面代码加到 `main()` 中。

```go
var results []result
fqdns := make(chan string, *flWorkerCount)
gather := make(chan []result)
tracker := make(chan empty)
```

通过用户提供的执行工作的数量来创建带有缓冲的 `fqdns` channel。这能让任务执行的稍微快些，因为在生产者使其阻塞前能持有超过1个信息。

### 使用**bufio**创建扫描器

接下来，打开用户提供的文件作为单条使用。打开文件后，使用 `bufio` 包创建一个 `scanner` 。该扫描器一次读取文件的一行数据。在 `main()` 中加入下面的代码：

```go
fh, err := os.Open(*flWordlist) 
if err != nil {
	panic(err) 
}
defer fh.Close()
scanner := bufio.NewScanner(fh)
```

如果返回的错误不为 `nil` 的话在这地方使用内置函数 `panic()` 。当写的包或程序给其他人使用时，应该考虑用更简洁的格式显示这些信息。

使用新创建的 `scanner` 从提供的词条抓取一行文本，然后结合该文本和用户提供的域来创建**FQDN**。将结果发送给 `fqdns` channel 。但是必须要先启动工做函数。这是非常重要的顺序。如果没有先启动工作函数就把任务发送给 `fqdns` channel，channel 很快就会满了，然后生产者就会阻塞。将下面代码加入到 `main()` 函数中。这段代码的作用就是启动工作 goroutine，读取输入文件，将任务发送给 `fqdns` channel 。

```go
for i := 0; i < *flWorkerCount; i++ {
    go worker(tracker, fqdns, gather, *flServerAddr)
}
for scanner.Scan() {
	fqdns <- fmt.Sprintf("%s.%s", scanner.Text()w, *flDomain)
}
```

使用这个方式创建工作者类似于在构建并发端口扫描器时所做的：使用for循环用户指定的次数。在第2个for循环中使用 `scanner.Scan()` 抓取文件的每行数据。当读完所有行时循环结束。使用 `scanner.Text()` 可以将被扫描的行转化成字符串形式。

开始工作了！享受一秒钟。在读下一段代码之前，想一想程序的进度和在本书中已完成了哪些内容。试着完成程序，然后继续下一节，在下一节中将带你完成剩余的部分。

### 收集并显示结果

在最后，先启动一个匿名goroutine来收集任务执行的结果。把下面代码添加到 `main()` 中：

```go
go func() {
	for r := range gather {
		results = append(results, r...v) 
    }
	var e empty 
    tracker <- e
}()
```

通过循环遍历 `gather` 管道，将接收到的结果追加到 `results` 切片中。因为将一个切片追加到另一个切片，所以必须要使用 `...`  。`gather` 管道关闭后循环结束，像之前那样发送一个空 `struct` 到 `tracker` 管道 。 这会防止在append()` 还没完成就把结果返回给用户。

剩下的就是关闭通道并显示结果。在 ` main()` 函数的底部包含以下代码，以便关闭通道并将结果显示给用户:

```go
close(fqdns)
for i := 0; i < *flWorkerCount; i++ {
	<-tracker 
}
close(gather) 
<-tracker
```

第一个应该关闭的是 `fqdns` 管道，因为已经把所有的任务发送给这个管道。接下来，从每个任务执行函数的 `tracker` 管道中接收，这样就知道执行函数已经完成退出了。收到和工作函数相等数量的信号就可以关闭 `gather` 管道了，因为不会再收到结果了。最后，再一次读取 `tracker` 管道让收集的goroutine完全完成。

现在结果还未呈现给用户。来完成吧。如果你愿意的话可以使用简单的循环来遍历 `results` 切片，使用 `fmt.Printf()` 打印 `Hostname` 和 `IPAddress` 。相反，我们使用几个Go的优秀内置包来显示数据；`tabwriter` 是其中之一。该包会友好的方式输出数据，甚至是按制表符分隔的列。在 `main()` 函数的最后加入下面的代码来使用 `tabwriter` 打印结果：

```go
w := tabwriter.NewWriter(os.Stdout, 0, 8, 4, ' ', 0) 
for _, r := range results {
	fmt.Fprintf(w, "%s\t%s\n", r.Hostname, r.IPAddress) 
}
w.Flush()
```

清单 5-4 是完整的程序代码

```go
package main

import (
	"bufio"
	"errors"
	"flag"
	"fmt"
	"os"
	"text/tabwriter"

	"github.com/miekg/dns"
)

func lookupA(fqdn, serverAddr string) ([]string, error) {
	var m dns.Msg
	var ips []string
	m.SetQuestion(dns.Fqdn(fqdn), dns.TypeA)
	in, err := dns.Exchange(&m, serverAddr)
	if err != nil {
		return ips, err
	}
	if len(in.Answer) < 1 {
		return ips, errors.New("no answer")
	}
	for _, answer := range in.Answer {
		if a, ok := answer.(*dns.A); ok {
			ips = append(ips, a.A.String())
		}
	}
	return ips, nil
}

func lookupCNAME(fqdn, serverAddr string) ([]string, error) {
	var m dns.Msg
	var fqdns []string
	m.SetQuestion(dns.Fqdn(fqdn), dns.TypeCNAME)
	in, err := dns.Exchange(&m, serverAddr)
	if err != nil {
		return fqdns, err
	}
	if len(in.Answer) < 1 {
		return fqdns, errors.New("no answer")
	}
	for _, answer := range in.Answer {
		if c, ok := answer.(*dns.CNAME); ok {
			fqdns = append(fqdns, c.Target)
		}
	}
	return fqdns, nil
}

func lookup(fqdn, serverAddr string) []result {
	var results []result
	var cfqdn = fqdn // Don't modify the original.
	for {
		cnames, err := lookupCNAME(cfqdn, serverAddr)
		if err == nil && len(cnames) > 0 {
			cfqdn = cnames[0]
			continue // We have to process the next CNAME.
		}
		ips, err := lookupA(cfqdn, serverAddr)
		if err != nil {
			break // There are no A records for this hostname.
		}
		for _, ip := range ips {
			results = append(results, result{IPAddress: ip, Hostname: fqdn})
		}
		break // We have processed all the results.
	}
	return results
}

func worker(tracker chan empty, fqdns chan string, gather chan []result, serverAddr string) {
	for fqdn := range fqdns {
		results := lookup(fqdn, serverAddr)
		if len(results) > 0 {
			gather <- results
		}
	}
	var e empty
	tracker <- e
}

type empty struct{}

type result struct {
	IPAddress string
	Hostname  string
}

func main() {
	var (
		flDomain      = flag.String("domain", "", "The domain to perform guessing against.")
		flWordlist    = flag.String("wordlist", "", "The wordlist to use for guessing.")
		flWorkerCount = flag.Int("c", 100, "The amount of workers to use.")
		flServerAddr  = flag.String("server", "8.8.8.8:53", "The DNS server to use.")
	)
	flag.Parse()

	if *flDomain == "" || *flWordlist == "" {
		fmt.Println("-domain and -wordlist are required")
		os.Exit(1)
	}

	var results []result

	fqdns := make(chan string, *flWorkerCount)
	gather := make(chan []result)
	tracker := make(chan empty)

	fh, err := os.Open(*flWordlist)
	if err != nil {
		panic(err)
	}
	defer fh.Close()
	scanner := bufio.NewScanner(fh)

	for i := 0; i < *flWorkerCount; i++ {
		go worker(tracker, fqdns, gather, *flServerAddr)
	}

	for scanner.Scan() {
		fqdns <- fmt.Sprintf("%s.%s", scanner.Text(), *flDomain)
	}
	// Note: We could check scanner.Err() here.

	go func() {
		for r := range gather {
			results = append(results, r...)
		}
		var e empty
		tracker <- e
	}()

	close(fqdns)
	for i := 0; i < *flWorkerCount; i++ {
		<-tracker
	}
	close(gather)
	<-tracker

	w := tabwriter.NewWriter(os.Stdout, 0, 8, 4, ' ', 0)
	for _, r := range results {
		fmt.Fprintf(w, "%s\t%s\n", r.Hostname, r.IPAddress)
	}
	w.Flush()
}
```

清单 5-4: 完整的猜域名程序 (https://github.com/bhg/ch-5/subdomain_guesser/main.go/)