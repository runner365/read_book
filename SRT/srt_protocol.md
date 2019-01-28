# SRT协议
srt是基于UDT传输协议，是用户级别的协议，其保留UDT的核心思想和机制，但是做了多项改进，包括控制报文的修改，针对直播流改进了流控，改进了拥塞算法，报文加密算法。本文介绍srt协议本身。更多的相关实现在：https://github.com/Haivision/srt

## 简介
srt传输协议为不可靠网络提供安全，可靠的数据传输，如因特网。任何数据都可以在srt协议上传输，特别是对音视频流数据优化最为明显。<br/>
在任何时候，srt都能用于视频流的汇聚/分发节点，提供几乎最好的质量和最低延时的视频。<br/>
当报文作为流在源与目的之间传输，srt对网络进行检测和适应。srt帮助补偿jitter和带宽抖动，针对噪声网络中的拥塞。其错误恢复机制对因特网的丢包影响最小化。srt也支持aes加密来保证端到端的安全。<br/>
SRT是基于UDT协议的(UDP-based data transfer)。虽然UDT是设计用来在公网中高吞吐的报文传输，UDT对视频直播并没有优势。SRT是一个大的改进UDT版本，其对视频直播有大的提升。<br/>
SRT在IP网络中的低延时视频传输，是以mpeg-ts格式作为UDP的单播/多播。其方案对网络提供可靠性，通过使能FEC来减轻任何丢包。在城市间，国家间设置跨洋间的低延时通信都带有更多的挑战。虽然用卫星通信，或MPLS专网来实现，但是价格太贵。公用因特网的连接，虽然便宜很多，但是需要大量的带宽来实现丢包恢复。<br/>
即使UDT并不是为直播流而设计，但是其丢包恢复机制能提供基本的丢包恢复功能。SRT的最早版本包括新报文的重传功能，能对直播流的丢包做快速响应。<br/>
为了达到低延时，SRT不得不引入分时机制。一个流在因特网中传输，很多特效会被完全影响，包括延时/jitter/丢包。进而导致解码问题，音视频解码器不能解码对应时间戳上的未收到的报文。如果应用buffer缓存来避免，但是却会带来延时。<br>

