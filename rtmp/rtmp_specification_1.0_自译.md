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