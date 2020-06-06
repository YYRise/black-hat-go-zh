许多安全工具都构建为 *frameworks* ——核心组件，抽象级别的构建可以容易地扩展他们的功能。仔细想想，这其实对安全从业人员来说很有意义。该行业在不断变化；社区总是在发明新的漏洞和技术，以避免检测，创造一个高度动态和有点不可预测的景观。但是，通过使用插件和扩展，工具开发人员可以在一定程度上保证他们的产品不会过时。通过重用工具的核心组件而不用再进行繁琐的重写，可以通过可插拔的系统优雅地处理行业发展。

这一点，再加上大量的社区参与，可以说是 **Metasploit Framework** 成功老化的原因。见鬼，即使是像Tenable这样的商业企业，也看到了创造可扩展产品的价值；Tenable 依赖于一个插件为基础的系统，在其 Nessus 漏洞扫描器中执行签名检查。

在本章中，将用Go创建两个漏洞扫描器扩展。首先，使用本地Go插件系统并显式地将代码编译为共享对象。然后，使用嵌入的Lua系统重新构建相同的插件，该系统比本地Go插件系统更早。记住，与用其他语言(如Java和Python)创建插件不同，在Go中创建插件是一个相当新的构造。本地支持插件从Go版本1.8开始。而且，直到版本1.10才可以将这些插件创建为Windows动态链接库(DLLs)。确保使用最新颁布的Go，以便本章中的例子可以正常运行。

## 使用Go本地插件系统

Go在版本1.8之前，不支持插件或动态运行时代码可扩展性。虽然像Java这样的语言允许执行程序来实例化导入的类型并调用它们的函数时加载类或JAR文件，但是Go没有提供这样的奢侈。虽然有时可以通过接口实现来扩展功能，但是不能真正动态地加载和执行代码本身。相反，需要在编译期时正确地包含它。作为一个例子，无法复制这里显示的Java功能，它从一个文件动态加载一个类，实例化类，并在实例上调用 `someMethod()` ：

```java
File file = new File("/path/to/classes/");
URL[] urls = new URL[]{file.toURL()};
ClassLoader cl = new URLClassLoader(urls);
Class clazz = cl.loadClass("com.example.MyClass"); clazz.getConstructor().newInstance().someMethod();
```

幸运的是，Go的新版本能够模拟这一功能，允许开发人员显式地编译代码以作为插件使用。具体来说，在版本1.10之前，插件系统只能在Linux上工作，因此必须在Linux上部署可扩展框架。Go的插件是在构建过程中作为共享对象创建的。要生成共享对象，输入以下构建命令，该命令将 `plugin` 提供给 `buildmode` 选项：

```shell
$ go build -buildmode=plugin
```

或者，要构建Windows DLL，使用 `c-shared` 作为 `buildmode` 选项:

```shell
$ go build -buildmode=c-shared
```

要构建Windows DLL，您的程序必须满足某些导出函数的约定，还必须导入C库。我们让您自己探索这些细节。在本章中，我们只关注Linux插件的变体，因为我们将在第12章演示如何加载和使用 DLL。

在编译到DLL或共享对象之后，一个单独的程序可以在运行时加载和使用插件。可以访问任何导出的函数。使用Go的插件包可以与共享对象的导出功能交互。包中的功能简单明了。要使用插件，请按照下列步骤：

