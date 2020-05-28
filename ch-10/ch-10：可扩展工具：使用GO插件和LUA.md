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