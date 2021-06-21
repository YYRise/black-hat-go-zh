# 第14章：构建命令和控制的RAT

在本章，结合前几章的课程来构建一个基本的命令和控制 (C2) 的 `remote access Trojan (RAT)`。RAT病毒是攻击者用来在受害机器上远程执行操作的工具，例如访问文件系统、执行代码和嗅探网络流量。

构建这个 RAT 需要构建三个独立的工具：一个客户端植入、一个服务器和一个管理组件。客户端植入是RAT病毒在一个被破坏的工作站上运行的部分。服务器与植入的客户端交互，很像Cobalt Strike的团队服务器——广泛使用的C2工具的服务器组件——向受影响的系统发送命令。与使用单一服务来促进服务器和管理功能的团队服务器不同，我们将创建一个单独的、独立的管理组件，用于实际发出命令。该服务器将充当中间人，编排被破坏系统和与管理组件交互的攻击者之间的通信。

设计 RAT 的方法有无数种。在本章中，我们重点介绍如何处理远程访问的客户端和服务器通信。出于这个原因，我们先展示如何构建简单粗暴的东西，然后让您去做显著的改进让您改进过的版本更健壮。在多数情况下，这些改进需要用到前面章节中的内容和代码示例。你将运用你的知识、创造力和解决问题的能力来提高你的执行能力。



## 开始

首先，让我们回顾一下将要做的事情：创建一个服务器，它以操作系统命令的形式从管理组件接收工作(也将创建管理组件)。 创建一个植入，定期轮询服务器获取新命令，然后将命令输出发布回服务器。 然后服务器将该结果返回给管理客户机，以便操作者(您)可以看到输出。 

首先安装工具，用于帮助我们处理这些网络交互，并检查这个项目的目录结构。

### 安装Protocol Buffers 用于定义 gRPC API

用 `gRPC` 构建所有的网络交互，`gRPC` 是有 Google 开发的一款高性能的远程过程调用 （RPC）框架。 RPC框架允许客户端通过标准和定义的协议与服务器通信，而无需知晓任何底层细节。 gRPC框架在HTTP/2上运行，以一种高效的二进制结构通信消息。

非常像其他的RPC机制，如 REST 或 SOAP，数据需要被定义，以便能够容易地序列化和反序列化。 幸运的是，有一种机制可以定义数据和API函数，因此可以与gRPC一起使用。 这个机制就是Protocol Buffers（也缩写为Protobuf），将API的标准语法和复杂的数据定义包含在 `.proto`文件中。 现有工具可将定义文件编译为Go支持的接口存根和数据类型。实际上，该工具可以生成各种语言的输出，也就是可以使用 `.proto` 文件生成 C# 存根和类型。

首先在系统中安装Protobuf编译器。 安装过程超出了本书的范围，但您可以在官方的 Go Protobuf库 https://github.com/golang/protobuf/ 上的“Installation”部分找到详细完整的安装信息。 同时，使用下面的命令安装gRPC包：

```sh
> go get -u google.golang.org/grpc
```

### 创建工程工作台

接下来，创建工程工作台。 创建四个子目录来存放三个组件（植入、服务器和管理组件）和定义 gRPC API 的文件。 每个组件的目录中，创建单个Go文件（与所在目录同名），该文件属于自己的 `main` 包。 这样可以作为独立的组件单独地编译和运行，且在组件上运行 `go build` 命令时会生成和目录同名的二进制。 同时，在 `grpcapi` 目录中创建名为 `implant.proto` 的文件。 该文件保存 Protobuf 架构和 gRPC API 定义。 目录结构应该像下面一样：

```sh
$ tree
.
|-- client
| |-- client.go
|-- grpcapi
| |-- implant.proto
|-- implant
| |-- implant.go
|-- server
| |-- server.go
```

创建了结构之后，我们就可以开始构建我们的实现了。在接下来的几个小节中，将带您浏览每个文件的内容。


## 定义并构建 gRPC API