![常规网络情况](https://github.com/runner365/read_book/blob/srt/SRT/pic/net_condition01.png)

SRT的机制在接收方新创建了重要的特性，极大的降低buffer的需要。这些机制是SRT协议自身的一部分，所以一旦报文从SRT的一端发到接收端，流自身状态已经被恢复成流本身的状态。<br/>

![srt网络情况](https://github.com/runner365/read_book/blob/srt/SRT/pic/net_condition02.png)

最初SRT协议又Haivision Systems公司开发，在2017年4月Wowza Media Systems将其开源。开源的SRT遵守MPL-2.0开源协议。选用MPL-2.0协议，因为想在对开源SRT的兼容性，和估计开源社区去改进SRT协议之间做好平衡。任何第三方开发者都能自由的使用SRT开源代码。但是如果他们修改和优化代码，就必须把这些优化代码提交到开源社区。<br/>
在2017年4月，Haivision和Wowza公司成立了SRT联盟www.srtalliance.org，致力于持续发展该协议。

## SRT的UDT4适配
UDT是一种ARQ(自动重传请求)协议。其应用的是ARQ的第三种演进方案(选择性重传)。 UDT4的应用在ietf中提出，在draft-gg-udt-03中。<br/>
UDT致力于容量的最大使用，也就是当发送数据的时候，应用必须保证输入buffer是足够的。当以确定的比特率发送实时视频，包的发送速度对比读取文件的速度是非常慢的。buffer耗尽会导致一些UDT算法的复位。也就是说，当拥塞发生，发送方的算法会挂住UDP API，也会把报文放入丢弃队列，这样新的报文就不能被处理。实时的视频不能被挂住的，所以报文可能被应用丢弃(因为发送API会被block住)。不像UDT，SRT对实时报文，重传报文，丢失或要丢弃的旧豹纹都共享当前的带宽。<br/>
SRT的早期开发就引入了大量针对UDT version4的改进: <br/>
* 基于字节统计
* 基于毫秒单位来做延时控制(测量buffer的时间单位，更好的适配延时，使其不受流比特率的影响)
* ACK消息的统计信息(接受速率和估计连接容量)
* 控制报文时间戳(这个在UDT的draft中有，但是不在UDT4的应用中)
* 时间戳漂移更正算法
* 周期的NAK报告(UDT4去使能这个特性，用于unACKed报文的超时重传，其对实时流应用会消耗过多的带宽)
* 基于时间戳的报文发送(可配置的延时)
* SRT的握手是基于UDT自定义的控制报文，用于交换点到点间的配置和应用信息，以确保无缝升级，当升级协议和保证向后兼容
* 基于配置和输入速率的最大输出速率
* 加密优化
早期SRT的开发，是在内网用Haivison的Makito X系列的编解码器，其能模拟包的丢弃。<br/>
当中间报文的丢失，会导致解码器的接收buffer耗尽(没有报文送去解码)。丢失的报文不能及时的重传。解决方法是更早的触发未确认收到报文的重传(ARQ的自动重传)。然而，这会导致带宽使用的迸发。基于超时重传的发生是当报文没有很快的得到ack确认。在UDT4中，重传发生只是当丢失表空的时候。<br/>
重传所有ack超时的报文，能解决这样的实验场景，当无网络拥塞，有随机丢包影响报文传输。在多次重传后，所有的报文都应该能被发送，接收者的队列不会被挂住。但这会消耗大量的带宽，因为没有加入速率控制。

## 包结构
SRT保有UDT的UDP报文结构，但是有很多改进。基于UDT IETF internet Draft(draft-gg-udt-03.txt)的细节。

### 数据和控制报文
每个承载SRT的UDP报文都带有SRT头(其紧跟在UDP头后面)。在所有的协议版本中，SRT头包含4个32bits字段：
* PH_SEQNO
* PH_MSGNO
* PH_TIMESTAMP
* PH_ID
SRT有两类报文，PH_SEQNO字段的第一个bit用来标识报文类型，0是数据报文，1是控制报文。如下的例子中，就是数据报文的例子，其'packet type' bit=0：<br/>
注意：在SRT version 1.3中的报文结构变化。为了提高早期版本的适应能力，新旧报文格式都在下面列出(大字节序)。<br/>

![srt_data_packet](https://github.com/runner365/read_book/blob/srt/SRT/pic/srt_data_packet.png)

* FF=(2bits) 如报文中的位置
<pre>
  1) 10b = 1st
  2) 00b = middle
  3) 01b = last
  4) 11b = single
</pre>
* O = (1bit) 表示消息是否按序发送，按序1，不按序0。在文件/消息模式下(传统的UDT)，当该bit为0，那么后面的消息是可以立刻发送而不用等待前面消息的发送完成。但是这在直播业务中是没有用的，因为当TSBPD模式开启时，为数据抽取而服务器的功能完全不同。
* KK=(2bits)表示是否数据已经被加密：
<pre>
  1) 00b: 不加密
  2) 01b: 加密，偶数key
  3) 10b: 加密，奇数key
</pre>
* R=(1bit)重传报文。如果是0，表示该报文是第一次发送；如果是1，就是重传报文。
在数据报文中，第3，4字段如下的定义：
* Timestamp: 报文时间戳
* ID: 报文分发的目的socket id，如果是connect的发起报文，该字段就是0

更多数据报文的细节在本文后面介绍。 <br/>
<br/>

SRT协议控制报文头("packet type" bit=1)，其结构如下(未包含udp头)：<br/>

