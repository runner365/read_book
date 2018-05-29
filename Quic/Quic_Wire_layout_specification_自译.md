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
&emsp;&emsp;32位表示QUIC协议的版本。该字段仅仅当public flag设置了FLAG_VERSION后才有(i.e public_flags & FLAG_VERSION !=0)。客户端设置这个flag后，且必须包含一个客户端推荐的quic version，包含任意数据(符合这个版本的)。服务器设置这个flag，仅当客户端推荐的quic version不支持，服务端返回一个列表包含可接受的quic version，但是不必后续带有数据。版本字段例子，"Q025"版本，"Q"在第9个字节，"0"在第10个字节，依次类推。(文档后有版本列表)

* Packet Number: <br/>
&emsp;&emsp;packet number的长度基于FLAG_BYTE_SEQUENCE_NUMBER的flag设置在public flag。每一个常规报文regular packet(也就是非public reset和version negotiation报文)都需要被发送方设置packet number。第一个被发送的报文的packet number应该设置成1，后续的报文的packet number应该+1递增。<br/>
&emsp;&emsp;packet number的64位被放在加密的内容中；因此，QUIC的一方不能发送报文，其packet number不在64bits内。如果QUIC的一方发送的packet number是2^64-1，报文产生CONNECTION_CLOSE报文，错误码是QUIC_SEQUENCE_NUMBER_LIMIT_REACHED，并且不会再发送其他的报文。<br/>
&emsp;&emsp;大部分情况packet number的48bits长度的传输，为了接收端能清晰的对packet number进行组包，QUIC发送端不应该发送packet number大于2^(bitlength-2)。因此48bits长度的packet number不应该大于(2^46)。<br/>
&emsp;&emsp;任何被截断的packet number都应该被推断为最接近已经收到最大packet number，其包含这个截断的packet number。这个packet number的传输比例与推断中的地位bits对应。<br/>
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

</pre><br/>

## Special Packets

### Version Negotiation Packet
&emsp;&emsp;version协商报文仅仅由服务端发送。version协商报文由8bit的public flag和64bit的connect ID。public flag必须设置PUBLIC_FLAG_VERSION，和64位bit的connect ID。报文后续是一个服务器支持version的信息列表，列表每项是4byte的version字段:<br/>
<pre>
--- src
     0        1        2        3        4        5        6        7       8
+--------+--------+--------+--------+--------+--------+--------+--------+--------+
| Public |    Connection ID (64)                                                 | ->
|Flags(8)|                                                                       |
+--------+--------+--------+--------+--------+--------+--------+--------+--------+

     9       10       11        12       13      14       15       16       17
+--------+--------+--------+--------+--------+--------+--------+--------+---...--+
|      1st QUIC version supported   |     2nd QUIC version supported    |   ...
|      by server (32)               |     by server (32)                |             
+--------+--------+--------+--------+--------+--------+--------+--------+---...--+

---
</pre><br/>

### Public Reset Packet
&emsp;&emsp;Public Reset报文由8bit的public flag和64bits的connect ID。public flag必须设置PUBLIC_FLAG_RESET，和64bit的connect ID。如果这是一个带tag PRST加密的握手消息，报文的剩余部分是被加密的(见 [QUIC-CRYPTO]):<br/>
<pre>
--- src
     0        1        2        3        4         8
+--------+--------+--------+--------+--------+--   --+
| Public |    Connection ID (64)                ...  | ->
|Flags(8)|                                           |
+--------+--------+--------+--------+--------+--   --+

     9       10       11        12       13      14       
+--------+--------+--------+--------+--------+--------+---
|      Quic Tag (32)                |  Tag value map      ... ->
|         (PRST)                    |  (variable length)                         
+--------+--------+--------+--------+--------+--------+---
---
</pre>
<br/>

Tag value map: 这个Tag value map有一下tar-values信息:<br/>
* RNON (public reset nonce proof) - a 64-bit unsigned integer. Mandatory.
* RSEQ (rejected packet number) - a 64-bit packet number. Mandatory.
* CADR (client address) - the observed client IP address and port number. 这当前只是用于调试目的，所以是可选的。<br/>