1. 调用 `plugin.Open(filename string) 来打开共享对象文件， 创建  `plugin.Plugin` 实例。
2.  `plugin.Plugin` 调用 `Lookup(symbolName string)` 通过名字来检索 Symbol （这是导出函数的变体）。
3. 使用类型断言将泛型的 Symbol 转换为程序期望的类型。
4. 根据需要使用转换后的结果对象。

可能已经注意到，对 `Lookup()` 的调用需要使用者提供符号名。这意味着使用者必须预定义，且希望能公开的命名方案。可以将其看作一个定义好的API或通用接口，插件将会遵循这些接口。如果没有标准的命名方案，新的插件将对使用者代码进行更改，从而破坏基于插件的系统的整个目的。

在下面的示例中，期望插件定义一个名为 `New()` 的导出函数，该函数返回特定的接口类型。这样，就可以对引导过程进行标准化。将句柄返回到接口能够以可预料的方式调用对象上的函数。

现在开始创建可插拔的漏洞扫描器。每个插件实现单个检测逻辑。主扫描器代码将通过从文件系统上的单个目录中读取插件来启动处理。要使这一切正常运行，将使用两个独立的存储库:一个用于插件，另一个用于使用插件的主程序。

### 创建主程序

从主程序开始，在其上添加插件。这有助于理解创建插件过程。设置存储库的目录结构，如下显示：

```shell
$ tree 
.
--- cmd
	--- scanner
		--- main.go
--- plugins
--- scanner
	--- scanner.go
```

*cmd/scanner/main.go* 文件是命令行工具。它加载插件并启动扫描。*plugins* 目录包含动态加载的所有共享对象，以调用各种漏洞签名检查。 *scanner/scanner.go* 文件中定义插件和主扫描器使用的数据类型。将数据存放在自己的包中更易用些。

清单10-1是 *scanner.go* 文件的内容。

```go
package scanner

// Scanner defines an interface to which all checks adhere 
type Checker interface {
	Check(host string, port uint64) *Result 
}
// Result defines the outcome of a check 
type Result struct {
    Vulnerable bool
    Details    string
}
```

清单 10-1: 定义核心扫描器类型 (https://github.com/blackhat-go/bhg/ch-10/plugin-core/scanner/scanner.go/)

在这个名为 *scanner* 的包中定义了两个类型。第一个是 `Checker` 接口。该接口定义了 `Check()` 这一个方法，参数为 `host` 和 `port`，返回值为 `Result` 指针。`Result` 类型被定义为 `struct`。其目的是追踪检查的结果。服务易受攻击吗?在记录、验证或利用缺陷时，哪些细节是相关的?

将接口视为某种契约或蓝图；插件随意实现 `Check()` 函数，只要返回 `Result` 指针。插件实现的逻辑基于每个插件的漏洞检查逻辑。例如，检查Java反序列化问题的插件可以实现适当的HTTP调用，而检查默认SSH凭据的插件可以对SSH服务发起密码猜测攻击。这就是抽象的力量！

接下来，回顾一下 *cmd/scanner/main.go 。该文件中使用插件(清单10-2)。

```go
const PluginsDir = "../../plugins/"

