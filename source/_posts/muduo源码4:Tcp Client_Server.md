---
title: muduo源码4-Tcp Client/Server 
categories: muduo
date: 2024-12-31 17:13:54
---

> 📌本文采用wolai制作， [link](https://www.wolai.com/ravenxrz/8vpH7tpYRdd8kxC4hn4XrM "link").

前文已经分析了完整的 事件循环 和 新建连接 流程，相当于下层基础设施已经完毕，现在看上层应用咋玩的。

<!--more-->

# 1 TcpConnection

`TcpConnection`是 Tcp连接的抽象实现。

## 1.1 建立连接

先看构造函数，看下有什么依赖需要注入：

```c++
  /// User should not create this object.
  TcpConnection(EventLoop* loop,
                const string& name,
                 int sockfd, 
                const InetAddress& localAddr,
                const InetAddress& peerAddr);
```

真正重要的是sockfd。这玩意代表本次连接, `caller`在 （可参考 [muduo源码分析3-Acceptor(新建连接) & Socket](https://www.wolai.com/tjiBrMJqR5LkYQaK5u7fWm "muduo源码分析3-Acceptor(新建连接) & Socket"))

```c++
void TcpServer::newConnection(int sockfd, const InetAddress& peerAddr)
{
  // FIXME poll with zero timeout to double confirm the new connection
  // FIXME use make_shared if necessary
  TcpConnectionPtr conn( new TcpConnection(ioLoop,
                                          connName,
                                          sockfd,
                                          localAddr,
                                          peerAddr) );
}
```

## 1.2 发送与读取

再看下其他重要函数:

```c++
  // void send(string&& message); // C++11
  void send(const void* message, int len);
  void send(const StringPiece& message);
  // void send(Buffer&& message); // C++11
  void send(Buffer* message);  // this one will swap data
  void shutdown(); // NOT thread safe, no simultaneous calling
  // void shutdownAndForceCloseAfter(double seconds); // NOT thread safe, no simultaneous calling
  void forceClose();
  void forceCloseWithDelay(double seconds);
  void setTcpNoDelay(bool on);
  // reading or not
  void startRead();
  void stopRead();
  bool isReading() const { return reading_; }; // NOT thread safe, may race with start/stopReadInLoop

```

这一组明显和读写有关。

### 1.2.1 发送

先看下发送:

```c++
void TcpConnection::send(const void* data, int len)
{
  send(StringPiece(static_cast<const char*>(data), len));
}

void TcpConnection::send(const StringPiece& message)
{
  if (state_ ==  kConnected )
  {
    if (loop_->isInLoopThread())
    {
       sendInLoop(message); 
    }
    else
    {
      void (TcpConnection::*fp)(const StringPiece& message) = &TcpConnection::sendInLoop;
      loop_->runInLoop(
          std::bind(fp,
                    this,     // FIXME
                    message.as_string()));
                    //std::forward<string>(message)));
    }
  }
}



```

kConnected 是在建立连接的时候设置的:

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

再看`sendInLoop`

```c++
void TcpConnection::sendInLoop(const void* data, size_t len)
{
  ...
  if (state_ == kDisconnected)
  {
    LOG_WARN << "disconnected, give up writing";
    return;
  }
  // if no thing in output queue, try writing directly
  if (!channel_->isWriting() && outputBuffer_.readableBytes() == 0)
  {
     nwrote = sockets::write(channel_->fd(), data, len);
     if (nwrote >= 0)
    {
      remaining = len - nwrote;
      if (remaining == 0 && writeCompleteCallback_)
      {
        loop_->queueInLoop(std::bind(writeCompleteCallback_, shared_from_this()));
      }
    }
    else // nwrote < 0
    {
       // ...
    }
  }

  assert(remaining <= len);
  if (!faultError && remaining > 0)
  {
     ...
     outputBuffer_.append(static_cast<const char*>(data)+nwrote, remaining);
    if (!channel_->isWriting())
    {
      channel_->enableWriting();
    } 
  }
}


```

两种情况：

1. channel本身没有监听可写事件，且没有需要发送的数据，则直接发送
2. 否则，放入buffer，开启write监听，等待可写回调。

有两个关注的点:

1. buffer怎么实现，它的作用？ 可以想到是**批量发送、读取聚合io**。 这个单独拿一篇文章来分析

> 📌TODO: 分析buffer实现

1. 写回调怎么玩的？

```c++
void TcpConnection::handleWrite()
{
  loop_->assertInLoopThread();
  if (channel_->isWriting())
  {
     ssize_t n = sockets::write(channel_->fd(),
                               outputBuffer_.peek(),
                               outputBuffer_.readableBytes()); 
    if (n > 0)
    {
      outputBuffer_.retrieve(n);
      if (outputBuffer_.readableBytes() == 0)
      {
         channel_->disableWriting();   // 已经发送完,关闭写（笔者注：这样看，每次发送完都有系统调用,  还是觉得io_uring更高效)
         if (writeCompleteCallback_)
        {
          loop_->queueInLoop(std::bind(writeCompleteCallback_, shared_from_this()));
        } 
        if (state_ == kDisconnecting)
        {
          shutdownInLoop();
        }
      }
    }
  }
}


```

还是调用`write`写，一旦写完，则回到 `writeCompleteCallback_`（这个cb是`caller`传进来的）

ok，新问题： 为什么不直接在`sendInLoop`死等`write`? 要绕一圈来回调写？

这是因为 `write`是同步阻塞接口，比如我有100个字节要发送，内核最多支持一次发送50字节，那么就要分两次发送，应用需要同步等待内核完成第一次发送，才能发送第二次，显然这是我们不想要的，所以通过注册 **可写 **监听，等到内核真正可写的时候再通知应用，此时应用在写。在此期间，应用可以去玩自己的其他业务逻辑。

### 1.2.2 读取

读启动， 通过如下接口开启可读监听：

```c++
void TcpConnection::startRead()
{
  loop_->runInLoop(std::bind(&TcpConnection::startReadInLoop, this));
}

void TcpConnection::startReadInLoop()
{
  loop_->assertInLoopThread();
  if (!reading_ || !channel_->isReading())
  {
    channel_-> enableReading ();
    reading_ = true;
  }
}

```

一旦可读：

```c++
void TcpConnection::handleRead(Timestamp receiveTime)
{
  loop_->assertInLoopThread();
  int savedErrno = 0;
   ssize_t n = inputBuffer_.readFd(channel_->fd(), &savedErrno);
   if (n > 0)
  {
     messageCallback_(shared_from_this(), &inputBuffer_, receiveTime); 
  }
  else if (n == 0)
  {
    handleClose();  //  可读事件包含了`close`事件 
  }
  else
  {
    errno = savedErrno;
    LOG_SYSERR << "TcpConnection::handleRead";
    handleError();
  }
}


```

读完后，cb到caller。

> 从上述代码可以看出，可读事件包括:
> 1\. 新连接建立， 参考 [muduo源码分析3-Acceptor(新建连接) & Socket](https://www.wolai.com/tjiBrMJqR5LkYQaK5u7fWm "muduo源码分析3-Acceptor(新建连接) & Socket")
> 2\. 新数据到达，真正可读
> 3\. 连接断开、关闭

### 1.2.3 关闭连接

和关闭相关的接口：

```c++
void shutdown(); // NOT thread safe, no simultaneous calling
// void shutdownAndForceCloseAfter(double seconds); // NOT thread safe, no
// simultaneous calling
void forceClose();
void forceCloseWithDelay(double seconds);

```

先看 `forceClose`:

```c++
void TcpConnection::forceClose()
{
  // FIXME: use compare and swap
  if (state_ == kConnected || state_ == kDisconnecting)
  {
    setState(kDisconnecting);
    loop_->queueInLoop(std::bind(&TcpConnection::forceCloseInLoop, shared_from_this()));
  }
}


void TcpConnection::forceCloseInLoop()
{
  loop_->assertInLoopThread();
  if (state_ == kConnected || state_ == kDisconnecting)
  {
    // as if we received 0 byte in handleRead();
    handleClose();
  }
}

void TcpConnection::handleClose()
{
  loop_->assertInLoopThread();
  LOG_TRACE << "fd = " << channel_->fd() << " state = " << stateToString();
  assert(state_ == kConnected || state_ == kDisconnecting);
  // we don't close fd, leave it to dtor, so we can find leaks easily.
  setState(kDisconnected);
   channel_->disableAll();
 
   TcpConnectionPtr guardThis(shared_from_this());  // 由于connectionCallback_可能析构，所以这里加shared_ptr guard
   connectionCallback_(guardThis); 
  // must be the last line
  closeCallback_(guardThis);
}



```

有两个值得看的点：

1. 这里保护对象生命周期的方式，用一个栈 `std::shared_pt` 来保护 `TcpConnection` 不被析构，因为`connectionCallback_` 可能析构`TcpConnection`对象。 比如如下代码:
   ```c++
   void PubSubClient::onConnection(const TcpConnectionPtr& conn)
   {
     if (conn->connected())
     {
       conn_ = conn;
       // FIXME: re-sub
     }
     else
     {
        conn_.reset();  // reset可能析构tcp connection
      }
    
   }


   ```
2. `connection`相关的cb都放在一个接口中，通过`bool connected() const` 接口判定当前链接是否还有用。 还是上面的代码:
   ```c++
   void PubSubClient::onConnection(const TcpConnectionPtr& conn)
   {
     if ( conn->connected() )  // 连接建立
     {
       conn_ = conn;
       // FIXME: re-sub
     }
     else  // 连接释放
     {
       conn_.reset();
     }
    
   }
   ```

简单总结： TcpConnection 为连接抽象，包含读写和关闭连接功能。对外全部提供为cb接口。

# 2 TcpServer

前文其实已经提到过几次。

```c++
#include "muduo/base/Atomic.h"
#include "muduo/base/Types.h"
#include "muduo/net/TcpConnection.h"

#include <map>

class Acceptor;
class EventLoop;
class EventLoopThreadPool;


/// TCP server, supports single-threaded and thread-pool models.
///
/// This is an interface class, so don't expose too much details.
class TcpServer : noncopyable
{
 public:
  typedef std::function<void(EventLoop*)> ThreadInitCallback;
  enum Option
  {
    kNoReusePort,
    kReusePort,
  };

  //TcpServer(EventLoop* loop, const InetAddress& listenAddr);
  TcpServer(EventLoop* loop,
            const InetAddress& listenAddr,
            const string& nameArg,
            Option option = kNoReusePort);
  ~TcpServer();   // force out-line dtor, for std::unique_ptr members. 

  
  private:
     void newConnection(int sockfd, const InetAddress& peerAddr); 
    
    
  std::unique_ptr<Acceptor> acceptor_; // avoid revealing Acceptor
  };
 #include "muduo/base/Atomic.h"#include "muduo/base/Types.h"#include "muduo/net/TcpConnection.h"#include <map>namespace muduo {namespace net {class Acceptor;class EventLoop;class EventLoopThreadPool;
```

上面代码高亮了两行：

1. 把析构定位 out-line (即在.cc文件中实现），这样 Acceptor 这类incomplete-type才能编译通过（因为Acceptor 是前置声明的）。 笔者尝试把 `TcpServer`的析构函数改为inline的，报错如下：

   ![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage_lusrEcDLSk.png)

   **这是为什么？** 这是因为如果把析构函数改为inline的，后续其他.cc（translation unit)包含这个头文件的时候，看到了`TcpServer`析构函数，这个析构函数又会调用`unique_ptr`的析构函数（见[std::unique\_ptr](https://www.wolai.com/5kf4Mci5ETPySzSafqkAiS "std::unique_ptr")), `unique_ptr`会调用`delete Acceptor*`, 此时是需要`Acceptor`的完整类型的。

   **如果换成****`std::shared_ptr`****还有问题吗？  **神奇的是，换成 `std::shared_ptr`就没问题了。 shared\_ptr 原理可见 [std::shared\_ptr](https://www.wolai.com/w8mFh9W9xfoz12i5ENRv74 "std::shared_ptr"). 这里看一行关键代码:
   ```c++
   ~__shared_count() noexcept {
     if (_M_pi != nullptr)  
       _M_pi->_M_release();  // 内部调用析构，但是析构都是虚函数
   }
   
   ```
2. `newConnection` 是核心函数，前面几篇文章页反复提到。

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

每次有新连接(Acceptor回调过来）， 创建`connection`对象，注册一些回调即可。



# 3 TcpClient

还是从构造函数入手:

```c++
TcpClient::TcpClient(EventLoop* loop,
                     const InetAddress& serverAddr,
                     const string& nameArg)
  : loop_(CHECK_NOTNULL(loop)),
     connector_(new Connector(loop, serverAddr)),
     name_(nameArg),
    connectionCallback_(defaultConnectionCallback),
    messageCallback_(defaultMessageCallback),
    retry_(false),
    connect_(true),
    nextConnId_(1)
{
  connector_->setNewConnectionCallback(
      std::bind(&TcpClient:: newConnection , this, _1));
  // FIXME setConnectFailedCallback
  LOG_INFO << "TcpClient::TcpClient[" << name_
           << "] - connector " << get_pointer(connector_);
}
```

重要的是 `connector_`对象。

```c++
  ConnectorPtr connector_; // avoid revealing Connector

```

这个和`Acceptor`的职责类型，`Acceptor`负责建立新连接，这个类作为Client的代理，负责发起连接。

## 3.1 Connector

客户端发起连接的具体实现。

```c++
class Connector : noncopyable,
                  public std::enable_shared_from_this<Connector>
{
 public:
  typedef std::function<void (int sockfd)> NewConnectionCallback;

  Connector(EventLoop* loop, const InetAddress& serverAddr);
  ~Connector();

  void setNewConnectionCallback(const NewConnectionCallback& cb)
  { newConnectionCallback_ = cb; }

   void start();  // can be called in any thread
   void restart();  // must be called in loop thread
  void stop();  // can be called in any thread
  
   ....
}
```

重点看下`start`函数:

```c++
void Connector::start()
{
  connect_ = true;
  loop_->runInLoop(std::bind(&Connector::startInLoop, this)); // FIXME: unsafe
}

void Connector::startInLoop()
{
  loop_->assertInLoopThread();
  assert(state_ == kDisconnected);
  if (connect_)
  {
    connect();
  }
  else
  {
    LOG_DEBUG << "do not connect";
  }
}


void Connector::connect()
{
  int sockfd = sockets::createNonblockingOrDie(serverAddr_.family());
   int ret = sockets::connect(sockfd, serverAddr_.getSockAddr());
   int savedErrno = (ret == 0) ? 0 : errno;
  switch (savedErrno)
  {
    case 0:
    case EINPROGRESS:
    case EINTR:
    case EISCONN:
       connecting(sockfd);
       break;

    case EAGAIN:
    case EADDRINUSE:
    case EADDRNOTAVAIL:
    case ECONNREFUSED:
    case ENETUNREACH:
      retry(sockfd);
      break;

    case EACCES:
    case EPERM:
    case EAFNOSUPPORT:
    case EALREADY:
    case EBADF:
    case EFAULT:
    case ENOTSOCK:
      LOG_SYSERR << "connect error in Connector::startInLoop " << savedErrno;
      sockets::close(sockfd);
      break;

    default:
      LOG_SYSERR << "Unexpected error in Connector::startInLoop " << savedErrno;
      sockets::close(sockfd);
      // connectErrorCallback_();
      break;
  }
}



```

负责发起`connect`:

```c++
void Connector::connecting(int sockfd)
{
  setState(kConnecting);
  assert(!channel_);
  channel_.reset(new Channel(loop_, sockfd));
  channel_->setWriteCallback(
      std::bind(&Connector::handleWrite, this)); // FIXME: unsafe
  channel_->setErrorCallback(
      std::bind(&Connector::handleError, this)); // FIXME: unsafe

  // channel_->tie(shared_from_this()); is not working,
  // as channel_ is not managed by shared_ptr
  channel_->enableWriting();
}


```

一旦连接连接，创建对应sockfd 的 `channel`， 后面这个一旦这个`fd`可写后，回调`handleWrite`:

```c++
void Connector::handleWrite()
{
  LOG_TRACE << "Connector::handleWrite " << state_;

  if (state_ == kConnecting)
  {
     int sockfd = removeAndResetChannel();
    int err = sockets::getSocketError(sockfd); 
    if (err)
    {
      LOG_WARN << "Connector::handleWrite - SO_ERROR = "
               << err << " " << strerror_tl(err);
      retry(sockfd);
    }
    else if (sockets::isSelfConnect(sockfd))
    {
      LOG_WARN << "Connector::handleWrite - Self connect";
      retry(sockfd);
    }
    else
    {
      setState(kConnected);
       if (connect_)
      {
        newConnectionCallback_(sockfd);
      } 
      else
      {
        sockets::close(sockfd);
      }
    }
  }
  else
  {
    // what happened?
    assert(state_ == kDisconnected);
  }
}


```

绕了一圈，**等待内核可写后**，就立即释放`channel`， 并回调`caller`。

## 3.2 TcpClient

代表客户端发起的连接。

`caller`通过`connect`函数发起连接。

```c++
void TcpClient::connect()
{
  // FIXME: check state
  LOG_INFO << "TcpClient::connect[" << name_ << "] - connecting to "
           << connector_->serverAddress().toIpPort();
  connect_ = true;
  connector_->start();
}

```

`connector_`一旦成功建立连接，回调`newConnection`

```c++
void TcpClient::newConnection(int sockfd)
{
  loop_->assertInLoopThread();
  InetAddress peerAddr(sockets::getPeerAddr(sockfd));
  char buf[32];
  snprintf(buf, sizeof buf, ":%s#%d", peerAddr.toIpPort().c_str(), nextConnId_);
  ++nextConnId_;
  string connName = name_ + buf;

  InetAddress localAddr(sockets::getLocalAddr(sockfd));
  // FIXME poll with zero timeout to double confirm the new connection
  // FIXME use make_shared if necessary
  TcpConnectionPtr conn(new TcpConnection(loop_,
                                          connName,
                                          sockfd,
                                          localAddr,
                                          peerAddr));

  conn->setConnectionCallback(connectionCallback_);
  conn->setMessageCallback(messageCallback_);
  conn->setWriteCompleteCallback(writeCompleteCallback_);
  conn->setCloseCallback(
      std::bind(&TcpClient::removeConnection, this, _1)); // FIXME: unsafe
  {
    MutexLockGuard lock(mutex_);
     connection_ = conn;
   }
  conn->connectEstablished();
}


```

后面的故事和`TcpServer`一模一样了。除了这里这里只保存了一份自己的连接外(注意这个`connection_`用了mutex保护，因为`connection_`还暴露给了外面：


```c++
  TcpConnectionPtr connection() const
  {
    MutexLockGuard lock(mutex_);
    return connection_;
  }

  TcpConnectionPtr connection_ GUARDED_BY(mutex_);

```

`connection_`本质上是一个`std::shared_ptr`， 因为[std::shared\_ptr](https://www.wolai.com/w8mFh9W9xfoz12i5ENRv74 "std::shared_ptr")不是线程安全的，所以加了锁。

> 不过笔者搜了下代码，似乎没人用这个`connection`函数，所以也许去掉`mutex` lock更好？



# 4 TcpServer usage

搜了下源码，在`TcpServer`之上还构建了 `HttpServer`和`RpcServer`。这俩不是笔者关注的重点，就跳过了。&#x20;

下一篇准备分析分析`muduo`的基础工具，包括**日志、buffer、定时器**，然后muduo就到此为止了。

