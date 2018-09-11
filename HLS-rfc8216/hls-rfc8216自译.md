* 1 HTTP Live Streaming简介
HTTP Live Streaming(HLS)是一种可靠的，高效的，在因特网中能持续长时间传输视频的协议。HLS允许接受者能针对网络状况调整媒体的比特率，来实现能提供最高质量的不间断播放。它支持内容间隔的边界。它也为加密的媒体提供灵活的框架。他能高效的提供同一个内容的多种演绎，例如音频的传输。它提供基于http缓存技术框架的容量能力，因而能支持大量的观众。<br/>
自从2009年第一个草案提交，HLS已经被大量的内容提供商，工具开发者，内容分发商和设备制造商。在今后的8年里，协议已经被多个应用者经过多次review和讨论而重定义。<br/>
本文的目的是通过描述hls传输协议，来帮助各个hls的应用者更好的完成协议互动。用这个协议，客户端可以接受到从服务端发来的持续的媒体流。

* 2 综述
多媒体的描述是通过URI定义到一个playlist中(RFC3986)。<br>
这个playlist可以是媒体播放列表，或关键信息列表。两者都是基于UTF8文本编码，内容包含URIs和描述tags。<br/>
一个媒体Playlist包含了一个媒体片段的列表，当其被按顺序播放，将播放出媒体的内容。<br/>
下面就是一个媒体playlist的例子:<br/>
<pre>
<code>	
#EXTM3U
#EXT-X-TARGETDURATION:10

#EXTINF:9.009,
http://media.example.com/first.ts
#EXTINF:9.009,
http://media.example.com/second.ts
#EXTINF:3.003,
http://media.example.com/third.ts
</code>
</pre>
第一行是格式定义的tag: #EXTM3U。第二行有#EXT-X-TARGETDURATION表示所有的媒体片段都是10秒左右。看后面，3个媒体片段有声明。第一个、第二个是9.009秒长；第三个是3.003秒长。<br/>
为了播放这个播放列表，客户端首先需要下载这个它，然后下载和播放每个在它里面的媒体片段。客户单需要重新下载这个playlist文件来获取新加入的媒体片段。数据传输通过http(RFC7230)，但是，总体来说，一个URI能根据特殊需要，被定义成任何可靠的传输协议。<br/>
