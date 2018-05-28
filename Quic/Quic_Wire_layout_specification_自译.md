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
&emsp;&emsp;TCP的连接由4元组定义: 源IP，源port，目的IP，目的port。TCP最著名的问题就是连接无法容忍IP地址变化(举例，WIFI迁移到移动网络)或者端口的变化(如当客户端的NAT绑定超时造成端口的变化)。当MPTCP导致TCP连接迁移，有个很大的困扰就是缺少中间件支持和缺少OS操作系统级别的支持。<br/>
&emsp;&emsp;QUIC连接由64bits的connectID定义，有客户端生成个随机数。QUIC能继续连接，即使IP变化或NAT重绑定发生，只要在迁移过程中connectID保持不变。QUIC也提供了自动的加密认证的客户端变化方式，因为迁移的客户端会继续用同一个会话key来进行加密和认证。<br/>
&emsp;&emsp;在某些特定场景中，如果连接可以被IP4元组唯一定义，且该4元组不会变化，可以选择不包含connectID进行连接。<br/>

# 包类型和格式
&emsp;&emsp;QUIC有特殊包(Special Packets)和常规包(Regular Packets)。<br/>
&emsp;&emsp;有两种特殊包(Special Packets):<br/>
* 版本协商报文(Version Negotiation Packets)
* public重置报文(Public Reset Packets)

&emsp;&emsp;常规包(Regular Packets)只包括数据报文。<br/>
&emsp;&emsp;所有的QUIC报文都应该适配传输路径的MTU大小，以避免IP分片。路径MTU发现还在研究中，当前推荐IPV6最大的MTU是1350字节，IPv4是1370字节。这里说的字节数是不包括IP头和UDP头的。

## QUIC的公共头(Public Packet Header)
&emsp;&emsp;所有QUIC报文的公共头都是1~51字节，格式如下:<br/>
<pre>
--- src
     0        1        2        3        4            8
+--------+--------+--------+--------+--------+---    ---+
| Public |    Connection ID (64)    ...                 | ->
|Flags(8)|      (optional)                              |
+--------+--------+--------+--------+--------+---    ---+

     9       10       11        12   
+--------+--------+--------+--------+
|      QUIC Version (32)            | ->
|         (optional)                |                           
+--------+--------+--------+--------+


    13       14       15        16      17       18       19       20
+--------+--------+--------+--------+--------+--------+--------+--------+
|                        Diversification Nonce                          | ->
|                              (optional)                               |
+--------+--------+--------+--------+--------+--------+--------+--------+

    21       22       23        24      25       26       27       28
+--------+--------+--------+--------+--------+--------+--------+--------+
|                   Diversification Nonce Continued                     | ->
|                              (optional)                               |
+--------+--------+--------+--------+--------+--------+--------+--------+

    29       30       31        32      33       34       35       36
+--------+--------+--------+--------+--------+--------+--------+--------+
|                   Diversification Nonce Continued                     | ->
|                              (optional)                               |
+--------+--------+--------+--------+--------+--------+--------+--------+

    37       38       39        40      41       42       43       44
+--------+--------+--------+--------+--------+--------+--------+--------+
|                   Diversification Nonce Continued                     | ->
|                              (optional)                               |
+--------+--------+--------+--------+--------+--------+--------+--------+


    45      46       47        48       49       50