### 常规报文(Regular Packets)
&emsp;&emsp;常规报文加上认证和加密的。Public header是加了认证信息，但是并未加密，常规报文的剩余部分是被加密的。在public header后面，常规报文包含AEAD(authenticated encryption and associated data，认证和被加密的数据)数据。这些数据应该按顺序被解密。解密后，明文应该由按顺序的frame组成。<br/>

### 数据报文(Frame Packet)
&emsp;&emsp;Frame报文的负载由一系列的type前缀的frames组成。报文type的格式后面会描述，总体的格式如下:<br/>
<pre>
--- src
+--------+---...---+--------+---...---+
| Type   | Payload | Type   | Payload |
+--------+---...---+--------+---...---+
---
</pre><br/>

# QUIC连接的生命周期(Life of a QUIC Connection)
## 连接建立(Connection Establishment)
&emsp;&emsp;QUIC客户端是一方发起连接的。QUIC的连接由version协商和加密、传输握手混合进行，以此降低连接的延时。我们下面先介绍version协商。<br/>
&emsp;&emsp;每个客户端发向服务端的初始化报文必须设置version flag，必须定义将要使用version。每个客户端发送的报文都不许带version flag，直到收到服务端返回一个不带version flag的报文。在服务端收到客户端第一个不带version flag的报文后，服务端就必须丢弃所有再收到version flag的报文。<br/>
&emsp;&emsp;当服务端收到一个新的connect ID，它将比较客户端的版本自己是否支持。如果客户端的版本自己自持，服务端将在整个连接周期内用该版本。然后，所有服务端的发送报文都应该清除version flag该标志位。<br/>
&emsp;&emsp;如果客户端的版本不被服务器接收，1个RTT的延时就会触发。服务端将发送Version协商报文给客户端。这个报文的version flag会被设置，并且会包含服务端支持的version列表。<br/>
&emsp;&emsp;当客户端收到version协商报文，会选择其中一个version并用这个version重发所有报文。这些报文必须也设置version flag和包含该version。最终，客户端接收到从服务器来的第一个常规报文开始，表示version协商的结束，客户端之后发送的所有报文都应该去使能version flag。<br/>
&emsp;&emsp;为了避免downgrade攻击，客户端定义在第一个报文的version和服务器支持的version列表都必须包含在加密的handleshake数据中。客户端需要确认在handshake中的version列表和version协商列表进行对比，得到相同一致的。服务端需要确认客户端发来的handshake中的version是否实际支持。<br/>
&emsp;&emsp;连接建立的后续部分将在handshake文档中介绍[QUIC-CRYPTO]。加密的handshake被分配固定的stream ID 1。<br/>
&emsp;&emsp;在连接建立过程中，handshake必须协商各种传输参数。当前已经定义的传输参数再本文后面有介绍。<br/>

## 数据传输(Data Transfer)
&emsp;&emsp;QUIC应用连接可靠性，拥塞控制和流控。QUIC流控基本上同HTTP/2的流控一样。QUIC可靠性和拥塞控制在相关的文档中描述。QUIC连接用唯一的packet sequence数字字段，对整个连接中的拥塞空着和丢包重传都一致。<br/>
&emsp;&emsp;在QUIC连接中传输的所有数据，包括加密的handshake，都是在stream中作为数据传输，ACK返回QUIC报文除外。<br/>
&emsp;&emsp;本节概念上对一个QUIC连接中数据传输中流的使用进行介绍。各个各样的报文会在Frame Type and Formats节进行介绍。<br/>