![srt_data_packet](https://github.com/runner365/read_book/blob/srt/SRT/pic/srt_control_packet.png)

对于控制报文，头两个字段分别解释如下：
* 头32bit：
<pre>
  1) bit0: 类型，1就是控制报文
  2) bits1-15: 消息类型
  3) bits16-31: 消息扩展类型
  -----------------------------------------------------------------------
  | type | Extended Type | description                                  |
  -----------------------------------------------------------------------
  |  0   |        0      | handshake                                    |
  -----------------------------------------------------------------------
  |  1   |        0      | keepalive                                    |
  -----------------------------------------------------------------------
  |  2   |        0      | ack                                          |
  -----------------------------------------------------------------------
  |  3   |        0      | nak(loss report)                             |
  -----------------------------------------------------------------------
  |  4   |        0      | congestion warning                           |
  -----------------------------------------------------------------------
  |  5   |        0      | shutdown                                     |
  -----------------------------------------------------------------------
  |  6   |        0      | ackack                                       |
  -----------------------------------------------------------------------
  |  7   |        0      | drop request                                 | 
  -----------------------------------------------------------------------
  |  8   |        0      | Peer Error                                   |
  -----------------------------------------------------------------------
  |0x7fff|        -      | Message Extension                            |
  -----------------------------------------------------------------------
  |0x7fff|        1      | SRT_HSREQ:SRT handleshake request            |
  -----------------------------------------------------------------------
  |0x7fff|        2      | SRT_HSRSP:SRT handleshake response           |
  -----------------------------------------------------------------------
  |0x7fff|        3      | SRT_KMREQ:Encryption Keying Material Request |
  -----------------------------------------------------------------------
  |0x7fff|        4      | SRT_KMRSP:Encryption Keying Material response|
  -----------------------------------------------------------------------
</pre><br/>
扩展消息机制是为未来的扩展，SRT可能因为某些原因今后会用到。后面的SRT扩展握手中会提及。
* 第二个32bits：
<pre>
  1) Additional info -- 其在控制消息中被用作扩展空间字段。它的解析依赖于特殊消息类型，握手消息不使用它。
</pre>

### Handshake报文
Handshake控制报文('packet type' bit=1) 是用来在两点之间建立连接的。早期的SRT用handshake来交换参数，在连接建立之后，但是1.3版本吧交换参数作为handshake的自身的一部分。后面的Handshake一节专门用来解释。<br/>

