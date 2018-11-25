## http1.x

#### 队头阻塞
在一个给定的TCP连接上，HTTP/1.0只允许发送一次，每次一个请求。

管道化：允许一次发送连续的请求，而不用等待应答返回。但其缺点造成浏览器默认关闭。
- 队头阻塞
- 并行处理需要占用缓存资源，服务器容易受到攻击
- 响应失败后，会断开tcp链接，并要求重发之后所有请求，造成资源浪费
- 中间代理对其兼容性不是很好

队头阻塞： 在请求应答过程中，如果出现任何状况，剩下所有的工作都会被阻塞在那次请求应答之后。它会阻碍网络传输和 Web 页面渲染，直至失去响应。

为了防止这种问题，现代浏览器会针对单个域名开启6个连接，通过各个连接分别发送请求。它实现了某种程度上的并行，但是每个连接仍会受到队头阻塞的影响。

#### 拥塞
tcp设置的初衷是：假设情况很保守，并能够公平对待同一网络的不同流量的应用。它的避免拥塞机制被设计成即使在最差的网络状况下仍能起作用，并且如果有需求冲突也保证相对公平。

拥塞窗口：在接收方确认数据包之前，发送方可以发出的 TCP 包的数量。如果拥塞窗口指定为1，发送方发送一个数据，只有接受方确认了数据之后才能发送下一个。

慢启动: 几何型增长拥塞窗口，拥塞之后减半，没有出现拥塞之后缓缓上涨。现代操作系统一般会取4~10数据包作为初始拥塞窗口大小。因为http1.1并不支持多路复用，所以浏览器一般会针对指定域名开启 6 个并发 连接。这意味着拥塞窗口波动也会并行发生6次。TCP协议保证那些连接都能正常工作，但是不能保证它们的性能是最优的。

#### 臃肿的消息首部
一般集中在460字节左右，100个资源的web页面，将近45k的消耗。导致初始TCP拥塞窗口被快速填满。当在一个新建立的TCP连接上发送多个请求时，这会产生极大的时延

## Overview
2015年5月24日HTTP/2协议正式版发布。http2总共经历了18个版本，从00到17，如h2-12说明是第十二版草案。如果某个软件上写着支持h2c表示运行在非加密通道上的http2(HTTP/2 Cleartext)
- Start http2
- Frame
- Header Compressed(HPack)
- Stream
- Http Message Exchange
- Server Push

## Start Http2
- http2使用相同的URL方案'http(80)'与'https(443)'，在处理对URI请求之前，需要确认上游服务器是否支持HTTP/2
- h2: 运行在TLS之上，被序列化成一个ALPN协议标识符(0x68，0x32)
- h2c: 明文TCP，运行在http1.1的upgrade首字段，以及任何在http上运行http2的场合

1. 不知下一跳是否支持http2
    使用upgrade机制发起请求，客户端先发起HTTP/1.1请求，该请求包含值为"h2c"的Upgrade首部字段，还必须包含一个HTTP2-Settings( 3.2.1节 )首部字段。
    ```
    GET / HTTP/1.1
    Host: server.example.com
    Connection: Upgrade, HTTP2-Settings
    Upgrade: h2c
    HTTP2-Settings: <base64url encoding of HTTP/2 SETTINGS payload>
    ```
    - 在客户端能发送HTTP/2帧之前，包含负载的请求必须全部被发送。
    - 如果一个初始请求的后续并发请求很重要，那么可以使用OPTIONS请求来执行升级到HTTP/2的操作，代价是一个额外的往返。
    - 不支持的情况下，直接使用http1.1请求
    - 支持HTTP/2的服务端返回101(Switching Protocols)响应，表示接受升级协议的请求。在结束101响应的空行之后，服务端可以开始发送HTTP/2帧。这些帧必须包含一个对升级请求的响应。
    - 服务端发送的第一个HTTP/2帧必须是由一个SETTINGS组成的服务端连接前言。客户端一收到101响应，也必须发送一个包含SETTINGS帧的连接前言。
    - HTTP2-Settings: SETTING帧的有效载荷被编码成的base64url串
2. https url
    客户端对"https"URI发起请求时使用带有应用层协议协商(ALPN)扩展的TLS。运行在TLS之上的HTTP/2使用"h2"协议标识符。TLS协商完成，两端发送前言。
    一定程度上可认为不存在试探是否支持或直接连接的烦恼，因为这个过程直接在TLS层协商而成。
    - TLS协商完成
    - 客户端发送前言
    - 服务端接收到之后再发送前言
    - 各自确认
    - 正常传输
3. 已知支持http2
    可以通过类似于[ALT-SVC]的机制了解是否支持http2。客户端必须先向这种服务端发送连接前言，然后可以立即发送HTTP/2帧。之后服务端也需发送连接前言。

#### Http/2 Connection Preface
http/2要求两端都必须发送连接前言，作为对使用协议的最终确认，并确定初始设置。任一端接收到SETTINGS帧之后，都需要返回一个包含确认标志位SETTIGN作为确认

```
PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n // 纯字符串表示，翻译成字节数为24个字节
SETTINGS帧                       // 其负载可能为空
```

允许客户端发送完连接前奏后就立即向服务端发送其他的帧，而不必等待服务端的连接前奏。

不过需要注意的是，服务端连接前奏的SETTINGS帧中可能包含一些期望客户端如何与服务端进行通信所必须修改的参数。在收到这些SETTINGS帧以后，客户端应当遵守所有设置的参数。在某些配置中，服务端是可以在客户端发送额外的帧之前传送 SETTINGS 帧的，这样就避免前边所说的问题。

## Frame
#### 帧格式
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
- Length: 帧有效载荷的长度，以24位无符号整数表示。除非接收端通过SETTINGS_MAX_FRAME_SIZE设置了更大的值，否则不能发送Length值大于2^14 (16,384)的帧。
- Type: 8bit的帧类型。帧类型决定了帧的格式和语义。必须忽略和丢弃任何未知的帧类型。
- Flags: 为帧类型保留的8bit布尔标识字段
- R: 1bit的保留字段。未定义该bit的语义。
- Stream Identifier: 流标识符是一个31bit的无符号整数。(值0x0是保留的，表明帧是与整体的连接相关的，而不是和单独的流相关。)

#### 帧定义
每种帧类型由唯一的8bit类型码标识。在建立和管理整个连接或者单独一个流时，每种帧类型都服务于特定的目的。传送特定的帧类型可以改变连接的状态。如果两端不能保持同步的连接状态，就不可能再成功地进行通信。