func main() {
	var (
		files []os.FileInfo
		err   error
		p     *plugin.Plugin
		n     plugin.Symbol
		check scanner.Checker
		res   *scanner.Result
	)
	if files, err = ioutil.ReadDir(PluginsDir); err != nil {
		log.Fatalln(err)
	}

	for idx := range files {
		fmt.Println("Found plugin: " + files[idx].Name())
		if p, err = plugin.Open(PluginsDir + "/" + files[idx].Name()); err != nil {
			log.Fatalln(err)
		}

		if n, err = p.Lookup("New"); err != nil {
			log.Fatalln(err)
		}

		newFunc, ok := n.(func() scanner.Checker)
		if !ok {
			log.Fatalln("Plugin entry point is no good. Expecting: func New() scanner.Checker{ ... }")
		}
		check = newFunc()
		res = check.Check("10.0.1.20", 8080)
		if res.Vulnerable {
			log.Println("Host is vulnerable: " + res.Details)
		} else {
			log.Println("Host is NOT vulnerable")
		}
	}
}
```

清单 10-2: 运行插件的扫描器客户端 (https://github.com/blackhat-go/bhg/ch-10/plugin-core/cmd/scanner/main.go/)

代码从定义插件所在的位置开始。这种情况下使用的是硬编码。当然也可以改进代码，从参数或环境变量中读入该值。使用该变量调用 `ioutil.ReadDir(PluginDir)` 来获取文件列表，然后再遍历插件文件。对每个文件，使用Go的 `plugin` 包通过调用 `plugin.Open()` 来读取插件。如果成功的话，就会有一个 `*plugin.Plugin` 实例，将该实例赋值给变量 `p` 。调用 `p.Lookup("New")` 来查找符号名为 `New` 的插件。

正如在前面的高级概述中提到的，这个符号查找约定需要主程序提供符号的明确的名称作为参数，意味着插件有相同名称的导出符号——在本例中，主程序查找的符号名字为 `New` 。此外，很快就会看到，代码期望符号是一个函数，该函数将返回 `scanner.Checker` 的具体实现，该接口在前一节中讨论过。

假定插件已经含有名为 `New` 的符号，对符号进行类型断言，将其转成 `func() scanner.Checker` 类型。即，期望符号是一个返回实现了 `scanner.Checker` 的对象的函数。将转换后的值赋给一个名为 `newFunc` 的变量。然后调用它，并将返回值赋值给变量 `check` 。幸亏类型断言，才能判断是否符合 `scanner.Checker` 接口，所以必须要实现 `Check()` 函数。调用并传参目标主机和端口。结果为 `*scanner.Result` ，赋值给变量 `res` 并检查以确定服务是否容易受到攻击。

注意，这个过程是通用的；它使用类型断言和接口来创建一个结构，通过这个结构可以动态地调用插件。代码中没有专门的单个漏洞签名或用于检查漏洞是否存在的方法。相反，已经对功能进行了足够的抽象，以至于插件开发人员可以创建独立的插件来执行工作单元，而不需要了解其他插件，甚至不需要了解使用应用程序。插件作者必须关心的惟一一件事是正确地创建导出的 `New()` 函数和实现 `scann . checker` 的类型。来看一下实现这一点的插件。

### 构建密码猜测插件

插件（清单10-3）对 Apache Tomcat Manager 登录入口进行密码猜测攻击。这也是黑客喜欢攻击的目标，这种入口通常将其配置为容易猜测的凭证。有了有效的凭证，黑客就可以在底层系统上可靠地执行任意代码。黑客非常容易获胜。

在对代码的审查中，没有涉及到漏洞测试的具体细节，因为实际上只是一系列发送到特定URL的HTTP请求。相反，主要专注于满足可插入扫描器的接口需求。

```go
import (
	// Some snipped for brevity 
    "github.com/bhg/ch-10/plugin-core/scanner" u
)

var Users = []string{"admin", "manager", "tomcat"}
var Passwords = []string{"admin", "manager", "tomcat", "password"}

// TomcatChecker implements the scanner.Check interface. Used for guessing Tomcat creds 
type TomcatChecker struct{}

