# 5 RTMP Chunk Stream
本节定义RTMP chunk stream。它为高层的媒体流协议提供多路复用和包封装的服务。<br/>
<br/>
当RTMP chuck stream被设计成为RTMP协议服务的时候，它能处理任何消息流的协议。每个消息包含时间戳和负载类型。RTMP chunk stream和RTMP在一起是非常适合各种音视频应用做点到点、点到多点的直播，或VOD服务，或会议应用。<br/>
<br/>
当用可靠传输协议来传输，如TCP(RFC0793)，RTMP chunk stream对所有流的消息提供可靠时序的点到点传输。RTMP chunk stream不提供任何优先级或类似控制，但可以在上层协议来提供优先级。例如，直播视频服务对网速慢的用户，可以选择丢弃视频消息以此保证音频消息的实时接收，无论基于时间发送还是回复每个消息的时间。<br/>
<br/>
RTMP chunk stream包括它自己的带内协议控制消息，并且也为高层协议提供机制内嵌用户控制消息。<br/>
<br/>

## 5.1 消息格式（Message Format）
消息格式能被分片成多个chunk，依次在上层协议上支持多路复用。消息格式因为应该包含下面几个字段，其是用来创建chunk的必要条件。<br/>
<br/>
* Timestamp: 消息的时间戳。这个字段是4个字节，单位毫秒;
* Length: 消息负载的长度。如果消息头被省略，它也应该包括在这个长度中。这个字段在chunk头中是3个字节。
* Type Id: type id的范围是为协议控制信息保留的。传播信息的消息体被RTMP chunk stream协议和上层协议处理。所有其他的type id都能为上层协议服务，并且可以作为透明值对rtmp chunk stream来说。实际上，在rtmp chunk stream里并不需要这些只；所有的非协议消息可以用同一个type，或者应用可以用这个值去传递其他的信息。这个字段在chunk header中占用一个字节。
* Message Stream ID: 它可以是个任意的值。不同的message流可以被复用到同一个chunk stream中，然后通过不同的message stream id被解复用成不同的message流。当然其也可以本当做透明的值。这个字段占用4个字节在chunk header中，以小字节序排列。

## 5.2 握手协议(Handshake)
一个RTMP连接开始于握手。握手完全不同于rtmp的其他协议。它有3个同等大小的chunk组成，而不是变长的chunk并带有头部。<br/>
<br/>
客户端和服务器各自发送同样3个chunk。客户端发送C0, C1和C2；服务端发送S0, S1和S2。

### 5.2.1 握手顺序
一开始客户端发送C0和C1的chunck。<br/>
<br/>
客户端发送C2前需要先收到服务端恢复的S1。客户端必须在收到S2之后，才能发送其他的数据。<br/>
<br/>
服务端必须收到C0后，才能发送S0和S1，也可以等到收到C0后才发送S0和S1。服务端必须等到收到C1才发送S2。服务端必须接收到C2后，才能发送其他的数据。<br/>
<br/>

### 5.2.2 C0和S0格式
C0和S0报文只有一个字节，也就是8-bit的字段:
<pre>
 0 1 2 3 4 5 6 7
+-+-+-+-+-+-+-+-+ 
|     version   | 
+-+-+-+-+-+-+-+-+
C0 and S0 bits
</pre>

C0/S0报文字段：
Version(8bits): 在C0中，这个字段是客户端定义RTMP版本。在S0中，这个字段由服务端选择支持的版本。在本文中，协议定义为3。值0-2是早期的版本好，已经不再使用；4-31是为未来的保留号；32-255不允许。服务端如果不认识客户端发来的版本好，就应该回复版本号3。客户端可以选择遵循协议3，或放弃握手。<br/>
<br/>