之后介绍的10种帧，为了表示简单，均只展示头部之后的帧载荷。其头部为通用帧头部。
##### DATA
用于传送某一个流的任意的、可变长度的字节序列。比如，用一个或多个DATA帧来携带HTTP请求或响应的载荷。DATA必须和某一个流关联，收到流量控制的限制。
type: 0x0
domain: 
- Pad Length: 只有padded标识才出现
- Data
- Padding: 填充，不包含应用语义，只有padded标识才出现
flag:
- END_STREAM: 当设置了该标识，第0位就指明了该帧是端点在指定流上发送的最后一帧。
- PADDED: 填充标识

```
+-----------------------------------------------+
|                Length (24)                    |
+---------------+---------------+---------------+
| 0x0 (8)       | 0000 ?00? (8) |
+-+-------------+---------------+-------------------------------+
|R|                Stream Identifier (31)                       |
+=+=============+===============================================+
|Pad Length? (8)|
+---------------+-----------------------------------------------+
|                            Data (*)                         ...
+---------------------------------------------------------------+
|                          Padding? (*)                       ...
+---------------------------------------------------------------+
```

##### HEADERS
HEADER帧用来打开一个流，再额外的携带一个首部片段。可以在一个流处于idel, reverse(local), open, half-closed(remote)状态下发送。HEADERS必须与某一个流关联。HEADER里的优先级信息等价与一个单独的PRIORITY帧，之后的HEADERS帧的PRIORITY域会变更流的优先级顺序。
type: 0x1
domain: 
- Pad Length
- E: 1bit标识，表示流依赖是否是专用的, 只有设置PRIORITY才会存在。
- Stream Dependecy: 流依赖, 只有设置PRIORITY才会存在。
- Weight: 权重, 只有设置PRIORITY才会存在。
- Header Block Fragment: 首部块片段
- Padding
flag: 
- END_STREAM: 携带此表明流要结束了。终端开始通过此流标识传输数据。但是在同一个流上，HEADER后存在的帧还可以跟随CONTINUATION帧。
- END_HEADER: 表示该帧包含整个首部块，后面不跟随CONTINUATION帧
- PADDED
- PRIORITY
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

##### PRIORITY
此帧指定流的发送方建议优先级。它可以在任何流状态下发送，包括空闲或封闭流。但限制是不能够在一个包含有报头块连续的帧里面出现，
type: 0x2
domain:
- E: 独占标志位
- Stream Dependency
- Weight
```
 +-+-------------------------------------------------------------+
 |E|                  Stream Dependency (31)                     |
 +-+-------------+-----------------------------------------------+
 |   Weight (8)  |
 +-+-------------+
```

##### RST_STREAM
RST_STREAM帧允许马上中止流。发送RST_STREAM以请求取消流或指示已发生错误情况。
type:0x3

在流上接收到RST_STREAM后，接收方不得为该流发送额外的帧，但PRIORITY除外。 但是，在发送RST_STREAM之后，发送端点必须准备好接收和处理在RST_STREAM到达之前可能已经由对等方发送的流上发送的附加帧。

```
 +---------------------------------------------------------------+
 |                        Error Code (32)                        |
 +---------------------------------------------------------------+
```

##### SETTINGS
用来传送影响两端通信的配置参数。接收者向发送者通告己方设定，服务器端在连接成功后必须第一个发送的帧。SETTINGS帧也用于通知对端自己收到了这些参数。SETTING帧只作用于整个连接。

type: 0x4

```
 +-------------------------------+
 |       Identifier (16)         |
 +-------------------------------+-------------------------------+
 |                        Value (32)                             |
 +---------------------------------------------------------------+
```

SETTING帧的处理流程：
1. 发送端发送需要两端都需要携带有遵守的SETTINGS设置帧，不能够带有ACK标志位
2. 接收端接收到无ACK标志位的SETTINGS帧，必须按照帧内字段出现顺序一一进行处理，中间不能够处理其它帧 
3. 接收端处理时，针对不受支持的参数需忽略
4. 接收端处理完毕之后，必须响应一个包含有ACK确认标志位、无负载的SETTINGS帧
5. 发送端接收到确认的SETTINGS帧，表示两端设置已生效。发送端等待确认若超时，报SETTINGS_TIMEOUT类型连接错误

SETTING帧的参数包括：
- SETTINGS_HEADER_TABLE_SIZE：报头表字节数最大值。报头块解码使用；初始值为4096个字节。
- SETTINGS_ENABLE_PUSH: 服务器推送
- SETTINGS_MAX_CONCURRENT_STREAMS：发送者允许可打开流的最大值，建议值100，默认可不用设置
- SETTINGS_INITIAL_WINDOW_SIZE: 发送端流控窗口大小
- SETTINGS_MAX_FRAME_SIZE: 单帧负载最大值
- SETTINGS_MAX_HEADER_LIST_SIZE:发送端通告自己准备接收的报头集合最大值，即字节数

##### PUSH_PROMISE
用于在发送方打算发起的流之前通知对等端点。服务器端通知对端初始化一个新的推送流准备稍后推送数据
- 要求推送流为打开或远端半关闭（half closed (remote)）状态
- 承诺的流不一定要按照其流打开顺序进行使用，仅用作稍后使用
- 受对端所设置SETTINGS_ENABLE_PUSH标志位决定是否发送
- 接收端一旦拒绝接收推送，会发送RST_STREAM帧告知对方推送无效
type: 0x5
domain:
- Pad Length: 帧填充长度
- R: 保留位
- Promise Stream ID: 承诺的流标识符必须是发送方发送的下一个流的有效选择
- Header Block Fragment: 请求头字块片段
- Padding: 帧填充
flag: 
- END_HEADERS: 位2表示该帧已经包含了整个标题位而且其后不跟CONTINUATION帧
- PADDED

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

##### PING
PING帧用以测量来自发送方的最小往返时间以及确定空闲连接是否仍然起作用的机制。可以从任何端点发送。
```
 +---------------------------------------------------------------+
 |                                                               |
 |                      Opaque Data (64)                         |
 |                                                               |
 +---------------------------------------------------------------+
```

- PING帧不与任何单个流相关联。
- 除了帧头之外，PING帧必须在有效载荷中包含8个八位字节的不透明数据。发件人可以包含它选择的任何值，并以任何方式使用这些八位字节。
- 不包含ACK标志的PING帧的接收器必须发送PING帧，其中ACK标志响应地设置，具有相同的有效载荷。 PING响应应该优先于任何其他帧。

