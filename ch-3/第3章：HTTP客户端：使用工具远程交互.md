、

## 摘要
在第2章中，您学习了如何用TCP创建可用的客户端和服务器。本章是探讨 OSI 模型较高层上的各种协议的第一章。
基于网络中的广泛性，~~其隶属关系宽松的出口控制(todo)~~，一般的灵活性，先从HTTP开始。

本章节专注于客户端。首先介绍构建和自定义HTTP请求并接收其响应的基础知识。然后，学习如何解析结构化的响应数据，以便客户可以访问数据以确定可行或相关数据。最后，通过构建与各种安全工具和资源进行交互的HTTP
客户端来学习如何应用这些基础知识。开发的客户端将查询和使用 Shodan，Bing 和 Metasploit 的 API ，并以类似于元数据搜索工具FOCA的方式搜索和解析文档元数据

## Go中HTTP原理

尽管没必要全面了解HTTP，但是开始时最好知道一些。

首先，HTTP是 *无状态协议*：服务器天生不会维护每个请求的状态。而是用多种方式跟踪状态，这些方式可能包括会话session标记，cookie，HTTP标头等。服务器和客户端需正确协商和验证此状态。

其次，客户端和服务器之间的通信可以是同步或异步的，但要以请求/响应周期运行。可以在请求中添加几个选项和header，以决定服务器的行为并创建可用的Web程序。最常见的是，服务器持有Web
浏览器渲染的文件，以生成数据的图形化，组织化和风格化的表现形式。但是服务器可以返回任意的数据类型。API通常通过结构化的数据编码（例如XML，JSON或MSGRPC）进行通信。某些情况下，数据可能是二进制格式，表示下载的文件是任意类型。

最后，使用Go提供的函数，可以快速轻松地构建HTTP请求并将其发送到服务器，然后获取并处理响应。通过前面几章中学到的一些技巧，使用结构化数据的约定非常方便地处理与HTTP API交互。

### 调用HTTP API

我们通过查看基本的请求开始HTTP的学习。Go的`net/http`包中有几个方便的函数能快速轻松地发送POST，GET和HEAD请求，这也是最常用的HTTP动词。这些函数的形式如下：
```go
Get(url string) (resp *Response, err error)
Head(url string) (resp *Response, err error)
Post(url string, bodyType string, body io.Reader) (resp *Response, err error)
```
每个函数都有一个参数——URL字符串，请求的目的地。`Post()`函数比`Get()`和`Head()`这两个函数稍微复杂点。`Post()`有两个额外的参数：`bodyType`，用于请求正文的Content-Type HTTP
 标头（通常是application/x-www-form-urlencoded）的字符串，`io.Reader`，在第2章学过。在清单3-1中有每个函数的示例实现。请注意，POST请求从表单创建请求主体并设置Content-Type
 标头。在每种情况下，都必须在从响应体读完数据后关闭它。
 ```go
r1, err := http.Get("http://www.google.com/robots.txt") // Read response body. Not shown.
defer r1.Body.Close()
r2, err := http.Head("http://www.google.com/robots.txt") // Read response body. Not shown.
defer r2.Body.Close() 
form := url.Values{} 
form.Add("foo", "bar") 
r3, err = http.Post(
    "https://www.google.com/robots.txt", 
    "application/x-www-form-urlencoded",
    strings.NewReader(form.Encode()w), 
)
// Read response body. Not shown. 
defer r3.Body.Close()
```
<center> 代码 3-1: Get(), Head(), 和 Post() 函数的示例 (https://github.com/blackhat-go/bhg
/ch-3/basic/main.go/) </center>

POST函数使用相当普通的调用模式，即URL编码表单数据时，设置Content-Type为application / x-www-form-urlencoded。

