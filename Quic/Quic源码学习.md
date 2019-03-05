# quic源码学习

## 1. chromium中的quic编译
### 1.1 depot_tools安装
export PATH="$PATH:/Users/shiwei9/Documents/code/depot/depot_tools"

### 1.2 编译
解压chromium后，进入src<br/>
cd src<br/>
gn gen out/Default && ninja -C out/Default quic_client quic_server net_unittests <br/>
<br/>
如果全部编译：<br/>
ninja -C out/Debug-iphonesimulator gn_all <br/>

## 2. quicclient源码
代码路径: chromium/src/net/third_party/quic <br/>
<pre>
core		platform	quartc		test_tools	tools
</pre>
QuicClient派生于QuicSpdyClientBase <br/>
QuicSpdyClientBase 派生于 QuicClientBase, QuicClientPushPromiseIndex::Delegate, QuicSpdyStream::Visitor <br/>
<br/>
主线：QuicClient :: QuicSpdyClientBase :: QuicClientBase <br/>

### 2.1 QuicClientBase类的关键信息
#### 2.1.1 关键成员
<pre>
<code>
  // Address of the server.
  QuicSocketAddress server_address_; //quic的socket address

  // If initialized, the address to bind to.
  QuicIpAddress bind_to_address_; //quic绑定的quic address

  // Local port to bind to. Initialize to 0.
  int local_port_; //quic的udp port

  // config_ and crypto_config_ contain configuration and cached state about
  // servers.
  QuicConfig config_; //quic连接配置
  QuicCryptoClientConfig crypto_config_; //quic加密配置

  // Writer used to actually send packets to the wire. Must outlive |session_|.
  std::unique_ptr< QuicPacketWriter> writer_; //用于发送报文的接口

  // Session which manages streams.
  std::unique_ptr< QuicSession> session_; //QuicSession是管理quic stream的会话类

  // The network helper used to create sockets and manage the event loop.
  // Not owned by this class.
  std::unique_ptr< NetworkHelper> network_helper_; //网络事件监听的event loop机制，也就是epoll的收发事件
</code>
</pre>

#### 2.1.2 关键成员函数
* bool QuicClientBase::Initialize() <br/>
  //初始化，接收窗口session 15MB, stream 6MB; 并且初始化network_helper_，网络事件机制；在StartConnect or Connect之前调用<br/>
  // Initializes the client to create a connection. Should be called exactly <br/>
  // once before calling StartConnect or Connect. Returns true if the <br/>
  // initialization succeeds, false otherwise.  <br/>
<pre>
<code>
bool QuicClientBase::Initialize() {
  //......
  // If an initial flow control window has not explicitly been set, then use the
  // same values that Chrome uses.
  const uint32_t kSessionMaxRecvWindowSize = 15 * 1024 * 1024;  // 15 MB
  const uint32_t kStreamMaxRecvWindowSize = 6 * 1024 * 1024;    //  6 MB

  //.....

  if (!network_helper_->CreateUDPSocketAndBind(server_address_,
                                               bind_to_address_, local_port_)) {
    return false;
  }

  initialized_ = true;
  return true;
}
</code>
</pre>

*  bool Connect(); <br/>
//连接quic server，包括 同步加密密钥，和handshake <br/>
会调用StartConnect() <br/>

* void QuicClientBase::StartConnect(); <br/>
//连接quic server，包括 同步加密密钥，和handshake; <br/>
创建QuicPacketWriter；<br/>
创建QuicSession；<br/>
创建QuicConnection(其最为QuecSession的成员对象)； <br/>
<pre>
<code>
void QuicClientBase::StartConnect() {
  .......
  QuicPacketWriter* writer = network_helper_->CreateQuicPacketWriter();//创建writer接口
  ......
  session_ = CreateQuicClientSession(
      supported_versions(),
      new QuicConnection(GetNextConnectionId(), server_address(), helper(),
                         alarm_factory(), writer,
                         /* owns_writer= */ false, Perspective::IS_CLIENT,
                         can_reconnect_with_different_version
                             ? ParsedQuicVersionVector{mutual_version}
                             : supported_versions()));
  // Reset |writer()| after |session()| so that the old writer outlives the old
  // session.
  set_writer(writer);
  InitializeSession();
  set_connected_or_attempting_connect(true);
}
</code>
</pre>
<br/>

* void QuicClientBase::InitializeSession() <br/>
  // Calls session()->Initialize(). Subclasses may override this if any extra <br/>
  // initialization needs to be done. Subclasses should expect that session() <br/>
  // is non-null and valid. <br/>
<pre>
<code>
void QuicClientBase::InitializeSession() {
  session()->Initialize();
}
</code>
</pre>
<br/>
quic session初始化，可能被继承覆盖，如果需要做更多工作的话。<br/>

### 2.2 QuicSpdyClientBase的关键信息
QuicSpdyClientBase继承QuicClientBase，和QuicClientPushPromiseIndex::Delegate， QuicSpdyStream::Visitor<br/>
#### 2.2.1 关键成员
<pre>
<code>
  // Keeps track of any data that must be resent upon a subsequent successful
  // connection, in case the client receives a stateless reject.
  std::vector< std::unique_ptr< QuicDataToResend>> data_to_resend_on_connect_;

  std::unique_ptr< ClientQuicDataToResend> push_promise_data_to_resend_;