#### GOAWAY
用于发起关闭连接，或者警示严重错误。一端通知对端较为优雅的方式停止创建流，同时还要完成之前已建立流的任务。
- 一旦发送，发送者将忽略接收到的流标识符大于Last-Stream-ID任何帧
- 接收者不能够在当前流上创建新流，若创建新流则创建新的连接
- 可用于服务器的管理行为，比如服务器进入维护阶段，不再准备接收新的连接
- 字段Last-Stream-ID为发送方取自最后一个正在处理或已经处理流的标识符
- 后续创建的流标识符高于Last-Stream-ID数据帧都不会被处理
- GOAWAY应用于当前连接，非具体流
- 发送端在发送GOAWAY时还有一些流任务没有完成，将保持连接为打开状态直到任务完成
- 流（标识符小于或等于已有编号的标识符）在连接关闭之前没有被完全关闭，需要创建新的连接进行重试

type: 0x7
```
+-+-------------------------------------------------------------+
|R|                  Last-Stream-ID (31)                        |
+-+-------------------------------------------------------------+
|                      Error Code (32)                          |
+---------------------------------------------------------------+
|                  Additional Debug Data (*)                    |
+---------------------------------------------------------------+
```

##### WINDOW_UPDATE
流量控制帧。

- 作用于单个流以及整个连接两个级别上运行。在前一种情况下，帧的流标识符指示受影响的流。而值0表示在整个连接上运行。
- WINDOW_UPDATE可以由发送带有END_STREAM标志的帧的对等体发送。这意味着接收器可以在“半封闭（远程）”或“关闭”流上接收WINDOW_UPDATE帧。
- 但只能影响两个端点之间传输的DATA数据帧。**中介不转发此帧**。但是，任何接收方对数据传输的限制都可能间接导致流控制信息向原始发送方传播。
- 在目前只影响DATA帧。其有效载荷是一个保留位加上无符号31位整数，表示除现有流控制窗口外，发送方可以发送的字节数。流量控制窗口增量的合法范围是1到231-1（2,147,483,647）个八位字节

type: 0x8
```
+-+-------------------------------------------------------------+
|R|              Window Size Increment (31)                     |
+-+-------------------------------------------------------------+
```

##### CONTINUATION
CONTINUATION帧(type=0x9)用于继续传送首部块片段序列。只要前面的帧在同一个流上，而且是一个没有设置END_HEADERS标志的HEADERS帧，PUSH_PROMISE帧，或者CONTINUATION帧，就可以发送任意数量的CONTINUATION帧。
type: 0x9
flag: 
- END_HEADERS
```
+---------------------------------------------------------------+
|                   Header Block Fragment (*)                 ...
+---------------------------------------------------------------+
```

#### 流量控制
HTTP/2中的流量控制功能通过每个流上的发送端各自维持的窗口来实现。流量控制窗口是一个简单的整数值，指出了准许发送端传送的数据的字节数。这样，窗口值衡量了接收端的缓存能力。
存在：
1. 流的流量窗口
2. 连接的流量窗口

- 即使这两种流量控制窗口都没有可用空间了，也可以发送长度为0、设置了END_STREAM标志的帧(即，空的 DATA 帧)。
- 9字节的帧首部不计入流量控制的计算。
- 发送了一个受流量控制影响的帧以后，发送端减少两个窗口的可用空间，减少量为已经发送的帧的长度大小。
- 当帧的接收端消耗了数据并释放了流量控制窗口的空间时，它发送一个WINDOW_UPDATE帧。对于流级别和连接级别的流量控制窗口，需要分别发送WINDOW_UPDATE帧。
- 收到WINDOW_UPDATE帧的发送端要更新相应的窗口，更新量已在WINDOW_UPDATE帧里指定。
- 发送端发送的受流量控制影响的帧和来自于接收端的WINDOW_UPDATE帧之间彼此完全异步。这个属性允许接收端剧烈地更新发送端维持的窗口大小，以防止流中止。
- 新创建流的初始流量控制窗口大小为65，535字节。连接的流量控制窗口也是65，535字节。
- 两端都可以通过组成连接序幕的 SETTING 帧里的 SETTINGS_INITIAL_WINDOW_SIZE 来调整新流的初始流量控制窗口的大小。
- 改变 SETTINGS_INITIAL_WINDOW_SIZE 可以引发流量控制窗口的可用空间变成负值。发送端必须追踪负的流量控制窗口，并且直到它收到了使流量控制窗口变成正值的 WINDOW_UPDATE 帧，才能发送新的受流量控制影响的帧。


## Stream
通常流指的是建立在单个http连接内独立并且双向的序列化的帧集，流有以下特点
- 单个http/2的连接中可以包含多个并行的打开的流，其中任一端点交叉来自多个流的帧。
- 流可以被单方面的使用，也可以由服务器与客户端共享
- 可以被任一端关闭
- 在流上发送帧的顺序非常重要。收件人按照收到的顺序处理框架。特别是，HEADERS和DATA帧的顺序在语义上是重要的。
- 流由整数标识。流标识符由启动流的端点分配给流。通常请求端请求流的streamid为奇数。

```
           
                             +--------+
                     send PP |        | recv PP
                    ,--------|  idle  |--------.
                   /         |        |         \
                  v          +--------+          v
           +----------+          |           +----------+
           |          |          | send H /  |          |
    ,------| reserved |          | recv H    | reserved |------.
    |      | (local)  |          |           | (remote) |      |
    |      +----------+          v           +----------+      |
    |          |             +--------+             |          |
    |          |     recv ES |        | send ES     |          |
    |   send H |     ,-------|  open  |-------.     | recv H   |
    |          |    /        |        |        \    |          |
    |          v   v         +--------+         v   v          |
    |      +----------+          |           +----------+      |
    |      |   half   |          |           |   half   |      |
    |      |  closed  |          | send R /  |  closed  |      |
    |      | (remote) |          | recv R    | (local)  |      |
    |      +----------+          |           +----------+      |
    |           |                |                 |           |
    |           | send ES /      |       recv ES / |           |
    |           | send R /       v        send R / |           |
    |           | recv R     +--------+   recv R   |           |
    | send R /  `----------->|        |<-----------'  send R / |
    | recv R                 | closed |               recv R   |
    `----------------------->|        |<----------------------'
                             +--------+

       send:   endpoint sends this frame
       recv:   endpoint receives this frame

       H:  HEADERS frame (with implied CONTINUATIONs)
       PP: PUSH_PROMISE frame (with implied CONTINUATIONs)
       ES: END_STREAM flag
       R:  RST_STREAM frame
```