下一个任务是定义gRPC API将使用的功能和数据。 与构建和使用 REST 端点不同，后者具有相当明确的一组期望（例如，它们使用 HTTP 动词和 URL 路径来定义对哪些数据采取哪些操作），gRPC 更随意。 可以有效地定义一个API服务，并将该服务的函数原型和数据类型与之绑定。 使用 Protobufs 定义API。 用Google搜索可以很快地找到Protobufs语法的完整说明，但是这里我们做个简短介绍。

至少，我们需要定义一个管理服务，操作者使用它向服务器发送操作系统命令(工作)。 同时也需要一个植入服务，用于从服务器获取工作，并将命令输出发送回服务器。 清单 14-1 是 `implant.proto` 文件的内容。 （所有的代码清单都在 github 仓库 https://github.com/blackhat-go/bhg/ 跟目录下。）

```proto
//implant.proto
syntax = "proto3";
package grpcapi;

// Implant defines our C2 API functions
service Implant {
	rpc FetchCommand (Empty) returns (Command);
	rpc SendOutput (Command) returns (Empty);
}

// Admin defines our Admin API functions
service Admin {
	rpc RunCommand (Command) returns (Command);
}

// Command defines a with both input and output fields
message Command {
	string In = 1;
	string Out = 2;
}

// Empty defines an empty message used in place of null
message Empty {
}
```

清单 14-1：用 Protobuf 定义 gRPC API (/ch-14/grpcapi/implant.proto)

还记得如何将这个定义的文件编译为Go特定的工件？ 好吧，明确地包含`package grpcapi` 来告诉编译器，我们希望在 `grpcapi` 包下创建这些工作。 包的名字是随意的。 这里使用是为了确保API代码与其他组件保持分离。

然后定义了一个名为 `Implant` 的服务和一个名为 `Admin` 的服务。 将其分开是因为 `Implant` 组件以不同于 `Admin` 客户端的方式与API交互。 举例来说，我们不希望 `Implant` 向我们的服务发送操作系统命令，就像我们不希望我们的 `Admin` 组件将命令输出发送到服务器一样。 

在 `Implant` 服务中定义了两个方法：`FetchCommand 和 Send Output`。 定义这些方法就像在Go中定义 `interface` 一样。 也可以说任何 `Implant` 服务的实现都需要实现这两个方法。 `FetchCommand` 使用 `Empty` 消息作为参数，且返回 `Command` 消息，将从服务器检索任何未完成的操作系统命令。 `SendOutput` 将 `Command` 消息（包含命令输出）发送回服务器。 刚刚提到的这些消息是任意的、复杂的数据结构，其中包含我们在端点之间来回传递数据所必需的字段。

`Admin` 服务定义了一个方法：`RunCommand` ，使用 `Command` 消息作为参数，并期望读回一个`Command` 消息。 其目的是让您，即 RAT 操作者，可以在具有运行植入程序的远程系统上运行操作系统命令。 

最后，定义了要传递的两个消息：`Command` 和 `Empty`。 `Command`有两个字段， 一个用于保存操作系统命令本身（名为 `In` 的字符串），另一用于保存命令输出（名为 `Out` 的字符串）。 注意，消息和字段的名字都是随意的，但是给每个字段都赋值了一个数字值。 您可能想知道，如果我们将`In` 和 `Out` 们定义为字符串，为何他们赋值数值。 答案是这只是模式定义，并非是实现。 这些数字表示消息中的字段在消息中的偏移。 也就是， `In` 先出现，`Out` 后出现。 `Empty` 中没有字段。 这是为了解决 Protobuf 不明确允许将空值传递到 RPC 方法或从 RPC 方法返回这一事实的技巧。 

现在有了模式。 需要编译该模式才算完成 gRPC的定义。 在 `grpcapi` 目录下执行下面的命令：

```sh
> protoc -I . implant.proto --go_out=plugins=grpc:./
```

在前面提到的初始化安装完成后这个命令才能用，该命令的作用是在当前目录下搜索名问 `implant.proto` 的 Protobuf 文件， 并在当前目录下生成特定于Go的输出。 一旦成功执行，在 `grpcapi` 目录下应该会生成名为 `implant.pb.go` 的新文件。 这个新文件包含了在Protobuf模式中创建的服务和消息的 `interface` 和 `struct` 定义。 我们将使用这些来构建我们的服务，植入和管理组件。 让我们一个一个来构建。