// Check attempts to identify guessable Tomcat credentials
func (c *TomcatChecker) Check(host string, port uint64) *scanner.Result {
    var (
        resp *http.Response err error
        url string
        res *scanner.Result 
        client *http.Client 
        req *http.Request
    )
    log.Println("Checking for Tomcat Manager...") 
    res = new(scanner.Result)
    url = fmt.Sprintf("http://%s:%d/manager/html", host, port) 
    if resp, err = http.Head(url); err != nil {
    	log.Printf("HEAD request failed: %s\n", err)
    	return res 
    }
	log.Println("Host responded to /manager/html request")
	// Got a response back, check if authentication required
	if resp.StatusCode != http.StatusUnauthorized || resp.Header.Get("WWW-Authenticate") == "" {
		log.Println("Target doesn't appear to require Basic auth.")
		return res 
    }
    
    // Appears authentication is required. Assuming Tomcat manager. Guess passwords... 
    log.Println("Host requires authentication. Proceeding with password guessing...") 
    client = new(http.Client)
    if req, err = http.NewRequest("GET", url, nil); err != nil {
    	log.Println("Unable to build GET request")
    	return res 
    }
    for _, user := range Users {
    	for _, password := range Passwords {
    		req.SetBasicAuth(user, password)
            if resp, err = client.Do(req); err != nil {
            	log.Println("Unable to send GET request")
            	continue 
            }
        	if resp.StatusCode == http.StatusOK {y
        		res.Vulnerable = true
        		res.Details = fmt.Sprintf("Valid credentials found - %s:%s", user, password) 
                return res
        	} 
        }
    }
    return res 
}
// New is the entry point required by the scanner 
func New() scanner.Checker {
    return new(TomcatChecker) 
}                                                                         
```

清单 10-3: 创建源生地 Tomcat 凭证猜测插件 (https://github.com/blackhat-go/bhg/ch-10/plugin-tomcat/main.go/)

首先，需要导入前面详细介绍过的 `scanner` 包。该包定义 `Checker` 接口和将要构建的 `Result` 结构体。首先定义名为 `TomcatChecker` 的一个空 `struct` 类型来实现 `Checker` 。要想满足 `Checker` 接口的实现需求，创建方法匹配 `Check(host string, port uint64) *scanner.Result` 。使用这些代码执行所有通用的漏洞检查逻辑。

由于期望返回 `*scanner.Result` ，初始化，并将其赋值给名为 `res` 的变量。如果条件满足——即，如果检查验证了可猜测的凭证，则确认了漏洞，将 `res.Vulnerable` 设置为 `true`，将 `res.Details` 设置为含有可识别的凭证的消息。如果漏洞不是可识别的，那么返回的实例的 `res.Vulnerable` 将被设置为默认的状态——`false` 。

最后，定义了需要导出的函数 `New() *scanner .Checker`。这符合扫描器调用 `Lookup()` 所设置的预期，以及实例化插件定义的`TomcatChecker` 所需要的类型断言和转换。这个基本入口点只是返回一个新的 `*TomcatChecker`（由于其实现了所必须的 `Check() 方法，因此恰好是一个 `scanner.Checker`）。

### 运行扫描器

既然已经创建了插件和使用它的主程序，那就编译插件，使用-o选项将编译后的共享对象定向到扫描程序的插件目录：

```shell
$ go build -buildmode=plugin -o /path/to/plugins/tomcat.so
```

然后运行扫描器 （cmd/scanner/main.go）来确认它能识别出插件，加载插件，并且执行插件的 `Check()` 方法：

```shell
$ go run main.go
Found plugin: tomcat.so
2020/01/15 15:45:18 Checking for Tomcat Manager...
2020/01/15 15:45:18 Host responded to /manager/html request
2020/01/15 15:45:18 Host requires authentication. Proceeding with password guessing... 2020/01/15 15:45:18 Host is vulnerable: Valid credentials found - tomcat:tomcat
```

能看到上面的输出吗？成功了！扫描器能够调用插件中的代码。可以在插件的目录中放入任意的插件了。扫描器能够尝试读取每个插件并启动漏洞检查功能。

代码有几处可以改进的地方。留给读者做为练习。可以从下面几个地方入手：

1. 创建检查不同漏洞的插件。 
2. 为方便更广泛的测试，添加动态主机列表及其开放端口。
3. 增强代码以只调用适用的插件。目前，代码将调用给定主机和端口的所有插件。这不是很理想。例如，如果目标端口不是HTTP或HTTPS，则不希望调用Tomcat检查。
4. 将插件系统转换为在Windows上运行，使用DLL作为插件类型。

在下一节中，将在一个不同的、非正式的插件系统：**Lua** 中构建相同的漏洞检查插件。

## 在**Lua**中构建插件

使用Go原生的`buildmode`创建可插拔的程序时有一定的局限性，特别由于可移植性不是很好，意味着插件不能很好的交叉编译。在本节中，我们将使用Lua创建插件来弥补这一不足。Lua是脚本语言，用来扩展各种的工具。Lua易于嵌入，强大，快速，并且友好的文档。像Nmap 和 Wireshark 这些安全工具使用Lua创建插件，和现在我们做的一样。更多的信息查看官方网站 *https://www.lua.org/* 。

要在Go中使用Lua，须使用第三方包 `gopher-lua` ，该包能够用Go直接编译并执行Lua。输入以下内容，将其安装在系统上：

```shell
$ go get github.com/yuin/gopher-lua
```

现在，请注意，可移植性带来的代价是复杂性的增加。