#### idle
流开始的阶段。

- 发送或者接受一个HEADERS会导致流从idle变为open状态。相同的HEADERS帧也可以使流立即变为“半封闭”
- PUSH_PROMISE帧只能在已有流上发送，在Promised Stream ID字段中引用新保留的流。导致创建的本地推送流处于"resereved(local)"状态
- 在已有流上接收PUSH_PORMISE帧，导致本地预留一个流处于"resereved(remote)"状态

#### reserved
1. local 
    此状态的流是为了推送而承诺的流。服务器端发送完PUSH_PROMISE帧本地预留的一个用于推送流所处于的状态。在此状态中，终端无法发送除了HEADERS,RSF_STREAM,PRIORITY的其他所有流。

    + 终端发送一个HEADERS帧，这会使其状态变为half-closed(remote)
    + 任何一个终端发送一个RSF_STREAM的帧都会造成其流进入关闭状态
2. remote
    客户端接收到PUSH_PROMISE帧，本地预留的一个用于接收推送流所处于的状态。在此状态中，终端无法发送除了WINDOW_UPDATE,RSF_STREAM,PRIORITY的其他所有流。

    + 终端收到一个HEADERS帧，这会使其状态变为half-closed(local)
    + 任何一个终端发送一个RSF_STREAM的帧都会造成其流进入关闭状态

#### open
两个对等体可以使用处于“打开”状态的流来发送任何类型的帧。在此状态下，发送对等方观察广告的流级流量控制限制。在此状态下，任何终端都可以通过END_STREAM来中断连接，会将连接变为其中一种half-closed状态。如果使用RSF_STREAM，会使其马上变为close状态。

#### half-closed

- 对流量控制窗口不可维护
- 只能接收RST_STREAM、PRIORITY、WINDOW_UPDATE帧
- 终端可以发送任何类型帧，但需要遵守对端的当前流的流量控制限制
- END_STREAM帧导致进入closed状态

1. local
发送包含有END_STREAM标志位帧的一端，流进入half-closed状态，

2. half-closed(remote)
接收到END_STREAM标志位帧的一端，


#### closed
封闭状态，端点绝不能在封闭流上发送PRIORITY以外的帧。接收到RST_STREAM后接收除PRIORITY之外的任何帧的端点必须将其视为STREAM_CLOSED类型的流错误。

- 只允许发送PRIORITY，对依赖关闭的流进行重新排序
- 终端接收RST_STREAM帧之后，只能接收PRIORITY帧，否则报STREAM_CLOSED流错误
- 接收的DATA/HEADERS帧包含有END_STREAM标志位，在一个很短的周期内可以接收WINDOW_UPDATE或RST_STREAM帧；超时后需要作为错误对待
- 终端必须忽略WINDOW_UPDATE或RST_STREAM帧
- 终端发送RST_STREAM帧之后，必须忽略任何接收到的帧
- 在RST_STREAM帧被发送之后收到的流量受限DATA帧，转向流量控制窗口连接处理。尽管这些帧可以被忽略，因为他们是在发送端接收到RST_STREAM之前发送的，但发送端会认为这些帧与流量控制窗口不符。
- 终端在发送RST_STREAM之后接收PUSH_PROMISE帧，尽管相关流已被重置，但推送帧也能使流变成“保留”状态。因此，可用RST_STREAM帧关闭一个不想要的承诺流

#### stream identifier
- 31个字节表示无符号的整数，1~2^31-1
- 客户端启动一个流必须从奇数流标识开始。服务端从偶数标识开始。
- 如果流的标识符是0x0, 此流用于连接控制信息，同时无法用于建立新流。
- 如果流标识符是0x1, 表示http/1.1通过upgrade升级成http/2，从HTTP / 1.1升级的客户端不能选择流0x1作为新的流标识符。
- 新建立的流的标识符必须在数值上大于发起端点已打开或保留的所有流。
- 新建流第一次被使用时，低于此标识符的并且处于空闲"idle"状态的流都会被关闭
- 已使用的流标识符不能被再次使用
- 长期连接可能导致端点耗尽可用的流标识符范围
    - 客户端: 可以通过新的连接建立新流
    - 服务端: 发送GOWAY帧，强制客户端打开新连接

#### stream concurrency
- 对等体可以使用SETTINGS帧内的SETTINGS_MAX_CONCURRENT_STREAMS参数限制并发活动流的数量。 
- 对等方接受并遵循最大流约定。
- 状态为open或者half closed计入总数
- 状态为reserved不计入总数
- 终端接收到HEADERS帧导致创建的流总数超过限制，需要响应PROTOCOL_ERROR或REFUSED_STREAM错误
- 终端想降低SETTINGS_MAX_CONCURRENT_STREAMS设置的活动流的上限，若低于当前已经打开流的数值，可以选择关闭超过新值的流或允许流完成。

#### Stream priority
- 客户端可以通过HEADER头部指定流的优先级在流打开的时候
- 在任何时候都可以通过PRIORITY帧来改变流的优先级
- 可以将流标记为依赖于其他流来确定流的优先级，为每个依赖项分配一个相对权重，该数字用于确定分配给依赖于相同流的流的可用资源的相对比例

#### Stream dependency
- 所有流默认依赖于0x0的流，0x0的流表示为根节点。推送流依赖于传输PUSH_PROMISE的关联流。
- 对当前不在树中的依赖，如处于'idel'状态的流，此流为默认优先级
- 对于依赖同一个流的子节点，可以被指定权重值，同时子节点顺序不是固定的
```
    A                 A
   / \      ==>      /|\
  B   C             B D C
```
- 设定专属标识(exclusive flag)为现有依赖插入一个水平依赖关系，其父级流只能被插入新流所依赖。
```
                      A
    A                 |
   / \      ==>       D
  B   C              / \
                    B   C
```
- 在依赖树内，只有当上级流被关闭的情况下子依赖流才能分配到资源。
- 流无法依赖自身
- 所有流权重控制范围在1-256，在相同父级上按照权重分配资源。如果流B与流C同样为A的子流，流B权重为4，流C权重为12，则当A结束之后，流B只能分配到流C的三分之一量的资源。
- 依赖模型中，父节点优先级以及专属依赖流的加入都会导致优先级重新排序
```
    ?                ?                ?                 ?
    |               / \               |                 |
    A              D   A              D                 D
   / \            /   / \            / \                |
  B   C     ==>  F   B   C   ==>    F   A       OR      A
     / \                 |             / \             /|\
    D   E                E            B   C           B C F
    |                                     |             |
    F                                     E             E
               (intermediate)   (non-exclusive)    (exclusive)
```

