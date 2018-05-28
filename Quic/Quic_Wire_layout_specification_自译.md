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

## 数据流与连接的流控
&emsp;&emsp;QUIC用的是流和连接级别的流控，类似HTTP/2's流控。QUIC流级别的流控工作如下。QUIC接受者发送字节偏移量，也就是接受者针对每条流能接受的字节数。当在某条流上的数据的收和发，接受者都会发送WINDOW_UPDATE报文增加流的字节偏移量，来允许对端发送更多的数据。<br/>
&emsp;&emsp;除了基于流的流控外，QUIC也提供连接级别的流控来限制聚合bffer，其控制QUIC接受者分配一个连接。连接流控的工作方式同流的流控方式一样，只是字节的发送和接收偏移量是针对所有流的。<br/>
&emsp;&emsp;同TCP的接收窗口自动调节机制一样，QUIC对流和连接的流控应用信用自动调节的机制。当接收应用比较慢时，如果需要限制发送者的速率，QUIC自动调节每个WINDOW_UPDATE报文的信用size。

## 多路复用
&emsp;&emsp;HTTP/2在TCP上有头部阻塞的问题。应为HTTP/2是多流复用的会造成头部阻塞的问题，一小片TCP报文的丢失会阻塞住后续所有的分片，直到这小片的重传能收到，完全不在乎后面的HTTP/2分片。
&emsp;&emsp;因为QUIC设计初衷就是为了多路复用，对于某一路流的丢包应该只影响该路流。每路流能马上被调度当收到报文，哪些没有报文丢失的流应该能被包重组和正常继续其应用。

## 认证和加密头和数据负载
对认证和加密不太熟悉，本节跳过。

## 连接迁移
&emsp;&emsp;TCP的连接由4元组定义: 源IP，源port，目的IP，目的port。TCP最著名的问题就是连接无法容忍IP地址变化(举例，WIFI迁移到移动网络)或者端口的变化(如当客户端的NAT绑定过去造成端口的变化)。当MPTCP导致TCP连接迁移，有个很大的困扰就是缺少中间件支持和缺少OS操作系统级别的支持。<br/>
&emsp;&emsp;QUIC连接由64bits的connectID定义，有客户端生成个随机数。QUIC能继续连接，即使IP变化或NAT重绑定发生，只要在迁移过程中connectID保持不变。QUIC也提供了自动的加密认证的客户端变化方式，因为迁移的客户端会继续用同一个会话key来进行加密和认证。<br/>
&emsp;&emsp;在某些特定场景中，如果连接可以被IP4元组唯一定义，且该4元组不会变化，可以选择不包含connectID进行连接。

# 包类型和格式
&emsp;&emsp;QUIC有特殊包(Special Packets)和常规包(Regular Packets)。
&emsp;&emsp;有两种特殊包(Special Packets):
* 版本协商报文(Version Negotiation Packets)
* public重置报文(Public Reset Packets)

&emsp;&emsp;常规包(Regular Packets)只包括数据报文。<br/>
&emsp;&emsp;所有的QUIC报文都应该适配传输路径的MTU大小，以避免IP分片。路径MTU发现还在研究中，当前推荐IPV6最大的MTU是1350字节，IPv4是1370字节。这里说的字节数是不包括IP头和UDP头的。

## QUIC的公共头(Public Packet Header)
&emsp;&emsp;所有QUIC报文的公共头都是1~51字节，格式如下:<br/>

