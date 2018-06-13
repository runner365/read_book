# 一点点从基础做起
从协议原文精读翻译做起，欢迎交流指导！

## 1. webrtc方向
&emsp;&emsp;webrtc技术栈比较长，内容比较多，如果只是拿一些开源来改改使用，无法真真的入门<br>
webrtc的主要内容:<br/>
+ 音视频编解码
+ RTP/RTCP传输
+ P2P
+ 各种信令协议
+ QUIC协议(未来用QUIC替代UDP的可能性)
开始自学英文协议文档方式，一点点的深入，一边提高英文能力，一边理解各种协议和具体实现。<br/>
首先，从关键的RTP/RTCP协议基础开始。<br/>

## 1.1 RTP/RTCP
**RTP：Audio and video for the Internet.pdf**，这本书比较全面的介绍RTP/RTCP协议在互联网中的应用，通过对关键章节的翻译，逐步深入理解RTP/RTCP协议。
## 1.1.1 英文自译
原文: [RTP/RTCP for internet](https://github.com/runner365/read_book/blob/master/RTP_RTCP/RTP%EF%BC%9AAudio%20and%20video%20for%20the%20Internet.pdf)<br>

### 1.1.1.1 RTP/RTC协议部分
自译文档连接:[RTP/RTC协议--精选翻译](https://github.com/runner365/read_book/blob/master/RTP_RTCP/RTP_RTCP%E5%8D%8F%E8%AE%AE%E5%86%85%E5%AE%B9--%E7%B2%BE%E9%80%89%E8%87%AA%E8%AF%91.md)<br/>
自译关键内容:
* RTCP报文介绍: SR, RR, BYE, APP
* JITTER计算方式
* RTT计算方式

### 1.1.1.2 RTP纠错方式
自译文档链接:[RTP纠错方式](https://github.com/runner365/read_book/blob/master/RTP_RTCP/RTP%E7%BA%A0%E9%94%99%E6%9C%BA%E5%88%B6--%E7%B2%BE%E9%80%89%E8%87%AA%E8%AF%91.md)<br/>
自译关键内容:
* RTP纠错方式: NACK
* RTP纠错方式:FEC(前向纠错)

## 1.2 RTP H264
**RFC6184是RFC描述RTP协议如何承载H264**<br/>
对于视频，主要承载的就是H264报文，rfc6184就显得非常的核心和重要。

## 1.2.1 英文自译
原文:[RFC6184.pdf](https://github.com/runner365/read_book/blob/master/RTP_H264/rfc6184.pdf)<br/>
自译文档链接:[翻译: rfc6184-RTP Payload Format for H.264 Video](https://github.com/runner365/read_book/blob/master/RTP_H264/rfc6184%E8%87%AA%E8%AF%91.md)<br/>
翻译主要内容:<br/>
* RTP承载H264的主要几种方式
* single NAL unit mode(单NAL单元模式)
* STAP-A方式(同一时间和顺序的聚合模式)
* FU-A(同一时间和顺序的分片模式)

对于STAP-B，MTAP，FU-B等，因为当前没有应用场景，在实际工作中用不到，因此不进行翻译。<br/>

## 1.3 QUIC
原文链接: [QUIC wire specification](https://docs.google.com/document/d/1WJvyZflAO2pq77yOLbp9NsGjC1CHetAXV8I0fQe-B_U/edit)<br/>
自译文档连接: [QUIC协议自译](https://github.com/runner365/read_book/blob/master/Quic/Quic_Wire_layout_specification_%E8%87%AA%E8%AF%91.md)<br/>