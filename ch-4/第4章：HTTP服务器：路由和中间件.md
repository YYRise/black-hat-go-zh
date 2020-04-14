## 摘要
如果您知道如何从零开始写HTTP服务器，就能够为社会工程，命令和控制（C2）传输，或者你自己的工具创建api和前端以及其他的内容创建自定义逻辑。幸运的是，Go有出色的标准包——`net/http`——构建HTTP服务；这是不仅仅是有效地写简单的服务所有需要的，以及复杂的、功能齐全的web应用程序所需要的。

除了标准包外，可以使用其他第三方包快速地开发和减少繁琐的程序，例如模式匹配。这些包有助于你使用路由，构建中间件，校验请求等。

本章中，首先探讨构建简单HTTP应用需要的技术。然后使用这些技术创建两个社会工程应用——凭证收集服务器和keylogging服务——和多路C2管道。

## HTTP服务的基础知识

本部分通过构建简单服务，路由，中间件来探讨`net/http`包和有用的第三方包。本章最后会扩展这些基础知识完成更多邪恶的示例。

### 构建简单的服务

代码4-1是启动了处理单个路由的服务。服务器根据`name`获取含有用户名的URL参数，并用定制的问候响应。
```go
package main

import (
	"fmt"
	"net/http"
)

func hello(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello %s\n", r.URL.Query().Get("name"))
}

func main() {
	http.HandleFunc("/hello", hello)
	http.ListenAndServe(":8000", nil)
}
```
代码 4-1: Hello World 服务 (https://github.com/blackhat-go/bhg/ch-4/hello_world/main.go/)

该示例在`/hello`处公开资源，资源关联参数并将其值回传给客户端。`main()`函数内的`http.HandleFunc()`带有两个参数：一个是字符串，指示服务器寻找的URL路径模式，另一个是实际上处理请求的函数。如果愿意的话，可以使用匿名内联函数。本例中，使用的是早定义好的`hello()`函数。

`hello()`函数处理请求，并返回给客户端一个hello消息。该函数本身需要两个参数。第一个是`http.ResponseWriter`，用于给请求写入响应。第二个参数是`http.Request`指针，用于从收到的请求中读取信息。注意，在`main()`中没有调用`hello()`函数。仅仅告诉HTTP服务器对于`/hello`的任何请求都由`hello()`函数处理。

实际上，`http.HandleFunc()`是做了什么呢? Go文档中指出在`DefaultServerMux`上处理。`ServerMux`是`server multiplexer`的缩写，只是个花哨的说法，底层代码能够处理多个请求模式和函数。使用协成处理，收到的每一个请求开启一个协成。导入`net/http`包就创建了ServerMux，并附加在包的命名空间上，即`DefaultServerMux`。

下行代码调用`http.ListenAndServe() `，该函数采用一个字符串和一个`http.Handler`作为参数。使用第一个参数作为地址启动一个HTTP服务。在本例中是`:8000`，也就是在所有接口中监听8000端口。对于第二个参数，即`http.Handler`，传入的是`nil`。结果是使用`DefaultServerMux`在底层处理。马上就实现自己的`http.Handler`，然后出入。但目前仅使用默认的即可。也可以使用`http.ListenAndServeTLS()`，如名字所描述那样，其使用`HTTPS` 和 `TLS`开启服务，但是需要另外的参数。

实现`http.Handler`接口需要单个方法：``ServeHTTP(http.ResponseWriter, *http.Request)`。这很棒，因为简化了自定义HTTP服务的创建。会发现很多第三方的实现，通过添加特色来扩展`net/http`功能，如中间件，认证，响应编码等等。

可以使用`curl`来测试服务：
```shell script
$ curl -i http://localhost:8000/hello?name=alice 
HTTP/1.1 200 OK
Date: Sun, 12 Jan 2020 01:18:26 GMT 
Content-Length: 12
Content-Type: text/plain; charset=utf-8 

Hello alice
```
太好了!所构建的服务器读取URL中的`name`参数并使用问候语回复。

### 构建简单的路由

接下来构建简单的路由，如代码4-2所示。演示了如何通过检查URL路径来动态处理收到的请求。取决于URL是否含有路径`/a, /b, 或 /c`，打印出` Executing /a, Executing /b, 或 Executing /c`。其他的打印`404 Not Found`。
```go
package main

import (
	"fmt"
	"net/http"
)

type router struct {
}

func (r *router) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	switch req.URL.Path {
	case "/a":
		fmt.Fprint(w, "Executing /a")
	case "/b":
		fmt.Fprint(w, "Executing /b")
	case "/c":
		fmt.Fprint(w, "Executing /c")
	default:
		http.Error(w, "404 Not Found", 404)
	}
}

func main() {
	var r router
	http.ListenAndServe(":8000", &r)
}
```
代码 4-2: 简单的路由 (https://github.com/blackhat-go/bhg/ch-4/simple_router/main.go/)

首先，定义了没有任何字段名为`router`的类型。用于实现`http.Handler`接口。为此，必须定义` ServeHTTP()`方法。该方法在请求的URL路径上使用一个`switch`语句，基于路径执行不同的逻辑。使用`404 Not Found`响应默认操作。在`main()`中，创建了一个`router`实例，并将指针传给`http.ListenAndServe()`。

在终端里执行下：
```shell script
$ curl http://localhost:8000/a
Executing /a
$ curl http://localhost:8000/d
404 Not Found
```

如期运行；程序对URL含`/a`路径的返回`Executing /a`，对不存在的路径返回`404 Not Found`响应。这是一个简单的例子。第三方路由会有更复杂的逻辑。但这应该能让你对它们的工作原理有一个基本的了解。


