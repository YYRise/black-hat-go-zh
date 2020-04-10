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
在对组织进行任何授权的对抗活动之前，优秀的黑客都先从侦察开始。典型地，不向目标发送数据包的被动技术开始；这样，几乎不可能检测到活动。黑客使用各种资源和服务——包括社交网络，公共记录，搜索引擎——来获取目标潜在的有用的信息。

令人难以置信的是，将环境上下文应用在连锁攻击场景中时，看似良性的信息如何变得至关重要。举例，仅将披露详细错误消息的Web应用程序视为低严重性。但是，如果错误消息公开了企业用户名格式，并且组织对VPN使用了单向身份验证，则这些错误消息可能会增加通过猜测密码攻击内部网络的可能性。在收集信息的同时保持低调，可确保目标的感知和安全态势保持中立，从而增加攻击成功的可能性。

Shodan (https://www.shodan.io/)，自称为“世界上第一个互联网连接设备搜索引擎，通过维护可搜索的网络设备和服务数据库来促进被动侦察，包括产品名称，版本，语言环境等元数据。将Shodan看作是扫描数据的存储库，即使它能做更多很多的事。

### 回顾构建API客户端的步骤

在接下来的几节构建和Shodan API交互的HTTP客户端，解析结果并展示相关信息。首先，需要Shodan API的key，在Shodan网站注册后就能获得。在撰写本文时，最低级别的费用非常合适，满足个人使用的需求，因此请注册吧。Shodan偶尔会有折扣，如果你想节省几块钱的话就要多关注点网站动态。

现在，从网站获得API key后设置为环境变量。接下来的例子使用的保存APT key的环境变量`SHODAN_API_KEY`。请参阅操作系统的用户手册，或者如果需要帮助设置变量，请参考第一章。

在研究代码之前，请了解本节演示的如何创建客户端的基本实现，而不是全部功能实现。不过，现在将要构建的基础的脚手架将使您可以轻松扩展演示的代码，以实现调用可能需要的其他API。

构建的客户端实现了两个API调用：一个查询订阅信用信息，另一个搜索含义某字符的hosts。使用后一个调用来标识主机。例如，与特定产品匹配的端口或操作系统。

幸运的是，Shodan API简单易懂，使用的易于理解的JSON格式的响应。这使其成为学习API交互的良好起点。以下是准备和构建API客户端的典型步骤的整体概述：
1. 查阅服务端API文档
2. 设计合理的代码结构以减少复杂性和重复性
3. 用GO按需定义请求和响应的类型
4. 创建辅助函数和类型，以简化简单的初始化，身份验证和通信，以减少冗余或重复的逻辑
5. 构建与API的函数和类型交互的客户端

我们在本节中没有明确指出每个步骤，但是您应该使用此列表作为指导来学习。开始快速地查阅Shodan网站的API文档。该文档很少，但是提供了创建客户端程序所需的一切。

### 设计项目结构
构建API客户端时，应对其进行结构设计，以使函数调用和逻辑独立。这可以在其他项目中作为单独的库来复用。这样就不会重复造轮子了。可复用性的架构会稍微改变项目的结构。以Shodan为例，项目的结构为：
```shell script
$ tree github.com/blackhat-go/bhg/ch-3/shodan github.com/blackhat-go/bhg/ch-3/shodan |---cmd
| |---shodan
| |---main.go |---shodan
|---api.go |---host.go |---shodan.go
```

`main.go`文件定义包`main`，主要用作构建的API的使用者；这样的话，主要使用该文件于客户端交互。

在`shodan`文件夹中的——`api.go, host.go, 和 shodan.go`——定义了包`shodan`，其包含了和`Shodan`通信所需要的类型和函数。这个包会是一个单独的库，可以导入的其他项目中使用。

### 清理API调用

当精读Shodan API文档时，您可能会注意到每个暴露的函数都需要发送您的API key。尽管可以肯定地将该值传递给所创建的函数，但该重复性的工作是乏味。类似的还有硬编码和操作URL (https://api.shodan.io/)。举例来说，如下所示定义了一个API函数，需要使用token和URL参数，这就有点不简洁：
```go
func APIInfo(token, url string) { --snip-- } 
func HostSearch(token, url string) { --snip-- }
```
可以选择一种惯用的解决方案来替代，更少的代码和更高的可读性。为此，创建`shodan.go`文件，然后使用代码3-7中的代码：
```go
package shodan
const BaseURL = "https://api.shodan.io"
type Client struct { 
    apiKey string
}
func New(apiKey string) *Client { return &Client{apiKey: apiKey}
}
```
代码 3-7: 定义Shodan客户端 (https://github.com/blackhat-go/bhg/ch-3/shodan.go/)

Shodan URL被定义为常量；这样，在函数中易于访问和复用。如果Shodan更改API的URL，只需要改动这一个地方就能改动整个代码库。接下来定义了`Client`结构体，保存请求API时使用的token。最后，定义了`New()`辅助函数，以API的token为参数，创建并返回已初始化了的Client实例。现在，无需将API代码创建为任意函数，而是将它们创建为Client的方法，这使就可以直接通过Client实例查询，而不必依赖过于冗长的函数参数。将稍后要讨论的API函数改为以下内容：
```go
func (s *Client) APIInfo() { --snip-- } 
func (s *Client) HostSearch() { --snip-- }
```
因为这都是Client的方法，可以直接通过`s.apiKey`获得API key，通过`BaseURL`得到URL。调用这些方法的唯一前提条件是首先创建Client实例。通过`shodan.go`文件中`New()`辅助函数就可以了。

### 查询Shodan订阅

现在可以开始和Shodan交互了。根据Shodan API文档，用于查询订阅计划信息的调用如下：
```html
https://api.shodan.io/api-info?key={YOUR_API_KEY}
```

返回的响应类似于下面的结构。显然，这基于计划详情和剩余的订阅支付有所不同。
```json
{
  "query_credits": 56, 
  "scan_credits": 0, 
  "telnet": true, 
  "plan": "edu", 
  "https": true, 
  "unlocked": true
}
```
首先，在`api.go`中，定义能够将JSON解析到Go的结构。没有它，将无法处理或查询响应体。该例中命名为`APIInfo`：
```go
type APIInfo struct {
    QueryCredits    int     `json:"query_credits"`
    ScanCredits     int     `json:"scan_credits"`
    Telnet          bool    `json:"telnet"` 
    Plan            string  `json:"plan"`
    HTTPS           bool    `json:"https"`
    Unlocked        bool    `json:"unlocked"`
}
```
Go的使结构体和JSON对应的功能棒极了。像第一章看到的那样，使用一些很棒的工具“自动”解析JSON——填充结构体中的字段。对于结构体暴露出的每一个字段，都可以用结构体标签明确地定义对应的JSON名字，这样能确保正确地映射和解析。

接下来需要执行代码3-8中的函数，该函数使用HTTP GET方法向Shodan请求，然后将响应解析到对应的`APIInfo`结构体。
```go
func (s *Client) APIInfo() (*APIInfo, error) {
    res, err := http.Get(fmt.Sprintf("%s/api-info?key=%s", BaseURL, s.apiKey))
    if err != nil {
        return nil, err 
    }
    defer res.Body.Close()
    var ret APIInfo
    if err := json.NewDecoder(res.Body).Decode(&ret); err != nil {
        return nil, err 
    }
    return &ret, nil }
```
代码 3-8: 发送 HTTP GET 请求并解析响应 (https://github.com/blackhat-go/bhg/ch-3/shodan/shodan/api.go/)

代码简短而精悍。先向/api-info发出HTTP GET请求。该URL由全局常量`BaseURL`和`s.apiKey`创建。然后将响应解析到`APIInfo`结构体中，最后返回给调用者。

在编写使用这种新颖逻辑的代码之前，请构建第二个更有用的API调用——主机搜索——将其添加到`host.go`中。根据API文档的描述，请求和响应如下：
```json
https://api.shodan.io/shodan/host/search?key={YOUR_API_KEY}&query={query}&facets={facets} 
{
  "matches": [ 
  {
    "os": null,
    "timestamp": "2014-01-15T05:49:56.283713", 
    "isp": "Vivacom",
    "asn": "AS8866",
    "hostnames": [ ],
    "location": {
        "city": null, 
        "region_code": null, 
        "area_code": null, 
        "longitude": 25, 
        "country_code3": "BGR", 
        "country_name": "Bulgaria", 
        "postal_code": null, 
        "dma_code": null, 
        "country_code": "BG", 
        "latitude": 43
    },
    "ip": 3579573318, 
    "domains": [ ],
    "org": "Vivacom",
    "data": "@PJL INFO STATUS CODE=35078 DISPLAY="Power Saver" ONLINE=TRUE", 
    "port": 9100,
    "ip_str": "213.91.244.70"
  },
  --snip--
  ], 
  "facets": {
    "org": [ 
    {
      "count": 286,
      "value": "Korea Telecom" 
    },
    --snip--
    ] 
  },
  "total": 12039 
}
```
与刚实现的初始 API 调用相比这个复杂多了。该请求不仅需要多个参数，而且JSON响应还包含嵌套的数据和数组。下面的实现忽略了`facets`选型的数据，重点放在基于字符串的主机搜索的处理过程，以仅匹配响应体的元素。

像之前做的那样，先创建处理响应体数据对应的Go的结构体；将代码3-9加到`host.go`文件中。
```go
type HostLocation struct{
    City            string      `json:"city"` 
    RegionCode      string      `json:"region_code"`
    AreaCode        int         `json:"area_code"`
    Longitude       float32     `json:"longitude"`
    CountryCode3    string      `json:"country_code3"`
    CountryName     string      `json:"country_name"`
    PostalCode      string      `json:"postal_code"`
    DMACode         int         `json:"dma_code"`
    CountryCode     string      `json:"country_code"`
    Latitude        float32     `json:"latitude"`     
}

type Host struct {
    OS        string        `json:"os"` 
    Timestamp string        `json:"timestamp"`
    ISP       string        `json:"isp"` 
    ASN       string        `json:"asn"`
    Hostnames []string      `json:"hostnames"`
    Location  HostLocation  `json:"location"`
    IP        int64         `json:"ip"`
    Domains   []string      `json:"domains"`
    Org       string        `json:"org"`
    Data      string        `json:"data"`
    Port      int           `json:"port"`
    IPString  string        `json:"ip_str"` 
}

type HostSearch struct {
    Matches []Host `json:"matches"`
}
```
代码 3-9: Host 搜索响应体数据类型 (https://github.com/blackhat-go/bhg/ch-3/shodan/shodan/host.go/)

代码中定义了三种类型：  
**HostSearch** 解析matches 数组   
**Host** 单个matches对象  
**HostLocation** host中的location元素 

注意，没有定义所有的响应字段类型。Go优雅地处理了仅使用关心的JSON字段来定义结构。因此，这些代码解析JSON刚刚好，同时还能减少代码量。通过代码3-10中定义的函数来初始化并返回结构体，和代码3-8中的`APIInfo()`方法类似。
```go
func (s *Client) HostSearch(q stringu) (*HostSearch, error) { 
    res, err := http.Get(
        fmt.Sprintf("%s/shodan/host/search?key=%s&query=%s", BaseURL, s.apiKey, q), 
    )
    if err != nil { 
        return nil, err
    }
    defer res.Body.Close()
    var ret HostSearch
    if err := json.NewDecoder(res.Body).Decode(&ret); err != nil {
        return nil, err
    }
    return &ret, nil
}
```
代码 3-10: 解析 host 搜索的响应头 (https://github.com/blackhat-go/bhg/ch-3/shodan/shodan/host.go/)

流程逻辑也和`APIInfo()`方法类似，除了需要搜索字符串作为参数发送到`/shodan/host/search`端点，然后将响应体解析到`HostSearch`。

对于要与之交互的每个 API 服务，重复定义相应的结构和实现函数。这里就不浪费纸张了，我们就跳过直接到最后一步：创建使用这些代码的客户端。

### 创建客户端

使用简单的方法创建客户端：将搜索词作为命令行参数，然后调用 APIInfo（） 和 HostSearch（） 方法，如代码 3-11 所示。
```go
func main() {
    if len(os.Args) != 2 {
        log.Fatalln("Usage: shodan searchterm") 
    }
    apiKey := os.Getenv("SHODAN_API_KEY")
    s := shodan.New(apiKey)
    info, err := s.APIInfo()w
    if err != nil {
        log.Panicln(err) 
    }
    fmt.Printf(
        "Query Credits: %d\nScan Credits: %d\n\n", 
        info.QueryCredits,
        info.ScanCredits)
    hostSearch, err := s.HostSearch(os.Args[1]) 
    if err != nil {
        log.Panicln(err) 
    }
    for _, host := range hostSearch.Matches { 
        fmt.Printf("%18s%8d\n", host.IPString, host.Port)
    } 
}
```
代码 3-11: 使用 `shodan` 包 (https://github.com/blackhat-go/bhg/ch-3/shodan/cmd/shodan/main.go/)

先从环境变量`SHODAN_API_KEY`中获得API 的key。然后用该值实例化 `Client` 结构体，命名为s，接下来调用 `APIInfo() `方法。调用 `HostSearch()`，使用命令行参数传递的搜索字符串。最后，遍历结果显示与查询字符串匹配的那些服务的IP和端口值。以下输出是搜索字符串tomcat的结果：
```shell script
$ SHODAN_API_KEY=YOUR-KEY go run main.go tomcat 
Query Credits:  100
Scan Credits:   100
  185.23.138.141  8081 
  218.103.124.239 8080 
  123.59.14.169   8081 
  177.6.80.213    8181 
  142.165.84.160  10000
--snip--
```

要向该项目中添加错误处理和数据验证，但这是使用新API提取和显示Shodan数据的一个很好的例子。现在
有一个可以易扩展的代码库，方便地添加其他Shodan功能并测试。

## 和Metasploit交互
Metasploit是用于执行各种黑客技术的框架，包括侦察，开发，指挥和控制，持久性，横向网络移动，有效负载的创建和交付，权限升级等待。更好的是，该产品的社区版本是免费的，可在Linux和macOS上运行，维护的也很活跃。是黑客的必备，Metasploit是渗透测试人员使用的基本工具，它公开了远程过程调用（RPC）API以允许与其功能进行远程交互。

在本节中，创建和远程Metasploit交互的客户端。非常像上节的Shodan代码。Metasploit客户端也不会实现所有的函数。当然，根据需要也可以在此基础上扩展。这将比Shodan的例子复杂点，也非常有挑战性。

### 环境搭建
本节开始之前，先下载和安装Metasploit编辑器。通过Metasploit中的`msgrpc`模块启动Metasploit控制台以及RPC侦听器。然后设置服务器地址——RPC服务监听的IP——和密码，如代码3-12：
```shell script
$ msfconsole
msf > load msgrpc Pass=s3cr3t ServerHost=10.0.1.6 [*] MSGRPC Service: 10.0.1.6:55552
[*] MSGRPC Username: msf
[*] MSGRPC Password: s3cr3t
[*] Successfully loaded plugin: msgrpc
```
代码 3-12: 启动 Metasploit 和  msgrpc 服务

为RPC实例设置以下环境变量，使代码更具可移植性且避免硬编码。这与在第58页“创建客户端”中用于与Shodan进行交互的Shodan API密钥的操作类似。
```shell script
$ export MSFHOST=10.0.1.6:55552 
$ export MSFPASS=s3cr3t
```
现在Metasploit 和 RPC 服务运行起来了。

鉴于开发和Metasploit使用的细节超出了本书的范围，假设通过欺骗已经危害了远程Windows系统，并利用Metasploit的Meterpreter进行高级的开发活动。在这里，专注于如何和Metasploit远程通信列出已建立的Meterpreter会话。如我们之前提到的，代码会有点复杂，因此我们只留下最核心的代码——足够以后按需去扩展了。

和Shodan例子的流程一样：查看Metasploit的API，将项目设计成库，定义数据类型，实现客户端API函数，最后用该库测试。

首先，在Rapid7官方网站(https://metasploit.help.rapid7.com/docs/rpc-api/)上查看Metasploit的API的开发文档。公开的功能很多，通过本地交互远程执行任何操作。不像Shodan使用JSON，Metasploit使用压缩高效的二进制格式的MessagePack通信。因为Go中没有标准的MessagePack 包，因此使用功能齐全的公开版本。通过执行下面的命令安装：
```shell script
$ go get gopkg.in/vmihailenco/msgpack.v2
```

代码中将要实现的称为`msgpack`。不用担心太多的MessagePack细节。很快就会知道构建客户端只需要很少的内容。Go杰出的原因之一是隐藏了很多细节，让开发者专注于业务逻辑。需要了解的是定义合适的类型正确解析MessagePack。此外，用另外的格式初始化编码和解码的代码，相 JSON 和 XML。

接下来，创建目录结构。本例只有两个Go文件：
```shell script
$ tree github.com/blackhat-go/bhg/ch-3/metasploit-minimal 
github.com/blackhat-go/bhg/ch-3/metasploit-minimal 
|---client
| |---main.go
|---rpc |---msf.go
```

`msf.go`文件是rpc包，`client/main.go`执行测试创建的库。

### 定义对象
现在需要定义用的对象了。为了简洁起见，实现调用RPC代码获取当前的Meterpreter回话——即Meterpreter开发文档中的`session.list`文件。请求格式定义如下：
```json
[ "session.list", "token" ]
```
最少的内容；期望接收要实现的方法的名称和`token`。`token`在这里占位用。如果通读过文档的话，这是成功登录到 RPC 服务器后生成的认证token。Metasploit 返回的`session.list`格式如下：
```
{
"1" => {
    'type' => "shell",
    "tunnel_local" => "192.168.35.149:44444", "tunnel_peer" => "192.168.35.149:43886", "via_exploit" => "exploit/multi/handler", "via_payload" => "payload/windows/shell_reverse_tcp", "desc" => "Command shell",
    "info" => "",
    "workspace" => "Project1",
    "target_host" => "",
    "username" => "root",
    "uuid" => "hjahs9kw",
    "exploit_uuid" => "gcprpj2a",
    "routes" => [ ]
    }
}
```

返回map类型的响应，key为Meterpreter的session标识符，value是session的细节。

在Go中创建相应的请求和响应数据类型。代码 3-13 定义了`sessionListReq`和 `SessionListRes`.
```go
type sessionListReq struct {
	_msgpack struct{} `msgpack:",asArray"`
	Method   string
	Token    string
}

type SessionListRes struct {
	ID          uint32 `msgpack:",omitempty"`
	Type        string `msgpack:"type"`
	TunnelLocal string `msgpack:"tunnel_local"`
	TunnelPeer  string `msgpack:"tunnel_peer"`
	ViaExploit  string `msgpack:"via_exploit"`
	ViaPayload  string `msgpack:"via_payload"`
	Description string `msgpack:"desc"`
	Info        string `msgpack:"info"`
	Workspace   string `msgpack:"workspace"`
	SessionHost string `msgpack:"session_host"`
	SessionPort int    `msgpack:"session_port"`
	Username    string `msgpack:"username"`
	UUID        string `msgpack:"uuid"`
	ExploitUUID string `msgpack:"exploit_uuid"`
}
```
代码 3-13: 定义Metasploit的session请求和响应 (https://github.com/blackhat-go/bhg/ch-3/metasploit-minimal/rpc/msf.go/)

使用`sessionListReq`将结构化数据序列化成和 Metasploit RPC 服务器一致的 MessagePack 格式——特别需要方法名字和token。注意，这里的字段没有任何描述符。这些数据通过数组而不是map传递，因此不是`key/value`格式的数据，RPC也需要的是数组中的值。这也就是为什么省略了这些属性的注释——不需要定义key的名字。然而，结构体编码时会默认使用属性的名字作为key编码成map。要避之并强制编码成数组，添加名为`_msgpack`的特殊字段并使用`asArray`描述符，明确地指示将数据编解码为数组。

`SessionListRes`类型将结构体属性和响应体字段一对一对应。像上例中响应体，数据本质上是嵌套的map。外层map是session详情的标识，内层map是session详情键/值对。不像请求体那样，响应体并不是构建成数组的结构，而是结构体的每个属性都使用描述符显式命名，并和 Metasploit的数据映射。代码将session标识符作为结构体的属性。然而，标识符实际的数据是键值，这将以稍微不同的方式填充，因此使用`omitempty`描述符，以使数据具有可选性，以便不影响编码或解码。这种平级的数据就没必要使用嵌套map了。

### 获取有效Token
现在还有一件事情未解决。那就是获取发送请求需要的`token`。为此，需要发送登录请求到`auth.login()`这个API，如下所示：
```shell script
["auth.login", "username", "password"]
```
使用初始化时在Metasploit加载`msfrpc`模块时的用户名和密码替换`username`和`password`（也就是将其加入到环境变量中了）。假如认证成功的话，服务器响应如下，其含有为接下来请求所用的认证token。
```shell script
{ "result" => "success", "token" => "a1a1a1a1a1a1a1a1" }
```
认证失败的响应如下：
```shell script
{
    "error" => true,
    "error_class" => "Msf::RPC::Exception",
    "error_message" => "Invalid User ID or Password" 
}
```
另外，我们还创建通过注销来使token过期的功能。该请求需要方法名，认证的token，第三个为可选参数，因为此处第三个参数不是必需的，就先忽略了：
```shell script
[ "auth.logout", "token", "logoutToken"]
```
成功的响应像下面这样：
```shell script
{ "result" => "success" }
```

### 定义请求和响应方法

正如为`session.list()`方法的请求和响应构造的Go类型一样，也需要对`auth.login()` 和 `auth.logout()`进行相同的操作（见代码 3-14）。和之前一样，使用描述符强制将请求序列化为数组，并将响应处理为map。
```go
type loginReq struct {
	_msgpack struct{} `msgpack:",asArray"`
	Method   string
	Username string
	Password string
}

type loginRes struct {
	Result       string `msgpack:"result"`
	Token        string `msgpack:"token"`
	Error        bool   `msgpack:"error"`
	ErrorClass   string `msgpack:"error_class"`
	ErrorMessage string `msgpack:"error_message"`
}

type logoutReq struct {
	_msgpack    struct{} `msgpack:",asArray"`
	Method      string
	Token       string
	LogoutToken string
}

type logoutRes struct {
	Result string `msgpack:"result"`
}
```
代码 3-14: 定义登录和登出 Metasploit 的类型 (https://github.com/blackhat-go/bhg/ch-3/metasploit-minimal/rpc/msf.go/)

值得注意的是，Go动态地序列化登录响应，只填充当前的字段，这意味着可以使用同一个结构格式来表示登录成功和失败。