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
类型3其实没有message header。stream ID，message length和timestamp都不存在。这个chunk类型都继承前一个相同chunk stream ID的chunk所有字段。当单个消息被切分多个chunk，所有的消息除了第一个chunk外都用这个类型。<br/>
对于示例2(5.3.2.2节)，流都是同等大小的消息组成，stream ID和时间间隔应该使用类型3，其紧跟着chunk类型2.<br/>
对于示例1(5.3.2.1节)。如果第一个消息与第二个消息的时间戳一样，那么type3可以作为第二个chunk紧跟着第一个chunk(type0)，二不需要type 2。如果type 3紧跟着type0的chunk，那么type3 chunk的时间戳与type0 chunk的时间戳绝对一致。<br/>

##### 5.3.1.2.5.  Common Header Fields
* timestamp delta (3 bytes):  针对type-1或type-2的chunk，前一个时间戳与后一个时间戳的差值，那么就该使用这个字段。如果这个delta大于等于16777215(16进制0xFFFFFF)，这个字段就必须是16777215，意味着Extended Timestamp字段会有32bits的值。此外这个字段就是表示准确的时间戳。
* message length (3 bytes):  针对type-0或type-1的chunk，消息的长度就在这个字段中。注意这个不是chunk负载的长度。chunk负载的长度是chunk size的最大值，针对所有chunk除了最后一个chunk，剩余的字节就放在最后一个chunk中(当然其也可能是整个长度，针对小消息来说)
* message type id (1 byte):  针对type-0或type-1，这个消息的type在这个字段中。注意
* message stream id (4 bytes):  针对type-0的chunk，消息stream ID会被发送。特别是，所有同一个chunk stream的消息都将来源于同一个消息流。也可能把多个不同的消息放入同一个chunk stream中，当然这就失去了头部压缩的好处。如果一个message流关闭，而另外下一个流打开，没有什么理由上次存在的chunk stream 不能被新的type-0 chunk重用。

#### 5.3.1.3.  Extended Timestamp
extended timestamp字段用于timestamp字段大于等于16777215(0xFFFFFF)；那是为了时间戳不能满足于在type0,1,2 chunk中24bits大小的字段。这个字段是完整的32bit的时间戳或时间戳差值。这个字段在chunk type 0中表示时间戳，或type-1或type-2 chunk中表示timestamp的差值，timestamp字段的值必须是16777215(0xFFFFFF)。这个字段当先前使用的type 0, 1, 或2 chunk对同一个chunk stream ID, 表示type3该字段是上次extended timesamp field。

