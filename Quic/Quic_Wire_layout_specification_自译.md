# QUIC概述
&emsp;&emsp;本节我们主要介绍QUIC的关键功能和优点。QUIC功能上等于TCP+TLS+HTTP/2，但是基于UDP传输的。QUIC优于TCP+TLS+HTTP/2的关键点有:<br/>
* connect连接建立的低延时
* 灵活的拥塞控制
* 无头部阻塞的多路复用(TCP是有头部阻塞的)
* 对头部和负载进行认证和加密
* 流和连接的流控
* 连接迁移

## connection连接低延时
&emsp;&emsp;QUIC把加密和传输的握手合并，降低了安全连接建立的通信来回次数。QUIC的连接建立过程是0-RTT，也就是说大部分的QUIC连接，数据能立马发送二不用等待服务器的返回，相比之下TCP+TLS的1-3次握手后才能通信。<br/>
&emsp;&emsp;QUIC提供一个特定的流(streamid=1)来进行握手，本文不详细描述握手协议。如果想要连接握手协议，可以访问[QUIC Crypto Handshake](https://docs.google.com/document/d/1g5nIXAIkN_Y-7XJW5K45IblHd_L2f5LTaDUDwvZ5L6g/edit)。当前的QUIC握手未来会被TLS1.3代替。

## 灵活的拥塞控制
&emsp;&emsp;QUIC比TCP有可插拔的拥塞控制和丰富的信令，相对于TCP，这些新信令能为QUIC提供很多的信息去做拥塞控制算法。当前，默认的拥塞控制是应用TCP Cubic；我们将会经历更多的可选的拥塞控制方式。<br/>
&emsp;&emsp;举例，每个quic报文，无论是源报文还是重传报文，都携带一个新的sequence号。不同的sequence号帮助发送端确认ACK信息是重传包的还是原始包的，因此避免了TCP重传模糊的问题。QUIC ACK也肯定产生包接收和ACK发送之间的延时，因为有递增的sequence，也就能准确计算出RTT。<br/>
&emsp;&emsp;最后，QUIC的ACK报文支持256个ack，所以QUIC的伸缩性强于TCP(用的SACK)，当乱序和丢失发生就能发送更多的字节。客户端和服务端都有更精确的哪些报文已经收到。