### 5.2.3 C1和S1格式
C1和S1包是1536字节长，由一下字段组成:
<pre>
0                    1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ 
|                        time (4 bytes)                         | 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ 
|                        zero (4 bytes)                         | 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ 
|                        random bytes                           | 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ 
|                        random bytes                           | 
|                            (cont)                             | 
|                             ....                              | 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                         C1 and S1 bits
</pre>
* Time(4字节)：这个字段包含时间戳，这个时间戳应该被后续所有的chuck作为时间开始。其可以是0，或者是其他任何数字。为了同步多个chuck流，服务端/客户端希望发送其他chunk流的时间戳。
* Zero(4字节): 这个字段必须是0。
* Random data (1528 bytes): 这个字段可以保护任何自定义信息。因为客户端和服务端为了区分发起握手的消息和响应握手的消息，这个数据块应该发送随机数值。但也不必一定是密码级别的随机值，或者可以是变量。

### 5.2.4 C2和S2格式
C2和S2都是1536字节长，是对S1/C1分别的回复，由以下字段组成:
<pre>
 0                   1                   2                     3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ 
|                        time (4 bytes)                         | 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ 
|                        time2 (4 bytes)                        | 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ 
|                        random echo                            | 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ 
|                        random echo                            | 
|                           (cont)                              |
|                            ....                               | 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                       C2 and S2 bits
</pre>
* Time (4 bytes):  这个字段是时间戳，S1的对C2，或C1对S2；
* Time2 (4 bytes):  这个字段必须包含上一个报文(s1或c1)发过来的时间戳；
* Random echo (1528 bytes):  这个字段必须包含随机数据对端发过来的，S1(对C2)或者S2(对C1)。双方都可以用time, time2字段来估计带宽和连接的延时，当然这比一定有用；

