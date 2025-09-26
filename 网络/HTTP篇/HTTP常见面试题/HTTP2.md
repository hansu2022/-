![[Pasted image 20250925175621.png]]
HTTP/2 在性能上的核心改进，旨在解决 HTTP/1.1 中存在的效率问题，特别是队头阻塞和冗余传输。

|改进点|核心机制|效果|解决的 HTTP/1.1 问题|
|---|---|---|---|
|**头部压缩**|HPACK 算法（头信息表和索引号）|减少头部传输数据量，提高速度。|头部冗余和重复传输。|
|**二进制格式**|帧（Frame）结构，头信息帧/数据帧|计算机直接解析，提高数据传输效率。|文本报文解析效率低，浪费字节。|
|**并发传输**|**Stream** 多路复用，乱序发送，有序组装|解决了 HTTP 层面上的队头阻塞问题。|HTTP/1.1 串行处理请求的队头阻塞。|
|**服务器推送**|服务器主动推送客户端需要的资源|减少消息往返次数（Round Trip Time, RTT）。|客户端多次请求依赖资源。|
### 1. 头部压缩（Header Compression）

**核心机制：** **HPACK** 算法。
- 客户端和服务器维护一张**头信息表**（包含**静态表**和**动态表**）。
- 将重复或相似的头部字段存入表中，并生成**索引号**。
- 后续传输时，**只发送索引号**而非完整的字段名和值。
- 对于**动态**表，可以添加新的键值对；对于**静态**表，则包含了常见的字段和值（如`:status: 200 ok`的编码是 `8`）。
- **效益：** 显著减少了头部数据的大小，特别是对于多个请求头部相似的情况。
### 2. 二进制格式（Binary Framing）

**核心机制：** 将所有的报文（包括头部和数据体）都采用**二进制格式**传输，并统一称为**帧（Frame）**。
- **帧的种类：** 主要分为**头信息帧（Headers Frame）**和**数据帧（Data Frame）**。 
- **对计算机友好：** 计算机无需将明文报文再转换为二进制，可以直接解析二进制帧，**提高了处理和传输效率**。
- **数据量减少：** 例如，状态码 `200` 在 HTTP/1.1 中需要 3 个字节（`'2','0','0'`），在 HTTP/2 中可能只需要 1 个字节（如 `10001000`）。

![HTTP/1 与 HTTP/2](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/%E7%BD%91%E7%BB%9C/http2/%E4%BA%8C%E8%BF%9B%E5%88%B6%E5%B8%A7.png)

这样虽然对人不友好，但是对计算机非常友好，因为计算机只懂二进制，那么收到报文后，无需再将明文的报文转成二进制，而是直接解析二进制报文，这**增加了数据传输的效率**。

比如状态码 200 ，在 HTTP/1.1 是用 '2''0''0' 三个字符来表示（二进制：00110010 00110000 00110000），共用了 3 个字节，如下图

![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/%E7%BD%91%E7%BB%9C/http2/http1.png)

在 HTTP/2 对于状态码 200 的二进制编码是 10001000，只用了 1 字节就能表示，相比于 HTTP/1.1 节省了 2 个字节，如下图：

![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/%E7%BD%91%E7%BB%9C/http2/h2c.png)

Header: :status: 200 OK 的编码内容为：1000 1000，那么表达的含义是什么呢？

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/network/http/index.png)

1. 最前面的 1 标识该 Header 是静态表中已经存在的 KV。（至于什么是静态表，可以看这篇：[HTTP/2 牛逼在哪？ (opens new window)](https://xiaolincoding.com/network/2_http/http2.html)）
2. 在静态表里，“:status: 200 ok” 静态表编码是 8，二进制即是 1000。

因此，整体加起来就是 1000 1000。

_3. 并发传输_

### 3. 并发传输（Stream Multiplexing）

**核心机制：** 引入 **Stream**（流）概念，实现了**多路复用**。
- **Stream 定义：** 1 个 **TCP 连接**可以包含**多个 Stream**。
    - **Message**：对应 HTTP/1 中的请求或响应，由头部和包体构成。
    - **Frame**：HTTP/2 的最小单位，以二进制格式存放头部或包体。
        
- **多路复用：** 1. **不同的 HTTP 请求**使用**独一无二的 Stream ID** 来区分。 2. **不同 Stream 的帧**可以在同一条 TCP 连接上**并行交错地**发送。 3. 接收端根据 Stream ID 将乱序到达的帧**有序组装**成完整的 HTTP 消息。
- **效益：** 彻底解决了 **HTTP/1.1 层面**的**队头阻塞**问题（即同一连接上，前一个请求阻塞会导致后续请求无法发送）。

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/%E7%BD%91%E7%BB%9C/http2/stream.png)