### QUIC流的生命周期(Life of a QUIC Stream)
&emsp;&emsp;QUIC流是双向发送的数据被分配到流分配包中的很多独立序列。stream能被客户单或服务器创建，能与其他的流一起并发发送数据，并且能停止发送。QUIC流的生命周期模型与HTTP/2的非常相似。[RFC7540]
(QUIC流的HTTP/2用法在本问题后面进行详细描述)<br/>
&emsp;&emsp;针对指定流发送一个流报文，就隐性的创建一个stream。为了避免stream ID冲突，如果是服务端发起stream的话，stream-ID必须是偶数;客户端发起stream的话，stream-ID必须是单数。0不是一个有效的stream-ID。Stream 1给加密的handshake作为第一个客户端端发起stream使用。当应用HTTP/2 over QUIC时，Stream 3为发送所有其他流的压缩头使用，从而确保可靠有序的发送和头部处理。<br/>
&emsp;&emsp;当新流被创建时，连接双方的stream ID应该连续的增长。举例，Stream2应该在Stream 3后创建(stream 3是客户端，stream2是服务端)，但是stream 7肯定不能再stream 9后才创建。对端可能接受的流是无序的。举例，如果在服务端接受packet9包含stream7前，接受到packet10包含stream9，服务器必须能从容处理这样的乱序情况。<br/>
&emsp;&emsp;如果一方收到一个stream包但并不想接收它，它可以立即返回一个RST_STREAM报文(下面会介绍)。注意，虽然发起方已经在该stream中发送数据，但这些数据会被丢弃。<br/>
&emsp;&emsp;一旦流被创建，它就能发送和接收数据。也就是说直到流在某方向结束前，这条流上的报文都能持续的被发送。<br/>
&emsp;&emsp;每个QUIC端都能正常终结stream。有3中终结stream的方法:<br/>
* 正常终结(Normal termination): 因为流是双向的，所以流能是单方向关闭或全关闭。当一方发送的报文带有FIN标志位，就代表单方向关闭。FIN标志着发送FIN的这一方不会再有数据要发送。当QUIC的一方发送并接受了FIN，这方也就被认为完全关闭了。FIN应该放在最后一个用户数据的报文中，但是FIT也能在最后一个用户数据报文后作为空报文发送(有点浪费)
* 突然结束(Abrupt termination):客户端和服务器能发送RST_STREAM在任何时候。RST_STREAM报文包含error错误码解释失败的原因(错误码列表在本文最后)。当RST_STREAM是流发起方发送，表明有错误发生且不会有更多的数据在该流发送。当RST_STREAM是接受者发送，流的发送方在接收到RST_STREAM报文后，应该立即停止任何数据在该流上的发送。流的接收方也应该意识到有个时间间隔在发送方已经发送的数据，和发送方接收到接收方发来的RST_STREAM报文。为了保证连接级别的流控能正确的被计数，即使RST_STREAM报文已经收到，发送方也需要确认：在该流上的FIN和所有数据字节对端已经收到；或对端收到RST_STREAM。也就是说，RST_STREAM的发送端需要继续用正确的WINDOW_UPDATEs响应这条流上的数据，保证发送方不会有流控阻塞，保证其完成FIN的发送。
* 当连接断开，流肯定也断开，在下面一节会详细介绍连接断开。
<br/>

## 连接断开(Connection Termination)
&emsp;&emsp;连接保持打开状态直到变成空闲状态一段设定的时间。当服务端要断开一个空闲连接，它不需要通知客户端，这回导致移动设备的唤醒信号。QUIC连接一段建立，有两种方式可以结束:<br/>
* 显式关闭(Explicit Shutdown): 一方发送CONNECTION_CLOSE报文给另外一方表明连接开始中断。一方也可以发送GOAWAY报文给另外一方，而不是用CONNECTION_CLOSE，GOAWAY表明连接很快将关闭。GOAWAY发送到对端后，对端继续对所有活跃的报文进行处理，但是GOAWAY的发送方不再发送新的报文，也不在接收任何新的数据报文。对活跃流的结束，也可以发送CONNECTION_CLOSE。如果当为结束的流是活跃的(没有FIN或RST_STREAM报文被发送或接收)，一方发送CONNECTION_CLOSE报文，那么对端就认为流未完成，已经被非正常结束。
* 隐式关闭(Implicit Shutdown):默认的QUIC连接的空闲超时是30秒，在连接协商中有个参数"ICSL"定义。最大值是10分钟。如果在空闲超时时间内没有任何网络活跃，连接会关闭。默认情况下CONNECTION_CLOSE将发送。当发送显示关闭太浪费，如移动网络会唤醒手机信号，"静音"关闭的选项被使能。
&emsp;&emsp;QUIC的一方在任何连接获取的时候，也能通过发送PUBLIC_RESET来终结连接。PUBLIC_RESET的PUBLIC_RESET是等价于TCP的RST。<br/>