### 5.3.2.  Examples
#### 5.3.2.1.  Example 1
本例展示了单个音频消息。这个例子展示了消息有很多重复信息。
![Sample audio messages to be made into chunks](https://github.com/runner365/read_book/blob/master/rtmp/pic/chunk%20audio%20example1.png)
<br/>
下一张图表显示chunk在流中的构成。从message 3往后，数据传输头部都优化了。在message 3后，每个message的头部只有1个字节
![Sample audio messages to be made into chunks](https://github.com/runner365/read_book/blob/master/rtmp/pic/chunk%20audio%20example2.png)

#### 5.3.2.2.  Example 2
本例演示了一个消息大于128字节的chunk，被切分成多个chunks。
![sample video message to be made into chunks](https://github.com/runner365/read_book/blob/master/rtmp/pic/chunk%20video%20example1.png)
下图是chunk的组成:<br/>
![Format of each of the chunks](https://github.com/runner365/read_book/blob/master/rtmp/pic/chunk%20video%20example2.png)
<br/>
chunk 1的头数据表明了这个message总共307字节长。<br/>
<br/>
从上面两个例子看，chunk type 3能有两种使用方法。第一种是连续的message。第二种是一个新消息其头部能被后续的数据继续使用。

## 5.4 Protocal Control Message(控制消息)
协议控制消息必须是message stream ID等于0(0表示控制协议)，并且chunk stream ID必须是2。协议控制消息在收到后需要即刻处理；他们的时间戳可以忽略。

### 5.4.1 Set Chunk Size（1）
控制消息1，Set Chunk Size，用于通知对端最新的chunk size。<br/>
<br/>
chunk size最大值默认是128bytes，但是客户端和服务器可以改变这个值，并用这个消息来通知对方。举个例子，假想一个客户端想要发送音频包131bytes，且其chunk size是128字节。在这种情况下，客户端可以发送这个控制消息给服务端，通知它chunk size现在修改为131字节了。然后，客户端就可以在一个chunk中发送音频数据了。<br/>
<br/>
最大的chunk size至少是128字节，内容至少1字节。chunk size的最大值每个方向独立维护。<br/>
<br/>
<pre>
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ 
|0|                    chunk size (31 bits)                     | 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        Payload for the ‘Set Chunk Size’ protocol message
</pre>
* 0: 这个bit必须是0;
* chunk size (31 bits): 这个字段包含chunk size的最大值，发送端接下来的chunk都以这个chunk size作为有效尺寸来发送，直到有新的set chunk size消息来到。值的范围是1到2147483647(0x7fffffff)；然后，所有值大于16777215(0xffffff)的都被认为是16777215，因为没有哪个chunk大小会比message还有大，因为message都是小于16777215字节的。

### 5.4.2 Abort Message(2)
协议控制消息2，Abort Message，被用于通知对端如果还在等待chunk包来完成message组播啊，可以丢弃掉已经接收到的chunk报文。对端接收到这个类型协议的报文，其负载是chunk stream ID。一个应用应该在开始关闭连接是发送这个消息，为了对端不用在等待未完成的message。<br/>
<br/>
<pre>
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ 
|                  chunk stream id (32 bits)                    | 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       Payload for the ‘Abort Message’ protocol message
</pre>
chunk stream ID (32 bits):  这个字段包含chunk stream ID。也就是这个chunk stream ID的消息将被丢弃。

### 5.4.3 Acknowledgement(3)
客户端或服务器必须在接收到window size消息后，发送Acknowledgement给对端。这个windows size是发送者还没有接收到acknowledgement前发送最大的字节数。消息定义了sequence number，其是目前已经接收到的字节数。<br/>
<br/>
<pre>
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ 
|                  sequence number (4 bytes)                    | 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       Payload for the ‘Acknowledgement’ protocol message
</pre>
<br/>
sequence number (32 bits):  这个字段表示当前接收到的sequence number。

### 5.4.4 Window Acknowledgement Size(5)
客户端或服务器发送此消息去通知对端window size在发送acknowledgments之前。发送端期望收到acknowledgment在发送端发送完window size的字节数后。接收端必须在接收到windows size的数据后，发送acknowledgement(5.4.3节)，或者从会话一开始还没有acknowledgement被发送前。<br/>
<br/>
<pre>
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ 
|           Acknowledgement Window size (4 bytes)               | 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 Payload for the ‘Window Acknowledgement Size’ protocol message
</pre>

### 5.4.5 Set Peer Bandwidth(6)
客户端或服务器发送这个消息以限制对端发送带宽。对端收到这个消息后通过限制发送数量来限制输出带宽，但是不需要针对这个消息的回复。对端接收到这个消息，如果window size与上次这个消息的发送者发送的不一样，就应该返回一个window acknowledgement size massage。
<br>
<pre>
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ 
|                  Acknowledgement Window size                  | 
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ 
|    Limit Type |
+-+-+-+-+-+-+-+-+
      Payload for the ‘Set Peer Bandwidth’ protocol message
</pre>
</br>
Limit type是如下几种:
* 0 - Hard: 对端应该限制出口带宽到window size。
* 1 - Soft: 对端应该限制其出口带宽到消息中的window size，或限制效果更小的带宽。
* 2 - Dynamic: 如果钱一个Limit Type是Hard，那么就认为这个消息类型是Hard，否则丢弃这个消息。

# 6. RTMP Message Formats

