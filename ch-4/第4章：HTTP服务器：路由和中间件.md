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

### 构建简单的中间件

现在来构建中间件，是一种执行所有传入的请求的包装，而不管目标函数是什么。代码4-3的例子中，将创建一个显示请求处理开始和结束时间的logger。
```go
package main

import (
	"fmt"
	"log"
	"net/http"
	"time"
)

type logger struct {
	Inner http.Handler
}

func (l *logger) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	log.Printf("start %s\n", time.Now().String())
	l.Inner.ServeHTTP(w, req)
	log.Printf("finish %s\n", time.Now().String())
}

func hello(w http.ResponseWriter, req *http.Request) {
	fmt.Fprint(w, "Hello\n")
}

func main() {
	f := http.HandlerFunc(hello)
	l := logger{Inner: f}
	http.ListenAndServe(":8000", &l)
}
```
代码 4-3: 简单的中间件 (https://github.com/blackhat-go/bhg/ch-4/simple_middleware/main.go/)

实际上是对于每个请求创建一个外部处理器，记录服务的一些信息，然后调用`hello()`函数。将日志逻辑到函数中。

如同简单的路由例子，定义了名为`logger`的新类型，但是这次带有一个`Inner`字段，其本身是`http.Handler`。在`ServeHTTP()`定义中，使用`log()`输出请求的开始和结束时间，在这之间调用内部的`ServeHTTP()`方法。对于客户端，请求会在内部的处理器结束。`main()`函数内，使用`http.HandlerFunc()`在函数外面创建了一个`http.Handler`。创建`logger`，给新创建的处理器设置`Inner`。最后，使用`logger`的指针启动服务。

运行代码，然后发送请求就好输出两个本次请求开始和结束时间的信息：
```shell script
$ go build -o simple_middleware
$ ./simple_middleware
2020/01/16 06:23:14 start
2020/01/16 06:23:14 finish
```
接下来的部分，深入研究中间件和路由，并使用我们喜欢的第三方包，其能创建更动态的路由和在链中执行中间件。我们还将讨论迁移到更复杂场景中的中间件的一些用例。


### 使用`gorilla/mux`包创建路由

如同代码4-2所示，通过路由将请求的路径匹配到函数。但是也可以用来将其他属性——HTTP方法或host头——与函数匹配。Go的生态中有几种第三方路由。这里介绍其中的一种：`gorilla/mux`。但就像其他事情一样，当遇到其他包时，我们鼓励通过研究来扩展知识。

`gorilla/mux`是个成熟的第三方包，支持简单和复杂模式的路由。包括正则表达式，模式匹配，方法匹配，和子路由，其他功能等。

通过几个简单的例子来看下如何使用该路由包。无需运行，因为很快就会在程序中使用它们，但是请随意尝试和试验。

使用`gorilla/mux`之前先安装：
```shell script
$ go get github.com/gorilla/mux
```
现在可以开始了。使用`mux.NewRouter()`创建路由：
```go
r := mux.NewRouter()
```

返回的类型实现了`http.Handler`，但是也有其他相关的方法。经常用的一个是`HandleFunc()`。举例，如果定义一个新路由处理对模式`/foo`的GET请求`，你可以这样使用：
```go
r.HandleFunc("/foo", func(w http.ResponseWriter, req *http.Request) {
    fmt.Fprint(w, "hi foo")
}).Methods("GET")
```
现在，因为调用了`Methods()`，该路由只匹配GET请求。对其他类型的请求返回404响应。还可以继续链接其他的限定，像`Host(string)`，匹配特定的host头。下面例子只匹配host头设置为`www.foo.com`的请求。
```go
r.HandleFunc("/foo", func(w http.ResponseWriter, req *http.Request) {
    fmt.Fprint(w, "hi foo")
}).Methods("GET").Host("www.foo.com")
```
有时，在请求路径中匹配并传递参数是很有帮助的（例如，实现RESTful API时）。使用`gorilla/mux`非常简单，下例将打印出请求路径中`/user/`后面的内容:
```go
r.HandleFunc("/users/{user}", func(w http.ResponseWriter, req *http.Request) {
    user := mux.Vars(req)["user"]
    fmt.Fprintf(w, "hi %s\n", user)
}).Methods("GET")
```
在路径定义中，使用大括号定义请求参数。可以将其看作一个已命名的占位符。然后，在处理函数内，调用`mux.Vars()`来解析请求体，返回`map[string] string`类型的数据，其值为请求的参数名字和各自的值。使用`user`作关键字。因此，`/users/bob`的请求就会产生对Bob的问候。
```shell script
$ curl http://localhost:8000/users/bob
hi bob
```
可以更进一步，使用正则表达式限定传递的模式。例如，可以限定user参数必须是小写字母：
```go
r.HandleFunc("/users/{user:[a-z]+}", func(w http.ResponseWriter, req *http.Request) {
    user := mux.Vars(req)["user"]
    fmt.Fprintf(w, "hi %s\n", user)
}).Methods("GET")
```
任何不匹配的模式现在都会返回404响应：
```shell script
$ curl -i http://localhost:8000/users/bob1 HTTP/1.1
404 Not Found
```
在下一节中，我们将扩展路由，包括一些使用其他库的中间件。这会增加处理HTTP请求的灵活性。

### 使用Negroni构建中间件

之前介绍的简单的中间件记录处理请求的开始和结束时间，然后返回响应。中间件无需对每个传入的请求都进行操作，且大多数情况下也都是这样的。使用中间件的理由很多，记录请求，身份认证，用户认证，映射资源。

例如，可以写个执行基础认证的中间件。能够解析每个请求的认证头，验证用户名和密码，如果凭据无效就返回401响应。还可以以这样的方式将多个中间件函数链接在一起使用，即在执行完一个后再运行下一个。对于本章前面创建的日志中间件，只封装了一个函数。实际上，不是很有用，因为要使用不止一个，为此，必须有可以在一个接一个的链中执行它们的逻辑。从零开始写也并不难，但是不用重复造轮子了。现在已经有成熟的包做到这一点了：`negroni`。

`negroni`，链接为`https://github.com/urfave /negroni/`，非常优秀，因为不是很大的框架。在其他框架中也很容易使用，也非常灵活。还附带了对程序都很有用的默认中间件。在使用之前先获取：
```shell script
$ go get github.com/urfave/negroni
```

虽然在技术上可以将negroni用于所有应用程序逻辑，但这样做并不理想，因为它是专门为充当中间件而构建的，并不是路由。相反，最好将`negroni`和其他包结合使用，如`gorilla/mux`或`net/http`。让我们使用`gorilla/mux`来构建个程序，再熟悉下`negroni`，并在他们穿越中间件练时能可视化操作的顺序。

首先，在一个目录中创建名为`main.go`的新文件，如`github.com/blackhat-go/bhg/ch-4/negroni _example/`。（在克隆BHG Github仓库时，已经创建了这个目录。）现在将下面的代码加到`main.go`文件中。
```go
package main

import (
    "net/http"

    "github.com/gorilla/mux"
    "github.com/urfave/negroni"
)

func main() {
    r := mux.NewRouter()
    n := negroni.Classic()
    n.UseHandler(r)
    http.ListenAndServe(":8000", n)
}
```
代码 4-4: Negroni 例子 (https://github.com/blackhat-go/bhg/ch-4/negroni_example/main.go/)

首先，像本章之前那样调用`mux.NewRouter()`来创建路由。接下来是和`negroni`第一次交互：调用`negroni.Classic()`。创建了一个Negroni实例的指针。

可以使用不同的方式。既可以使用 `negroni.Classic()`，也可以调用`negroni.New()`。第一个，`negroni.Classic()`，设置默认中间件，包括请求日志，被拦截和panic时恢复中间件，在同一目录中从公用文件夹中提供文件的中间件。`negroni.New()`函数不会创建任何默认的中间件。

在`negroni`包中，每种类型的中间件都是可用的。例如，可以通过下面的步骤使用恢复包:
```go
n.Use(negroni.NewRecovery())
```

接下来，通过调用`n.UseHandler(r)`将路由添加到中间件上。当继续规划和构建中间件时，考虑下执行顺序。例如，认证中间件要在需要认证的处理函数之前执行。在路由之前的中间件要先处理函数之前执行；在路由之后的中间件要在处理函数之后执行。顺序很重要。本例中，还没有定义任何中间件，但很快就会了。

继续编译之前创建的代码4-4，然后运行。然后向服务监听的地址`http://localhost:8000`发送web请求。`negroni`日志中间件就好输出下面的信息。输出带有时间戳，响应码，处理时间，host，和HTTP方法。
```shell script
$ go build -s negroni_example
$ ./negroni_example
[negroni] 2020-01-19T11:49:33-07:00 | 404 | 1.0002ms | localhost:8000 | GET
```

有默认中间件非常好，但真正的能力是构建自己的中间件。使用`negroni`调用几个方法就能添加中间件。看一下下面的代码。其创建了`trivial`中间件，打印信息，然后将执行传递给链中的下一个中间件:
```go
type trivial struct {
}
func (t *trivial) ServeHTTP(w http.ResponseWriter, r *http.Request, next http.HandlerFunc) {
    fmt.Println("Executing trivial middleware")
    next(w, r)
}
```

这里的实现和之前的例子有点不一样。之前实现了`http.Handler`接口，`ServeHTTP()`方法接收两个参数：`http.ResponseWriter`和` *http.Request`。在本例中，实例`negroni.Handler`接口来替换了` http.Handler`接口。稍微不同的地方是`negroni.Handler`接口要实现`ServeHTTP()`方法，其接收的参数不是两个，而是三个：`http.ResponseWriter`, `*http.Request`, 和 `http.HandlerFunc`。`http.HandlerFunc`这个参数用来指示链中的下一个中间件函数。便于理解将其命名为`next`。在`ServeHTTP()`中处理，然后调用`next()`。传递原先接收到的`http.ResponseWriter`和`*http.Request`的值。这有效地将执行向下转移。

但是仍然要让`negroni`知道将您的实现作为中间件链的一部分。通过调用negroni的`Use`方法，并传递实现的`negroni.Handler`实例就可以了：
```go
n.Use(&trivial{})
```

使用这种简便的方法来写自己的中间件，因为这很容易地将执行传递给下一个中间件。缺点是：必须使用`negroni`。举个例子，如果写了个给响应添加安全头的中间件，希望其实现`http.Handler`，因此可以在其他的应用中复用，又由于多数的应用没有negroni.Handler。关键是，不管中间件的用途是什么，当在在无`negroni`应用中使用negroni中间件时可能会出现兼容性问题，反之亦然。

有两种方式告诉`negroni`使用你的中间件。`UseHandler (handler http.Handler)`，已经非常熟悉了，这是第一种。第二种是调用` UseHandleFunc(handlerFunc func(w http.ResponseWriter, r *http.Request))`。后者不经常使用，不允许放弃执行链中的下一个中间件。举例，如果写了个执行认证的中间件，如果有任何凭证或session信息无效的话就直接返回401响应；使用第二种方式的话不可能实现的。

