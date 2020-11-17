# http2-理论知识

[TOC]

GO的http库默认支持HTTP/2协议，只要使用TLS则会默认启动HTTP/2特性。


HTTP/2强制使用TLS。生成证书：

`openssl req -newkey rsa:2048 -nodes -keyout server.key -x509 -days 365 -out server.crt`



**基于标准库 http 启用 http2 server**

```go
// 基于标准库 http 启用 http2 server
func server1() {
   srv := &http.Server{
      Addr:":8848",
      Handler: http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
         // 记录请求协议
         log.Printf("Got connection: %s", r.Proto)
         // 向客户发送一条消息
         w.Write([]byte("Hello"))
      })}

   // http/2，基于TLS对。
   log.Printf("Serving on https://localhost:8848")
   // GO的http库默认支持HTTP/2协议，只要使用TLS则会默认启动HTTP/2特性。
   log.Fatal(srv.ListenAndServeTLS("CA/server.crt", "CA/server.key"))
}
```



## HTTP2 握手协议

HTTP/2协议握手分2种方式，一种叫h2，一种叫h2c。

h2要求必须使用TLS加密，在TLS握手期间会顺带完成HTTPS/2协议的协商，如果协商失败（比如客户端不支持或者服务端不支持），则会使用HTTPS/1继续后续通讯。

h2c不使用TLS，而是多了一次基于HTTP协议的握手往返来完成向HTTP/2协议的升级，一般不建议使用。

GO标准库http库默认支持HTTP/2协议，只要我们使用TLS则会默认启动HTTP/2特性。

http库在设计API时并没有支持用户使用h2c，而是鼓励使用h2。只要我们使用TLS，则http库就会默认进行HTTPS/2协商，协商失败则蜕化为HTTPS/1。


