---
title: muduo源码3-Acceptor(新建连接) & Socket
categories: muduo
date: 2024-12-29 11:55:54
---

> 📌本文采用wolai制作， [link](https://www.wolai.com/ravenxrz/tjiBrMJqR5LkYQaK5u7fWm "link")

<!--more-->

# 1 回顾socket接口

在分析muduo源码前，简单回顾下linux下的网络编程接口，这里用下csapp的slide:

![image-2020083117322399|550](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/obsidian_img/image-20200831173223990.png "image-2020083117322399|550")

这里解释下一些关键接口的含义:

- socket:  用于创建一个socket descriptor（就像open返回一个fd一样)
- bind: 将ip地址绑定到socket
  ```c
  int bind(int socket, const struct sockaddr *address, socklen_t address_len);
  ```
- listen:  将一个socket从 `active socket`转换为 `listening socket`
  ```c
  int listen(int socket, int backlog);
  
  ```
  默认情况下，创建的socket叫做`active socket`，这种socket只能用在client端。 通过`listen`调用，将其转换为`listening socket`，这种socket可以接收来自client的请求，用于server端。
- accept: server端用于等待client端的连接函数，连接信息放在`address`中。
  ```c
  int accept(int socket, struct sockaddr * address,
             socklen_t * address_len);
  
  ```
- connect: client端发起连接
  ```c
  int connect(int socket, const struct sockaddr *address, socklen_t address_len);
  ```

# 2 Acceptor源码

Acceptor 用于处理新连接，正如官方注释写的:

```typescript
///
/// Acceptor of incoming TCP connections.
///

```

对外提供的接口有:

```c++
  Acceptor(EventLoop* loop, const InetAddress& listenAddr, bool reuseport);
  ~Acceptor();

  void setNewConnectionCallback(const NewConnectionCallback& cb)
  { newConnectionCallback_ = cb; }

  void listen();

  bool listening() const { return listening_; }

```

包含一个处理链接的回调，和一个`listen`监听接口。

主要成员包括:

```c++
  EventLoop* loop_;
  Socket acceptSocket_;
  Channel acceptChannel_;
   NewConnectionCallback newConnectionCallback_;
  bool listening_;
  int idleFd_;
```

先看构造函数:

```c++
Acceptor::Acceptor(EventLoop* loop, const InetAddress& listenAddr, bool reuseport)
  : loop_(loop),
     acceptSocket_(sockets::createNonblockingOrDie(listenAddr.family())),
    acceptChannel_(loop, acceptSocket_.fd()),
     listening_(false),
    idleFd_(::open("/dev/null", O_RDONLY | O_CLOEXEC))
{
  assert(idleFd_ >= 0);
   acceptSocket_.setReuseAddr(true);
  acceptSocket_.setReusePort(reuseport); 
  acceptSocket_.bindAddress(listenAddr);
   acceptChannel_.setReadCallback(
      std::bind(&Acceptor::handleRead, this)); 
}


```

acceptSocket\_ 是muduo关于socket的封装。 关于socket的细节下文再说。本节重点关于Acceptor的逻辑即可。