### 5.2.5.  Handshake图解
![handshake图解](https://github.com/runner365/read_book/blob/master/rtmp/pic/rtmp%20handshake.png)
<br/>
下面的描述握手图解的流程，3个状态:
* Uninitialized: 协议版本在过程中发送。客户端和服务端都没有初始化。客户端用C0发送协议版本。如果服务器支持该版本，服务端回复S0/S1。如果没有，服务端回复该有的行为。在RTMP中，就是中断连接。
* Version Sent:  客户端和服务端在非初始化状态后，能进入确定版本状态。客户端等待S1，服务端等待C1。当收到报文，客户端发送C2，服务端发送S2。状态就成Ack sent状态。
* Ack Sent: 客户端和服务端相互等待S2/C2状态。
* Handshake Done:  握手完成，客户端和服务端可以开始交换信息。

## 5.3 Chunking
握手后，连接可以服用一路或多路chunk stream。每个chunk stream都能从一个消息流中承载一种类型消息。每个chunk都能由唯一的ID定义，也就是chunk stream ID。chunk是通过网络发送。当网络传输时，每个chunk在前一个chunk发送完成后才发送。在接收端，chunk分片会被根据chunk stream id组包成完整的消息体。<br/>
<br/>
Chunking运行上层协议把大报文分片成小报文，以此防止大尺寸低优先级报文(如视频)阻塞到小尺寸高优先级报文(如控制报文)。<br/>
<br/>
Chunking也允许小报文能节省的报文头发送，也就是chunk header能压缩头信息，信息能包含在消息本身中。<br/>
<br/>
chunk size是可配置的。它能通过set chunk size控制协议来定义(在5.4.1中)。大的chunk size能降低CPU占用率，但是大尺寸的chunk能对低带宽连接导致高延时。而过小的chunk对高带宽流不是很好。chunk size是双方单方向来独立维护的。<br/>
<br/>
### 5.3.1 Chunk格式
每个chunk由header和data组成。头由3部分组成:
<pre>
+--------------+----------------+--------------------+--------------+ 
| Basic Header | Message Header | Extended Timestamp | Chunk Data   | 
+--------------+----------------+--------------------+--------------+ 
|<-------------------         Chunk Header        ----------------->|
</pre>
* Basic Header (1 to 3 bytes):  这个字段包括stream id和chunk type。chunk type定义了message header的格式。长度完全由chunk stream id决定，其是可变长度的字段
* Message Header (0, 3, 7, or 11 bytes):  这个字段包含消息要发送的信息(无论是全部还是部分)。该长度通过chunk header中的chunk type来定义
* Extended Timestamp (0 or 4 bytes): 这个字段chunk message header中的时间戳。 5.3.1.3中有详细信息。
* Chunk Data (variable size):  chunk的负载数据，数据填充到chunk size的大小。

#### 5.3.1.1.  Chunk Basic Header
这个chunk basic header包含chunk stream id 和chunk type(有fmt字段表示)。chunk type定义message header的格式类型。chunk basic header字段可以是1，2或3字节，其有由chunk stream id决定。<br/>
<br/>
协议支持到65597，streamid的范围是3--65599。ID中0，1，2是保留的。值0定义id为两个字节，64-319(第二个字节+64)。值1定义3个字节，范围64-65599(第三个字节*256+第二个字节+64)。值范围3-63代表完整的streamid。chunk stream id为2表示底层协议的控制消息和命令。<br/>
<br/>
bit 0-5(低字节)在chunk basic header中代表chunk stream id。<br/>
<br/>
chunk streamid 2-63 是这个字段的第一个版本。<br/>
<pre>
 0 1 2 3 4 5 6 7 
+-+-+-+-+-+-+-+-+ 
|fmt|   cs id   | 
+-+-+-+-+-+-+-+-+
Chunk basic header 1
</pre>
<br/>
chunk stream id值范围64-319是其头中的两个字节。ID为第二个字节+64。<br/>
<pre>
 0                   1
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ 
|fmt|     0     |   cs id - 64  | 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     Chunk basic header 2
</pre>
<br/>
chunk streamid 54-65599范围在3个字节的版本中编码。ID等于：第三个字节*256+第二个字节+64。<br/>
<pre>
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ 
|fmt|     1     |         cs id - 64            | 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
               Chunk basic header 3
</pre>
<br/>
* cs id (6 bits):  这个字段包含chunk stream ID，值2-63。值0和1的是2或3字节的版本。
* fmt (2 bits):  这个字段定义4中类型的chunk message header。这个chunk message header的类型介绍在下一节。
* cs id - 64 (8 or 16 bits):  这个字段包含最小为64的chunk stream id。举例，ID 365就是一个1的cs id字段和16bit的301字段。
<br/><br/>
chunk stream ID是值64-319，是头中两个字节或三个字节的格式模式。<br/>

#### 5.3.1.2 chunk message header
有chunk message header的不同格式，其有chunk basic header中的fmt字段定义。<br/>
<br/>
在应用中，应该用选择使用压缩比例最高的chunk message header。

##### 5.3.1.2.1 Type 0
Type 0 chunk header是11字节长。这个type 0类型必须是在chunk stream的最开始使用，并且无论什么时候stream timestamp都应该是向后发展的。<br/>
<pre>
0                    1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ 
|                timestamp                      |message length | 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ 
|     message length (cont)     |message type id| msg stream id | 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ 
|             message stream id (cont)          | 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                       Chunk Message Header - Type 0
</pre>
* timestamp (3 bytes):  对于type-0的chunk，消息使用绝对时间戳。如果时间戳大于等于16777215 (16进制0xFFFFFF)，这个字段必须是16777215，意味着Extended Timestamp字段32bit的时间戳。此外，这个字段代表完整的时间戳。
##### 5.3.1.2.2 Type 1
Type 1 chunk headers是7字节长。message stream ID不在其内。这个chunk带有上个chunk相同的chunk stream id。流都是变长的消息(举例，多种视频格式)应该用这个格式作为第二个stream的chunk报文。
<pre>
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ 
|                timestamp delta                |message length | 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ 
|      message length (cont)    |message type id| 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
               Chunk Message Header - Type 1
</pre>
##### 5.3.1.2.3 Type 2
类型2chunk headers是3字节长。stream ID和message length都不包含；chunk有与前一个chunk相同的stream ID和message length。流是定长的消息(例如，音频和数据格式)应该用这个类型，作为第二个stream的chunk报文。
<pre>
 0                   1                   2
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ 
|                timestamp delta                | 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                       Chunk Message Header - Type 2
</pre>
##### 5.3.1.2.4 Type 3
类型3其实没有message header。stream ID，message length和timestamp都不存在。这个chunk类型都继承前一个相同chunk stream ID的chunk所有字段。当单个消息被切分多个chunk，所有的消息除了第一个chunk外都用这个类型。