![srt_handshake_packet](https://github.com/runner365/read_book/blob/srt/SRT/pic/srt_handshake_packet.png)

### KM 错误反馈报文
Key Messge Error Response控制报文('packet type' bit=1)是用来交换错误状态消息。在加密一节中详细介绍。<br/>

![km_error_response](https://github.com/runner365/read_book/blob/srt/SRT/pic/KM_err_response.png)

### ACK报文
ACK控制报文('packet type' bit=1) 是用来提供报文发送状态和RTT信息的。在SRT数据传输和控制一节中详细介绍。<br/>

![srt_ack_packet](https://github.com/runner365/read_book/blob/srt/SRT/pic/srt_ack_packet.png)

### Keep-alive报文
Keep-alive报文('packet type' bit=1) 是用来每10ms交换信息，来保证SRT流在连接断开后字段重连的。<br/>

![srt_keepalive_packet](https://github.com/runner365/read_book/blob/srt/SRT/pic/srt_keepalive_packet.png)

### NAK控制报文
NAK控制报文('packet type' bit=1) 是用来报告失败的报文传输。在SRT数据传输和控制一节中详细介绍。<br/>

![srt_nak_packet](https://github.com/runner365/read_book/blob/srt/SRT/pic/srt_nak_packet.png)

### SHUTDOWN控制报文
shutdown控制报文('packet type' bit=1) 用来发器关闭SRT连接。<br/>

![srt_shutdown_packet](https://github.com/runner365/read_book/blob/srt/SRT/pic/srt_shutdown_packet.png)

### ACKACK控制报文
ACKACK控制报文('packet type' bit=1) 用来回复收到ACK报文，并且可以用来计算RTT。在SRT数据传输和控制一节中介绍。<br/>

![srt_ackack_packet](https://github.com/runner365/read_book/blob/srt/SRT/pic/srt_ackack_packet.png)

### 扩展控制报文
扩展控制报文('packet type' bit=1) 用来为原始UDT用户控制消息。它们被用在SRT扩展握手报文中，可以通过独立的消息，或内嵌在HANDSHAKE中。<br/>

![srt_extend_ctrl](https://github.com/runner365/read_book/blob/srt/SRT/pic/srt_extended_ctrl_packet.png)

## SRT数据交互
下表描述数据交互(包括控制数据)。注意，两点间角色的变换。举例，在会话过程中节点可以作为发起者，和监听者，然后也能成为发送和接受者在数据传输过程中。<br/>

![srt_exchange_packet](https://github.com/runner365/read_book/blob/srt/SRT/pic/srt_data_exchange.png)

## SRT数据传输和控制
本节介绍直播的音视频中，如何处理控制/数据报文的关键思想。

### Buffers
当应用(编码器)提供数据给srt来发送，它们被放入一个环状的发送buffer中。它们用seqid来编号。报文放在buffer中，直到收到对端的ack，万一它们会需要被重传。每个报文都有一个时间戳，其基于连接时间(其在handshake中定义，在第一个报文发送之前)。
StartTime是应用创建SRT socket的时刻。报文时间戳是介于StartTime和报文被加到send buffer之间。

![starttime](https://github.com/runner365/read_book/blob/srt/SRT/pic/starttime.png)

注意：这里的时间是从左到右的，最新的报文在右边。<br/>
接收者也有个一样的buffer。报文都放在buffer的队列中，直到旧的报文被上层应用获取。当配置的延时刚好到packet 2，那么packet 1就应该送个上次应用而出队了。<br/>

![starttime](https://github.com/runner365/read_book/blob/srt/SRT/pic/receive_buffer.png)

时间戳是和连接关联的。传输并不是基于绝对时间。调度执行时间应该是基于实际时钟时间。时间的基准应该转换每个报文的时间戳到本地时钟时间。报文都是从发生方StartTime的便宜。任何时间相关参数都是基于本地StartTime来维护的，用来计算RTT，时间区和便宜，通过nanoseconds和其他。<br/>

### Send Buffer Management
发送队列(SndQ)是动态长度大小的，其包含了发送者buffer的内容相关多个引用。当发送队列有内容发送，相关引用信息就被加入到SndQ中。SndQ有多个buffers使用一个相同的channel(UDP socket)。<br/>
下表显示出send buffer与SRT socket(CUDT)相关。SndQ包含socket引用，时间戳引用，本地索引信息(其是在SndQ中位置)。相关引用对象也是SndQ的对象的一部分。<br/>

![sndq_info](https://github.com/runner365/read_book/blob/srt/SRT/pic/SndQ_Info.png)

SndQ是双向链表，其有send buffer的CSnode的入口点。CSnode是SndQ类的一个对象(SndQ是哥队列，但是也有其他的类成员)。CSnode并不与buffer内容相关。它有指针指向它的socket，timestamp 和buffer中的位置(其被插入到SndQ的位置)。<br/>
SndQ有个发送线程，其用来检查是否有报文要发送。基于在入口中包含的数据，它什么socket有报文ready可以被发送了。它检查时间戳引用，判断是否packet真的需要发送。如果没有，还把它放入list中。如果ready，线程就把他从list中移除掉。<br/>
每次发送线程发送报文，发送线程会把报文重新插入list。它然后去查看send buffer中的下一个packet。packet的时间戳决定SndQ中插入的位置，其是按照timestamps排序的。<br/>
控制报文是直接发送的。它们病不通过SndQ或发送队列。这个SndQ更新变量，为了追踪packet插入的位置，和那个packet是最后被对端ack的。<br/>

### SRT Buffer延时
发生者和接受者有大量的buffer，其在SRT程序代码中定义。在发送方，延时是时间，其是根据发送速率，SRT保存报文，直到给他时机发送，延时的影响对发送方比较小，如果ack晚到或没到，发送方只是按照上下文规定来丢弃。但是接收方的延时影响就明显很多。<br/>
延时是用毫秒来定义的值，其可以用来计算成百上千的高速率。延时能被认为是一个窗口，窗口滑移是基于时间，基于一个时间来进行窗口的滑移。<br/>
举例，在下表中，packet #2是最旧的报文，也是接受者队列的对头(packet #1已经被上层应用取走)。 <br/>
延时窗口在队列中从左向右滑动。当packet #2移除窗口外，它就需要被上次应用取走(如去用作解码)。<br/>
考虑到接收buffer存储一系列的packets。我们也就说我们定义延时，其是一个有6个报文长度的周期。延时能被认为是6个报文窗口长度。 <br/>
在延时窗口中恢复报文的能力，依赖与传输的时间。延时窗口高效通过RTT决定什么报文能恢复，多少次被恢复。<br>

![rcv_buffer_latency](https://github.com/runner365/read_book/blob/srt/SRT/pic/rcv_buffer_latency.png)

在准确的时刻，接受者buffer释放第一个报文给上层应用。当延时窗口滑向下一个报文间隔，接受者释放第二个报文给上层应用，以此类推。<br/>
现在我们看一下如果packet没有收到(packet#4)会发生什么。当延时窗口滑动，packet就应该可以上送给上层应用，但是该packet不存在。这就会导致跳到下个packet。其不会被恢复，那么它也将被移出丢弃list，并且永不会在要求重传。 <br/>

![rcv_latency_drop](https://github.com/runner365/read_book/blob/srt/SRT/pic/rcv_latency_drop.png)

滑移延时窗口可以被认为是一个区间，SRT能在区间内恢复大部分的packet。<br/>
另外一方面，发送者的buffer也有一个延时窗口。当时间流逝，最旧的报文就会移出延时窗口，永不会被恢复。因为即使它们再次被发送，它们将到达接收方太晚，而不能成功被接受者处理。<br/>
如果延时滑动窗口移除过期报文，其嗨没有被成功发送的(接收方没有ack成功)，太晚而不用再去发送它们了。因为它们会在接受者的发送窗口之外--即使它们后面到达，它们也会被丢弃。所以这些报文应该被移除到发送buffer外。<br/>
接收方和发送方应该有相同的延时值是非常重要的，以便协调packet的及时到达和丢弃。在早期SRT版本中，发送方把延时参数放在发送给接受者的handshake中。如果接受者上配置的延时参数较大，它会把这个延时参数放在packet中，发送response给发送方。在SRT 1.3版本后，双方都能在handshake的中立刻配置(不需要进一步的resoponse)。<br/>
当一个packet已经发送但是没有收到ack并且超时，这个packet就被移出发生方的延时窗口。它曾是发送者buffer的延时窗口，但是ackpos不会前移(在最后一个packet被ack确认其被收到，ACKPOS节点会指向下个packet)。当延时窗口前移并最旧的packet移出窗口，这个packet就可以被移除。SndQ的清理是手动的移动ACKPOS到延时窗口的下个packet。<br/>
所有再发送buffer的packets有保留的位置，其带有配置的长度(7个188字节，也就是1316，在加上udt头)。有很多发送buffer，每个都包含连续按顺序排列的packet。一个发送buffer是一个环形队列，开始和结束节点可以在任何节点，但是不会重合。协议保留sequence信息(基于ACK)。在传送过程中，所有packet的seq number能达到更高的数字，实际buffer位置不能达到的。发送者的buffer使用位置来计算。一个在buffer中的item都有一个开始节点start position(STARTPOS)。发送buffer中的positon和packet中的sequence是可以互相转化的，因为两者都是同时增长。 <br/>

### SRT Sockets, Send List & Channel
考虑到socket 1 和 2， 每个都有自己的发送buffer。SndQ包含一个packets的列表来发送。有一个线程来持续检查这个发送buffer。当一个报文可以被发送，一个CSnode被创建，其确认一个报文的socket，和在SndQ中一个相关的对象，SndQ讲指向发送队列尾部。<br/>

![srt_socket_bufferlist_1](https://github.com/runner365/read_book/blob/srt/SRT/pic/srt_socket_bufferlist_1.png)

每个packet都有timestamp，依赖timestamp来确定何时发送。SndQ列表是用timestamp来排列的。如果发送线程决定socket 1发送bufer有报文ready，它就把packet放入SndQ的队列中。如果SndQ队列是空，packet就被放在队列头，并带上自己的时间戳，其决定报文什么时候该被处理。<br/>
socket 2的发送buffer也能被加到SndQ中。发送线程将向buffer中要packet发送，线程会根据速率来计算packet的发送间隔。<br/>

![srt_socket_bufferlist_2](https://github.com/runner365/read_book/blob/srt/SRT/pic/srt_socket_bufferlist_2.png)