#### Prioritization State Management
- 从依赖关系树中删除流时，可以移动其依赖关系以依赖于关闭流的父级。通过基于其依赖性的权重按比例分配闭合流的依赖性的权重来重新计算新依赖性的权重。
- 保留未计入SETTINGS_MAX_CONCURRENT_STREAMS设置的限制的流的优先级信息可能会给端点带来很大的状态负担。
- 流可能会关闭，而创建对该流的依赖的优先级信息正在传输中。如果依赖项中标识的流没有关联的优先级信息，则为依赖流分配默认优先级。这可能会产生次优的优先级，因为流可以被赋予与预期不同的优先级。为了避免这些问题，端点应该在流关闭后的一段时间内保留流优先级状态。保留的状态越长，为流分配错误或默认优先级值的可能性就越小。

#### Default Priorities
最初为流0x0分配了所有流的非排他依赖性。其默认的优先级分配为16。这时可以变成其它流的父节点，可以指派新的优先级值

## HTTP Message Exchange
客户端通过一个预先未被使用的新流传递http request请求，而服务端在相同的流上返回http response。一个典型的http请求/响应包括
1. (only response)零个或多个HEADERS帧（每个帧后跟零个或多个CONTINUATION帧）包含信息（1xx）HTTP响应的消息头
2. 或者一个HEADERS帧包含着完整的头部信息
3. 零或多个包含载荷信息的数据帧
4. (option)拖尾首部字段： 一个HEADERS帧，后面跟随零个或多个包含有报尾（trailer-part）的CONTINUATION帧
序列的最后一帧携带END_STREAM标记，注意一个携带了END_STREAM标记的HEADERS帧后面可以跟多个CONTINUATION帧来携带首部块的其余部分。

无论是来自哪个流的其它的帧都不能出现在 HEADERS 帧和后面可能跟随的 CONTINUATION 帧之间。

HEADERS帧 (及其相关的CONTINUATION帧)只能出现在一个流的开始或结尾处。一个终端在接收到一个最终的 (final) (非信息性的)状态码之后，接收到了一个没有设置END_STREAM的HEADERS帧，则它必须将对应的请求或响应当作是已损坏的。

一个HTTP请求/响应交换完全消耗一个流。一个请求以一个HEADERS帧开始，而将流放进“打开”状态。请求以一个携带了END_STREAM的帧结束，而使得流对于客户端变为"half-closed(local)"，对于服务器变为"half-closed (remote)"。一个响应以一个HEADERS帧开始，以一个携带了END_STREAM的帧结束，而将流放进”closed”状态。

一个HTTP响应完成指的响应帧是包含有END_STREAM标志，在服务器发送并且客户端接收成功。若响应不依赖于客户端的请求，服务器端可以在先于客户端发送请求之前发送完成，之后服务器通过再次发送一个RST_STREAM流（错误代码为NO_ERROR）请求客户端放弃发送请求。这要求客户端在接收到RST_STREAM帧后必须不能够丢弃响应，无论是处于什么原因。

#### 例子
服务端:
1. SETTINGS：服务端设置连接参数
2. WINDOW_UPDATE: 服务端应答更新窗口大小
3. SETTINGS：认同客户端SETTING的ACK
4. HEADERS(streamid=1): 响应头
5. DATA(END_STREAM=1)：数据帧，只有一帧，结束流
6. HEADERS(streamid=3): 响应头
7. DATA
...

浏览器: 
1. SETTINGS: 客户端设置连接参数
2. WINDOW_UPDATE：客户端发送流控窗口大小
3. HEADERS(index.html streamid=1):请求头
4. HEADERS(main.css streamid=3):请求头

## HTTP2 首部字段
1. ASCII值表示 
2. 字段要求在http编码之前编译为小写
3. http2提供伪首部, 通过':'定义，其不属于http首部，只在定义它们的上下文有效。伪首部字段 必须出现在首部块中普通的首部字段之前。
4. 连接属性专用字段（Connection-Specific Header Fields）不再被使用（但Transfer-Encoding可以允许出现在请求头中），比如Keep-Alive, Proxy-Connection, Transfer-Encoding和Upgrade等

#### 伪首部
伪首部的概念被应用，替换了http1.x的请求行与响应行，只在本地的上下文进行表示，实际上发送时通过HPACK算法的索引表进行发送。
- :method
- :scheme :不限于http和https计划的URI。代理或网关可以转换对非HTTP方案的请求，从而允许使用HTTP与非HTTP服务进行交互。
- :authority :目标URI的权限部分
- :path
- :status

HTTP/2没有定义携带HTTP/1.1请求行里包含的版本号的方法。
对于HTTP/2响应，只定义了一个:status伪首部字段，其携带了HTTP状态码(参见 [RFC7231]，第6章 )。所有的响应都必须包含该伪首部字段。

为了更好的压缩效率，可以将Cookie首部字段拆分成单独的首部字段，每一个都有一个或多个cookie对。如果解压缩后有多个Cookie首部字段，在将其传入一个非HTTP/2的上下文(比如：HTTP/1.1连接，或者通用的HTTP服务器应用)之前，必须使用两个字节的分隔符0x3B，0x20(即ASCII字符串"; ")将这些Cookie首部字段连接成一个字符串。

比较一下http1.1中与http2中request/response的不同

需要注意的是，组成任何给定首部字段的数据可以在首部块片段之间散布。本例帧中首部字段的分配仅仅是示例性的。

简单的图片请求:
```
GET /resource HTTP/1.1           HEADERS
  Host: example.org          ==>     + END_STREAM
  Accept: image/jpeg                 + END_HEADERS
                                       :method = GET
                                       :scheme = https
                                       :path = /resource
                                       host = example.org
                                       accept = image/jpeg

```

大于16字节的报头:
```
 POST /resource HTTP/1.1          HEADERS
  Host: example.org          ==>     - END_STREAM
  Content-Type: image/jpeg           - END_HEADERS
  Content-Length: 123                  :method = POST
                                       :path = /resource
  {binary data}                        :scheme = https

                                   CONTINUATION
                                     + END_HEADERS
                                       content-type = image/jpeg
                                       host = example.org
                                       content-length = 123

                                   DATA
                                     + END_STREAM
                                   {binary data}
```

