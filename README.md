# 一点点从基础做起
从协议原文精读翻译做起，欢迎交流指导！

## 1. webrtc方向
&emsp;&emsp;webrtc技术栈比较长，内容比较多，如果只是拿一些开源来改改使用，无法真真的入门<br>
webrtc的主要内容:<br/>
+ 音视频编解码
+ RTP/RTCP传输
+ P2P
+ 各种信令协议

开始自学英文协议文档方式，一点点的深入，一边提高英文能力，一边理解各种协议和具体实现。<br/>
首先，从关键的RTP/RTCP协议基础开始。<br/>

## 1.1 RTP/RTCP
**RTP：Audio and video for the Internet.pdf**，这本书比较全面的介绍RTP/RTCP协议在互联网中的应用，通过对关键章节的翻译，逐步深入理解RTP/RTCP协议。
## 1.1.1 英文自译
原文: [RTP/RTCP for internet](https://github.com/runner365/read_book/blob/master/RTP_RTCP/RTP%EF%BC%9AAudio%20and%20video%20for%20the%20Internet.pdf)<br>
自译: <br>
### [RTP/RTC协议--精选翻译](https://github.com/runner365/read_book/blob/master/RTP_RTCP/RTP_RTCP%E5%8D%8F%E8%AE%AE%E5%86%85%E5%AE%B9--%E7%B2%BE%E9%80%89%E8%87%AA%E8%AF%91.md)<br/>
自译关键内容:
* RTCP报文介绍: SR, RR, BYE, APP
* JITTER计算方式
* RTT计算方式

### [RTP纠错方式](https://github.com/runner365/read_book/blob/master/RTP_RTCP/RTP%E7%BA%A0%E9%94%99%E6%9C%BA%E5%88%B6--%E7%B2%BE%E9%80%89%E8%87%AA%E8%AF%91.md)<br/>
自译关键内容:
* RTP纠错方式: NACK
* RTP纠错方式:FEC(前向纠错)
