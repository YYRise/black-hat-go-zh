# 第3章：HTTP客户端和使用工具远程交互

## 摘要

在第2章中，您学习了如何用TCP创建可用的客户端和服务器。本章是探讨 OSI 模型较高层上的各种协议的第一章。 基于网络中的广泛性，宽松的出口控制联系，一般的灵活性，就先从HTTP开始。

本章节专注于客户端。首先介绍构建和自定义HTTP请求并接收其响应的基础知识。然后，学习如何解析结构化的响应数据，以便客户可以访问数据以确定可行或相关数据。最后，通过构建与各种安全工具和资源进行交互的HTTP 客户端来学习如何应用这些基础知识。开发的客户端将查询和使用 Shodan，Bing 和 Metasploit 的 API ，并以类似于元数据搜索工具FOCA的方式搜索和解析文档元数据

## Go中HTTP基础

尽管没必要全面了解HTTP，但是开始时最好知道一些。

首先，HTTP是 _无状态协议_：服务器天生不会维护每个请求的状态。而是用多种方式跟踪状态，这些方式可能包括会话session标记，cookie，HTTP标头等。服务器和客户端需正确协商和验证此状态。

其次，客户端和服务器之间的通信可以是同步或异步的，但要以请求/响应周期运行。可以在请求中添加几个选项和header，以决定服务器的行为并创建可用的Web程序。最常见的是，服务器持有Web 浏览器渲染的文件，以生成数据的图形化，组织化和风格化的表现形式。但是服务器可以返回任意的数据类型。API通常通过结构化的数据编码（例如XML，JSON或MSGRPC）进行通信。某些情况下，数据可能是二进制格式，表示下载的文件是任意类型。

最后，使用Go提供的函数，可以快速轻松地构建HTTP请求并将其发送到服务器，然后获取并处理响应。通过前面几章中学到的一些技巧，使用结构化数据的约定非常方便地处理与HTTP API交互。

### 调用HTTP API

我们通过查看基本的请求开始HTTP的学习。Go的`net/http`包中有几个方便的函数能快速轻松地发送POST，GET和HEAD请求，这也是最常用的HTTP动词。这些函数的形式如下：

```go
Get(url string) (resp *Response, err error)
Head(url string) (resp *Response, err error)
Post(url string, bodyType string, body io.Reader) (resp *Response, err error)
```