channel已经在[muduo源码分析1-事件循环(上)](https://www.wolai.com/fgMUbiv6DPt5w4HU9RdPvE "muduo源码分析1-事件循环(上)")中分析过。

额外关注的是 `acceptor`开启了ReusePort.

打开看`acceptSocket_` 的Reuse相关实现：


```c++
void Socket::setReuseAddr(bool on)
{
  int optval = on ? 1 : 0;
  ::setsockopt(sockfd_, SOL_SOCKET,  SO_REUSEADDR ,
               &optval, static_cast<socklen_t>(sizeof optval));
  // FIXME CHECK
}


void Socket::setReusePort(bool on)
{
#ifdef SO_REUSEPORT
  int optval = on ? 1 : 0;
  int ret = ::setsockopt(sockfd_, SOL_SOCKET,  SO_REUSEPORT ,
                         &optval, static_cast<socklen_t>(sizeof optval));
  if (ret < 0 && on)
  {
    LOG_SYSERR << "SO_REUSEPORT failed.";
  }
#else
  if (on)
  {
    LOG_ERROR << "SO_REUSEPORT is not supported.";
  }
#endif
}



```

重点参数是两个flag：

- `SO_REUSEADDR`选项允许在TCP连接中，立即重用处于`TIME_WAIT`状态的端口。这在服务器程序需要频繁重启时特别有用，因为它避免了必须等待旧连接完全关闭才能重新绑定端口的问题。
- `SO_REUSEPORT`选项允许多个套接字绑定到同一个IP地址和端口。这对于实现负载均衡非常有用，因为可以创建多个监听套接字，每个监听套接字可以在不同的线程或进程中处理请求，从而提高并发处理能力。

> 这里提到了 `TIME_WAIT`, 所以再说下TCP的四次回收断开连接过程。如下图：
>
> ![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/obsidian_img/57)
>
> 可以看到每个主动发起close的一端，在最后总会进入`TIME_WAIT`状态，等待2MSL时间才能真正断开，在此期间，原链接的ip+port 组是不能被复用的，对于高并发多连接的场景，是不可接受的，所以`SO_REUSEADDR`的作用就是对于处于这个状态的连接，可以重用。

回到`Acceptor`。 `Acceptor` 还提供了`listen函数`, 其`caller`为

```c++
void TcpServer::start()
{
  if (started_.getAndSet(1) == 0)
  {
    threadPool_->start(threadInitCallback_);

    assert(!acceptor_->listening());
    loop_->runInLoop(
         std::bind(&Acceptor::listen, get_pointer(acceptor_) ));
  }
}

void Acceptor::listen()
{
  loop_->assertInLoopThread();
  listening_ = true;
  acceptSocket_.listen();
  acceptChannel_.enableReading(); // 注册read事件监听，这样accept sockfd上有read事件即会回调 std::bind(&Acceptor:: handleRead , this)
}


```

再看 `handleRead`

```c++
void Acceptor::handleRead()
{
  loop_->assertInLoopThread();
  InetAddress peerAddr;
  //FIXME loop until no more
   int connfd = acceptSocket_.accept(&peerAddr); // 创建新连接
   if (connfd >= 0)
  {
    // string hostport = peerAddr.toIpPort();
    // LOG_TRACE << "Accepts of " << hostport;
    if (newConnectionCallback_)
    {
       newConnectionCallback_(connfd, peerAddr);  // 回调给caller
     }
    else
    {
      sockets::close(connfd);
    }
  }

}
```

`newConnectionCallback_` 哪来的？在`TcpServer`构造函数中注册进来的:

```c++
TcpServer::TcpServer(EventLoop* loop,
                     const InetAddress& listenAddr,
                     const string& nameArg,
                     Option option)
  : ...
{
   acceptor_->setNewConnectionCallback(
      std::bind(&TcpServer::newConnection, this, _1, _2)); 
}


```

`newConnection` 函数已经在[muduo源码分析1-事件循环(上)](https://www.wolai.com/fgMUbiv6DPt5w4HU9RdPvE "muduo源码分析1-事件循环(上)")和[muduo源码分析2-事件循环（下)](https://www.wolai.com/64a1q1Q3cCT7Dbzm6Bz5Ax "muduo源码分析2-事件循环（下)")都提到了, 这里再提一下：

```c++
void TcpServer::newConnection(int sockfd, const InetAddress& peerAddr)
{
  loop_->assertInLoopThread();
  EventLoop* ioLoop = threadPool_->getNextLoop();
  char buf[64];
  snprintf(buf, sizeof buf, "-%s#%d", ipPort_.c_str(), nextConnId_);
  ++nextConnId_;
  string connName = name_ + buf;

  LOG_INFO << "TcpServer::newConnection [" << name_
           << "] - new connection [" << connName
           << "] from " << peerAddr.toIpPort();
  InetAddress localAddr(sockets::getLocalAddr(sockfd));
  // FIXME poll with zero timeout to double confirm the new connection
  // FIXME use make_shared if necessary
   TcpConnectionPtr conn(new TcpConnection(ioLoop,
                                          connName,
                                          sockfd,
                                          localAddr,
                                          peerAddr));
   connections_[connName] = conn;
  conn->setConnectionCallback(connectionCallback_);
  conn->setMessageCallback(messageCallback_);
  conn->setWriteCompleteCallback(writeCompleteCallback_);
  conn->setCloseCallback(
      std::bind(&TcpServer::removeConnection, this, _1)); // FIXME: unsafe
   ioLoop->runInLoop(std::bind(&TcpConnection::connectEstablished, conn));
 }
```

拿到sockfd后，创建`TcpConnection`对象，内部生成`channel`，后面关于这个连接的读写，都通过`channel`来监听，一旦出现读写事件，又会回调这里的 `messageCallback_`和 `writeCompleteCallback_`回调。 同时，当出现新建连接时，还会进入 `connectEstablished`:

```c++
void TcpConnection::connectEstablished()
{
  loop_->assertInLoopThread();
  assert(state_ == kConnecting);
  setState(kConnected);
  channel_->tie(shared_from_this());
  channel_->enableReading();

   connectionCallback_(shared_from_this());
 }


```

这里的`connectionCallback_`就是最外层应用层逐层传递下来的，以`HtppServer`为例:

```c++
HttpServer::HttpServer(EventLoop* loop,
                       const InetAddress& listenAddr,
                       const string& name,
                       TcpServer::Option option)
  : server_(loop, listenAddr, name, option),
    httpCallback_(detail::defaultHttpCallback)
{
   server_.setConnectionCallback(
      std::bind(&HttpServer::onConnection, this, _1));
  server_.setMessageCallback(
      std::bind(&HttpServer::onMessage, this, _1, _2, _3)); 
}


```

## 2.1 一图串起来

现在我们完全了解了 **创建一个新连接 **是怎么玩的了，画个图来总结。

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage_7ft2lsZAN7.png)

# 3 Socket

这个类是对底层`socket` 接口的wrap， 其实没什么好说的，贴下源码吧。

```c++
///
/// Wrapper of socket file descriptor.
///
/// It closes the sockfd when desctructs.
/// It's thread safe, all operations are delagated to OS.
class Socket : noncopyable
{
 public:
  explicit Socket(int sockfd)
    : sockfd_(sockfd)
  { }

  // Socket(Socket&&) // move constructor in C++11
  ~Socket();

  int fd() const { return sockfd_; }
  // return true if success.
  bool getTcpInfo(struct tcp_info*) const;
  bool getTcpInfoString(char* buf, int len) const;

  /// abort if address in use
  void bindAddress(const InetAddress& localaddr);
  /// abort if address in use
  void listen();

  /// On success, returns a non-negative integer that is
  /// a descriptor for the accepted socket, which has been
  /// set to non-blocking and close-on-exec. *peeraddr is assigned.
  /// On error, -1 is returned, and *peeraddr is untouched.
  int accept(InetAddress* peeraddr);

  void shutdownWrite();

  ///
  /// Enable/disable TCP_NODELAY (disable/enable Nagle's algorithm).
  ///
  void setTcpNoDelay(bool on);

  ///
  /// Enable/disable SO_REUSEADDR
  ///
  void setReuseAddr(bool on);

  ///
  /// Enable/disable SO_REUSEPORT
  ///
  void setReusePort(bool on);

  ///
  /// Enable/disable SO_KEEPALIVE
  ///
  void setKeepAlive(bool on);

 private:
  const int sockfd_;
};


Socket::~Socket()
{
  sockets::close(sockfd_);
}

bool Socket::getTcpInfo(struct tcp_info* tcpi) const
{
  socklen_t len = sizeof(*tcpi);
  memZero(tcpi, len);
  return ::getsockopt(sockfd_, SOL_TCP, TCP_INFO, tcpi, &len) == 0;
}

bool Socket::getTcpInfoString(char* buf, int len) const
{
  struct tcp_info tcpi;
  bool ok = getTcpInfo(&tcpi);
  if (ok)
  {
    snprintf(buf, len, "unrecovered=%u "
             "rto=%u ato=%u snd_mss=%u rcv_mss=%u "
             "lost=%u retrans=%u rtt=%u rttvar=%u "
             "sshthresh=%u cwnd=%u total_retrans=%u",
             tcpi.tcpi_retransmits,  // Number of unrecovered [RTO] timeouts
             tcpi.tcpi_rto,          // Retransmit timeout in usec
             tcpi.tcpi_ato,          // Predicted tick of soft clock in usec
             tcpi.tcpi_snd_mss,
             tcpi.tcpi_rcv_mss,
             tcpi.tcpi_lost,         // Lost packets
             tcpi.tcpi_retrans,      // Retransmitted packets out
             tcpi.tcpi_rtt,          // Smoothed round trip time in usec
             tcpi.tcpi_rttvar,       // Medium deviation
             tcpi.tcpi_snd_ssthresh,
             tcpi.tcpi_snd_cwnd,
             tcpi.tcpi_total_retrans);  // Total retransmits for entire connection
  }
  return ok;
}

void Socket::bindAddress(const InetAddress& addr)
{
  sockets::bindOrDie(sockfd_, addr.getSockAddr());
}

void Socket::listen()
{
  sockets::listenOrDie(sockfd_);
}

int Socket::accept(InetAddress* peeraddr)
{
  struct sockaddr_in6 addr;
  memZero(&addr, sizeof addr);
  int connfd = sockets::accept(sockfd_, &addr);
  if (connfd >= 0)
  {
    peeraddr->setSockAddrInet6(addr);
  }
  return connfd;
}

void Socket::shutdownWrite()
{
  sockets::shutdownWrite(sockfd_);
}

void Socket::setTcpNoDelay(bool on)
{
  int optval = on ? 1 : 0;
  ::setsockopt(sockfd_, IPPROTO_TCP, TCP_NODELAY,
               &optval, static_cast<socklen_t>(sizeof optval));
  // FIXME CHECK
}

void Socket::setReuseAddr(bool on)
{
  int optval = on ? 1 : 0;
  ::setsockopt(sockfd_, SOL_SOCKET, SO_REUSEADDR,
               &optval, static_cast<socklen_t>(sizeof optval));
  // FIXME CHECK
}

void Socket::setReusePort(bool on)
{
#ifdef SO_REUSEPORT
  int optval = on ? 1 : 0;
  int ret = ::setsockopt(sockfd_, SOL_SOCKET, SO_REUSEPORT,
                         &optval, static_cast<socklen_t>(sizeof optval));
  if (ret < 0 && on)
  {
    LOG_SYSERR << "SO_REUSEPORT failed.";
  }
#else
  if (on)
  {
    LOG_ERROR << "SO_REUSEPORT is not supported.";
  }
#endif
}

void Socket::setKeepAlive(bool on)
{
  int optval = on ? 1 : 0;
  ::setsockopt(sockfd_, SOL_SOCKET, SO_KEEPALIVE,
               &optval, static_cast<socklen_t>(sizeof optval));
  // FIXME CHECK
}


```