```
  HTTP/1.1 200 OK                  HEADERS
  Content-Type: image/jpeg   ==>     - END_STREAM
  Content-Length: 123                + END_HEADERS
                                       :status = 200
  {binary data}                        content-type = image/jpeg
                                       content-length = 123

                                   DATA
                                     + END_STREAM
                                   {binary data}

```

请求或响应和所有的 DATA 帧被发送完之后，拖尾的首部字段被当做一个首部块发送。开始拖尾首部块的 HEADERS 帧设置了 END_STREAM 标志:
```
HTTP/1.1 100 Continue            HEADERS
 Extension-Field: bar       ==>     - END_STREAM
                                    + END_HEADERS
                                      :status = 100
                                      extension-field = bar

 HTTP/1.1 200 OK                  HEADERS
 Content-Type: image/jpeg   ==>     - END_STREAM
 Transfer-Encoding: chunked         + END_HEADERS
 Trailer: Foo                         :status = 200
                                      content-length = 123
 123                                  content-type = image/jpeg
 {binary data}                        trailer = Foo
 0
 Foo: bar                         DATA
                                    - END_STREAM
                                  {binary data}

                                  HEADERS
                                    + END_STREAM
                                    + END_HEADERS
                                      foo = bar
```

如果请求expect中包含101-continue标识，那么应在其响应里发送一个100(Continue)状态码:
```
 HTTP/1.1 100 Continue            HEADERS
 Extension-Field: bar       ==>     - END_STREAM
                                    + END_HEADERS
                                      :status = 100
                                      extension-field = bar

 HTTP/1.1 200 OK                  HEADERS
 Content-Type: image/jpeg   ==>     - END_STREAM
 Transfer-Encoding: chunked         + END_HEADERS
 Trailer: Foo                         :status = 200
                                      content-length = 123
 123                                  content-type = image/jpeg
 {binary data}                        trailer = Foo
 0
 Foo: bar                         DATA
                                   - END_STREAM
                                  {binary data}

                                  HEADERS
                                    + END_STREAM
                                    + END_HEADERS
                                      foo = bar
```

#### 请求可靠机制
在HTTP/1.1里，发生错误时，HTTP客户端不能重试一个非幂等的请求，因为没有办法判定错误的性质。相比错误，一些服务端可能优先处理已经发生的请求，如果重试发生错误的请求，可能会导致不可预料的影响。
对于请求尚未被处理的客户端，HTTP/2提供了两种判断机制：
- GOAWAY帧会携带上流标识符的最大值，低于此值的请求已经被执行过，高于此值的请求帧，则可以再次放心重试
- 包含有REFUSED_STREAM错误代码的RST_STREAM帧说明当前流早于任何处理发生之前就已经被关闭，因此发生在当前流上的请求可以安全重试。

## 首部压缩和解压缩
与HTTP1.x一致，HTTP/2中也是一个键具有一个或者多个值。这些首部字段用于HTTP请求和响应消息，也用于服务端推送操作。

- 首部列表是零个或多个首部字段的集合。
- 通过连接传送时，首部列表被HTTP首部压缩(HPack)序列化成首部块。
- 序列化的首部块又被划分成一个或多个叫做首部块片段的字节序列
- 通过HEADERS, PUSH_PROMISE, CONTINUATION帧的有效载荷发送，因为这些帧携带的数据能修改并维护端的上下文。
- cookie需要特殊对待
- 接受端连接片段重组首部块，然后解压重建首部列表

完整的首部块包括两者之一:
- 一个单独的、设置了END_HEADERS标识的HEADERS或PUSH_PROMISE帧
- 一个单独的、清除了END_HEADERS标识的HEADERS或PUSH_PROMISE帧，并有一个或多个 CONTINUATION帧，且最后一个CONTINUATION帧设置了END_HEADERS标识。

首部块被当做离散单元处理，但是首部块必须作为连续的帧序列发送。而没有交错任何其他类型或其他流的帧。(通过END_HEADERS标识实现)

## HPACK
#### 格式
首部字段列表视为可包含重复对的名-值对的有序集合。名字和值被认为是不透明的字节序列，并且头部字段的顺序在压缩和解压后被保留。

编码通过将头部字段映射到索引值的头部字段表来获知。这些头部字段表可以随着新头部字段的编码和解码而增量地更新。

字面量值直接编码或使用一个静态Huffman码。

编码器负责决定将哪些头部字段作为新条目插入头部字段表。解码器执行由编码器规定的对头部字段表的修改，及重建头部字段列表的过程。这使解码器保持简单，并能与各种编码器的互操作。

#### 术语
- 头部字段： key-value。不透明的字节序列
- 动态表： 将存储的头部字段和索引值关联起来的表。动态的，特定于编码或者上下文
- 静态表： 静态地将频繁出现的头部字段与索引值关联起来的表。有序的，只读的，总是可访问的。且可以在所有的编码或解码上下文间共享的。
- 头部列表： 联合编码的头部字段可出现重复的有序头部字段集合。一个HTTP/2 HEADER 中包含的完整的头部字段列表是一个头部列表。
- 头部字段表示： 头部字段可以以字面量或索引的编码形式表示
- 头部块： 头部字段表示的有序列表，当解码时，产生完整的头部列表。

#### 压缩过程
HPACK保留头部列表中头部字段的顺序。

- 编码器： 根据头部字段在原始头部列表中的顺序在头部块中对头部字段表示进行排序
- 解码器： 根据头部字段表示在头部块中的顺序在解码的头部列表中对头部字段进行排序。

解压缩头部块的时候，解码器只需维护作为解码上下文。

当被用于双向通信时，比如在HTTP中，编码和解码动态表完全独立地由一端维护，比如，请求和响应的动态表是分开的。

#### 索引表
通过两个表将索引与头部字段关联，静态表是预定义的，包含常用的字段。动态表是动态的，可被编码器用于在编码的头部列表中重复地索引头部字段。两个表组合起来，用于定义单个索引值的地址空间。

```
<----------  Index Address Space ---------->
<-- Static  Table -->  <-- Dynamic Table -->
+---+-----------+---+  +---+-----------+---+
| 1 |    ...    | s |  |s+1|    ...    |s+k|
+---+-----------+---+  +---+-----------+---+
                       ^                   |
                       |                   V
                Insertion Point      Dropping Point

```

#### 静态表
预定义的头部静态列表