每个函数都有一个参数——URL字符串，请求的目的地。`Post()`函数比`Get()`和`Head()`这两个函数稍微复杂点。`Post()`有两个额外的参数：`bodyType`，用于请求正文的Content-Type HTTP 标头（通常是application/x-www-form-urlencoded）的字符串，`io.Reader`，在第2章学过。在清单3-1中有每个函数的示例实现。请注意，POST请求从表单创建请求主体并设置Content-Type 标头。在每种情况下，都必须在从响应体读完数据后关闭它。

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

 代码 3-1: Get\(\), Head\(\), 和 Post\(\) 函数的示例 \(https://github.com/blackhat-go/bhg /ch-3/basic/main.go/\)

POST函数使用相当普通的调用模式，即URL编码表单数据时，设置Content-Type为application / x-www-form-urlencoded。

Go中有另外的POST请求简便函数叫做`PostForm()`，移除了设置这些值和手动编码每个请求的繁琐工作；语法如下：

```go
func PostForm(url string, data url.Values) (resp *Response, err error)
```

如果想使用`PostForm()`函数替代代码3-1中的`Post()`，使用像代码3-2中的代码那样。

```go
form := url.Values{}
form.Add("foo", "bar")
r3, err := http.PostForm("https://www.google.com/robots.txt", form) // Read response body and close.
```

代码 3-2: 使用`PostForm()`函数替代`Post()` \([https://github.com/blackhat](https://github.com/blackhat) -go/bhg/ch-3/basic/main.go/\)

不幸的是，没有其他的HTTP方法（例如PATCH，PUT或DELETE）没有这种简便函数。主要使用这些方法与RESTful API进行交互，RESTful API 提供了服务器使用这些方法的方式和原因的一般指导；但没有什么是一成不变的，而在方法方面，HTTP就像古老的西方。实际上，我们经常想到创建一个专门使用DELETE进行所有操作的新网络框架的想法。我们称它为`DELETE.js`，毫无疑问，这将是Hacker News的热门话题。读到这，即表示您同意不窃取这个想法！

### 创建请求

使用 `NewRequest()`函数为这些HTTP方法创建请求。该函数创建`Request`结构体，接下来使用客户端函数`Do()`方法发送。其实要比听起来简单多了。`http.NewRequest()`函数原型如下：

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

代码 3-3: 发送 DELETE 请求 \([https://github.com/blackhat-go/bhg/ch-3/basic](https://github.com/blackhat-go/bhg/ch-3/basic) /main.go/\)

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

代码 3-4: 发送 PUT 请求 \([https://github.com/blackhat-go/bhg/ch-3/basic](https://github.com/blackhat-go/bhg/ch-3/basic) /main.go/\)

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

代码 3-5: 处理 HTTP 响应体 \([https://github.com/blackhat-go/bhg/ch-3/basic/main.go/](https://github.com/blackhat-go/bhg/ch-3/basic/main.go/)\)

收到响应后，在上面代码命名为 `resp`，通过访问暴露出的 `Status` 参数就能获取到状态字符串（例如，200 ok）；有一个类似的 `StatusCode` 参数，该参数仅访问状态字符串的整数部分，未在示例中展示。

响应中还暴露出一个`io.ReadCloser`类型的`Body`参数。`io.ReadCloser` 既是`io.Reader`又是 `io.Closer` 接口，或者是需要实现 `Close()` 函数的接口以关闭读取和执行清理。现在先不用考虑具体的细节；只是从`io.ReadCloser` 读取完数据后记得调用响应体`Close()`函数来关闭。通常使用`defer`关闭响应体；这能确保退出函数之前不会忘记关闭响应体。

现在回到终端查看错误状态和响应体内容：

```text
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

```javascript
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

代码 3-6: 解码 JSON 响应体 \([https://github.com/blackhat-go/bhg/ch-3/basic-parsing/main.go/](https://github.com/blackhat-go/bhg/ch-3/basic-parsing/main.go/)\)

代码先定义`Status`结构体，含义预期的服务器返回的元素。`main()`函数先发送`POST`请求，然后解析响应体。这之后就可以正常的使用`Status`结构——访问暴露出的`Statu`和`Message`数据类型

其他编码格式解析这种结构化数据也是一样的，像XML，甚至是二进制。先定义期望响应数据的结构体，然后将数据解析到该结构体。解析其他样式的细节和实现过程留给你自己去研究了。

下一节将应用这些基本概念来构建工具与第三方 API 进行交互，顺便增强黑客技术。

## 构建和Shodan交互的HTTP客户端

在对组织进行任何授权的对抗活动之前，优秀的黑客都先从侦察开始。典型地，不向目标发送数据包的被动技术开始；这样，几乎不可能检测到活动。黑客使用各种资源和服务——包括社交网络，公共记录，搜索引擎——来获取目标潜在的有用的信息。

令人难以置信的是，将环境上下文应用在连锁攻击场景中时，看似良性的信息如何变得至关重要。举例，仅将披露详细错误消息的Web应用程序视为低严重性。但是，如果错误消息公开了企业用户名格式，并且组织对VPN使用了单向身份验证，则这些错误消息可能会增加通过猜测密码攻击内部网络的可能性。在收集信息的同时保持低调，可确保目标的感知和安全态势保持中立，从而增加攻击成功的可能性。

Shodan \([https://www.shodan.io/\)，自称为“世界上第一个互联网连接设备搜索引擎，通过维护可搜索的网络设备和服务数据库来促进被动侦察，包括产品名称，版本，语言环境等元数据。将Shodan看作是扫描数据的存储库，即使它能做更多很多的事。](https://www.shodan.io/%29，自称为“世界上第一个互联网连接设备搜索引擎，通过维护可搜索的网络设备和服务数据库来促进被动侦察，包括产品名称，版本，语言环境等元数据。将Shodan看作是扫描数据的存储库，即使它能做更多很多的事。)

### 回顾构建API客户端的步骤

在接下来的几节构建和Shodan API交互的HTTP客户端，解析结果并展示相关信息。首先，需要Shodan API的key，在Shodan网站注册后就能获得。在撰写本文时，最低级别的费用非常合适，满足个人使用的需求，因此请注册吧。Shodan偶尔会有折扣，如果你想节省几块钱的话就要多关注点网站动态。

现在，从网站获得API key后设置为环境变量。接下来的例子使用的保存APT key的环境变量`SHODAN_API_KEY`。请参阅操作系统的用户手册，或者如果需要帮助设置变量，请参考第一章。

在研究代码之前，请了解本节演示的如何创建客户端的基本实现，而不是全部功能实现。不过，现在将要构建的基础的脚手架将使您可以轻松扩展演示的代码，以实现调用可能需要的其他API。

构建的客户端实现了两个API调用：一个查询订阅信用信息，另一个搜索含义某字符的hosts。使用后一个调用来标识主机。例如，与特定产品匹配的端口或操作系统。

幸运的是，Shodan API简单易懂，使用的易于理解的JSON格式的响应。这使其成为学习API交互的良好起点。以下是准备和构建API客户端的典型步骤的整体概述： 1. 查阅服务端API文档 2. 设计合理的代码结构以减少复杂性和重复性 3. 用GO按需定义请求和响应的类型 4. 创建辅助函数和类型，以简化简单的初始化，身份验证和通信，以减少冗余或重复的逻辑 5. 构建与API的函数和类型交互的客户端

我们在本节中没有明确指出每个步骤，但是您应该使用此列表作为指导来学习。开始快速地查阅Shodan网站的API文档。该文档很少，但是提供了创建客户端程序所需的一切。

### 设计项目结构

构建API客户端时，应对其进行结构设计，以使函数调用和逻辑独立。这可以在其他项目中作为单独的库来复用。这样就不会重复造轮子了。可复用性的架构会稍微改变项目的结构。以Shodan为例，项目的结构为：

```text
$ tree github.com/blackhat-go/bhg/ch-3/shodan github.com/blackhat-go/bhg/ch-3/shodan |---cmd
| |---shodan
| |---main.go |---shodan
|---api.go |---host.go |---shodan.go
```

`main.go`文件定义包`main`，主要用作构建的API的使用者；这样的话，主要使用该文件于客户端交互。

在`shodan`文件夹中的——`api.go, host.go, 和 shodan.go`——定义了包`shodan`，其包含了和`Shodan`通信所需要的类型和函数。这个包会是一个单独的库，可以导入的其他项目中使用。

### 清理API调用

当精读Shodan API文档时，您可能会注意到每个暴露的函数都需要发送您的API key。尽管可以肯定地将该值传递给所创建的函数，但该重复性的工作是乏味。类似的还有硬编码和操作URL \([https://api.shodan.io/\)。举例来说，如下所示定义了一个API函数，需要使用token和URL参数，这就有点不简洁：](https://api.shodan.io/%29。举例来说，如下所示定义了一个API函数，需要使用token和URL参数，这就有点不简洁：)

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

代码 3-7: 定义Shodan客户端 \([https://github.com/blackhat-go/bhg/ch-3/shodan.go/](https://github.com/blackhat-go/bhg/ch-3/shodan.go/)\)

Shodan URL被定义为常量；这样，在函数中易于访问和复用。如果Shodan更改API的URL，只需要改动这一个地方就能改动整个代码库。接下来定义了`Client`结构体，保存请求API时使用的token。最后，定义了`New()`辅助函数，以API的token为参数，创建并返回已初始化了的Client实例。现在，无需将API代码创建为任意函数，而是将它们创建为Client的方法，这使就可以直接通过Client实例查询，而不必依赖过于冗长的函数参数。将稍后要讨论的API函数改为以下内容：

```go
func (s *Client) APIInfo() { --snip-- } 
func (s *Client) HostSearch() { --snip-- }
```

因为这都是Client的方法，可以直接通过`s.apiKey`获得API key，通过`BaseURL`得到URL。调用这些方法的唯一前提条件是首先创建Client实例。通过`shodan.go`文件中`New()`辅助函数就可以了。

### 查询Shodan订阅

现在可以开始和Shodan交互了。根据Shodan API文档，用于查询订阅计划信息的调用如下：

```markup
https://api.shodan.io/api-info?key={YOUR_API_KEY}
```

返回的响应类似于下面的结构。显然，这基于计划详情和剩余的订阅支付有所不同。

```javascript
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

代码 3-8: 发送 HTTP GET 请求并解析响应 \([https://github.com/blackhat-go/bhg/ch-3/shodan/shodan/api.go/](https://github.com/blackhat-go/bhg/ch-3/shodan/shodan/api.go/)\)

代码简短而精悍。先向/api-info发出HTTP GET请求。该URL由全局常量`BaseURL`和`s.apiKey`创建。然后将响应解析到`APIInfo`结构体中，最后返回给调用者。

在编写使用这种新颖逻辑的代码之前，请构建第二个更有用的API调用——主机搜索——将其添加到`host.go`中。根据API文档的描述，请求和响应如下：

```javascript
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

代码 3-9: Host 搜索响应体数据类型 \([https://github.com/blackhat-go/bhg/ch-3/shodan/shodan/host.go/](https://github.com/blackhat-go/bhg/ch-3/shodan/shodan/host.go/)\)

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

代码 3-10: 解析 host 搜索的响应头 \([https://github.com/blackhat-go/bhg/ch-3/shodan/shodan/host.go/](https://github.com/blackhat-go/bhg/ch-3/shodan/shodan/host.go/)\)

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

代码 3-11: 使用 `shodan` 包 \([https://github.com/blackhat-go/bhg/ch-3/shodan/cmd/shodan/main.go/](https://github.com/blackhat-go/bhg/ch-3/shodan/cmd/shodan/main.go/)\)

先从环境变量`SHODAN_API_KEY`中获得API 的key。然后用该值实例化 `Client` 结构体，命名为s，接下来调用 `APIInfo()`方法。调用 `HostSearch()`，使用命令行参数传递的搜索字符串。最后，遍历结果显示与查询字符串匹配的那些服务的IP和端口值。以下输出是搜索字符串tomcat的结果：

```text
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

要向该项目中添加错误处理和数据验证，但这是使用新API提取和显示Shodan数据的一个很好的例子。现在 有一个可以易扩展的代码库，方便地添加其他Shodan功能并测试。

## 和Metasploit交互

Metasploit是用于执行各种黑客技术的框架，包括侦察，开发，指挥和控制，持久性，横向网络移动，有效负载的创建和交付，权限升级等待。更好的是，该产品的社区版本是免费的，可在Linux和macOS上运行，维护的也很活跃。是黑客的必备，Metasploit是渗透测试人员使用的基本工具，它公开了远程过程调用（RPC）API以允许与其功能进行远程交互。

在本节中，创建和远程Metasploit交互的客户端。非常像上节的Shodan代码。Metasploit客户端也不会实现所有的函数。当然，根据需要也可以在此基础上扩展。这将比Shodan的例子复杂点，也非常有挑战性。

### 环境搭建

本节开始之前，先下载和安装Metasploit编辑器。通过Metasploit中的`msgrpc`模块启动Metasploit控制台以及RPC侦听器。然后设置服务器地址——RPC服务监听的IP——和密码，如代码3-12：

```text
$ msfconsole
msf > load msgrpc Pass=s3cr3t ServerHost=10.0.1.6 [*] MSGRPC Service: 10.0.1.6:55552
[*] MSGRPC Username: msf
[*] MSGRPC Password: s3cr3t
[*] Successfully loaded plugin: msgrpc
```

代码 3-12: 启动 Metasploit 和 msgrpc 服务

为RPC实例设置以下环境变量，使代码更具可移植性且避免硬编码。这与在第58页“创建客户端”中用于与Shodan进行交互的Shodan API密钥的操作类似。

```text
$ export MSFHOST=10.0.1.6:55552 
$ export MSFPASS=s3cr3t
```

现在Metasploit 和 RPC 服务运行起来了。

鉴于开发和Metasploit使用的细节超出了本书的范围，假设通过欺骗已经危害了远程Windows系统，并利用Metasploit的Meterpreter进行高级的开发活动。在这里，专注于如何和Metasploit远程通信列出已建立的Meterpreter会话。如我们之前提到的，代码会有点复杂，因此我们只留下最核心的代码——足够以后按需去扩展了。

和Shodan例子的流程一样：查看Metasploit的API，将项目设计成库，定义数据类型，实现客户端API函数，最后用该库测试。

首先，在Rapid7官方网站\([https://metasploit.help.rapid7.com/docs/rpc-api/\)上查看Metasploit的API的开发文档。公开的功能很多，通过本地交互远程执行任何操作。不像Shodan使用JSON，Metasploit使用压缩高效的二进制格式的MessagePack通信。因为Go中没有标准的MessagePack](https://metasploit.help.rapid7.com/docs/rpc-api/%29上查看Metasploit的API的开发文档。公开的功能很多，通过本地交互远程执行任何操作。不像Shodan使用JSON，Metasploit使用压缩高效的二进制格式的MessagePack通信。因为Go中没有标准的MessagePack) 包，因此使用功能齐全的公开版本。通过执行下面的命令安装：

```text
$ go get gopkg.in/vmihailenco/msgpack.v2
```

代码中将要实现的称为`msgpack`。不用担心太多的MessagePack细节。很快就会知道构建客户端只需要很少的内容。Go杰出的原因之一是隐藏了很多细节，让开发者专注于业务逻辑。需要了解的是定义合适的类型正确解析MessagePack。此外，用另外的格式初始化编码和解码的代码，相 JSON 和 XML。

接下来，创建目录结构。本例只有两个Go文件：

```text
$ tree github.com/blackhat-go/bhg/ch-3/metasploit-minimal 
github.com/blackhat-go/bhg/ch-3/metasploit-minimal 
|---client
| |---main.go
|---rpc |---msf.go
```

`msf.go`文件是rpc包，`client/main.go`执行测试创建的库。

### 定义对象

现在需要定义用的对象了。为了简洁起见，实现调用RPC代码获取当前的Meterpreter回话——即Meterpreter开发文档中的`session.list`文件。请求格式定义如下：

```javascript
[ "session.list", "token" ]
```

最少的内容；期望接收要实现的方法的名称和`token`。`token`在这里占位用。如果通读过文档的话，这是成功登录到 RPC 服务器后生成的认证token。Metasploit 返回的`session.list`格式如下：

```text
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

代码 3-13: 定义Metasploit的session请求和响应 \([https://github.com/blackhat-go/bhg/ch-3/metasploit-minimal/rpc/msf.go/](https://github.com/blackhat-go/bhg/ch-3/metasploit-minimal/rpc/msf.go/)\)

使用`sessionListReq`将结构化数据序列化成和 Metasploit RPC 服务器一致的 MessagePack 格式——特别需要方法名字和token。注意，这里的字段没有任何描述符。这些数据通过数组而不是map传递，因此不是`key/value`格式的数据，RPC也需要的是数组中的值。这也就是为什么省略了这些属性的注释——不需要定义key的名字。然而，结构体编码时会默认使用属性的名字作为key编码成map。要避之并强制编码成数组，添加名为`_msgpack`的特殊字段并使用`asArray`描述符，明确地指示将数据编解码为数组。

`SessionListRes`类型将结构体属性和响应体字段一对一对应。像上例中响应体，数据本质上是嵌套的map。外层map是session详情的标识，内层map是session详情键/值对。不像请求体那样，响应体并不是构建成数组的结构，而是结构体的每个属性都使用描述符显式命名，并和 Metasploit的数据映射。代码将session标识符作为结构体的属性。然而，标识符实际的数据是键值，这将以稍微不同的方式填充，因此使用`omitempty`描述符，以使数据具有可选性，以便不影响编码或解码。这种平级的数据就没必要使用嵌套map了。

### 获取有效Token

现在还有一件事情未解决。那就是获取发送请求需要的`token`。为此，需要发送登录请求到`auth.login()`这个API，如下所示：

```text
["auth.login", "username", "password"]
```

使用初始化时在Metasploit加载`msfrpc`模块时的用户名和密码替换`username`和`password`（也就是将其加入到环境变量中了）。假如认证成功的话，服务器响应如下，其含有为接下来请求所用的认证token。

```text
{ "result" => "success", "token" => "a1a1a1a1a1a1a1a1" }
```

认证失败的响应如下：

```text
{
    "error" => true,
    "error_class" => "Msf::RPC::Exception",
    "error_message" => "Invalid User ID or Password" 
}
```

另外，我们还创建通过注销来使token过期的功能。该请求需要方法名，认证的token，第三个为可选参数，因为此处第三个参数不是必需的，就先忽略了：

```text
[ "auth.logout", "token", "logoutToken"]
```

成功的响应像下面这样：

```text
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

代码 3-14: 定义登录和登出 Metasploit 的类型 \([https://github.com/blackhat-go/bhg/ch-3/metasploit-minimal/rpc/msf.go/](https://github.com/blackhat-go/bhg/ch-3/metasploit-minimal/rpc/msf.go/)\)

值得注意的是，Go动态地序列化登录响应，只填充当前的字段，这意味着可以使用同一个结构格式来表示登录成功和失败。

### 创建配置结构体和RPC方法

代码3-15中，定义了相关类型并使用，创建必要的方法向Metasploit发送RPC命令。和Shodan的例子一样，可以定义能保存相关配置和认证信息的任意类型。这样，就不必直接反复传递`host`, `port`和认证`token`这些通用的数据。相反，定义成类型和该类型的方法使数据不公开使用。

```go
type Metasploit struct {
    host  string
    user  string
    pass  string
    token string
}
func New(host, user, pass string) (*Metasploit, error) {
    msf := &Metasploit{
        host: host,
        user: user,
        pass: pass,
    }

    if err := msf.Login(); err != nil {
        return nil, err
    }

    return msf, nil
}
```

代码 3-15: 定义Metasploit 客户端 \([https://github.com/blackhat-go/bhg/ch-3/metasploit-minimal/rpc/msf.go/](https://github.com/blackhat-go/bhg/ch-3/metasploit-minimal/rpc/msf.go/)\)

现在有了结构体，为了方便起见，还有一个初始化并返回新实例的New\(\)的函数。

### 执行远程调用

给`Metasploit`添加方法来执行远程调用。代码3-16中，为减少重复代码添加序列化，反序列化，HTTP通信逻辑的相关方法。这样就不必在每个RPC函数中添加重复的逻辑代码了。

```go
func (msf *Metasploit) send(req interface{}, res interface{}) error {
    buf := new(bytes.Buffer)
    msgpack.NewEncoder(buf).Encode(req)
    dest := fmt.Sprintf("http://%s/api", msf.host)
    r, err := http.Post(dest, "binary/message-pack", buf)
    if err != nil {
        return err
    }
    defer r.Body.Close()

    if err := msgpack.NewDecoder(r.Body).Decode(&res); err != nil {
        return err
    }

    return nil
}
```

代码 3-16: 复用序列化和反序列化的普通`send()`方法 \([https://github.com/blackhat-go/bhg/ch-3/metasploit-minimal/rpc/msf.go/](https://github.com/blackhat-go/bhg/ch-3/metasploit-minimal/rpc/msf.go/)\)

`send()`方法接收`interface{}`类型的请求和响应参数。使用`interface{}`可以将任意的请求结构体传递给函数，接下来序列化并发送给服务器。使用`res interface{}`读取解码后的HTTP响应体数据存储在内存中，而不是直接返回该响应体。

下一步使用`msgpack`包编码请求。该逻辑与其他标准的结构化数据类型相似:先使用`NewEncoder()`创建编码器，然后调用`Encode()`方法。这样请求结构体就会被编码到`buf`变量中。编码后，使用`Metasploit`接收器`msf`中的数据构建URL，显示地设置内容类型为`binary/message-pack`，和设置包体为序列化后的数据。最后，解码响应体。如前所述，解码后的数据被写入到传入到方法中的响应接口的内存位置。这种灵活，可复用的方法不需要明确地知道请求和响应结构体类型就能编码和解码。

代码 3-17，是逻辑的精髓所在。

```go
func (msf *Metasploit) Login() error {
    ctx := &loginReq{
        Method:   "auth.login",
        Username: msf.user,
        Password: msf.pass,
    }
    var res loginRes
    if err := msf.send(ctx, &res); err != nil {
        return err
    }
    msf.token = res.Token
    return nil
}

func (msf *Metasploit) Logout() error {
    ctx := &logoutReq{
        Method:      "auth.logout",
        Token:       msf.token,
        LogoutToken: msf.token,
    }
    var res logoutRes
    if err := msf.send(ctx, &res); err != nil {
        return err
    }
    msf.token = ""
    return nil
}

func (msf *Metasploit) SessionList() (map[uint32]SessionListRes, error) {
    req := &sessionListReq{Method: "session.list", Token: msf.token}
    res := make(map[uint32]SessionListRes)
    if err := msf.send(req, &res); err != nil {
        return nil, err
    }

    for id, session := range res {
        session.ID = id
        res[id] = session
    }
    return res, nil
}
```

代码 3-17: Metasploit API 调用实现 \([https://github.com/blackhat-go/bhg/ch-3/metasploit-minimal/rpc/msf.go/](https://github.com/blackhat-go/bhg/ch-3/metasploit-minimal/rpc/msf.go/)\)

定义了三个方法`Login()` , `Logout()`, 和 `SessionList()`。 每个方法使用相同的通用流程：创建并初始化请求结构体，创建响应结构体，调用辅助函数发送请求并接收解码后的响应体。`Login()`和`Logout()`方法操作`token`属性。方法中逻辑上唯一显著的差异在SessionList\(\)方法中，该方法中定义了`map[uint32]SessionListRes`类型的响应`res`，然后再遍历`res`，赋值结构体的`ID`属性，而后保存在map中。

`session.list()`这个RPC函数需要有效的认证token，也就是在调用`SessionList()`前就要登录成功。代码 3-18使用`Metasploit`类型的的`msf`来访问token，此时还是无效的值——是个空字符串。由于这不是功能齐全的代码，只能在`SessionList`方法中显示调用`Login()`方法，但是对于实现另外的认证方法，必须检查存在有效的认证token，且显示调用`Login()`。这不是好的编码，因为花费大量的时间写重复的代码，这里只是作为引导。

作为引导已经实现了`New()`函数，因此要添加认证，新实现的函数如下（见代码3-18）。

```go
func New(host, user, pass string) (*Metasploit, error) {
    msf := &Metasploit{
        host: host,
        user: user,
        pass: pass,
    }

    if err := msf.Login(); err != nil {
        return nil, err
    }

    return msf, nil
}
```

代码 3-18: 带有登录Metasploit的初始化客户端\([https://github.com/blackhat-go/bhg/ch-3/metasploit-minimal/rpc/msf.go/](https://github.com/blackhat-go/bhg/ch-3/metasploit-minimal/rpc/msf.go/)\)

修补后的代码带有错误返回。用来提醒认证可能会失败。同样地添加了显示调用 `Login()`方法。只要使用`New()`实例化`Metasploit`对象，通过调用认证方法就能访问有效的认证token。

### 创建实用程序

在本例的最后，最后一项工作是使用实现的新库创建实用的程序。将3-19中的代码添加的`client/main.go`中并允许，看看奇迹发生了。

```go
package main

import (
    "fmt"
    "log"
    "os"

    "github.com/bhg/ch-3/metasploit-minimal/rpc"
)

func main() {
    host := os.Getenv("MSFHOST")
    pass := os.Getenv("MSFPASS")
    user := "msf"

    if host == "" || pass == "" {
        log.Fatalln("Missing required environment variable MSFHOST or MSFPASS")
    }

    msf, err := rpc.New(host, user, pass)
    if err != nil {
        log.Panicln(err)
    }
    defer msf.Logout()

    sessions, err := msf.SessionList()
    if err != nil {
        log.Panicln(err)
    }
    fmt.Println("Sessions:")
    for _, session := range sessions {
        fmt.Printf("%5d  %s\n", session.ID, session.Info)
    }
}
```

Listing 3-19: 使用 msfrpc 包 \([https://github.com/blackhat-go/bhg/ch-3/metasploit-minimal/client/main.go/](https://github.com/blackhat-go/bhg/ch-3/metasploit-minimal/client/main.go/)\)

首先，启动RPC和初始化Metasploit实例。记住，只是更新了这个函数来执行在初始化时认证。下一步，通过defer调用`Logout()`方法确保`main`退出时正确执行清理工作。然后调用`SessionList()`方法，再遍历响应体列出有效的Meterpreter session。

通常调用其他API需要很多代码，但幸运的是，现在工作量应该会大大减少，因为只需定义请求和响应类型并构建库方法来发出远程调用。下面是直接从我们的客户端工具生成的输出示例，显示了一个已建立的Meterpreter session:

```text
$ go run main.go 
Sessions:
  1 WIN-HOME\jsmith @ WIN-HOME
```

好了。已经成功的创建了库和客户端程序来与远程的Metasploit实例交互，并获取到了Meterpreter session。接下来，继续探索抓取搜索引擎的响应和解析文档元数据。

## 使用Bing抓取并解析文档元数据

正如我们在Shodan部分所强调的，在正确的上下文中查看相对无害的信息是至关重要的，这增加了成功攻击组织的可能性。像员工名字，电话，邮件和客户端软件版本这些信息通常是最受重视的，因为它们提供了具体或可操纵的信息，黑客可以直接利用或使用这些信息来设计更有效、更有针对性的攻击。FOCA推广的数据源之一就是文档元数据。

应用程序将任意信息保存在磁盘的文件中。有时候会含有地理坐标，应用版本，操作系统信息，用户名。更好的是，搜索引擎有高级查询过滤器，能检索系统内特定文件。本章剩余部分主要来构建`scrapes`工具——或者是官方称为`indexes`——Bing搜索的结果来检索目标组织的Microsoft Office文档，然后依次提取相关的元数据。

### 建立环境和规划

在深入讨论细节之前，我们将先陈述下目标。首先，只关注以`xlsx, docx, pptx`等为扩展名的Open XML文档。尽管的确含有合法的Office数据类型，但二进制格式使它们的复杂性呈指数级增长，增加代码复杂度，减少可读性。对PDF文件来说也是一样的。此外，开发的代码不会处理Bing分页，而是只解析搜索结果的开始页面。我们鼓励您将其构建到工作示例中，并探索Open XML之外的文件类型。

为什么只使用Bing搜索API来构建，而不是抓取HTML？因为已经学会了如何构建客户端来和结构化的API交互。有一些用于抓取HTML页面的用例，特别是在不存在API的情况下。顺便利用这个机会介绍一种提取数据的新方法，而不是重复已经学会的内容。将使用优秀的包，`goquery`，模仿`jQuery`的功能，jQuery是一个JavaScript库，直观的语法来遍历HTML文档并在其中选择数据。从安装`goquery`开始：

```text
$ go get github.com/PuerkitoBio/goquery
```

幸运的是，这是完成开发所需的惟一需要的软件。使用标准的Go包就能和Open XML 文件交互。这些文件尽管后缀都属于ZIP归档文件，提取后都含有XML文件。元数据存储在归档文件`docProps`目录中的两个文件中:

```text
$ unzip test.xlsx $ tree
--snip-- 
|---docProps
| |---app.xml
| |---core.xml 
--snip—
```

`core.xml`文件包含作者信息和修改细节。结构如下：

```markup
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<cp:coreProperties xmlns:cp="http://schemas.openxmlformats.org/package/2006/metadata /core-properties"
xmlns:dc="http://purl.org/dc/elements/1.1/" xmlns:dcterms="http://purl.org/dc/terms/" xmlns:dcmitype="http://purl.org/dc/dcmitype/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
<dc:creator>Dan Kottmann</dc:creator>u
<cp:lastModifiedBy>Dan Kottmann</cp:lastModifiedBy>v
<dcterms:created xsi:type="dcterms:W3CDTF">2016-12-06T18:24:42Z</dcterms:created> <dcterms:modified xsi:type="dcterms:W3CDTF">2016-12-06T18:25:32Z</dcterms:modified>
</cp:coreProperties>
```

`creator`和`lastModifiedBy`元素是主要关系的。这些字段含有员工或用户名，可用在社交工程或猜密码活动中。`app.xml`中有创建Open XML 文档的应用类型和版本的详细信息。结构如下：

```markup
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<Properties xmlns="http://schemas.openxmlformats.org/officeDocument/2006/extended-properties"
xmlns:vt="http://schemas.openxmlformats.org/officeDocument/2006/docPropsVTypes"> <Application>Microsoft Excel</Application>u
<DocSecurity>0</DocSecurity>
<ScaleCrop>false</ScaleCrop>
<HeadingPairs>
<vt:vector size="2" baseType="variant">
            <vt:variant>
                <vt:lpstr>Worksheets</vt:lpstr>
            </vt:variant>
            <vt:variant>
                <vt:i4>1</vt:i4>
            </vt:variant>
        </vt:vector>
    </HeadingPairs>
    <TitlesOfParts>
<vt:vector size="1" baseType="lpstr"> <vt:lpstr>Sheet1</vt:lpstr>
        </vt:vector>
    </TitlesOfParts>
<Company>ACME</Company>v <LinksUpToDate>false</LinksUpToDate> <SharedDoc>false</SharedDoc> <HyperlinksChanged>false</HyperlinksChanged> <AppVersion>15.0300</AppVersion>w
</Properties>
```

主要关注的元素很少：`Application`, `Company`, 和 `AppVersion`。版本本身和Office版本名称没有明显的联系，如Office 2013, Office 2016,等。但是在该字段和更可读、更常见的替代之间确实存在逻辑映射。编写代码来维护这个映射。

### 定义元数据包

代码3-20，在新包`metadata`中定义和这些XML数据集一致的Go类型，代码在`openxml.go`文件中，即要解析的每个 XML 文件的一种类型。然后添加数据映射和便利函数，用于确定与`AppVersion`对应的可识别的Office版本。

```go
type OfficeCoreProperty struct {
    XMLName        xml.Name `xml:"coreProperties"`
    Creator        string   `xml:"creator"`
    LastModifiedBy string   `xml:"lastModifiedBy"`
}

type OfficeAppProperty struct {
    XMLName     xml.Name `xml:"Properties"`
    Application string   `xml:"Application"`
    Company     string   `xml:"Company"`
    Version     string   `xml:"AppVersion"`
}

var OfficeVersions = map[string]string{
    "16": "2016",
    "15": "2013",
    "14": "2010",
    "12": "2007",
    "11": "2003",
}

func (a *OfficeAppProperty) GetMajorVersion() string {
    tokens := strings.Split(a.Version, ".")

    if len(tokens) < 2 {
        return "Unknown"
    }
    v, ok := OfficeVersions[tokens[0]]
    if !ok {
        return "Unknown"
    }
    return v
}
```

代码 3-20: 定义Open XML 类型和版本映射 \([https://github.com/blackhat-go/bhg/ch-3/bing-metadata/metadata/openxml.go/](https://github.com/blackhat-go/bhg/ch-3/bing-metadata/metadata/openxml.go/)\)

定义 `OfficeCoreProperty` 和 `OfficeAppProperty` 类型后，再定义`map`类型的`OfficeVersions`，用于维护主版本号和可识别发布年份间的关系。为使用该map，在 `OfficeAppProperty`类型上定义了`GetMajorVersion()`方法。该方法分割XML数据的`AppVersion`值来检索主版本号，然后使用该值在`OfficeVersions`检索出发布的年份。

### 将数据映射到结构体

现在已经构建了处理和检查关注的XML数据的逻辑和类型，是时候写代码来读取文件并将内容赋值给结构体了。为此，定义`NewProperties()`和`process()`函数，如代码 3-21。

```go
func process(f *zip.File, prop interface{}) error {
    rc, err := f.Open()
    if err != nil {
        return err
    }
    defer rc.Close()
    if err := xml.NewDecoder(rc).Decode(&prop); err != nil {
        return err
    }
    return nil
}

func NewProperties(r *zip.Reader) (*OfficeCoreProperty, *OfficeAppProperty, error) {
    var coreProps OfficeCoreProperty
    var appProps OfficeAppProperty

    for _, f := range r.File {
        switch f.Name {
        case "docProps/core.xml":
            if err := process(f, &coreProps); err != nil {
                return nil, nil, err
            }
        case "docProps/app.xml":
            if err := process(f, &appProps); err != nil {
                return nil, nil, err
            }
        default:
            continue
        }
    }
    return &coreProps, &appProps, nil
}
```

代码 3-21: 处理开放的XML归档文件和嵌入的XML文档 \([https://github.com/blackhat-go/bhg/ch-3/bing-metadata/metadata/openxml.go/](https://github.com/blackhat-go/bhg/ch-3/bing-metadata/metadata/openxml.go/)\)

`NewProperties()`函数接收`*zip.Render`，表示ZIP归档文档的`io.Reader`。使用`zip.Reader`实例，遍历归档中的所有文件，检查文件名。如果文件名匹配两个属性文件名中的一个，则调用`process()`函数，实参为文件和希望解析成的任意结构类型——`OfficeCoreProperty`或`OfficeAppProperty`。

`process()`接收两个参数：`*zip.File`和`interface{}`。和之前开发的Metasploit相似，这段代码也是用了通用的`interface{}`类型，这样能把文件内容赋值给任何任意的数据类型。这增加了代码重用，因为在`process()`函数中没有特定于类型的内容。函数中的代码读取文件内容，然后将XML数据解码到结构体中。

### 用Bing搜索和接收文件

现在已经有了打开、读取、解析和提取Office Open XML文档所需的所有代码，并且也了解用该文件能做什么。现在应该弄明白使用Bing如何搜索和检索文件。以下是你应该遵循的行动规划: 1. 向Bing提交一个搜索请求，使用适当的过滤器来检索目标结果。 2. 获取HTML响应，提取HREF \(link\)数据来获得文档的直接url。 3. 直接向每个文档的URL提交HTTP请求。 4. 解析响应体创建`zip.Reader`。 5. 将`zip.Reader`传递给已开发的提取元数据的代码。

以下各节按顺序讨论这些步骤。

业务的第一步是构建一个搜索查询模板。非常像Google，Bing包含高级查询参数用于在大量的变量中过滤搜索结果。大多数过滤器都是以`filter_type: value`格式提交的。无需解释所有的过滤类型，重点放在有助于实现目标的方面。下面列出了需要的三种类型。注意，也可以使用另外的过滤器，但是在撰写本文时，它们的行为有些不可预测。

* **site** 用于过滤特定域的结果
* **filetype** 用于根据资源文件类型筛选结果
* **instreamset** 用于过滤只包含某些文件扩展名的结果

从`nytimes.com`查询并保存为`docx`文件的示例查询如下:

```text
site:nytimes.com && filetype:docx && instreamset:(url title):docx
```

提交查询后，在浏览器中查看结果的URL。类似于图3-1。后面可能会出现其他参数，但是对于本例来说是无关紧要的，所以可以忽略它们。

现在知道了URL和参数格式，可以查看HTML响应体，但是首先需要确定Document Object Model\(DOM\)文档链接的位置。可以通过直接查看源代码，或者限制猜想，只使用浏览器的开发者工具。下图是所需要的HREF的完整HTML元素的路径。像图3-1那样，使用元素检查器快速地选择连接来查看其完整的路径。

![](https://github.com/YYRise/black-hat-go/raw/dev/ch-3/images/3-1.jpg) 图3-1：浏览器开发者工具显示完整的元素路径

有了该路径信息，就能使用`goquery`系统地提取和HTML路径匹配的数据元素。闲话少说！代码3-22集成在一起了：检索，抓取，解析，提取。将代码保存到`main.go`中。

```go
func handler(i int, s *goquery.Selection) {
    url, ok := s.Find("a").Attr("href")
    if !ok {
        return
    }

    fmt.Printf("%d: %s\n", i, url)
    res, err := http.Get(url)
    if err != nil {
        return
    }

    buf, err := ioutil.ReadAll(res.Body)
    if err != nil {
        return
    }
    defer res.Body.Close()

    r, err := zip.NewReader(bytes.NewReader(buf), int64(len(buf)))
    if err != nil {
        return
    }

    cp, ap, err := metadata.NewProperties(r)
    if err != nil {
        return
    }

    log.Printf(
        "%21s %s - %s %s\n",
        cp.Creator,
        cp.LastModifiedBy,
        ap.Application,
        ap.GetMajorVersion())
}

func main() {
    if len(os.Args) != 3 {
        log.Fatalln("Missing required argument. Usage: main.go <domain> <ext>")
    }
    domain := os.Args[1]
    filetype := os.Args[2]

    q := fmt.Sprintf(
        "site:%s && filetype:%s && instreamset:(url title):%s",
        domain,
        filetype,
        filetype)
    search := fmt.Sprintf("http://www.bing.com/search?q=%s", url.QueryEscape(q))
    doc, err := goquery.NewDocument(search)
    if err != nil {
        log.Panicln(err)
    }

    s := "html body div#b_content ol#b_results li.b_algo div.b_title h2"
    doc.Find(s).Each(handler)
}
```

代码 3-22: 抓取 Bing 结果并解析文档元数据 \([https://github.com/blackhat-go/bhg/ch-3/bing-metadata/client/main.go/](https://github.com/blackhat-go/bhg/ch-3/bing-metadata/client/main.go/)\)

创建了两个函数。第一个`handler()`，接受`goquery.Selection`实例（在本例中，使用锚HTML元素填充）然后查找并提取`href`属性。这个属性包含直接链接到使用Bing搜索返回的文档。然后代码使用该URL发送GET请求检索该文档。假设没有错误发生的话，就能读取响应体并创建`zip.Reader`。回想下之前在`metadata`包中创建的`NewProperties()`函数，参数为`zip.Reader`。现在有了合适的数据类型，传给该函数，其属性被文件中的数据填充，然后在屏幕上输出。

`main()`函数启动并控制整个流程；通过命令行参数传递域名和文件类型。然后使用输入的数据构建了带有合适过滤条件的Bing查询。过滤字符串被编码并用于构建完整的Bing搜索URL。使用`goquery.NewDocument()`函数发送搜索查询，该函数使用HTTP GET请求并返回`goquery`易读的HTML响应文档。该文档能够使用`goquery`检索。最后，使用HTML元素选择器字符串来查找和迭代匹配的HTML元素，该字符串由浏览器开发者工具标识。对于每个匹配的元素，调用`handler()`函数。

运行代码产生的输出示例类似如下：

```text
$ go run main.go nytimes.com docx
0: http://graphics8.nytimes.com/packages/pdf/2012NAIHSAnnualHIVReport041713.docx 2020/12/21 11:53:50 Jonathan V. Iralu Dan Frosch - Microsoft Macintosh Word 2010 
1: http://www.nytimes.com/packages/pdf/business/Announcement.docx
2020/12/21 11:53:51 agouser agouser - Microsoft Office Outlook 2007
2: http://www.nytimes.com/packages/pdf/business/DOCXIndictment.docx
2020/12/21 11:53:51 AGO Gonder, Nanci - Microsoft Office Word 2007 
3: http://www.nytimes.com/packages/pdf/business/BrownIndictment.docx
2020/12/21 11:53:51 AGO Gonder, Nanci - Microsoft Office Word 2007 
4: http://graphics8.nytimes.com/packages/pdf/health/Introduction.docx
2020/12/21 11:53:51 Oberg, Amanda M Karen Barrow - Microsoft Macintosh Word 2010
```

现在，可以针对特定的域名从所有Open XML文件中搜索并提取文档元数据。我鼓励您对这个示例进行扩展，能操纵多个Bing搜索结果的逻辑，处理除Open XML外的其他文件类型，提高代码能并发下载支持的文件。

## 总结

本章介绍了Go中的基本的HTTP概念，并用其创建和远程API交互的可用工具，及抓取任意的HTML数据。在下一章，通过创建服务器而非客户端来继续学习HTTP专题。

