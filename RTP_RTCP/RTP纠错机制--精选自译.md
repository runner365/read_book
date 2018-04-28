# 9. 纠错机制
## 9.1 前向纠错(FEC)
&emsp;&emsp;前向纠错(FEC)算法能让二进制流在传输过程中保持健壮性。传送大量二进制流在松散的媒介或网络下。额外的信息会加在二进制流中，能让接受者正确的重构在传输中丢失的数据。前向就戳算法主要应用在广域网，如手机网络、或包交换网络、或存储系统(如压缩盘、电脑硬盘或内存)。因为因特网是一个松散的媒介，因为媒体应用的信息对丢失非常敏感，FEC方案就被提议和编程RTP应用的标准。FEC方案能完成解压，和准确重构二进制流，取决FEC类型、应用数量，和丢失的方式。<br/>
&emsp;&emsp;当RTP应用FEC，就必须设计一下加入多少个FEC报文，取决于网络丢失数据的大小。一个方法就是，用RTCP RR报文中的loss fraction(丢失率)来决定加入多少FEC数据到媒体流中。<br/>
&emsp;&emsp;理论上，通过改变媒体编码，是有可能确保一定丢失率下的纠错。但实际上，很多情况都表明，FEC只提供可能性的修复。关键点在于FEC报文的加入也增加了带宽消耗。带宽增加局限了FEC加入到有容量限制网络的总数，如果是因为网络拥塞造成丢包，FEC有可能造成更坏的影响。因为，带宽消耗增加会加重网络拥塞。这个会在本章后面的章节讨论。<br/>
&emsp;&emsp;注意虽然FEC的数量根据接收者质量报告而变化，对报文丢失并不发送反馈，并且也不能保证所有的丢失报文都能修复。目的是让丢失率降低到能接受的丢失率，然后让错误隐藏解决剩下的错误率。<br/>
&emsp;&emsp;如果FEC正常工作的话，错误纠正肯定是局限的，而且只在某些局限的模式下，丢失报文才能被纠正。比如说，FEC方案只能去修复小于5%的丢失率，如果报文丢失率高达10%就无法正常纠错了。另外一个限制是，在%5丢失率下，只有非连续报文的丢失才能被修复(sequence连续的报文丢失，FEC是无法修复的)。<br/>
&emsp;&emsp;FEC最大的优点是，它能覆盖很大的通信组，或者通信组直接不需要错误反馈。基于丢失率和丢失模式场景，加入一定量的纠错冗余的报文，所有接收者都能兼容。<br/>
&emsp;&emsp;FEC缺点是过分依赖平均报文丢失率。当前报文丢失率低于平均的接收者接收到FEC冗余纠错报文，这些纠错报文就仅仅是浪费带宽，并肯定当做无用而丢弃。而高于平均丢失率的接收者就无法修复这些错误，只能靠错误隐藏来解决这些错误。如果各个接收者的错误率是不一样的，仅仅一种FEC方式是无法让所有接收者满意。<br/>
&emsp;&emsp;FEC的其他缺点是，FEC会引入演示，因为修复必须要等到收到FEC报文。如果保护某些RTP报文的FEC报文一直没有收到，那接收者只能要么播放破损的RTP报文，要么等待FEC到来，但等待肯定就增加了延时。对于交互性很强的应用影响很大，因为这些应用非常需要低延时。<br/>
&emsp;&emsp;FEC的方案很多，有几种FEC对RTP协议非常适合。我们先过一下一些独立于音视频的FEC技术--校验FEC和里德.所罗门编码，今后在讨论针对特定音频、视频的FEC。<br/>