+--------+--------+--------+--------+--------+--------+
|           Packet Number (8, 16, 32, or 48)          |
|                  (variable length)                  |
+--------+--------+--------+--------+--------+--------+
</pre>
<br/>
&emsp;&emsp;负载会包含类型独立的头部字节，描述如下。<br/>
&emsp;&emsp;公共头字段如下:<br/>
* Public Flags<br/>
&emsp;&emsp;* 0x01 = PUBLIC_FLAG_VERSION. 这个flag的含义在于报文由服务器还是客户端发出。当报文由客户端发出，设置改bit意味着头部包含有QUIC version(如下)。客户端必须设置该bit，直到服务端返回运行的version。服务端同意客户端的version，但服务端发送的报文中并不设置该标志位。如果服务端发送的报文设置该bit，意味该报文是version协商报文。version的协商将在后面进行讨论。<br/>
&emsp;&emsp;* 0x02 = PUBLIC_FLAG_RESET. 该bit位表示Public Reset packet报文。<br/>
&emsp;&emsp;* 0x04 表示在头部有32字节的多元化标志。<br/>
&emsp;&emsp;* 0x08 表示报文有全8字节的connect ID。该bit必须在所有报文中设置，直到有不同的值产生(举例，客户端可能需要connect id更少的字节)<br/>
&emsp;&emsp;* 0x30 这两个字节的占位表示packet number需要字节的数量。这两个bit仅仅正对数据报文。对于public reset和version negotiation报文(服务端发送的)，这两个字节的占位设置为0。<br/>
&emsp;&emsp;&emsp;&emsp;* 0x30 表示packet number字段有6个字节的长度<br/>
&emsp;&emsp;&emsp;&emsp;* 0x20 表示packet number字段有4个字节的长度<br/>
&emsp;&emsp;&emsp;&emsp;* 0x10 表示packet number字段有2个字节的长度<br/>
&emsp;&emsp;&emsp;&emsp;* 0x00 表示packet number字段有1个字节的长度<br/>
&emsp;&emsp;* 0x40 保留为多路径用途<br/>
&emsp;&emsp;* 0x80 未使用，必须设置为0

* Connection ID: <br/>
&emsp;&emsp;这个是客户端生成的64位bit的随机数，标识连接的唯一性。因为QUIC的连接设计初衷是即使客户端IP迁移，连接也不中断，IP4元组(源IP，源port，目的IP，目的port)并不需要去确定连接的唯一性。如果对于某个传输的方向，IP4元组能代表连接的唯一性(其实就是不可能发生IP迁移等)，connect ID字段也就不需要了。

* QUIC Version: <br/>
&emsp;&emsp;32位表示QUIC协议的版本。该字段仅仅当public flag设置了FLAG_VERSION后才有(i.e public_flags & FLAG_VERSION !=0)。客户端设置这个flag后，且必须包含一个客户端推荐的quic version，包含任意数据(符合这个版本的)。服务器设置这个flag，仅当客户端推荐的quic version不支持，服务端返回一个列表包含可接受的quic version，但是不必后续带有数据。版本字段例子，"Q025"版本，"Q"在第9个字节，"0"在第10个字节，依次类推。(文档后有版本列表)

* Packet Number: <br/>
&emsp;&emsp;packet number的长度基于FLAG_BYTE_SEQUENCE_NUMBER的flag设置在public flag。每一个常规报文regular packet(也就是非public reset和version negotiation报文)都需要被发送方设置packet number。第一个被发送的报文的packet number应该设置成1，后续的报文的packet number应该+1递增。<br/>
&emsp;&emsp;packet number的64位被放在加密的内容中；因此，QUIC的一方不能发送报文，其packet number不在64bits内。如果QUIC的一方发送的packet number是2^64-1，报文产生CONNECTION_CLOSE报文，错误码是QUIC_SEQUENCE_NUMBER_LIMIT_REACHED，并且不会再发送其他的报文。<br/>
&emsp;&emsp;大部分情况packet number的48bits长度的传输，为了接收端能清晰的对packet number进行组包，QUIC发送端不应该发送packet number大于2^(bitlength-2)。因此48bits长度的packet number不应该大于(2^46)。<br/>
&emsp;&emsp;人也被截断的packet number都应该被推断为最接近已经收到最大packet number，其包含这个截断的packet number。这个packet number的传输比例与推断中的地位bits对应。<br/>
&emsp;&emsp;Public Flag的处理流程如下: <br/>
<pre>
--- src
Check the public flags in public header
                 |
                 |
                 V
           +--------------+
           | Public Reset |    YES
           | flag set?    |---------------> Public Reset Packet
           +--------------+
                 |
                 | NO
                 V
           +------------+          +-------------+
           | Version    |   YES    | Packet sent |  YES
           | flag set?  |--------->| by server?  |--------> Version Negotiation
           +------------+          +-------------+               Packet
                 |                        |
                 | NO                     | NO
                 V                        V
           Regular Packet         Regular Packet with 
                              QUIC Version present in header
---

</pre>