http 标准库对http2实现也是基于 [golang.org/x/net/http2](https://godoc.org/golang.org/x/net/http2), 它是相对比较底层的包，

通常的应用通过 http 包 实现 http2，当然如果需要

## HTTP2带来的变化

###  HTTP/1.x 的局限

HTTP/1.x 客户端需要使用多个连接才能实现并发和缩短延迟；

HTTP/1.x 不会压缩请求和响应标头，从而导致不必要的网络流量；

HTTP/1.x 不支持有效的资源优先级，致使底层 TCP 连接的利用率低下；等等。

在 HTTP/1.x 中，如果客户端要想发起多个并行请求以提升性能，则必须使用多个 TCP 连接，这是 HTTP/1.x 交付模型的直接结果，该模型可以保证每个连接每次只交付一个响应（响应排队）。 更糟糕的是，这种模型也会导致队首阻塞，从而造成底层 TCP 连接的效率低下。

### HTTP2的变化

与 HTTP/1.1 相比，HTTP/2 的主要变化在于性能提升。`HTTP/2 所有性能增强的核心在于新的二进制分帧层`，它定义了如何封装 HTTP 消息并在客户端与服务器之间传输。

HTTP/2 的主要目标是通过支持完整的请求与响应复用来减少延迟，通过有效压缩 HTTP 标头字段将协议开销降至最低，同时增加对请求优先级和服务器推送的支持。

与 HTTP/1.x 相比，可以使用更少的 TCP 连接。这意味着与其他流的竞争减小，并且连接的持续时间变长，这些特性反过来提高 了可用网络容量的利用率。

`HTTP/2 没有改动 HTTP 的应用语义。 HTTP 方法、状态代码、URI 和标头字段等核心概念一如往常。`不过，HTTP/2 修改了数据格式化（分帧）以及在客户端与服务器间传输的方式。这两点统帅全局.



需要注意的是，HTTP/2 仍是对之前 HTTP 标准的扩展，而非替代。 HTTP 的应用语义不变，提供的功能不变，HTTP 方法、状态代码、URI 和标头字段等这些核心概念也不变。 

### 每个来源一个连接

有了新的分帧机制后，HTTP/2 不再依赖多个 TCP 连接去并行复用数据流；每个数据流都拆分成很多帧，而这些帧可以交错，还可以分别设定优先级。 因此，所有 HTTP/2 连接都是永久的，而且仅需要每个来源一个连接，随之带来诸多性能优势。

大多数 HTTP 传输都是短暂且急促的，而 TCP 则针对长时间的批量数据传输进行了优化。 通过重用相同的连接，HTTP/2 既可以更有效地利用每个 TCP 连接，也可以显著降低整体协议开销。 不仅如此，使用更少的连接还可以减少占用的内存和处理空间，也可以缩短完整连接路径（即，客户端、可信中介和源服务器之间的路径） 这降低了整体运行成本并提高了网络利用率和容量。 因此，迁移到 HTTP/2 不仅可以减少网络延迟，还有助于提高通量和降低运行成本。

注：连接数量减少对提升 HTTPS 部署的性能来说是一项特别重要的功能：可以减少开销较大的 TLS 连接数、提升会话重用率，以及从整体上减少所需的客户端和服务器资源。

## 多路复用

在 HTTP/1 中，如果客户端要想发起多个并行请求以提升性能，则必须使用多个 TCP 连接。在单个链接中，HTTP/1 对每个请求每次交付一个响应，并且必须受到影响后，才能继续发起请求。就算是通过管道，单个连接的多个请求也只能顺次发送，顺次接受，顺序不能乱。

HTTP/2 中，HTTP 消息分解为互不依赖的帧，通过不同的 Stream 交错发送，最后再在另一端把它们重新组装起来。

在 HTTP/1.x 中，每个连接每次只交付一个响应（响应排队），HTTP/2  引入一个新的二进制分帧层，以实现请求和响应复用、优先级和标头压缩，目的是更有效地利用底层 TCP 连接；客户端和服务器可以将 HTTP 消息分解为互不依赖的帧，然后交错发送，最后再在另一端把它们重新组装起来。

![Snip20200815_8](/img/Snip20200815_8.png)



从图中，在HTTP2中，同一个连接内并行的多个数据流，多个请求之间不再有严格的顺序要求，这就大大减少了请求排队的时间，也能更充分发挥TCP的传输能力；

`将 HTTP 消息分解为独立的帧，交错发送，然后在另一端重新组装是 HTTP 2 最重要的一项增强。光这个机制就带来巨大的性能提升`。

## 二进制分帧 

HTTP/2 最革命性的原因就在于这个二进制分帧的引入。

- `Frame`，帧，HTTP/2 协议里通信的最小单位，每个帧有自己的格式，不同类型的帧负责传输不同的消息
- `Message`, 消息，类似 Request/Response 消息，每个消息包含一个或多个帧
- `Stream`，流，建立链接后的一个双向字节流，用来传输消息，可以承载一条或多条消息。

通过多路复用，所有的通信都在一个 tcp 链接上完成，会建立一个或多个 stream 来传递数据；

每个 stream 都有唯一的 id 标识和一些优先级信息，客户端发起的 stream 的 id 为单数，服务端发起的 stream id 为偶数；

每个 message 就是一次 Request 或 Response 消息，包含一个或多个帧，比如只返回 header 帧，相当于 HTTP 里 HEAD method 请求的返回；或者同时返回 header 和 Data 帧，就是正常的 Response 响应。

Frame 是最小的通信单位，承载着特定类型的数据，例如 Headers， Data, Ping, Setting 等等。 来自不同 stream 的 frame 可以交错发送，然后再根据每个 Frame 的 header 中的数据流标识符重新组装。

**连接，流，消息，帧关系图**

![Snip20200815_7](/img/Snip20200815_7.png)



### Frame 结构

```
+-----------------------------------------------+
|                 Length (24)                   |
+---------------+---------------+---------------+
|   Type (8)    |   Flags (8)   |
+-+-------------+---------------+-------------------------------+
|R|                 Stream Identifier (31)                      |
+=+=============================================================+
|                   Frame Payload (0...)                      ...
+---------------------------------------------------------------+
```

**Frame 的go表示**

```go
// A FrameHeader is the 9 byte header of all HTTP/2 frames.
type FrameHeader struct {
   valid bool // caller can access []byte fields in the Frame

   //Type: 表示 Frame 的类型, 目前定义了 0-9 共 10 种类型。
   Type FrameType

   //Flags 为一些特定类型的 Frame 预留的标志位，比如 Header, Data, Setting, Ping 等，都会用到。
   Flags Flags

   //Length: 表示 Frame Payload 的大小，即帧长度，不包括9字节头，最大不超过16M (uint24)，
   //但其实 payload 默认的大小是不超过 2^14 字节，可以通过 SETTING Frame 来设置 SETTINGS_MAX_FRAME_SIZE 修改允许的 Payload 大小。
   Length uint32

   // 帧所属流 id，其中 0 用来传输控制信息，比如 Setting, Ping
   StreamID uint32
}
```

R: 1-bit 的保留位，目前没用，值必须为 0

Stream Identifier: Steam 的 id 标识，表明 id 的范围只能为 0 到 2^31-1 之间，其中 0 用来传输控制信息，比如 Setting, Ping；

客户端发起的 Stream id 必须为奇数，

服务端发起的 Stream id 必须为偶数；

每次建立新 Stream 的时候，id 必须比上一次的建立的 Stream 的 id 大；

当在一个连接里，如果无限建立 Stream，最后 id 大于 2^31 时，必须从新建立 TCP 连接，来发送请求。如果是服务端的 Stream id 超过上限，需要对客户端发送一个 GOWAY 的 Frame 来强制客户端重新发起连接。

### 帧类型

```go
// A FrameType is a registered frame type as defined in
// http://http2.github.io/http2-spec/#rfc.section.11.2
type FrameType uint8

const (
   FrameData         FrameType = 0x0
   FrameHeaders      FrameType = 0x1
   FramePriority     FrameType = 0x2
   FrameRSTStream    FrameType = 0x3
   FrameSettings     FrameType = 0x4
   FramePushPromise  FrameType = 0x5
   FramePing         FrameType = 0x6
   FrameGoAway       FrameType = 0x7
   FrameWindowUpdate FrameType = 0x8
   FrameContinuation FrameType = 0x9
)

var frameName = map[FrameType]string{
	FrameData:         "DATA",
	FrameHeaders:      "HEADERS",
	FramePriority:     "PRIORITY",
	FrameRSTStream:    "RST_STREAM",
	FrameSettings:     "SETTINGS",
	FramePushPromise:  "PUSH_PROMISE",
	FramePing:         "PING",
	FrameGoAway:       "GOAWAY",
	FrameWindowUpdate: "WINDOW_UPDATE",
	FrameContinuation: "CONTINUATION",
}
```

**不同类型帧用到的标志为Flag**

```go
const (
	// Data Frame
	FlagDataEndStream Flags = 0x1  //标志用来表示 Data Frame 的传输是否结束
	FlagDataPadded    Flags = 0x8

	// Headers Frame
	FlagHeadersEndStream  Flags = 0x1
	FlagHeadersEndHeaders Flags = 0x4
	FlagHeadersPadded     Flags = 0x8
	FlagHeadersPriority   Flags = 0x20

	// Settings Frame
	FlagSettingsAck Flags = 0x1

	// Ping Frame
	FlagPingAck Flags = 0x1

	// Continuation Frame
	FlagContinuationEndHeaders Flags = 0x4

	FlagPushPromiseEndHeaders Flags = 0x4
	FlagPushPromisePadded     Flags = 0x8
)

```

#### DATA 帧

DATA 帧用来传输可变长度的二进制流，这部分最主要的用途就是用来传递之前 HTTP/1 中的 Request 或 Response 的 Body 部分。

`Pad Length 和 Padding` 是干什么用的？HTTP/2 在设计的时候就更多的考虑了数据的安全性，所以默认使用 HTTPS，除此之外，协议本身也对传输的数据做了一些安全考虑，填充就是其中一个。填充可以模糊帧的大小，使攻击者更难通过帧的数量来猜测传输内容的长度，减少破解的可能性。

`END_STREAM` (0x1)Flag，这个标志用来表示 Data Frame 的传输是否结束，当该标志位为 1 时，表示 Stream 的传输结束，发起 Stream 的一方会进入 half-closed(local) 或者 closed 状态，END_STREAM 在 Header 帧中也有用到，含义一样。

```
 +---------------+
 |Pad Length? (8)|
 +---------------+-----------------------------------------------+
 |                            Data (*)                         ...
 +---------------------------------------------------------------+
 |                           Padding (*)                       ...
 +---------------------------------------------------------------+
```



```go
type DataFrame struct {
   FrameHeader
   data []byte
}
```



#### HEADERS 帧

HEADERS Frame(type=0x1) 用于开启一个 Stream，当然也用于传输正常 HTTP 请求中的 Header 信息。

```
 +---------------+
 |Pad Length? (8)|
 +-+-------------+-----------------------------------------------+
 |E|                 Stream Dependency? (31)                     |
 +-+-------------+-----------------------------------------------+
 |  Weight? (8)  |
 +-+-------------+-----------------------------------------------+
 |                   Header Block Fragment (*)                 ...
 +---------------------------------------------------------------+
 |                           Padding (*)                       ...
 +---------------------------------------------------------------+
```

`Header Block Fragment` 字段用于存储正常的 Http Header 头信息，`E、Stream Dependency、Weight` 字段都是用于权重控制。由于 HTTP/2 是支持多路复用，也就是多个流同时进行传输，那么这个时候哪个流更重要，应该优先传输哪个，就需要用这些字段来进行控制了。

```go
// HeadersFrameParam are the parameters for writing a HEADERS frame.
type HeadersFrameParam struct {
   StreamID uint32
   // BlockFragment is part (or all) of a Header Block.
   BlockFragment []byte

   // EndStream 即 `END_STREAM` (0x1)Flag，这个标志用来表示 Data Frame 的传输是否结束，
   // 当该标志位为 1 时，表示 Stream 的传输结束，发起 Stream 的一方会进入 half-closed(local) 或者 closed 状态
   EndStream bool

   // END_HEADERS 标识，这个帧的包含一个完整的header快，不可以接着 CONTINUATION 帧
   EndHeaders bool

   // 补到这个帧的零值
   PadLength uint8

   // 流优先级
   Priority PriorityParam
}
```



```go
  FrameHeaders: {// Headers Frame
    FlagHeadersEndStream:  "END_STREAM",
    FlagHeadersEndHeaders: "END_HEADERS",
    FlagHeadersPadded:     "PADDED",
    FlagHeadersPriority:   "PRIORITY",
  }
```

#### PRIORITY 帧

PRIORITY Frame(type=0x2) 用于指定 Stream 的优先级，PRIORITY 帧不能在 id 为 0 的 stream 上发送。它可以以任何流状态发送，包括空闲或关闭的流。PRIORITY 帧 没有任何Flag 标志位。

一个优先帧的有效负载包含以下字段:

E: 指示流依赖关系是排他独占的单位标志

Stream Dependencies：31位流标识符，标示此流所依赖的流

Weight：表示流优先级权重的uint8整数。值1到256之间。

```
 +-+-------------------------------------------------------------+
 |E|                  Stream Dependency (31)                     |
 +-+-------------+-----------------------------------------------+
 |   Weight (8)  |
 +-+-------------+
```

```go
// A PriorityFrame specifies the sender-advised priority of a stream.
type PriorityFrame struct {
   FrameHeader
   PriorityParam
}
```

流优先级参数：

```go
type PriorityParam struct {
   // StreamDep 是 31-bit 的流标识，表示依赖的流。0 表示没有依赖
   StreamDep uint32
   // Exclusive 表示依赖是否是独占的
   Exclusive bool
   // Weight 即权重，和 StreamDep 一起使用，范围是1到256
   Weight uint8
}
```



#### RST_STREAM 帧

RST_STREAM Frame(type=0x3) 用于立即终止 Stream. 主要用来取消流，或者发生异常时表明需要终止。

```
 +---------------------------------------------------------------+
 |                        Error Code (32)                        |
 +---------------------------------------------------------------+
```

```go
// A RSTStreamFrame allows for abnormal termination of a stream.
type RSTStreamFrame struct {
   FrameHeader
   ErrCode ErrCode
}
```



#### SETTINGS 帧

SETTINGS Frame(type=0x4) 用来控制客户端和服务端之间通信的一些配置。

SETTINGS 帧必须在连接开始时由通信双方发送，并且可以在任何其他时间由任一端点在连接的生命周期内发送;

SETTINGS 帧必须在 id 为 0 的 stream 上进行发送，不能通过其他 stream 发送；

SETTINGS 影响的是整个 TCP 链接，而不是某个 stream；

在 SETTINGS 设置出现错误时，必须当做 connection error 重置整个链接;

SETTINGS 帧带有 Ack 的 Flag，接收方必须收到 ack 为 0 的 SETTINGS 后，应马上启用 SETTING 的配置并返回一个 Ack 为 1 的 SETTINGS 帧。

```go
type SettingsFrame struct {
   FrameHeader
   p []byte
}
```

```
 +-------------------------------+
 |       Identifier (16)         |
 +-------------------------------+-------------------------------+
 |                        Value (32)                             |
 +---------------------------------------------------------------+
```

**常用的 SETTINGS 有几类：**

- `SETTINGS_HEADER_TABLE_SIZE` (0x1): 控制每个 Header 帧中的 HTTP 头信息的大小
- `SETTINGS_ENABLE_PUSH` (0x2): 是否启用服务端推送 (Server Push)，默认开启；不管是服务端还是客户端发送了禁用的配置，那么服务端就不应该发送 PUSH_PROMISE 帧
- `SETTINGS_MAX_CONCURRENT_STREAMS` (0x3): 用来控制多路复用中 Stream 并发的数量，这个主要是用来限制单个链接对服务端的资源的占用过大，这个值默认是没有限制，如果做一个 server 服务，那么建议一定要设置这个值，RFC 文档中建议不要小于 100，那么我们设置 100 就可以了。
- SETTINGS_INITIAL_WINDOW_SIZE(0x4)、
- SETTINGS_MAX_FRAME_SIZE(0x5)、 
- SETTINGS_MAX_HEADER_LIST_SIZE(0x6) 

#### PUSH_PROMISE 帧

PUSH_PROMISE Frame(type=0x5) 用于服务端在发送 PUSH 之前先发送 PUSH_PROMISE 帧来通知客户端将要发送的 PUSH 信息。PUSH_PROMISE 涉及到 server push 的相关信息.

```
+---------------+
 |Pad Length? (8)|
 +-+-------------+-----------------------------------------------+
 |R|                  Promised Stream ID (31)                    |
 +-+-----------------------------+-------------------------------+
 |                   Header Block Fragment (*)                 ...
 +---------------------------------------------------------------+
 |                           Padding (*)                       ...
 +---------------------------------------------------------------+
```

```go
// A PushPromiseFrame is used to initiate a server stream.
type PushPromiseFrame struct {
   FrameHeader
   PromiseID     uint32
   headerFragBuf []byte // not owned
}
```



Flag: "END_HEADERS","PADDED",

#### PING 帧

PING Frame(type=0x6) 是用来测量来自发送方的最小往返时间以及确定空闲连接是否仍然起作用的机制。 PING 帧可以从任何一方发送。PING 帧跟 SETTINGS 帧非常类似，一个是必须在 id 为 0 的 stream 上发送，另一个就是它也包含一个 Ack 的 Flag，发送方发送 ack=0 的 PING 帧，接收方必须响应一个 ack=1 的 PING 帧，并且 PING 帧的响应 应该 优先于任何其他帧。

Flag： “ACK”

```
 +---------------------------------------------------------------+
 |                                                               |
 |                      Opaque Data (64)                         |
 |                                                               |
 +---------------------------------------------------------------+
```

```go
type PingFrame struct {
   FrameHeader
   Data [8]byte
}
```





#### GOAWAY 帧

GOAWAY frame(type=0x7) 用于关闭连接，GOAWAY 允许端点优雅地停止接受新流，同时仍然完成先前建立的流的处理。

当服务端需要维护时，发送一个 GOAWAY 的 Frame 给客户端，那么发送之前的 Stream 都正常处理了，发送 GOAWAY 后，客户端会新启用一个链接，继续刚才未完成的 Stream 发送。这样就可以做到完全不影响运行中的业务而进行服务端维护。

```
 +-+-------------------------------------------------------------+
 |R|                  Last-Stream-ID (31)                        |
 +-+-------------------------------------------------------------+
 |                      Error Code (32)                          |
 +---------------------------------------------------------------+
 |                  Additional Debug Data (*)                    |
 +---------------------------------------------------------------+
```

#### WINDOW_UPDATE 帧

WINDOW_UPDATE frame(type=0x8) 用于流控 (flow control)。

流控制在两个级别上运行:在每个单独的流上和在整个连接上。这两种类型的流控制都是逐跳的，也就是说，只在两个端点之间进行。中介不转发依赖连接之间的WINDOW_UPDATE帧。然而，节流任何接收器的数据传输可能间接导致流控制信息向原始发送方传播。

```go
type WindowUpdateFrame struct {
   FrameHeader
   Increment uint32 // never read with high bit set
}
```

```
 +-+-------------------------------------------------------------+
 |R|              Window Size Increment (31)                     |
 +-+-------------------------------------------------------------+
```

#### CONTINUATION 帧

CONTINUATION frame(type=0x9) 用于持续的发送未发送完的 HTTP header 信息. 如果前边是这三个帧 (HEADERS, PUSH_PROMISE, or CONTINUATION)，并且未携带 END_HEADERS 的 flag，就可以继续发送 CONTINUATION 帧。

```
 +---------------------------------------------------------------+
 |                   Header Block Fragment (*)                 ...
 +---------------------------------------------------------------+
```

Flag: "END_HEADERS"

## 流控制

流控制是一种阻止发送方向接收方发送大量数据的机制，以免超出后者的需求或处理能力：发送方可能非常繁忙、处于较高的负载之下，也可能仅仅希望为特定数据流分配固定量的资源。

例如，客户端可能请求了一个具有较高优先级的大型视频流，但是用户已经暂停视频，客户端现在希望暂停或限制从服务器的传输，以免提取和缓冲不必要的数据。 再比如，一个代理服务器可能具有较快的下游连接和较慢的上游连接，并且也希望调节下游连接传输数据的速度以匹配上游连接的速度来控制其资源利用率；等等。

 不过，由于 HTTP/2 数据流在一个 TCP 连接内复用，`TCP 流控制既不够精细`，是连接级别的流控，无法提供必要的应用级 API 来调节各个数据流的传输。 

为了解决这一问题，`HTTP/2允许客户端和服务器实现其自己的数据流和连接级流控制`：

- 流控制具有方向性。 每个接收方都可以根据自身需要选择为每个数据流和整个连接设置任意的窗口大小。
- 流控制基于信用。 每个接收方都可以公布其初始连接和数据流流控制窗口（以字节为单位），每当发送方发出 `DATA` 帧时都会减小，在接收方发出 `WINDOW_UPDATE` 帧时增大。
- 流控制无法停用。 建立 HTTP/2 连接后，客户端将与服务器交换 `SETTINGS` 帧，这会在两个方向上设置流控制窗口。 流控制窗口的默认值设为 65,535 字节，但是接收方可以设置一个较大的最大窗口大小（`2^31-1` 字节），并在接收到任意数据时通过发送 `WINDOW_UPDATE` 帧来维持这一大小。
- 流控制为逐跃点控制，而非端到端控制。 即，可信中介可以使用它来控制资源使用，以及基于自身条件和启发式算法实现资源分配机制。



## 服务端推送

TTP/2 新增的另一个强大的新功能是，服务器可以对一个客户端请求发送多个响应。 即服务器推送。

HTTP/2 打破了严格的请求-响应语义，支持一对多和服务器发起的推送工作流，在浏览器内外开启了全新的互动可能性。 这是一项使能功能，对我们思考协议、协议用途和使用方式具有重要的长期影响。

在网页使用许多资源（HTML，样式表，脚本，图像等）在 HTTP/1.x 中都必须显式的逐个请求这些资源。这通常是一个缓慢的过程。浏览器首先获取 HTML，然后在解析和评估页面时逐步监测到需要获取的更多资源。由于服务器必须等待浏览器发出每个文件获取请求，因此网络通常处于空闲状态且利用率较低。

为了减少延迟，HTTP/2 引入了 Server Push ，它允许服务器在显式请求资源之前将资源推送到浏览器。服务器通常知道页面需要的许多其他资源，并且可以在响应初始请求时开始推送这些资源。这使服务器可以充分利用原本空闲的网络并缩短页面加载时间。

HTTP 2.0允许服务器为单个客户端请求(又称服务器推送)发送多个响应。

![Snip20200815_5](/img/Snip20200815_5.png)

推送资源可以进行以下处理：

- 由客户端缓存
- 在不同页面之间重用
- 与其他资源一起复用
- 由服务器设定优先级
- 被客户端拒绝

**PUSH_PROMISE**

所有服务器推送数据流都由 `PUSH_PROMISE` 帧发起，表明了服务器向客户端推送所述资源的意图，并且需要先于请求推送资源的响应数据传输。 这种传输顺序非常重要：客户端需要了解服务器打算推送哪些资源，以免为这些资源创建重复请求。 满足此要求的最简单策略是先于父响应（即，`DATA` 帧）发送所有 `PUSH_PROMISE` 帧，其中包含所承诺资源的 HTTP 标头。

在客户端接收到 `PUSH_PROMISE` 帧后，它可以根据自身情况选择拒绝数据流（通过 `RST_STREAM` 帧）。 （例如，如果资源已经位于缓存中，便可能会发生这种情况。） 这是一个相对于 HTTP/1.x 的重要提升。 相比之下，使用资源内联（一种受欢迎的 HTTP/1.x“优化”）等同于“强制推送”：客户端无法选择拒绝、取消或单独处理内联的资源。

使用 HTTP/2，客户端仍然完全掌控服务器推送的使用方式。 客户端可以限制并行推送的数据流数量；调整初始的流控制窗口以控制在数据流首次打开时推送的数据量；或完全停用服务器推送。 这些优先级在 HTTP/2 连接开始时通过 `SETTINGS` 帧传输，可能随时更新。

推送的每个资源都是一个数据流，与内嵌资源不同，客户端可以对推送的资源逐一复用、设定优先级和处理。 浏览器强制执行的唯一安全限制是，推送的资源必须符合原点相同这一政策：服务器对所提供内容必须具有权威性。

所以在HTTP/2时代，资源内嵌反而不高效。

在协议级别，HTTP/2 服务器推送由 `PUSH_PROMISE` 帧实现。`PUSH_PROMISE` 描述服务器预测浏览器将在不久的将来发出的请求。一旦浏览器收到 PUSH_PROMISE，它就会知道服务器将交付资源。如果浏览器后来发现它需要此资源，它将等待推送完成，而不是发送新请求。这减少了浏览器在网络上花费的时间。

server push go 代码demo `go get golang.org/x/blog/content/h2push/server`

```go
var httpAddr = flag.String("http", ":8080", "Listen address")

func serverPush() {
   flag.Parse()
   http.Handle("/static/", http.StripPrefix("/static/", http.FileServer(http.Dir("static"))))
   http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
      if r.URL.Path != "/" {
         http.NotFound(w, r)
         return
      }
      pusher, ok := w.(http.Pusher)
      if ok { 
         //如果判断支持服务端推送，则直接把css，js文件推送给终端，而不是等终挨个端请求
         //这样可以提高页面加载效率
         if err := pusher.Push("/static/app.js", nil); err != nil {
            log.Printf("Failed to push: %v", err)
         }
         if err := pusher.Push("/static/style.css", nil); err != nil {
            log.Printf("Failed to push: %v", err)
         }
      }
      // 写index的响应内容前，进行推送
      fmt.Fprintf(w, indexHTML)
   })
   // 配置了 TLS，http 标准库会自动有http2协议
   log.Fatal(http.ListenAndServeTLS(*httpAddr, "CA/cert.pem", "CA/key.pem", nil))
}

const indexHTML = `<html>
<head>
   <title>Hello World</title>
   <script src="/static/app.js"></script>
   <link rel="stylesheet" href="/static/style.css"">
</head>
<body>
Hello, gopher!
</body>
</html>
`
```

当访问 https://localhost:8080/ 时，服务端会同时把 app.js 和 style.css 一起发给客户端，而不是等客户端请求。 

![Snip20200815_4](/img/Snip20200815_4.png)

注意事项。首先，能推送服务器具有权威性的资源，即不能推送第三方服务器或 CDN 上托管的资源。

其次，除非确信客户端实际需要这些资源否则不要推送资源，否则反而会浪费带宽。例如是当客户端可能已经缓存了资源时则不要再推送。第三，在页面上不加思考的推送所有资源通常会使性能变差。



扩展阅读

[HTTP/2 Server Push](https://blog.golang.org/h2push)

[HTTP/2 服务端推送](https://learnku.com/docs/go-blog/h2push/6514)

[HTTP/2 Push: The details](https://calendar.perfplanet.com/2016/http2-push-the-details/)

[Innovating with HTTP 2.0 Server Push](https://www.igvita.com/2013/06/12/innovating-with-http-2.0-server-push/)



## 流优先级

将 HTTP 消息分解为很多独立的帧之后，我们就可以复用多个数据流中的帧，客户端和服务器交错发送和传输这些帧的顺序就成为关键的性能决定因素。 为了做到这一点，HTTP/2 标准允许每个数据流都有一个关联的`权重`和`依赖关系`：

- 可以向每个数据流分配一个介于 1 至 256 之间的整数。
- 每个数据流与其他数据流之间可以存在显式依赖关系。

其中依赖关系的优先级要高于权重，而权重的不同，所得的分配资源的配额也不同。

数据流依赖关系和权重的组合让客户端可以构建和传递“优先级树”，表明它倾向于如何接收响应。 反过来，服务器可以使用此信息通过控制 CPU、内存和其他资源的分配设定数据流处理的优先级，在资源数据可用之后，带宽分配可以确保将高优先级响应以最优方式传输至客户端。

![Snip20200815_9](/Users/hongxingxing/www/hugo/myblog/content/go/http2/img/Snip20200815_9.png)

HTTP/2 内的数据流依赖关系通过将另一个数据流的唯一标识符作为父项引用进行声明；如果忽略标识符，相应数据流将依赖于“根数据流”。 声明数据流依赖关系指出，应尽可能先向父数据流分配资源，然后再向其依赖项分配资源。 换句话说，“请先处理和传输响应 D，然后再处理和传输响应 C”。

共享相同父项的数据流（即，同级数据流）应按其权重比例分配资源。 例如，如果数据流 A 的权重为 12，其同级数据流 B 的权重为 4，那么要确定每个数据流应接收的资源比例，请执行以下操作：

1. 将所有权重求和：`4 + 12 = 16`
2. 将每个数据流权重除以总权重：`A = 12/16, B = 4/16`

因此，数据流 A 应获得四分之三的可用资源，数据流 B 应获得四分之一的可用资源；数据流 B 获得的资源是数据流 A 所获资源的三分之一。

我们来看一下上图中的其他几个操作示例。 从左到右依次为：

1. 数据流 A 和数据流 B 都没有指定父依赖项，依赖于隐式“根数据流”；A 的权重为 12，B 的权重为 4。因此，根据比例权重：数据流 B 获得的资源是 A 所获资源的三分之一。
2. 数据流 D 依赖于根数据流；C 依赖于 D。 因此，D 应先于 C 获得完整资源分配。 权重不重要，因为 C 的依赖关系拥有更高的优先级。
3. 数据流 D 应先于 C 获得完整资源分配；C 应先于 A 和 B 获得完整资源分配；数据流 B 获得的资源是 A 所获资源的三分之一。
4. 数据流 D 应先于 E 和 C 获得完整资源分配；E 和 C 应先于 A 和 B 获得相同的资源分配；A 和 B 应基于其权重获得比例分配。

如上面的示例所示，数据流依赖关系和权重的组合明确表达了资源优先级，这是一种用于提升浏览性能的关键功能，网络中拥有多种资源类型，它们的依赖关系和权重各不相同。 不仅如此，HTTP/2 协议还允许客户端随时更新这些优先级，进一步优化了浏览器性能。 换句话说，我们可以根据用户互动和其他信号更改依赖关系和重新分配权重。

注：数据流依赖关系和权重表示传输优先级，而不是要求，因此不能保证特定的处理或传输顺序。 即，客户端无法强制服务器通过数据流优先级以特定顺序处理数据流。 尽管这看起来违反直觉，但却是一种必要行为。 我们不希望在优先级较高的资源受到阻止时，还阻止服务器处理优先级较低的资源。

## 头压缩

每个 HTTP 传输都承载一组标头，这些标头说明了传输的资源及其属性。 在 HTTP/1.x 中，此元数据始终以纯文本形式，通常会给每个传输增加 500–800 字节的开销。如果使用 HTTP Cookie，增加的开销有时会达到上千字节。 （请参阅[测量和控制协议开销](https://hpbn.co/http1x/#measuring-and-controlling-protocol-overhead)。） 为了减少此开销和提升性能，HTTP/2 使用 HPACK 压缩格式压缩请求和响应标头元数据，这种格式采用两种简单但是强大的技术：

1. 这种格式支持通过静态霍夫曼代码对传输的标头字段进行编码，从而减小了各个传输的大小。
2. 这种格式要求客户端和服务器同时维护和更新一个包含之前见过的标头字段的索引列表（换句话说，它可以建立一个共享的压缩上下文），此列表随后会用作参考，对之前传输的值进行有效编码。

利用霍夫曼编码，可以在传输时对各个值进行压缩，而利用之前传输值的索引列表，我们可以通过传输索引值的方式对重复值进行编码，索引值可用于有效查询和重构完整的标头键值对。

作为一种进一步优化方式，HPACK 压缩上下文包含一个静态表和一个动态表：静态表在规范中定义，并提供了一个包含所有连接都可能使用的常用 HTTP 标头字段（例如，有效标头名称）的列表；动态表最初为空，将根据在特定连接内交换的值进行更新。 因此，为之前未见过的值采用静态 Huffman 编码，并替换每一侧静态表或动态表中已存在值的索引，可以减小每个请求的大小。

注：在 HTTP/2 中，请求和响应标头字段的定义保持不变，仅有一些微小的差异：所有标头字段名称均为小写，请求行现在拆分成各个 `:method`、`:scheme`、`:authority` 和 `:path` 伪标头字段。



参考与扩展阅读

[http2 demo server](https://http2.golang.org/)

[golang.org/x/net/http2](https://godoc.org/golang.org/x/net/http2)

[HTTP/2 RFC7540](https://tools.ietf.org/html/rfc7540)

[HTTP/2 简介](https://developers.google.com/web/fundamentals/performance/http2)

[High Performance Browser Networking](https://www.oreilly.com/library/view/high-performance-browser/9781449344757/)

[“HTTP/2”](https://hpbn.co/http2/) – Ilya Grigorik 所著的完整文章

[“HTTP/2 推送的经验法则”](https://docs.google.com/document/d/1K0NykTXBbbbTlv60t5MyJvXjqKGsCVNYHyLEXIxYMv0/edit) – Tom Bergan、Simon Pelchat 和 Michael Buettner 对何时以及如何使用推送的分析。