#### 动态表
动态表由以FIFO顺序维护的头部字段列表组成。动态表中第一个且最新的条目索引值最低，动态表最旧的条目索引值最高。

每个动态表只针对一个连接，每个连接的压缩解压缩的上下文有且仅有一个动态表。

- 动态表本来是空的，条目随着解压头部块过程增加
- 动态表可以包含重复的条目
- 编码器决定如何更新动态表，并因此可以控制动态表使用多少内存。
- 处理 头部字段表示时更新动态表

##### 动态表管理
动态表条目大小是它名字的字节长度及它值的字节长度的总和，外加32(预估开销，如果条目结构使用两个64位的指针引用条目的名字和值，及两个64位整数来计数引用了名字和值的引用数目，这将有32字节的开销。

任何条目的大小使用未应用Huffman编码条件下，它的名字和值的长度来计算。

##### 最大表大小
通过SETTINGS_HEADER_TABLE_SIZE管理。无论何时动态表最大大小减小，将条目从动态表的结尾处逐出，直到动态表的大小小于或等于最大大小。

## 原始数据类型表示
HPACK算法表示的对象，主要有原始数据类型的整型值和字符串，头部字段，以及头部字段列表。

HPACK编码使用两种原始数据类型：无符号的变长整数和字节串

#### 整数
整数用于表示
- 名字索引
- 头部字段索引
- 字符串长度。

整数由两部分表示
- 填满当前字节的前缀
- 在前缀不足以表示整数时的一个可选字节列表

为HPACK希望每一个整数的表示能够从某个8比特位字节(octet，下文将其简写为"字节")中的任何一个比特位开始，但总是要在某个字节的最后一个比特位结束。

如果小于前缀，则直接通过前缀表示。
```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| ? | ? | ? |       Value       |
+---+---+---+-------------------+

```
如果大于前缀，则前缀表示2^n-1的数，剩下的数通过剩余帧表示，剩余帧中首位通过0与1表示是否是结束。

```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| ? | ? | ? | 1   1   1   1   1 |
+---+---+---+-------------------+
| 1 |    Value-(2^N-1) LSB      |
+---+---------------------------+
               ...
+---+---------------------------+
| 0 |    Value-(2^N-1) MSB      |
+---+---------------------------+
```

要由字节列表解码整数值，首先需要将列表中的字节顺序反过来。然后，移除每个字节的最高有效位。连接字节的剩余位，再将结果加2^N-1获得整数值。

如果我们用3个bit标识前置位
```
5 = ?????101
(101b = 5)
8 = ?????111 00000001
(111b + 1 = 8)
135 = 7 + 128 = ?????111 10000000 00000001
(111b + 0 + 128 * 1 = 135)
```

#### 字符串
```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| H |    String Length (7+)     |
+---+---------------------------+
|  String Data (Length octets)  |
+-------------------------------+
```

- H: 表示是否是huffman编码，1为是
- StringLength: 表示随后跟随的字符串的长度，用上述的整数编码方式编码
- StringData： 如果是 huffman 编码，则使用 huffman 编码后的字符串，否则就是原始串。

##### huffman编码
根据字符出现的概率重新编排字符的二进制代码，从而压缩概率高的字符串进而压缩整个串的长度。这里的 huffman 编码是静态的，是根据过去大量的 Http 头的数据从而选出的编码方案。

#### 二进制编码压缩

#### 已被索引的头部
```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| 1 |        Index (7+)         |
+---+---------------------------+
```

第一个bit为1，随后紧跟一个 无整数的编码表示 Index，即为静态表或者是动态表中的索引值。

#### name在索引，value不在索引且允许保存
```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| 0 | 1 |      Index (6+)       |
+---+---+-----------------------+
| H |     Value Length (7+)     |
+---+---------------------------+
| Value String (Length octets)  |
+-------------------------------+
```

第一个字节的前两个bit为01，随后无符号整数编码Index表示name的索引。下面紧随一个字面字符串的编码，表示value 。这个 Header 会被两端都加入动态表中。

#### name和value都没被索引而且允许保存
```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| 0 | 1 |           0           |
+---+---+-----------------------+
| H |     Name Length (7+)      |
+---+---------------------------+
|  Name String (Length octets)  |
+---+---------------------------+
| H |     Value Length (7+)     |
+---+---------------------------+
| Value String (Length octets)  |
+-------------------------------+
```
第一个字节为01000000, 然后紧随2个字面字符串的编码表示。
这个 Header 会被两端都加入动态表中。

#### name被索引， value未索引且不保存
```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| 0 | 0 | 0 | 0 |  Index (4+)   |
+---+---+-----------------------+
| H |     Value Length (7+)     |
+---+---------------------------+
| Value String (Length octets)  |
+-------------------------------+
```

#### name与value都未被索引且不保存
```
    0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| 0 | 0 | 0 | 0 |       0       |
+---+---+-----------------------+
| H |     Name Length (7+)      |
+---+---------------------------+
|  Name String (Length octets)  |
+---+---------------------------+
| H |     Value Length (7+)     |
+---+---------------------------+
| Value String (Length octets)  |
+-------------------------------+
```

#### name在索引表中， value不在，且绝对不允许被索引
```
0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| 0 | 0 | 0 | 1 |  Index (4+)   |
+---+---+-----------------------+
| H |     Value Length (7+)     |
+---+---------------------------+
| Value String (Length octets)  |
+-------------------------------+
```
这个和之前仅仅在于，中间是否通过了代理。如果没有代理那么表现是一致的。如果中间通过了代理，协议要求代理必须原样转发这个 Header 的编码，不允许做任何修改，这个暗示中间的代理这个字面值是故意不压缩的，比如为了敏感数据的安全等。而之前则允许代理重新编码等。

#### name 和 value 都不在索引表中，且绝对不允许被索引
```
 0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| 0 | 0 | 0 | 1 |       0       |
+---+---+-----------------------+
| H |     Name Length (7+)      |
+---+---------------------------+
|  Name String (Length octets)  |
+---+---------------------------+
| H |     Value Length (7+)     |
+---+---------------------------+
| Value String (Length octets)  |
+-------------------------------+
```

#### 修改动态表的大小
```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| 0 | 0 | 1 |   Max size (5+)   |
+---+---------------------------+
```

## 栗子
有一个编码的header如下
```
:method: GET
:scheme: http
:path: /
:authority: www.example.com
```

已被索引的头部：
:method: GET -> 静态表Index = 2 -> 10000010 -> 82
:scheme: http -> 静态表Index = 6 -> 10000110 -> 86 
:path: / -> 静态表Index = 4 -> 10000100 -> 84
:authority(name被索引，值未) -> 静态表 = 1 -> 01000001 -> 41
之后一个字符串类型的解码解析:authority对应的value，使用huffman编码，其字符串长度为12所以得到 -> 10001100 = 8c
接着解析12个字节为 huffman 编码后的字符f1e3 c2e5 f23a 6ba0 ab90 f4ff

编码之后16进制如下
```
8286 8441 8cf1 e3c2 e5f2 3a6b a0ab 90f4 ff     
```

## Server Push
HTTP/2允许服务端抢先向客户端发送(或『推送』)响应(以及相应的『被允诺的』请求)，这些响应跟先前客户端发起的请求有关。为了完整地处理对最初请求的响应，客户端将需要服务端推送的响应，当服务端了解到这一点时，服务端推送功能就会是有用的。

客户端可以要求关闭服务端推送功能，但这是每一跳独立协商地。SETTINGS_ENABLE_PUSH 设置为0，表示关闭服务端推送功能。

被允诺的请求必须是可缓存的，必须是安全的，而且不能包含请求体。如果客户端收到的被允诺的请求是不可缓存的，不安全的，或者含有请求体，那么客户端必须使用类型为 PROTOCOL_ERROR 的流错误来重置该被允诺的流。注意，如果客户端认为一个新定义的方法是不安全的，这将会导致被允诺的流被重置。

服务端必须在:authority伪首部字段中包含一个值，以表明服务端是权威可信的。客户端必须将不可信的服务端的 PUSH_PROMISE 帧当做类型为 PROTOCOL_ERROR 的流错误。

中介可以接收来自服务端的推送，并选择不向客户端转发这些推送。换句话说，怎样利用被推送来的信息取决于该中介。同样，中介可以选择向客户端做额外的推送，不需要服务端采取任何行动。

#### 推送请求
服务端推送在语义上等价于服务端对请求的响应。请求也被服务端以 PUSH_PROMISE 帧的形式发送。

PUSH_PROMISE帧包含一个首部块，该首部块包含完整的一套请求首部字段，服务端将这些字段归因于请求。被推送的响应总是和明显来自于客户端的请求有关联。服务端在该请求所在的流上发送 PUSH_PROMISE帧。

PUSH_PROMISE帧也包含一个被允诺的流标识符，该流标识符是从服务端可用的流标识符里选出来。服务端必须在:method伪首部字段里包含一个安全的可缓存的方法。在发送任何跟被允诺的响应有关联的帧之前，服务端应该优先发送PUSH_PROMISE帧。

这就避免了一种竞态条件，即客户端在收到任何 PUSH_PROMISE 帧之前就发送请求。

发送完PUSH_PROMISE帧，服务器需要马上发送具体DATA数据帧

客户端接收完PUSH_PROMISE帧后，选择接收PUSH响应内容，这期间不能触发请求承诺的响应内容，直到承诺流关闭

客户端可以通过设置SETTINGS_MAX_CONCURRENT_STREAMS限制响应数，值为0禁用。但不能阻止服务器发送PUSH_PROMISE帧

客户端不需要接收推送内容时，可以选择发送RST_STREAM帧，包含CANCEL/REFUSED_STREAM代码，以及PUSH流标识符发送给服务器端，重置推送流

PUSH_PROMISE 流的响应以 HEADERS 帧开始，这会立即将流在服务端置于 半关闭(远端)(half-closed(remote)) 状态，在客户端置于 半关闭(本地)(half-closed(local)) 状态，最后以携带 END_STREAM 的帧结束，这会将流置于 关闭(closed) 状态。

例如，如果服务端收到了一个对文档的请求，该文档包含内嵌的指向多个图片文件的链接，且服务端选择向客户端推送那些额外的图片，那么在发送包含图片链接的 DATA 帧之前发送 PUSH_PROMISE 帧可以确保客户端在发现内嵌的链接之前，能够知道有一个资源将要被推送过来。同样地，如果服务端推送被首部块引用的响应(比如，在链接的首部字段里)，在发送首部块之前发送一个 PUSH_PROMISE 帧，可以确保客户端不再请求那些资源。

#### connect方法
在http1.x里，CONNECT伪方法用于将远端主机的HTTP连接转换为隧道。CONNECT主要用于HTTP代理和源服务器建立TLS会话，其目的是和https资源交互。

在HTTP/2中,CONNECT方法用于在一个到远端主机的单独的HTTP/2流之上建立隧道。HTTP首部字段映射以像在请求的伪首部字段定义的那样起作用，但有一些不同：
- :method 伪首部字段设置为 CONNECT
- 必须忽略:scheme 和 :path伪首部字段。
- :authority伪首部字段包含要连接的主机和端口

## 连接管理
http/2是持久连接，对于给定的一对主机和端口，客户端应该只打开一个HTTP/2连接，其中主机或者来自于一个URI ，一种选定的可替换的服务[ALT-SVC]或者来自于一个已配置的代理。

客户端可以创建额外的连接作为替补，或者用来替换将要耗尽可用流标识符空间的连接，为一个 TLS 连接刷新关键资料，或者用来替换遇到错误的连接。

客户端可以和相同的IP地址和TCP端口建立多个连接，这些地址和端口使用不同的服务器名称指示值，或者提供不同的 TLS 客户端证书。但是应该避免使用相同的配置创建多个连接。

鼓励服务端尽可能长地维持打开的连接，但是准许服务端在必要时可以关闭空闲的连接。当任何一端选择关闭传输层 TCP 连接时，主动关闭的一端应该先发送一个 GOAWAY 帧，这样两端都可以确定地知道先前发送的帧是否已被处理

#### 连接复用
对于连接到源服务器的连接，无论是直接或者通过CONNECT方法连接，都可能重复使用于多个不同URL权限请求组件。只要源服务器是权威的，连接就可以被复用。如果没有TLS的TCP连接，取决于同一个IP地址已经解析的主机端。

对于HTTP资源，连接复用还额外取决于是否拥有一个对于URL中的主机已经验证过的证书。该证书由服务器提供,在URL主机中形成一个新的TLS连接时客户端将执行任何检查。

源服务器可能提供一个拥有多个subjectAltName属性或者通配符的证书，其中之一是有效授权的URL。例如：一个带有*.example.com的subjectAltName证书将允许a.example.com和b.example.com使用同一个连接。