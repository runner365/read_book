[原版英文书链接(RTP:Audio and video for the internet.pdf)](https://github.com/runner365/read_book/blob/master/RTP_RTCP/RTP%EF%BC%9AAudio%20and%20video%20for%20the%20Internet.pdf)<br/>
RTP协议比较简单，因此从第5章节RTCP开始。

## 5.1 RTCP的组件
&emsp;&emsp;一个RTCP的应用有3个部分: 报文结构、时间规则，和参与者数据库。<br/>
有几种类型的RTCP报文。5种标准的报文类型会在5.3 RTCP报文格式中描述，并随后附有报文集成放入复合报文中被发送。验证RTCP报文正确性的算法会在报文有限性的章节中描述。<br/>
&emsp;&emsp;这样复合的报文会被周期发送，基于规定的准则，这个会在后面章节Timing rules中描述。发送报文的间隔被叫做"报告间隔"。所有的RTCP活动都是在这些报告间隔中发生。除了报文发送间隔外，基于这个时间间隔，接受者的质量统计也会被计算出来，并且更新源描述和音视频同步之间的时间也需要计算出来。这些时间间隔根据会话大小或场景中的报文格式而变化。<br/>
&emsp;&emsp;典型的例子，针对小型的会话，间隔时间可以设置成5秒；而那些大型的会话组，可以把时间间隔设置成几分钟。<br/>
&emsp;&emsp;发送者应该特别考虑设定的报告间隔，因为source description和音视频同步信息发送比较频繁。而接受者的报告就没有那么频繁。<br/>
&emsp;&emsp;每个会话应用都会维护一个数据库，数据库的信息来源于接收到的RTCP报文。这个数据库被用于填写周期发送的RR接受者报告报文发，也用于接收音频、视频的同步和维护source description信息。<br/>
&emsp;&emsp;参与者数据库隐私安全部分会在后面的安全与隐私章节描述。

## 5.2 RTCP报文的传输
&emsp;&emsp;每个RTP的会话是由一个IP地址和一对UDP port组成: <br/>
* 一个为rtp
* 一个为rtcp<br/>
&emsp;&emsp;rtp的数据端口号是偶数，rtcp的端口号为rtp端口号+1。举例，如果媒体数据发送到udp port 5004，那么rtcp就会发送到5005. 所有会话中的参与者都应该发送组合的RTCP报文，相反的，也将接收到其他参与者发送的RTCP组合报文。既然所有的反馈都会在多方会话中被发送给所有参与者:单播就发送给中转服务器，它会中转数据，或者可以直接通过组播。RTCP的点到点性质让每个会话中的参与者都能得到其他参与者的信息: 是否在线、接收质量、个人信息，比如姓名、email、地址和电话号码。
## 5.3 RTCP报文格式
在RTP规范中，定义了5种RTCP报文:
* SR: 发送者报告, type-200
* RR: 接受者报告, type-201
* SDES: source description 源信息报告, type-202
* BYE: 成员管理报告, type-203
* APP: 自定义报告, type-204
* RTP_Feedback: RTCP Transport Layer Feedback Packet, type-205
* PS_Feedback: RTCP Payload Specific Feedback Packet, type-206
<br/>
RTP fmt:<br/>
* RTCP_PLI_FMT(1): picture重传, type-206
* RTCP_SLI_FMT(2): Slice重传, type-206
* RTCP_FIR_FMT(4): 关键帧重传, type-206
* RTCP_AFB(15): 带宽估计, type-206
* RTCP_RTP_FB_NACK_FMT(1): NACK重传, type-205

图5.1描述基本的RTCP格式。
![basic rtcp format](https://github.com/runner365/read_book/blob/master/RTP_RTCP/pic/basic_rtcp_packet_format.png)
<br/>
&emsp;&emsp;5种格式都是4字节为长度单位，组成部分如下:
+ Version number(V). 当前版本为2，也可以说没有其他版本在用。
+ Padding(P). 这个标志位表示是否补位。如果这个位置1，表示报文最后会有补齐。关于这个的使用注意事项，会在后面的章节详细描述
+ Item count(IC). 除了一些固定的报文信息，一些报文后面会有项目列表。IC就是表示项目列表的数量。
(这个字段对于不同的RTCP类型有不同的含义)。最大支持31个项，当然也受限于网络单元。如果超过31项要发送，需要分解成多个RTCP来<br/>
发送。IC=0以为项为空(但是并不以为报文为空)。报文的一些类型IC做其他的用途。
+ Packet type(PT). PT表示报文类型，5中标准的报文类型已经被定义，更多的报文类型会在未来被协议定义。
+ Length. 这个字段表示报文头后面内容的长度，单位是32bits，也就是4字节。所以如果你用字节来标识长度的话，可定会造成不连续<br/>
的问题。0也是合理的值，表示只有头信息不符(这时头中的IC字段也肯定是0)

&emsp;&emsp;在RTCP头后面的就是RTCP数据(数据基于具体的RTCP类型)和可选的pad补充字段。多个RTCP头和数据的合成在一起就是一个RTCP报文，当然只一个RTCP头和数据也可以。5种标准报文在本章会介绍。RTCP报文很少单独发送;相反RTCP是一起打包发送。每一个RTCP报文会被UDP/IP封装发送。如果RTCP报文需要加密,那么RTCP报文会有前缀。这个RTCP报文的结构会在图5.2中显示:<br/>
![RTCP_transport](https://github.com/runner365/read_book/blob/master/RTP_RTCP/pic/RTCP_Trasnport.png)<br/>
&emsp;&emsp;RTCP报文的组合打包发送，具体的组包方式会在本章的"Packing Issues"中描述。下面详细介绍5中标准的RTCP报文。
## 5.4 RTCP RR: Receiver Reports
&emsp;&emsp;RTCP报文中重要的一项是接收者质量报告，它通过RR报文来完成，由数据接收方发送。<br/>
### 5.4.1 RTCP RR的格式
&emsp;&emsp;RR报文的RTCP类型是201，报文格式在图5.3所示。接收者报文包含了rtp接收者的SSRC，后面跟着0个或多个报告数据，具体个数由RC字段定义<br/>
![rtcp_rr_packet](https://github.com/runner365/read_book/blob/master/RTP_RTCP/pic/RTCP_RR_PACKET.png)<br/>
&emsp;&emsp;每个RR报文内容都描述了一个时间间隔接收RTP报文的质量情况。每个RR头后面最大能携带31个RR报告块。如果超过31个报告块，就要划分成多个RTCP报文发送。每个RR的报告数据库有7个字段，共24个字节。(这里说的RR数据块是报告发送方SSRC字段后面的数据)<br/>
&emsp;&emsp;reportee SSRC定义的是接收RR报文方的SSRC。<br/>
&emsp;&emsp;cumulative number of packets lost(RTP报文丢失数)是24bit大小的整数，是期待接收到的RTP报文个数-实际收到报文个数。期待接收到报文的个数由：RTP中最新的sequence-第一个sequence。实际接收到的报文包括晚到，或重复到的，所以实际接收到的报文个数是有可能比大于期待的理论个数，这个字段有可能是负数。这个字段是基于整个RTP会话过程来计算的，而不是基于RTCP RR发送周期来计算的。这个字段最大的正数是0x7fffff。<br/>
&emsp;&emsp;loss fraction是在RR报告周期内，用丢失报文个数/期望收到报文个数。这个字段的表示方式是，丢失报文个数/期待报文个数x256。举例，如果丢失率是1/4，那么这个字段就是1/4x256=64。如果实际接收的个数大于期待接收的个数(因为会有重复报文、晚到的报文)，那么这个字段填写0。<br/>
&emsp;&emsp;interarrival jitter是网络传输时间估计值，表示rtp发送方报文发送的网络时间。该字段用时间戳表示，所以它用32位bits无符号整型表示，同RTP时间戳。<br/>
&emsp;&emsp;为了计算网络传输时间的变化，必须测量网络传输时间。因为RTP发送者和接受者都没有同步时钟，所以也不大可能精确计算出绝对的报文发送时间。相反相对的报文发送时间就要通过RTP报文中的时间戳，与接收方本地时间戳来进行计算。计算需要rtp接受者为每个源都维护一个时间，并且时间用与源都一致的时间频率。因为缺少rtp发送者和接受者的时间同步，相对的发送时间就会有未知的变数。但这不是问题，因为我们就是对发送时间的变化感兴趣: rtp接收方收到两个报文的时间差，对rtp发送方发送两个报文的时间差。接下来的算法，就是通过两个未同步的时间进行相减。<br/>
&emsp;&emsp;如果Si是RTP报文内的时间戳，Ri是服务器接收到RTP报文的时间，那么**相对的**传输时间就是(Ri-Si)。对于两个连续的RTP报文i和j，差值就是应该如下表示:&emsp;D(i,j) = (Rj-Sj) - (Ri-Si) <br/>
&emsp;&emsp;需要注意的是，Rx和Sx都是无符号32位整型，但是D(i,j)却是有符号整型。<br/>
&emsp;&emsp;那么到达间隔的jitter就是，每次报文到达，都用当前报文和上次到达报文计算出D(i,j)(这两个报文不一定是sequence连续的)。Jitter就被维护成一个平均值，计算公式如下:<br/>
**J(i) = J(i-1) + (|D(i-1, i)|-J(i-1))/16**;<br/>
&emsp;&emsp;无论RR报文什么时候生成，当前最新的J(i)就应该被放入interarrival jitter字段中。<br/>
&emsp;&emsp;LSR(Last sender report timestamp)，是最近收到发送方的RTCP SR报文中64位时间戳(NTP时间戳)中间的32位(有点变态，为什么是中间的32位)，如果没有SR收到，该字段填写0.<br/>
&emsp;&emsp;DLSR(delay since last sender report)，单位是1/65536秒，是接收到最新SR报文和当前发送RR报文的时间差。如果没有收到SR报文，该字段为0。<br/>
### 5.4.2 详解一下RR数据
&emsp;&emsp;通过RR进行接收质量不仅对发送者有帮助，也对参与者和第三方管理有好处。通过RR的反馈，发送者可以调整发送速度。此外，其他的参与者能判断出问题出在本地还是，其他人的质量也如此，并且网络监控着可以通过RR的和其他RTCP报文得知网络的状况。<br/>
&emsp;&emsp;发送者能通过RR中的LSR和DLSR字段计算出**RTT**(round-trip time 发送者与接受者之间网络来回耗时)。发送者接收到RR后，通过当前时间戳减去RR中的LSR字段，就得到发送SR和接收RR之间的时间差。然后，再减去RR中的DLSR，也就是去掉RR发送方内部接收RR和发送SR的时间差，这样就得到了RTT.<br/>
**RTT= RTP发送方本地时间 - RR中LSR - RR中DLSR** <br/>
(注意，这里的计算需要计算前统一一下时间单位) <br/>
具体如图5.4所示:<br/>
![rtt](https://github.com/runner365/read_book/blob/master/RTP_RTCP/pic/my_rtt.png)<br/>
&emsp;&emsp;注意RTT指的是网络来回的耗时，不包括发送、接收两端的延时。举例说明，如果接收方为了播放防止抖动而缓存了数据(具体的缓存、播放等细节在第6章描述)<br/>
&emsp;&emsp;RTT非常的重要，因为延时是交互最大的障碍。研究发现，如果总体上RTT大于300ms后会话就很难正常的完成了(当然，这个数字是估计值，也依赖听众和对音视频的处理方法)。发送者可以用RTT去优化编码--比如，当RTT比较大时，通过减少数据(应该是降低编码质量)来减少打包延时--或者通过错误修复的代码来完成(详情见第9章, 纠错)<br/>
&emsp;&emsp;丢包百分比能给接受者一个短期丢包率的信息。通过观测报告数据的趋势，发送者能判断当前的丢包是暂时现象，还是长期现象。很多RR的数据字段都是累积值，也就是可以去平均总数据。通过两个RR数据的不同点，能同时得到短期、长期的质量数据，让报告数据更具有弹性。<br/>
&emsp;&emsp;比如说，从两个间隔RR报文的累积统计字段(cumulative)就能得出报文丢失率，这是能直接得出的。两个RR报文的累积字段(cumulative)的差值也就表示了间隔时间内的包丢包数，除以间隔时间内最新sequence差值(得到期望收到的包数)，这样就能得到丢包率。这个算法的结果也等于RR报文中的Loss fraction字段，不过丢包率也只是估计的统计，因为可能出现负数，原因是有可能出现重复发包。Loss fraction字段的好处是能从单一RR报文得到丢包信息，在大型会话中是非常有用的，因为会话过程中，RR报文有可能出现丢失。<br/>
&emsp;&emsp;丢包率能是编码格式和纠错编码方式选择的基础(纠错，在第9章)。高丢包率就要选择高丢包率的编码方式，并且如果可能，比特率最好能降低(因为丢包率都是网络拥塞造成)<br/>
&emsp;&emsp;jitter字段也能用作检测正在发送的拥塞: 一个突增的jitter将会预示着有丢包发生。这个主要依赖网络拓扑，和流个数，流多路复用的程度，降低多路复用的程度就能减少jitter和将要发生拥塞的关联性。<br/>
&emsp;&emsp;发送者应该意识到，jitter字段是通过报文中的时间戳来标识的。如果发送者延迟发送某些报文，那么也会被记录到jitter中。关于视频报文，因为视频帧比较大，通常会被分成同一个时间戳的多包来发送，而不是单个打包一次发送。但这并不是个问题，因为jitter的测量本来就是要告诉接收者应该加入多大的接收buffer(这个接收buffer需要能容忍jitter和包间延迟)<br/>
## 5.5 RTCP SR: Sender Reports
&emsp;&emsp;除了接收者报告外，RTCP协议也提供让音视频发送方传输发送者报告(SR)。SR提供发送音视频媒体信息，主要为了接收方能同步多个媒体流(比如说，音视频同步)<br/>
### 5.5.1 SR格式
&emsp;&emsp;发送者报告的包类型是200，格式如图5.5所示。除常规RTCP头外，含24个字节，当然24字节后可以跟其他的RR报文，数量有RC字段设置，同RR报文类似。SR和RR一起发送仅仅当发送者同时也是接受者。<br/>
![RTCP SR format](https://github.com/runner365/read_book/blob/master/RTP_RTCP/pic/RTCP_SR.png)<br/>
&emsp;&emsp;NTP时间戳是64位长度，表示SR的发送时刻。它是NTP时间戳格式，是从1900年1月1日到现在的秒数，高32位代表秒数，低32位表示秒的小数部分(这是64位的定点指，定点在32位上)。UNIX时间戳到NTP时间戳，需要加上2,208,988,800秒。<br/>
具体计算方式: <br/>
<pre>
struct ntptime 
{
    unsigned int integer; //1900年以来的秒数
    unsigned int fraction;//小数部份，单位是微秒数的4294.967296(=2^32/10^6)倍
};
#define JAN_1970 2208988800
timeval到ntp时间戳的转换: 
ntptime.integar=timeval.tv_sec+JAN_1970;
ntptime.fraction=timeva.tv_usec* 0x100000000/1000000;
</pre><br/>
&emsp;&emsp;RTP的时间戳字段记录的时刻和NTP的时刻一致，但是单位是RTP报文时间戳单位。这个字段的值同RTP音视频流中时间戳字段值不一定一样，因为RTCP与RTP发送的时间不一定一样。见图5.6，显示SR报文时间戳的例子。SR报文有RTP时间戳，记录的是SR报文发送的时刻，与发送的RTP报文时间戳是独立的。<br/>
![SR TIMESTAMP](https://github.com/runner365/read_book/blob/master/RTP_RTCP/pic/SR%20TIMESTAMP.png)<br/>
&emsp;&emsp;sender's packet count是某个SSRC从会话开始发送报文的个数。<br/>
&emsp;&emsp;sender's octet count是某个SSRC从会话开始发送报文的字节数(不包括RTP头和pad)<br/>
&emsp;#emsp;如果发送者改变SSRC，sender's packet count和octet count会被清理(比如发生崩溃)。如果源持续发送很长时间，这两个字段可能出现翻转，但是这不是个问题。字段最新值减去上次的值，如果拿结果去模32，结果应该是一个2的32次方内的数字，即使翻转了也在这个范围内。这两个字段能让接收者计算出发送端的平均数据量。</br>
### 5.5.1 SR具体含义
&emsp;&emsp;通过SR信息，接收方能在一个时间间隔计算出发送方的平均数据量和包量，即使没有收到具体的数据。平均数据量除以包量，就得到平均包大小。如果嘉定丢包是独立于包大小，用接收方实际收到的报文个数乘以平均包大小，就能得到平均的吞吐量。<br/>
&emsp;&emsp;时间戳信息，就拿来产生RTP时钟与外部时钟(NTP clock)的关联。这对音视频同步有帮助，具体见第7章。
## 5.6 RTCP SDES: Source Description
&emsp;&emsp;这章的知识，在实际中暂时没有用到，先放一下。
## 5.7 RTCP BYE: Membership Control
&emsp;&emsp;RTCP通过RTCP BYE来提供比较松散的成员控制，BYE报文被用来表示成员离开会话。发送BYE报文，意味着有成员离开，或者成员SSRC改变--比如SSRC冲突，需要修改自己的SSRC来继续通信。BYE报文也是可能在传送中丢失，一些应用并不产生它；因此接受者必须提供超时机制，在一段时间后没有收到数据就认为会话结束，即使没有收到BYE报文。<br/>
&emsp;&emsp;BYE的重要性对应用来说是种扩展。它直接表示成员离开RTP会话，但它也能对其他协议成员做关系参考。但是RTCP BYE不终结其他协议的会话。<br/>
&emsp;&emsp;BYE的报文类型是203，具体格式如图5.10。RC字段表示有几个SSRC要离开。RC字段为0，有效但是没什么用。如果收到BYE报文，就知道BYE报文中的SSRC列表中的SSRC都离开会话，不在处理其对应的RTP/RTCP报文。当然，在收到BYE报文后继续保留在线状态一段时间，延迟对应数据的处理。<br/>
![RTCP BYE](https://github.com/runner365/read_book/blob/master/RTP_RTCP/pic/RTCP_BYE.png)<br/>
&emsp;&emsp;本章后面的成员数据库会介绍状态维护的华庭，这和超时机制和RTCP BYE报文有关。<br/>
&emsp;&emsp;BYT报文也包含了text可选字段，可用来表示离开会话原因描述，可以很方便的在用户界面上显示。text字段是可选的，但是程序肯定要接收它(及时不处理它)
## 5.8 RTCP APP: Application-Defined RTCP Packets
&emsp;&emsp;最后介绍的RTCP类型(APP)是可以应用自定义的扩展。报文类型是204，格式如图5.11。APP报文中packet name是一个4字节的字符串，每个字符都是ASCII码，区分大小写。推荐packet name用于标识改应用类型，当然也可以用subtype字段来表示应用类型。报文最后部分是应用自定义。<br/>
![rtcp app format](https://github.com/runner365/read_book/blob/master/RTP_RTCP/pic/RTCP_APP_FORMAT.png)<br/>
&emsp;&emsp;APP报文用于非标准的RTCP扩展，用作新特性的扩展。目的还是，开发者可以把APP用来尝试新的特性，并且如果某些新特性被广泛使用后再注册成新的报文类型。一些应用开发产生APP报文，程序应该丢弃不认识的APP报文<br/>
