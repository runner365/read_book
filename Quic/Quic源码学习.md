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
  std::unique_ptr<QuicPacketWriter> writer_; //用于发送报文的接口

  // Session which manages streams.
  std::unique_ptr<QuicSession> session_; //QuicSession是管理quic stream的会话类

  // The network helper used to create sockets and manage the event loop.
  // Not owned by this class.
  std::unique_ptr<NetworkHelper> network_helper_; //网络事件监听的event loop机制，也就是epoll的收发事件
</code>
</pre>

#### 2.1.2 关键成员函数
* bool QuicClientBase::Initialize() 
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

*  bool Connect(); //连接quic server，包括 同步加密密钥，和handshake
会调用StartConnect() <br/>

* void QuicClientBase::StartConnect(); //连接quic server，包括 同步加密密钥，和handshake; 
创建QuicPacketWriter；创建QuicConnection(其最为QuecSession的成员对象)；创建QuicSession； <br/>
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
