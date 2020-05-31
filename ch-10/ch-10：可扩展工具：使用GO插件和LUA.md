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