</code>
</pre>
<br/>

#### 2.2.2 关键成员函数
* std::unique_ptr<QuicSession> QuicSpdyClientBase::CreateQuicClientSession; <br/>
CreateQuicClientSession是基类QuicClientBase中的纯虚函数.<br/>
创建QuicSpdyClientSession类对象，需要输入connection对象等。<br/>
<pre>
<code>
std::unique_ptr< QuicSession> QuicSpdyClientBase::CreateQuicClientSession(
    const quic::ParsedQuicVersionVector& supported_versions,
    QuicConnection* connection) {
  return QuicMakeUnique< QuicSpdyClientSession>(
      *config(), supported_versions, connection, server_id(), crypto_config(),
      &push_promise_index_);
}
</code>
</pre>
<br/>
*   void SendRequest(const spdy::SpdyHeaderBlock& headers, QuicStringPiece body, bool fin); <br/>
发送http request的报文，headers是http的头部额外信息，body是http的body信息；<br/>
创建Quic client stream类对象，通过其发送. <br/>
<pre>
<code>
void QuicSpdyClientBase::SendRequest(const SpdyHeaderBlock& headers,
                                     QuicStringPiece body,
                                     bool fin) {
  //设置重传的部分代码
  ......
  QuicSpdyClientStream* stream = CreateClientStream(); //创建QuicSpdyClientStream类对象
  if (stream == nullptr) {
    QUIC_BUG << "stream creation failed!";
    return;
  }
  stream->SendRequest(headers.Clone(), body, fin);//通过QuicSpdyClientStream的SendRequest发送信息
  //记录重传信息
  .....
}
</code>
</pre>
<br/>

*   void SendRequestAndWaitForResponse(const spdy::SpdyHeaderBlock& headers,QuicStringPiece body,bool fin); <br/>
// Sends an HTTP request and waits for response before returning.
发送http request后，通过WaitForEvents做异步等待返回！<br/>
WaitForEvents依靠基类QuicClientBase中的实现完成：network_helper_->RunEventLoop();
<pre>
<code>
void QuicSpdyClientBase::SendRequestAndWaitForResponse(
    const SpdyHeaderBlock& headers,
    QuicStringPiece body,
    bool fin) {
  SendRequest(headers, body, fin);
  while (WaitForEvents()) {
  }
}
</code>
</pre>
<br/>

*  QuicSpdyClientStream* CreateClientStream(); <br/>
创建一个QuicSpdyClientStream类对象<br/>
<pre>
<code>
QuicSpdyClientStream* QuicSpdyClientBase::CreateClientStream() {
  if (!connected()) {
    return nullptr;
  }

  auto* stream = static_cast< QuicSpdyClientStream*>(
      client_session()->CreateOutgoingBidirectionalStream());
  if (stream) {
    stream->SetPriority(QuicStream::kDefaultPriority);
    stream->set_visitor(this);
  }
  return stream;
}
</code>
</pre>
<br/>

### 2.3 QuicClient
QuicClient继承QuicSpdyClientBase<br/>
#### 2.3.1 关键成员
基本都是继承基类的成员变量. <br/>

#### 2.3.1 关键成员函数
* std::unique_ptr<QuicSession> CreateQuicClientSession(const ParsedQuicVersionVector& supported_versions, QuicConnection* connection) override; <br/>
继承和覆盖基类的CreateQuicClientSession函数。<br/>
通过输入，QuicConnection和ParsedQuicVersionVector来进行创建quic session。<br/>
在基类QuicSpdyClientBase中的CreateQuicClientSession返回的对象是QuicSpdyClientSession。<br/>
在基类QuicClientBase中CreateQuicClientSession是一个纯虚函数<br/>

### 3. Quic Session相关代码
* QuicSpdyClientSession <br/>
代码路径: net/third_party/quic/core/http <br/>
class QuicSpdyClientSession : public QuicSpdyClientSessionBase <br/>
* QuicSpdyClientSessionBase  <br/>
代码路径: net/third_party/quic/core/http <br/>
<pre>
<code>
  class QUIC_EXPORT_PRIVATE QuicSpdyClientSessionBase
    : public QuicSpdySession,
      public QuicCryptoClientStream::ProofHandler
</code>
</pre><br/>
* QuicSpdySession <br/>
代码路径: net/third_party/quic/core/http <br/>
<pre>
<code>
class QUIC_EXPORT_PRIVATE QuicSpdySession
    : public QuicSession,
      public QpackEncoder::DecoderStreamErrorDelegate,
      public QpackEncoderStreamSender::Delegate,
      public QpackDecoder::EncoderStreamErrorDelegate,
      public QpackDecoderStreamSender::Delegate
</code>
</pre><br/>
* QuicSession<br/>
代码路径: net/third_party/quic/core<br/>
<pre>
<code>
class QUIC_EXPORT_PRIVATE QuicSession : public QuicConnectionVisitorInterface,
                                        public SessionNotifierInterface,
                                        public QuicStreamFrameDataProducer
</code>
</pre><br/>

### 3.1 QuicSession
#### 3.1.1 类成员变量
* 