从上图可以看到，1 个 TCP 连接包含多个 Stream，Stream 里可以包含 1 个或多个 Message，Message 对应 HTTP/1 中的请求或响应，由 HTTP 头部和包体构成。Message 里包含一条或者多个 Frame，Frame 是 HTTP/2 最小单位，以二进制压缩格式存放 HTTP/1 中的内容（头部和包体）。

比如下图，服务端**并行交错地**发送了两个响应： Stream 1 和 Stream 3，这两个 Stream 都是跑在一个 TCP 连接上，客户端收到后，会根据相同的 Stream ID 有序组装成 HTTP 消息。

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/network/http/http2%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8.jpeg)

_4、服务器推送_

**核心机制：** 服务器可以在客户端未明确请求的情况下，**主动**向客户端推送其可能需要的资源。
- **主动性：** 服务端不再是被动响应，可以主动发起消息。
- **Stream ID 规则：**
    - 客户端建立的 Stream ID 必须是**奇数**号（如请求 HTML 的 Stream 1）。
    - 服务器建立的 Stream ID 必须是**偶数**号（如推送 CSS 的 Stream 2 或 4）。
        
- **效益：** 减少了客户端为了获取依赖资源（如 HTML 依赖的 CSS、JS）而发起的**二次请求**，从而减少了 RTT（往返时延），**加快了页面加载速度**。

![](https://cdn.xiaolincoding.com//mysql/other/83445581dafe409d8cfd2c573b2781ac.png)

再比如，客户端通过 HTTP/1.1 请求从服务器那获取到了 HTML 文件，而 HTML 可能还需要依赖 CSS 来渲染页面，这时客户端还要再发起获取 CSS 文件的请求，需要两次消息往返，如下图左边部分：

![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/%E7%BD%91%E7%BB%9C/http2/push.png)

如上图右边部分，在 HTTP/2 中，客户端在访问 HTML 时，服务器可以直接主动推送 CSS 文件，减少了消息传递的次数。

> HTTP/2 有什么缺陷？

HTTP/2 通过 Stream 的并发能力，解决了 HTTP/1 队头阻塞的问题，看似很完美了，但是 HTTP/2 还是存在“队头阻塞”的问题，只不过问题不是在 HTTP 这一层面，而是在 TCP 这一层。

**HTTP/2 是基于 TCP 协议来传输数据的，TCP 是字节流协议，TCP 层必须保证收到的字节数据是完整且连续的，这样内核才会将缓冲区里的数据返回给 HTTP 应用，那么当「前 1 个字节数据」没有到达时，后收到的字节数据只能存放在内核缓冲区里，只有等到这 1 个字节数据到达时，HTTP/2 应用层才能从内核中拿到数据，这就是 HTTP/2 队头阻塞问题。**

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/network/quic/http2%E9%98%BB%E5%A1%9E.jpeg)

举个例子，如下图：

![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/%E7%BD%91%E7%BB%9C/http3/tcp%E9%98%9F%E5%A4%B4%E9%98%BB%E5%A1%9E.gif)

图中发送方发送了很多个 packet，每个 packet 都有自己的序号，你可以认为是 TCP 的序列号，其中 packet 3 在网络中丢失了，即使 packet 4-6 被接收方收到后，由于内核中的 TCP 数据不是连续的，于是接收方的应用层就无法从内核中读取到，只有等到 packet 3 重传后，接收方的应用层才可以从内核中读取到数据，这就是 HTTP/2 的队头阻塞问题，是在 TCP 层面发生的。

所以，一旦发生了丢包现象，就会触发 TCP 的重传机制，这样在一个 TCP 连接中的**所有的 HTTP 请求都必须等待这个丢了的包被重传回来**。