Go中有另外的POST请求简便函数叫做`PostForm()`，移除了设置这些值和手动编码每个请求的繁琐工作；语法如下：
```go
func PostForm(url string, data url.Values) (resp *Response, err error)
```
如果想使用` PostForm() `函数替代代码3-1中的`Post()`，使用像代码3-2中的代码那样。
```go
form := url.Values{}
form.Add("foo", "bar")
r3, err := http.PostForm("https://www.google.com/robots.txt", form) // Read response body and close.
```
代码 3-2: 使用`PostForm()`函数替代`Post()` (https://github.com/blackhat -go/bhg/ch-3/basic/main.go/)

不幸的是，没有其他的HTTP方法（例如PATCH，PUT或DELETE）没有这种简便函数。主要使用这些方法与RESTful API进行交互，RESTful API
提供了服务器使用这些方法的方式和原因的一般指导；但没有什么是一成不变的，而在方法方面，HTTP就像古老的西方。实际上，我们经常想到创建一个专门使用DELETE进行所有操作的新网络框架的想法。我们称它为` DELETE.js
`，毫无疑问，这将是Hacker News的热门话题。读到这，即表示您同意不窃取这个想法！

### 创建请求

使用 ` NewRequest()`函数为这些HTTP方法创建请求。该函数创建`Request`结构体，接下来使用客户端函数` Do()`方法发送。其实要比听起来简单多了。`http.NewRequest()`函数原型如下：
```go
func NewRequest(method,url string,body io.Reader) (resp *Response, err error)
```
前两个字符串参数分别为HTTP方法名和URL地址。很像代码3-1中的POST例子。可以选择通过传入io.Reader作为第三个也是最后一个参数来提供请求体。

代码3-3是无HTTP请求体——DELETE请求。
```go
req, err := http.NewRequest("DELETE", "https://www.google.com/robots.txt", nil) var client http.Client
resp, err := client.Do(req)
// Read response body and close.
```
代码 3-3: 发送 DELETE 请求 (https://github.com/blackhat-go/bhg/ch-3/basic /main.go/)

现在，代码3-4是使用 `io.Reader` 的 PUT请求（看起来像PATCH请求）。
```go
form := url.Values{} form.Add("foo", "bar")
var client http.Client
req, err := http.NewRequest(
"PUT", "https://www.google.com/robots.txt", strings.NewReader(form.Encode()),
)
resp, err := client.Do(req)
// Read response body and close.
```
代码 3-4: 发送 PUT 请求 (https://github.com/blackhat-go/bhg/ch-3/basic /main.go/)

标准的Go `net/http` 库包含一些请求在发送到服务器之前对其进行操作的函数。通过阅读本章中的实战示例，您将学到一些更相关和实用的变体。但是，先来演示下如何有目的地处理服务器收到的HTTP请求。

### 使用结构体解析响应

之前的章节已经学会了用Go创建并发送HTTP请求。示例代码中都未处理响应，基本上都忽略了。但是检测HTTP响应的组成是任何处理HTTP相关任务的关键，像读取响应体，访问cookies 和 headers，或者简单的检查HTTP状态码。

为了演示状态码和响应体——本例中是Google的 robots.txt 文件，代码3-5简化了代码3-1中GET请求，用`ioutil.ReadAll()` 函数从响应体中读取数据，并检查错误，然后打印HTTP的状态码和响应体的内容。
```go
resp, err := http.Get("https://www.google.com/robots.txt") 
if err != nil {
    log.Panicln(err) 
}
// Print HTTP Status 
fmt.Println(resp.Status)
// Read and display response body
body, err := ioutil.ReadAll(resp.Body) 
if err != nil {
    log.Panicln(err) 
}
fmt.Println(string(body))
resp.Body.Close()
```
代码 3-5: 处理 HTTP 响应体 (https://github.com/blackhat-go/bhg/ch-3/basic/main.go/)

收到响应后，在上面代码命名为 `resp`，通过访问暴露出的 `Status` 参数就能获取到状态字符串（例如，200 ok）；有一个类似的 `StatusCode` 参数，该参数仅访问状态字符串的整数部分，未在示例中展示。

响应中还暴露出一个`io.ReadCloser`类型的`Body`参数。`io.ReadCloser` 既是`io.Reader`又是 `io.Closer` 接口，或者是需要实现 `Close()` 函数的接口以关闭读取和执行清理。现在先不用考虑具体的细节；只是从`io.ReadCloser` 读取完数据后记得调用响应体`Close()`函数来关闭。通常使用`defer`关闭响应体；这能确保退出函数之前不会忘记关闭响应体。

现在回到终端查看错误状态和响应体内容：
```shell script
$ go run main.go
200 OK
User-agent: * 
Disallow: /search 
Allow: /search/about 
Disallow: /sdch 
Disallow: /groups 
Disallow: /index.html? 
Disallow: /?
Allow: /?hl=
Disallow: /?hl=*&
Allow: /?hl=*&gws_rd=ssl$ 
Disallow: /?hl=*&*&gws_rd=ssl 
--snip--
```

如果需要解析结构化数据——很可能会的——使用第2章的约定读取响应体并解码。举例，假如和端口使用JSON和API通信交互（如`/ping`），返回以下响应表明服务器状态：
```json
{"Message":"All is good with the world","Status":"Success"}
```
使用代码 3-6 中的程序与此端点交互并解码 JSON 消息。
```go
package main
import { 
    encoding/json"
    log
    net/http 
}
type Status struct { 
    Message string 
    Status string
}
func main() {
    res, err := http.Post(
        "http://IP:PORT/ping", 
        "application/json", 
        nil,
    )
    if err != nil {
        log.Fatalln(err) 
    }
    var status Status
    if err := json.NewDecoder(res.Body).Decode(&status); err != nil {
        log.Fatalln(err) 
    }
    defer res.Body.Close()
    log.Printf("%s -> %s\n", status.Status, status.Message) }
```
代码 3-6: 解码 JSON 响应体 (https://github.com/blackhat-go/bhg/ch-3/basic-parsing/main.go/)

代码先定义`Status`结构体，含义预期的服务器返回的元素。`main()`函数先发送`POST`请求，然后解析响应体。这之后就可以正常的使用`Status`结构——访问暴露出的`Statu`和`Message`数据类型

其他编码格式解析这种结构化数据也是一样的，像XML，甚至是二进制。先定义期望响应数据的结构体，然后将数据解析到该结构体。解析其他样式的细节和实现过程留给你自己去研究了。

下一节将应用这些基本概念来构建工具与第三方 API 进行交互，顺便增强黑客技术。

## 构建和Shodan交互的HTTP客户端


