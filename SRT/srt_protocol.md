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

![常规网络情况](https://github.com/runner365/read_book/blob/master/SRT/pic/net_condition01.png)

SRT的机制在接收方新创建了重要的特性，极大的降低buffer的需要。这些机制是SRT协议自身的一部分，所以一旦报文从SRT的一端发到接收端，流自身状态已经被恢复成流本身的状态。<br/>

![srt网络情况](https://github.com/runner365/read_book/blob/master/SRT/pic/net_condition02.png)

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
<br/>
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

![srt_data_packet](https://github.com/runner365/read_book/blob/master/SRT/pic/srt_data_packet.png)

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

![srt_data_packet](https://github.com/runner365/read_book/blob/master/SRT/pic/srt_control_packet.png)

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

![srt_handshake_packet](https://github.com/runner365/read_book/blob/master/SRT/pic/srt_handshake_packet.png)

### KM 错误反馈报文
Key Messge Error Response控制报文('packet type' bit=1)是用来交换错误状态消息。在加密一节中详细介绍。<br/>

![km_error_response](https://github.com/runner365/read_book/blob/master/SRT/pic/KM_err_response.png)

### ACK报文
ACK控制报文('packet type' bit=1) 是用来提供报文发送状态和RTT信息的。在SRT数据传输和控制一节中详细介绍。<br/>

![srt_ack_packet](https://github.com/runner365/read_book/blob/master/SRT/pic/srt_ack_packet.png)

### Keep-alive报文
Keep-alive报文('packet type' bit=1) 是用来每10ms交换信息，来保证SRT流在连接断开后字段重连的。<br/>

![srt_keepalive_packet](https://github.com/runner365/read_book/blob/master/SRT/pic/srt_keepalive_packet.png)

### NAK控制报文
NAK控制报文('packet type' bit=1) 是用来报告失败的报文传输。在SRT数据传输和控制一节中详细介绍。<br/>

![srt_nak_packet](https://github.com/runner365/read_book/blob/master/SRT/pic/srt_nak_packet.png)

### SHUTDOWN控制报文
shutdown控制报文('packet type' bit=1) 用来发器关闭SRT连接。<br/>

![srt_shutdown_packet](https://github.com/runner365/read_book/blob/master/SRT/pic/srt_shutdown_packet.png)

### ACKACK控制报文
ACKACK控制报文('packet type' bit=1) 用来回复收到ACK报文，并且可以用来计算RTT。在SRT数据传输和控制一节中介绍。<br/>

![srt_ackack_packet](https://github.com/runner365/read_book/blob/master/SRT/pic/srt_ackack_packet.png)

### 扩展控制报文
扩展控制报文('packet type' bit=1) 用来为原始UDT用户控制消息。它们被用在SRT扩展握手报文中，可以通过独立的消息，或内嵌在HANDSHAKE中。<br/>

![srt_extend_ctrl](https://github.com/runner365/read_book/blob/master/SRT/pic/srt_extended_ctrl_packet.png)

## SRT数据交互
下表描述数据交互(包括控制数据)。注意，两点间角色的变换。举例，在会话过程中节点可以作为发起者，和监听者，然后也能成为发送和接受者在数据传输过程中。<br/>

![srt_exchange_packet](https://github.com/runner365/read_book/blob/master/SRT/pic/srt_data_exchange.png)

## SRT数据传输和控制
本节介绍直播的音视频中，如何处理控制/数据报文的关键思想。

### Buffers
当应用(编码器)提供数据给srt来发送，它们被放入一个环状的发送buffer中。它们用seqid来编号。报文放在buffer中，直到收到对端的ack，万一它们会需要被重传。每个报文都有一个时间戳，其基于连接时间(其在handshake中定义，在第一个报文发送之前)。
StartTime是应用创建SRT socket的时刻。报文时间戳是介于StartTime和报文被加到send buffer之间。

![starttime](https://github.com/runner365/read_book/blob/master/SRT/pic/starttime.png)

注意：这里的时间是从左到右的，最新的报文在右边。<br/>
接收者也有个一样的buffer。报文都放在buffer的队列中，直到旧的报文被上层应用获取。当配置的延时刚好到packet 2，那么packet 1就应该送个上次应用而出队了。<br/>

![starttime](https://github.com/runner365/read_book/blob/master/SRT/pic/receive_buffer.png)

时间戳是和连接关联的。传输并不是基于绝对时间。调度执行时间应该是基于实际时钟时间。时间的基准应该转换每个报文的时间戳到本地时钟时间。报文都是从发生方StartTime的便宜。任何时间相关参数都是基于本地StartTime来维护的，用来计算RTT，时间区和偏移，通过nanoseconds和其他。<br/>

### Send Buffer Management
发送队列(SndQ)是动态长度大小的，其包含了发送者buffer的内容相关多个引用。当发送队列有内容发送，相关引用信息就被加入到SndQ中。SndQ有多个buffers使用一个相同的channel(UDP socket)。<br/>
下表显示出send buffer与SRT socket(CUDT)相关。SndQ包含socket引用，时间戳引用，本地索引信息(其是在SndQ中位置)。相关引用对象也是SndQ的对象的一部分。<br/>

![sndq_info](https://github.com/runner365/read_book/blob/master/SRT/pic/SndQ_Info.png)

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

![rcv_buffer_latency](https://github.com/runner365/read_book/blob/master/SRT/pic/rcv_buffer_latency.png)

在准确的时刻，接受者buffer释放第一个报文给上层应用。当延时窗口滑向下一个报文间隔，接受者释放第二个报文给上层应用，以此类推。<br/>
现在我们看一下如果packet没有收到(packet#4)会发生什么。当延时窗口滑动，packet就应该可以上送给上层应用，但是该packet不存在。这就会导致跳到下个packet。其不会被恢复，那么它也将被移出丢弃list，并且永不会在要求重传。 <br/>

![rcv_latency_drop](https://github.com/runner365/read_book/blob/master/SRT/pic/rcv_latency_drop.png)

滑移延时窗口可以被认为是一个区间，SRT能在区间内恢复大部分的packet。<br/>
另外一方面，发送者的buffer也有一个延时窗口。当时间流逝，最旧的报文就会移出延时窗口，永不会被恢复。因为即使它们再次被发送，它们将到达接收方太晚，而不能成功被接受者处理。<br/>
如果延时滑动窗口移除过期报文，其还没有被成功发送的(接收方没有ack成功)，太晚而不用再去发送它们了。因为它们会在接受者的发送窗口之外--即使它们后面到达，它们也会被丢弃。所以这些报文应该被移除到发送buffer外。<br/>
接收方和发送方应该有相同的延时值是非常重要的，以便协调packet的及时到达和丢弃。在早期SRT版本中，发送方把延时参数放在发送给接受者的handshake中。如果接受者上配置的延时参数较大，它会把这个延时参数放在packet中，发送response给发送方。在SRT 1.3版本后，双方都能在handshake的中立刻配置(不需要进一步的resoponse)。<br/>
当一个packet已经发送但是没有收到ack并且超时，这个packet就被移出发生方的延时窗口。它曾是发送者buffer的延时窗口，但是ackpos不会前移(在最后一个packet被ack确认其被收到，ACKPOS节点会指向下个packet)。当延时窗口前移并最旧的packet移出窗口，这个packet就可以被移除。SndQ的清理是手动的移动ACKPOS到延时窗口的下个packet。<br/>
所有再发送buffer的packets有保留的位置，其带有配置的长度(7个188字节，也就是1316，在加上udt头)。有很多发送buffer，每个都包含连续按顺序排列的packet。一个发送buffer是一个环形队列，开始和结束节点可以在任何节点，但是不会重合。协议保留sequence信息(基于ACK)。在传送过程中，所有packet的seq number能达到更高的数字，实际buffer位置不能达到的。发送者的buffer使用位置来计算。一个在buffer中的item都有一个开始节点start position(STARTPOS)。发送buffer中的positon和packet中的sequence是可以互相转化的，因为两者都是同时增长。 <br/>

### SRT Sockets, Send List & Channel
考虑到socket 1 和 2， 每个都有自己的发送buffer。SndQ包含一个packets的列表来发送。有一个线程来持续检查这个发送buffer。当一个报文可以被发送，一个CSnode被创建，其确认一个报文的socket，和在SndQ中一个相关的对象，SndQ讲指向发送队列尾部。<br/>

![srt_socket_bufferlist_1](https://github.com/runner365/read_book/blob/master/SRT/pic/srt_socket_bufferlist_1.png)

每个packet都有timestamp，依赖timestamp来确定何时发送。SndQ列表是用timestamp来排列的。如果发送线程决定socket 1发送bufer有报文ready，它就把packet放入SndQ的队列中。如果SndQ队列是空，packet就被放在队列头，并带上自己的时间戳，其决定报文什么时候该被处理。<br/>
socket 2的发送buffer也能被加到SndQ中。发送线程将向buffer中要packet发送，线程会根据速率来计算packet的发送间隔。<br/>

![srt_socket_bufferlist_2](https://github.com/runner365/read_book/blob/master/SRT/pic/srt_socket_bufferlist_2.png)

<br/>
带有包间隔的时间戳决定packet重新插入SndQ的位置(在从socket1 buffer的packet之前，或之后)。<br/>
SndQ决定从哪个SRT socket去取下一个packet来发送。send buffer和socket绑定，而SndQ却是跟channle更加相关。几个socket都发送到同一目的地，所以它们是多路复用的。<br/>
当packet被加到socket，SndQ也会被更新。当一个packet已经可以被发送，其也被基于时间戳重新插入到SndQ中的正确位置。<br/>
这个处理过程在SndQ中发生。对每个packet报文，有一个线程去检查是否该发送。如果没有，就什么也不错。否则，线程要求SRT socket把这个packet发送到channel。在SndQ的条目有SRT socket的引用。<br/>
当一个packet被写到send buffer，它也被加入到SndQ中，其也通过CSnode来确保其不会产生重复的条目项。SndQ重复的移除条目项，并同时插入新的packet到正确的位置上。<br/>
在不同SRT socket的packet的时间戳是本地的，其定义的时间是发送时对比当前时间。在packet加入到SndQ的时刻，他的时间戳对比当前的时间决定其在SndQ中的位置。<br/>
Send buffer的操作和SndQ的操作是分离的。packet被加入到buffer，且然后SndQ被通知有packet需要发送。它们各自有自己的行为。<br/>
send buffer的内容会被加入到应用线程中(sender线程)。然后有另外一个线程和SndQ互动，其通过输入buffer的速率负责控制packet间隔。输出是通过buffer中的packet间隔来调整控制。<br/>

### Packet Acknowledgement(ACKs)
在确定的间隔(与ACKs, ACKACKs 和 Round Trip Time相关)，接收方发送ACK给发送方，使得发送方把收到ack的packet从sender buffer中移除，其在buffer中的空间点将被回收。ACK包含了packet的sequence number，其是刚最新收到报文的seq+1。当没有报文丢失的情况下，ack返回的seq应该是n+1(n是接收到的packet的seq number)。<br/>
举例，如果接受者发送packet 6的ACK(如下)，意味着比这个sequence数小的报文都收到了，能从发送者的buffer中移除。<br/>

![srt_ack](https://github.com/runner365/read_book/blob/master/SRT/pic/SRT_ACK.png)

在丢失的案例中，ACK(seq)就是丢失列表中的第一个报文，其就是最新收到报文的seq+1。<br/>

### Packet Retransmission (NAKs)
如果packet 4到达了接受者的buffer，但是packet 3并没有到达，NAK报文就需要发送给发送着。NAK被加到一个列表(周期的NAK报告)，其周期的发送给发送方，以此避免NAK报文本身传输中丢失或延迟到达。<br/>

![srt_nak](https://github.com/runner365/read_book/blob/master/SRT/pic/SRT_NAKs.png)

如果packet 2到达，但是packet 3没有，那么当packet 4到达后，NAK就应该按照规则发送来发起要求重传。<br/>

![srt_nak2](https://github.com/runner365/read_book/blob/master/SRT/pic/SRT_NAK2.png)

### Packet Acknowledgment in SRT
UDT草案定义周期发送的NAK控制报文，其包含一个丢失报文的列表。UDT4应用去使能这个特性，而用定时重传的方法来代替。NAK的发送仅仅发生在一个丢失报文被检测到(也就是下一个报文都收到了，但是上一个报文未能收到)。如果NAK本身丢失，ACK会阻塞在这个packet，同时阻止发送更多的报文给接收端直到丢弃list为空。在发送方，因为如果没收到NAK报文，丢失的报文也不会被加入到丢失list中去，并会影响到没有收到ACK报文的重传。<br/>
UDT处理拥塞的方法是通过阻止重传直到丢失列表为空，这个做法基本上是错的。因为重传丢失列表报文优先会很大可能阻塞住接收方。<br/>
在SRT接下来的修改中(举例NAK周期发送，基于时间戳的发送，太晚报文丢弃等等)，会降低ACK-timeout重传的发生。<br/>

### Timestamp-based Packet Delivery(TsbPD)
这个特性是使用UDT packet头中的timestamp。早期的SRT TsbPD设计是想复制编码器的输出，来作为解码器的输入。这个设计没有考虑到传输的瓶颈，报文传输越快，丢失的报文重传也就越快，避免了低延时。但是SRT协议是基于网络带宽受限的情况下开发的，能占有网络没有任何限制。<br/>
另外一个问题是原始SRT TsbPd的设计是基于CPU限制的。ts packet的时间戳是基于系统能产生和标识packet的时间戳。如果接收者没有同样的CPU容量，也就不能复制发送者的模式。<br/>
SRT的当前版本，TsbPD允许接受者以相同的速率发送packet报文给接收者，其速率是SRT发送者编码器的发送速率。基本上，在接收者把收到的报文上送给应用前，发送者在报文中的时间戳会被调整成接收者的本地时间(补偿时间偏移或不同的时间轴)。packet能被SRT基于配置的接收者延时来保有。更高的延时能容忍更大的报文丢失发生率，或更大的报文丢失突发率。接收到的报文在它们被play后再丢弃掉。<br/>
packet的timestamp(微秒)是关联到SRT的连接建立时间。原始的UDT 编码用packet 发送时间来作为packet的timestamp。对于TsbPD特性来说，是不正确的，因为如果一个新的时间(当前的发送时间)用来重传报文，会导致乱序，因为按时间把重传报文插入到队列中。报文应该基于sequence number来插入。<br/>
当一个packet在应用把packet报文放入SRT的原始时间(微秒)就是packet的timestamp。TsbPD特性用这个时间来作为packet第一次发送的timestamp，和接下来任何时间重传报文的timestamp。时间戳和配置的延时控制控制恢复buffer和实时发送到对方的packet。<br/>
UDT协议本身并不使用packet timestamp，所以这个修改对UDT协议并不影响，也不会影响到当前已存在的拥塞控制方法。<br/>

### Fast Retransmit：快速重传
原始的UDT4未确认ack的报文重传是基于超时机制，这样对于实时数据并不友好。只要在loss list中还有报文，没有收到ack的报文就不会被重传。因为丢失报文的重传是并发发生的，当重传定时器到超时时刻，会开始一波并发事件，其会影响到实时数据(拥塞窗口，发送者buffer慢，丢包等等)。<br/>
快速重传在拥塞窗口满之前，通过重传没有收到ACK的报文来解决这个问题。发送着把在合理时间内没有收到ACK的报文都放入loss list中，合理的时间基于RTT和丢弃报文的timestamp。<br/>
快速重传机制，减少了接收者buffer size，和延时。其也让丢包数量变量变化更平滑，对比于拥塞窗口满时的重传。然而，这个特性也是对带宽非常饥渴的。SRT发送者的这个特性是与SRT1.0接收者可以互通配合的。当有了周期的NAK报告后，这个特性就很少用了，或仅仅作为看门狗。<br/>

### Periodic NAK Reports：周期发送NAK
SRT1.0在报文丢包率大于2%后是非常不高效的。很多报文重传都不止一次，其实都没有完全确认清楚该报文是否真的丢失。造成的情况是，带宽的瓶颈和延时无法承受这样丢包率造成的重传。<br/>
Periodic NAK Reports在UDT4的源码中被去使能了。这个特性在SRT1.1.0的接收者中被重新开启，用来提高SRT适应高丢包环境，和所有的丢包环境。因为重传带宽的超出，这样的应用会造成大概两倍的丢包率。对于SRT配置参数的限制内，10%的抗丢包是可以达到的。<br/>
SRT的Periodic NAK Reports是基于RTT/2为周期，最小20ms(UDT设定是300ms)。一个NAK控制报文含有一个丢失报文的压缩列表。所以，仅仅丢失的报文会被重传。通过使用RTT/2作为NAK报告的周期，这会导致丢失报文可能会被重传一次以上，但是这样会保持低延时在NAK报文丢失的情况下。

### Too-Late-Packet-Drop: 报文太晚丢弃
这个特性在SRT1.0.5中开始有，允许发送端丢弃报文，如果其没有机会及时发送。在SRT发送端，如果Too-Late-Packet-Drop使能，报文的时间戳比SRT最大延时的125%还要大，就会被认为太晚而不用在发送了，会被编码器丢弃。IFrame尾部的报文就有可能被丢弃。<br/>
接收者(SRT>=1.1.0)可以不让使能发送端的包丢弃，来防止发送端的产生bug。发送端(srt>-1.1.0)会保留报文至少1000ms，如果SRT的最大延时小于1000ms(对最大的RTT不够的话)。
在接收端，大IFrame的尾部可能晚到，并且不会被SRT接收buffer缓冲到。接受buffer耗尽，如果有发现丢包发生，就没有时间可以重传。丢失的报文被接收者忽略。

### Bidirectional Transmission Queues：双向传输队列
SRT也支持这样的场景，接收者也有自己的发送队列，发送着也有相应的接受队列，支持双向通信。<br/>
这是很有用的，能支持标准点到点的SRT会话，两端都有发送/接受buffer。发送端的Tx buffer对应接收端的Rx buffer，而接收端的Tx buffer对应发送端的Rx buffer。和普通单方向的会话一样，Tx/RX的延时相互匹配。<br/>
在handshake报文中，发送端提供自己的Tx latency和假想对端的latency(接收端的Tx buffer值)。接收端也响应回复对应的参数。提议的latency值是在单个RTT周期内，双方评估的结果(尽量选择大一点的值)。<br/>

![bidirectional trans queue](https://github.com/runner365/read_book/blob/master/SRT/pic/bidirectional_trans_queue.png)

### ACKs, ACKACKs & Round Trip Time
Round Trip Time(RTT)是时间的度量，表示报文一个来回的耗时。SRT不能测量单方向的耗时，所以只能用RTT/2来表示单方向耗时。一个ACK(从接收方)会触发ACKACK(从发送方)的发送，几乎不带其他延时。ACK的发送时间与对应ACKACK收到时间的差值就是RTT。<br/>

![acks send](https://github.com/runner365/read_book/blob/master/SRT/pic/ACK_send.png)

ACKACK告诉接收者停止发送对应便宜点的ACK，因为发送端已经知道接收端收到了。否则，ACK(带有过时信息)将被持续的周期发送。类似的，如果发送端没有收到ACK，它自己也会周期发送没有收到ACK的packet。<br/>
有两种情况发送ACK。一个full ACK是基于10ms(ACK周期)发送。对于高bitrate的传输，一种"light ACK"就能被发送，期是多个packet的一个sequence。在10ms的间隔里，经常有大量packet的发送和接收，以至于发送端ACK的偏移点不能够快的移动。为了减轻这个问题，在收到64packets后(即使ACK发送周期还没到)，发送端发送一个light ACK。

![ackack send](https://github.com/runner365/read_book/blob/master/SRT/pic/ackack_send.png)

ACK动作像ping报文，而ACKACK像ping back回复，以此可以度量出RTT。每个ACK都有一个数值，而ACKACK也有相同的一个数值。接收方有一个ACK的列表去匹配ACKACK。不像full ACK报文(包含当前的RTT和多个其他的控制信息参数)，light ACK包含sequence数值(如下表所示)。在接收端，所有控制消息被直接发送和处理，但是ACKACK的处理时间是微不足道的(因为它的处理时间被包括在RTT里面)。<br/>
RTT是在接收端被计算出来的，并且发送下一个full ACK。注意，第一个ACK包含的RTT值默认是100ms，因为早期的计算可能不准确。<br/>

![ack_ackack_formats](https://github.com/runner365/read_book/blob/master/SRT/pic/ack_ackack_formats.png)

发送端永远都是通过接收端获取到RTT。没有一个方法来模拟ACK/ACKACK机制(举例，不可能发送一个消息，这个消息不处理而立刻返回)。

### Dirft Management: 偏移管理
当发送方进入“连接”状态，它就告诉上层应用有个socket interface接口已经ready可以发送了。在这个时间的，应用可以开始发送数据packet了。它以一定的输入速率把packet加入到SRT发送方的buffer，通过这个buffer，packet以规定的时间发送到接收者。<br/>
同步的时间是需要来保证，接收者/发送者buffer的登记，需要考虑到时间轴和RTT。考虑到加/减RTT，和可能的不同步的系统时间，一个都能认同的时间基准，每分钟约几个微秒的偏移。这样的偏移累积起来可能需要几天才能让发送/接收的buffer耗尽或溢出，从而严重影响视频质量。SRT有时间管理机制来补偿这个偏移量。<br/>
当packet收到后，SRT能得出packet timestamp和期望timestamp之间的差值。时间戳timestamp的计算是在接收方进行。RTT告诉接收方报文要消耗多少时间。SRT在发送者buffer的延时窗口的边缘时间和接收者对应时间之间维护一个参数。这样能允许基于本地时间实时的能调度事件。 <br/>
接收者采样时间偏移数据，和周期的计算packet timestamp纠正参数，两者就能用来对每个收到的data packet来调整报文间隔。<br/>
当一个packet收到后，不能马上上送上层应用。当时间推移，接收者就能算出丢失报文预计的时间，并且可以用这些信息去用特殊报文来填补这些“丢包的空洞”。<br/>
接收者用本地时间去调度事件---举例，去决定是否现在上送一个packet。每个packet里的timestamp都与会话开始时候有个差值。当收到一个packet(packet内带有发送方的timestamp)，接收者就用本地时间与会话开始时间之间的差值，来重新计算packet的timestamp。start time就是会话开始时的本地时间。packet timestamp就等于当前时间减去start time(packet_timestamp=now-start_time)，start time就是socket的创建时刻。<br/>

### Loss List
发送方通过NAK报告来建立lost packe list。当调度到发送，先看lost list中是否有报文需要发送，有的话先发送lost list中的。否则，就发送SndQ list中的。注意，当packet发送后，仍旧在buffer中保留以免对方没有收到。<br/>
收到NAK报文后，就把其中的报文放入lost list。当延时窗口前移，packet将被移出send queue，需要检查一下是否丢弃或重传的packet在lost list中，以此来决定是否把这些报文移出lost list，因为它们没有必要再重传。在send queue和lost list的操作是通过ACKPOS决定。<br/>

![ack_pos1](https://github.com/runner365/read_book/blob/master/SRT/pic/ACKPOS1.png)

当ACKPOS前移到一个点，所有比这个点旧的packet都可以被移出send queue。<br/>

![ack_pos2](https://github.com/runner365/read_book/blob/master/SRT/pic/ACKPOS2.png)

当接收者遇到遇到这种情况，下一个应该被play的packet没有收到，它就应该跳过这个packet，并且发送一个fake ACK。对于发送端，fake ACK就是真的ACK，也就是说发送端就认为这个packet真的被成功接收了。这个方法有利于帮助发送者和接收者之间的同步。实际上packet的丢失对于发送端是不知道的。跳过这个报文在接收端的统计statistics中有记录。<br/>
当发送端收到NAK packet。也有个packet的计数器。如果packet没有对应的ACK，它就在lost list中保留，有可能被多次发送。在lost list中的packet优先级更高一些。<br/>
如果在lost list中的packet持续的阻塞住send queue，在一些情况下就会造成send queue被填满。当send queue满了，发送端首先就丢弃packet而不是发送它们。编码器(或上层应用)可能持续的生成packet，但是send queue没有空间放入，所以packet会被直接丢弃。SRT本身不对未发送报文敏感的，其也不会在SRT statistics中显示。<br/>
这种packet上层丢弃的情况几乎不会发生。放在send buffer中packet的数量是基于配置延时参数的。旧的packet，没有机会再被重传或被play的，会被丢弃，为上层应用的新报文留空间。当低延时配置被配置，最小一秒的延时是被使能的。一秒的限制是来源于MPEG I-frame用SRT传输的情况。IFrame是比较大的(通常8倍于其他packet)，需要更多的时间来发送。其太大而不能在延时窗口中保留，可能会造成queue中的报文丢弃。为了避免这种情况，SRT应用在丢弃packet前，最少有1秒等等(或延时变量值)。这就可以当应用最小延时变量，仍可以应用到IFrame中。<br/>

### SRT Packet Pacing: SRT按速率发送packet
UDT用最大带宽设置来控制packet输出速率。这个statics设定对于动态的输入不太友好，特别当你改变编码器的编码bitrate的时候。SRT控制packet速率基于输入速率，和重传的负荷(这些都基于输出来计算)。<br/>
早期，SRT用配置的输入速率作为编码器的输出速率(根据packet的音频和最高的bitrate)。但是有时对于编码器经常会输出过高的速率。在低速率输出时，编码器有时太过于乐观，就会输出超过预期过高的bitrate。在这些情况下，SRT packet就不会足够快的输出，因为SRT会最终被过低的错误配置影响到。<br/>
这可以通过计算bitrate来缓解这个问题。SRT检测到要发送的packet，并且计算出它的平均移动bitrate。然而，这样的操作可能会有一些延时。也就是说，如果编码器遇到黑屏或者静帧，它就会大幅度的降低bitrate，就会降低SRT bitrate。而当编码器输出突然增大，SRT也不会立刻增大很快。packet可能会延时，解码器收到就会晚，从而造成问题。<br/>
一个提议的解决方案是，让SRT把编码器的input rate配置，和测量实际input rate，使用这两者的中最大值。<br/>
输出是通过控制packet周期(两个packet之间的间隔)。对于一个指定的bitrate，SRT计算出一个packet的平均大小。packet的间隔是通过比较持续packet中的timestamp来定义。packet的输出是通过packet size和间隔来调度。<br/>

![packet_pacing_interval](https://github.com/runner365/read_book/blob/master/SRT/pic/packet_pacing_interval.png)

传输速度是通过packet之间的定时器来控制的。在老的代码中，packet周期是通过拥塞控制模块来调整的。基于从网络的反馈，packet间隔可以突然减小来加速，或突然增加来减速。但是这在SRT的直播模式中并不适合。音视频流bitrate是在mpegts packet形成，修改出口packet pace病不能影响到接收方单个的流rate--它会影响到解码。早期SRT是在一个周期里，输出rate与输入input相当。默认情况下，SRT测量input流的bitrate，根据这个来调整packet period。 <br/>
SRT需要一个确定的足够带宽(比预想的带宽略高)，为了有空间插入更多的重发packet，而不影响SRT发送者的主流输出速度太多而导致packet不能正确发送。唯一的办法是通过network的反馈来降低拥塞，来控制编码器的输出(SRT的输入)。不太可能对已经打包到预想bitrate的packet进行调整，因为这个预设定的速度已经是解码器想要的速度了。<br/>
这里有3个配置项：INPUTBW, MAXBW 和 OVERHEAD(%)。 <br/>

![maxbw_inputbw_overhead](https://github.com/runner365/read_book/blob/master/SRT/pic/maxbw_inputbw_overhead.png)

设置输入带宽(INPUTBW)参数为0，意味着用内部测量值(smpinpbw)来设置packet周期，同时与output的overhead最大配置配合来完成。<br/>
一个绝对的最大带宽(MAXBW)能被配置成一个能力极限值。设定MAXBW为0的话，就是只用INPUTBW来调整输出。<br/>
这里有个情况是SRT需要考虑的。问题就是测量输入带宽的方法有一个延后问题，这个测量是平均值。所以如果输入速率突然降为0(举例，因为可能会有黑屏在视频流中)，测量结果就会掉入低值的rate。但是当视频速率突然增长，input rate就突然增长。SRT的输出速率就会落后于编码器的输入速率。报文就会累计在SRT发送者的buffer中，因为SRT要花时间来测量速度。如果packet太晚输出，就会导致问题。<br/>

![SRT_blackscreen](https://github.com/runner365/read_book/blob/master/SRT/pic/SRT_blackscreen.png)

为了解决这个问题，SRT的发送方的输出是通过视频编码器的速率来配置的。因为可能通过编码器配置的bitrate来配置SRT的bitrate，任何修改都能被直接copy到SRT的配置中。这个是全局的。

![sender_encoder](https://github.com/runner365/read_book/blob/master/SRT/pic/sender_encoder.png)

如下表所示，SRT的发送者bitrate因为黑屏而降低，但是不会一直降低。 <br/>

![SRT_blackscreen2](https://github.com/runner365/read_book/blob/master/SRT/pic/SRT_blackscreen2.png)

但是这个解决方案有它自己的弱点。在低bitrate，SRT从编码器来的输入速率经常会超过预先配置的bitrate。因为SRT发送方的输出是基于配置编码器的bitrate来调整，输入突然过高会导致packet再send buffer中的堆积。buffer的堆积比packet的发送更快。但是SRT输出还会依赖配置的速度，但是被累积的packet不得不超时发送。它们会在buffer中累计，不会即使的输出，最后导致一些packet会被发送过晚。<br/>

![SRT_packet_pacing_loss_retransmission](https://github.com/runner365/read_book/blob/master/SRT/pic/SRT_packet_pacing_loss_retransmission.png)

图中橘黄色线以上表示，基于延时需要利用overhead space进行重传的pakets。橘黄色线表示的区域是丢失packet的数量。这些packet不得不加入到图表的上部，实时直播流不得不一直维护这些信息。<br/>
基于这个原则，packet之间的空间决定重传packet在什么地方插入，overhead代表可以提供的边界。有个经验的计算，定义packet之间的间隔从而得出输出bitrate。它是负载的功能(负载包含音频/视频)。<br/>
SRT尝试基于解码器输出来重新定义发送者的输入带宽，重发packet是透明的，就好像它们从来没被重传过。当到达一个临界点，但是一旦packet pacing下降，有很大的累积在send buffer。它突然变满，并且不能足够快的清空，到某个时间点packet就开始丢包。 <br/>
SRT版本1.3合并了两种packet pacing的方法。输入rate的检测，但是如果其掉速太多，SRT发送者输出的配置bitrate是公认的。如果测量低于配置速率，它并不遵循测量速率(如果完全没有packet发送，SRT就不发送)。然而，如果编码器输入速率大于配置速率，它就遵循编码器配置速率，越快越去遵循。绑定这两种方法来克服各自的问题。<br/>

![SRT_packet_pacing_loss_retransmission2](https://github.com/runner365/read_book/blob/master/SRT/pic/SRT_packet_pacing_loss_retransmission2.png)

理论上，带宽能力是带宽overhead的值。如果有太多packdet而不能发出，SRT不能发出。但是带宽能力的机制不能正常工作。带宽限制就被轻易的击溃。<br/>
另外一个SRT版本1.3的变化，是加了一个配置option，叫OUTPACEMODE，其使能其他pacing项的组合。设置MAXBW模式就是以绝对带宽容量值，其并不随编码器的输入而波动。设置MAXBW为0，意味着使用INPUTBW。相反的，设置INPUTBW为0意味着完全使用内部测量值。<br/>
SRT 1.3版本用这些技术的捆绑来作为配置输入，而不是只用测量值，或使用两者间的最大值。OUTPACEMODE是用来使能其他配置项的合并使用。它定义了MAXBW，INPUTBW，和两者的合并，或两者都不使能(不限制)。<br/>
SRT packet pacing总结如下。默认，直播模式下的输出rate是基于输入rate的(也就是流的输出)。输入rate(sendmsg API)是内部测量到的，输出rate是动态调整的，包括配置overhead(25%)，其被加到重传packet的迸发rate中。

### Packet Probes
当SRT用很多技术方法在流控上，但有很多限制在于无法准确的计算链路的带宽容量。这里的Packet probe技术是每16个packet报文间隔发送一次“packet probe”其没有报文间隔的延时。如果这两个连着的packet到达接收者是有间隔的，意味着网络有影响的背景流量，其也意味着足够的网络带宽有所下降。这也帮助去测量网络带宽能力。但是在packet之间的空间的计算是无法控制的，也就是说足够的带宽量是很难定义出来的。<br/>

![SRT_PacketProbes](https://github.com/runner365/read_book/blob/master/SRT/pic/SRT_PacketProbes.png)