# 报文类型和格式(Frame Types and Formats)
&emsp;&emsp;QUIC报文是以方式存在，报文都有报文类型(frame type)，类型有完全独立的解释，后面跟随fream header字段。所有的frame都被包含在QUIC报文中，没有哪个frame会越过QUIC报文的边界。<br/>
## 报文类型(Frame Types)
&emsp;&emsp;对于报文类型有两种解释，也就由此定义两种报文类型:
* 特殊报文(Special Frame Types)<br/>
特殊报文包含frame type和flag信息在frame type的字段中<br/>
* 常规报文(Regular Frame Types)<br/>
常规报文只包含frame type在frame type的字段中<br/>

当前定义的特殊报文类型(Special Frame Types):<br/>
<pre>
--- src
   +------------------+-----------------------------+
   | Type-field value |     Control Frame-type      |
   +------------------+-----------------------------+
   |     1fdooossB    |  STREAM                     |
   |     01ntllmmB    |  ACK                        |
   |     001xxxxxB    |  CONGESTION_FEEDBACK        |
   +------------------+-----------------------------+
---
</pre><br/>
当前定义的常规报文类型(Regular Frame Types):
<pre>
--- src
   +------------------+-----------------------------+
   | Type-field value |     Control Frame-type      |
   +------------------+-----------------------------+
   | 00000000B (0x00) |  PADDING                    |
   | 00000001B (0x01) |  RST_STREAM                 |
   | 00000010B (0x02) |  CONNECTION_CLOSE           |
   | 00000011B (0x03) |  GOAWAY                     |
   | 00000100B (0x04) |  WINDOW_UPDATE              |
   | 00000101B (0x05) |  BLOCKED                    |
   | 00000110B (0x06) |  STOP_WAITING               |
   | 00000111B (0x07) |  PING                       |
   +------------------+-----------------------------+
---
</pre><br/>

## 流报文(STREAM Frame)
&emsp;&emsp;流报文被隐式的创建流并发送报文，格式如下:<br/>
<pre>
--- src
     0        1       …               SLEN
+--------+--------+--------+--------+--------+
|Type (8)| Stream ID (8, 16, 24, or 32 bits) |
|        |    (Variable length SLEN bytes)   |
+--------+--------+--------+--------+--------+

  SLEN+1  SLEN+2     …                                         SLEN+OLEN   
+--------+--------+--------+--------+--------+--------+--------+--------+
|   Offset (0, 16, 24, 32, 40, 48, 56, or 64 bits) (variable length)    |
|                    (Variable length: OLEN  bytes)                     |
+--------+--------+--------+--------+--------+--------+--------+--------+

  SLEN+OLEN+1   SLEN+OLEN+2
+-------------+-------------+
| Data length (0 or 16 bits)|
|  Optional(maybe 0 bytes)  |
+------------+--------------+
---
</pre><br/>
流报文头部的各个字段描述如下:
* Frame Type: 报文类型是8bit大小，包含各种flag信息(1fdooossB)<br/>
&emsp;&emsp; * 最左边的bit设置1，表示这是个流报文。The leftmost bit must be set to 1 indicating that this is a STREAM frame.<br/>
&emsp;&emsp; * f标志位是FIN表示，当设置为1，表示发送端完成该流的发送，并希望半双工关闭(后续详细介绍)<br/>
&emsp;&emsp; * d表示Data length会在STREAM头部存在，如果设置为0，表示流报文的长度会是一直到报文末尾。<br/>
&emsp;&emsp; * ooo表示offset字段的长度，000~111分别表示0, 16, 24, 32, 40, 48, 56, or 64 bits长度。<br/>
&emsp;&emsp; * ss表示Stream ID的长度，00~11分别表示8, 16, 24, or 32 bits长度。<br/>
* Stream ID: 可变长度的无符号整型ID，标识唯一的流。</br>
* Offset: 可变长度的无符号整型，表示流数据块开始的偏移位置。</br>
* Data length: (可选)16bit长的无符号整型，标识报文中的数据长度。如果不需要此字段，表示offset后到末尾的所有字节都是数据，后面没有padding数据。<br/>
一个流报文肯定要么有非0的数据长度，要么数据长度为0但是FIN标志位被设置1。<br/>