### 9.1.1 Parity FEC(校验FEC)
&emsp;&emsp;一种最简单的FEC编码方式就是校验编码。校验计算数学上叫做比特流的异或，XOR。XOR是基于比特位的逻辑操作，两个输入下的操作是。<br/>
<pre>
0 XOR 0 = 0
1 XOR 0 = 1
0 XOR 1 = 1
1 XOR 1 = 0
</pre>
计算公式能容易扩展到多个XOR输入:<br/>
<pre>
A XOR B XOR C = (A XOR B) XOR C = A XOR (B XOR C)
</pre>
&emsp;&emsp;改变一个XOR的输入会改变结果，那么用XOR的结果就可以去检测错误。这个XOR的能力靠这些值的多少来限制，但是当多个比特位都进行XOR，就能监测和修正错误。<br/>
&emsp;&emsp;把XOR用在RTP系统中，关键点就是错误限制在丢包，而不是包内部比特位错误--也就是必须发送单独的保护媒体数据的XOR报文。如果有足够的XOR比特数据，这些XOR比特数据就能拿来修复那个丢失的报文，这个修复的方法如下:<br/>
<pre>
A XOR B OXR B = A 
</pre>
&emsp;&emsp;对于所有类型bit值，如果我们分别发送3个值：A, B 和 A X0R B，这3个报文只要收到其中两个，另外一个都能被恢复。图9.1显示7为bit的修复过程，当然其实能对任意长度的bit流进行修复。这个过程会被直接应用到RTP报文中，就是把整个RTP报文当做一组bit流，然后计算几个RTP报文的XOR结果，这个XOR结果就能用作丢失恢复。<br/>
![Parity_error](https://github.com/runner365/read_book/blob/master/RTP_RTCP/pic/Parity_revover.png)<br/>
&emsp;&emsp;RTP的校验FEC标准在RFC2733。这个RFC标准内容是定义RTP报文总体FEC方案，满足所有类型的负载类型payload type，并且后向兼容那些不处理FFEC报文的接受者。FEC报文是通过源RTP报文计算得到的。这些FEC报文作为独立的RTP报文来发送，FEC报文能用来修复源报文的丢失，如图9.2。<br/>
![Parity_FEC_Repair](https://github.com/runner365/read_book/blob/master/RTP_RTCP/pic/Parity_FEC_Repair.png)<br/>
### 9.1.2 校验FEC报文格式
&emsp;&emsp;FEC报文格式如图9.3，主要有3部分:<br/>
* 标准RTP头
* FEC负载头
* 负载数据本身

&emsp;&emsp;FEC报文是通过对被保护的RTP报文组进行XOR计算产生的，RTP报文计算内容是算上rtp header extension相关信息的。也就是对所有的RTP报文对应内容进行XOR计算，结果保存在FEC中。<br/>
![FEC_Packet_Format](https://github.com/runner365/read_book/blob/master/RTP_RTCP/pic/FEC_Packet_Format.png)<br/>
&emsp;&emsp;RTP头的字段具体如下：<br>
* version, playload type, sequence number 和timestamp和普通的RTP报文方式一样。payload type可以基于RTP规程动态定义；sequence对每个FEC报文进行自增;timestamp是要保护RTP报文的时间戳时钟，具体数值是记录FEC的时刻。(FEC的timestamp不一定等于媒体RTP的timestamp)。也就是说，FEC报文的时间戳单独自增，完全独立。
* padding，extension，CC和marker bits这4个字段，是通过对原始报文对应字段进行XOR计算得到。如果原始报文丢失，这些原始报文中的字段就能被恢复。
* CSRC列表和header extension不在FEC头中，CC和X字段完全独立。如果csrc列表和header extension在源RTP报文中，这些内容会被包含在FEC报文的payload负载内容中(就是在FEC payload header后)<br/>

&emsp;&emsp;FEC payload header保护源RTP头字段，这些字段没有在FEC报文的RTP header中出现，具体有这6个字段:<br/>
* sequence number base. 被保护RTP报文组最小的sequence number
* Length recovery. 原始RTP负载报文组各自长度的XOR结果。各自的长度是指payload数据，csrc列表，header extension，和padding。当不知道丢失报文的payload长度是，通过FEC计算就可以得出。
* Extension(E). 表示FEC头中是否有额外的字段。通常该字段设置为0，表示没有额外字段(ULP格式，在本章后面介绍，用extension字段表示多层FEC)
* Mask. 这个是bit mask，表示在sequence number base只有有哪些报文是在FEC保护中。如果bit i被设置成1，就是说 sequence number N+i已经被计算在FEC报文中，属于被保护对象了，这里的N就是sequence number base。最小i=0，最大i=23，也就是说FEC最大能一次保护24个RTP报文，这些报文可以是不连续的。
* Timestamp recovery. 该字段是原始报文的timestamp XOR结果。<br/>

&emsp;&emsp;FEC payload data 是 CSRC list(如果存在)，header extension(如果存在) 和 源RTP报文payload data的XOR结果。如果一组RTP报文的数据长度都各不相同，最小的那个RTP报文就用pad来填充到最大那个RTP报文的长度(这个padding bits具体内容并不重要，只要每次保持用一样的值就可以了，通常使用0值就可以了)。

## 9.3 纠错重传
&emsp;&emsp;如果接受者向发送者发送消息，索要丢失的报文，丢失的报文就能避免。对于纠错机制来说，重传是常规的方法，它能在各种场景中应用。当然重传也有其应用的局限性。重传并不是RTP标准协议的一部分；但是，RTP配套的文档也在发展中，它应用基于RTCP的框架来发送重传请求和其他的即时反馈机制。<br/>
### 9.3.1 RTCP的重传机制
&emsp;&emsp;因为RTP包括了基于RTCP通道的反馈机制，如RR和其他的数据，很子软的也就用RTCP通道来进行重传请求。需要两部分: <br/>
+ RTCP重传请求报文格式定义
+ 允许立即反馈的时间准则
### 9.3.2 RTCP重传请求格式
&emsp;&emsp;RTP配套体系中对于重传请求，定义了两种额外的RTCP报文：<br/>
+ 收包反馈: 反馈一组已经收到的报文
+ 丢包反馈: 最常用的是丢包反馈，向发送者报告一组丢失的报文<br/>
&emsp;&emsp;丢包反馈格式(negative acknowledgment，今后简称NACK)，在图9.11中显示。NACK包含一个packeg identifier，其标识一个丢失的报文，其后面的字段bitmap of lost packets标识该报文后面16个报文哪几个丢失了，1代表丢失，0表示收到。即使收到NACK报文中bitmap字段中有0，发送者也不应该假想接受者已经收到该报文；只能确认到接受者当前并没有上报该RTP报文丢失。当接收到一个NACK，发送者应该重传NACK中标识丢失的RTP报文，虽然协议中没有强制这么做。<br/>
![RTCP_NACK](https://github.com/runner365/read_book/blob/master/RTP_RTCP/pic/RTCP_NACK.png)<br/>
&emsp;&emsp;ACK格式如图9.12所示。ACK报文包含RTP sequnece，其代表已经收到的RTP报恩，和bitmap或后面的报文个数。如果R设置1，表示后面的报文个数，如果R设置0，表示bitmap后面15个具体哪几个已经也收到了。这两个选择允许ACK针对少量丢失(R=1)，和ACK分散的丢失(R=0)。<br/>
![RTCP_ACK](https://github.com/runner365/read_book/blob/master/RTP_RTCP/pic/RTCP_ACK.png)<br/>
&emsp;&emsp;ACK和NACK用哪一个取决于具体修复算法和期望的机制。ACK表示部分报文收到了;发送者认为其他的丢失了。相反，NACK表示部分报文丢失了，而其他的报文也许收到，也许没收到(比如，当某个重要报文丢失需要重传，接受者针对这个报文发送NACK，但忽略其他不重要的报文信息)。<br/>
&emsp;&emsp;这样的反馈报文的发送也是作为RTCP组合报文的一部分，同其他的RTCP报文一样。反馈报文放在组合报文的最后，如在SR/RR和SDES后面。(见第五章，RTCP，具体RTCP格式介绍)<br/>