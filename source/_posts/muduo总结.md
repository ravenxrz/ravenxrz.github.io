---
title: muduo总结
categories: muduo
date: 2025-01-12 19:29:54
---


> 📌wolai制作，原文：[https://www.wolai.com/ravenxrz/d4CYcWtC6M86RQyVAuKT2d](https://www.wolai.com/ravenxrz/d4CYcWtC6M86RQyVAuKT2d "https://www.wolai.com/ravenxrz/d4CYcWtC6M86RQyVAuKT2d")

花了1\~2 week的时间过完了经典的C++网络库muduo实现，和学习LevelDB的时候不同，那个时候主要学习C++的编程知识，而muduo主要学习的是设计，当然还有一些编程技巧。本篇总结下通过muduo学习到了哪些内容。

<!--more-->

# 1 IO多路复用

通过redis和muduo的学习，以及公司内部的一些网络库实现。IO多路复用是必备的，且基本都用epoll实现。回顾下什么是io多路复用，有哪些io多路复用方式，优劣势。

**IO多路复用指的是一个线程可以同时监听多个fd**，一旦这些fd中任一一个有响应，系统即可通知该线程来处理，这样每个线程可以同时处理多个链接，提高系统性能（iops和带宽）。



当前**io多路复用的方式** 主要有三种：

1. select：select维护一个fd\_set(底层实现就是一个bitmap），通过`FD_SET`绑定要监听的fd，通过`select`系统调用等待事件ready，再通过`FD_ISSET`判定哪个fd ready，最后处理。

   缺点：
   1. 由于fd\_set底层是一个bitmap，这个bitmap最多接受1024个fd，所以有长度限制
   2. fd在用户态和内核态各维护了一份，每次select需要将fd拷贝一份到内核，内核就绪后，又会拷贝一份到用户态，这里存在拷贝开销
   3. 处理时，通过循环判定fd是否ready，这里有多余开销。
2. poll: poll和select基本一致，但是底层不使用bitmap，而是链表，从而打破1024的上限限制。其余缺点和select一样。
3. epoll：
   1. 优点： 1. epoll监控fd的数据结构是红黑树，相比select和poll，每次修改要监听的fd时的开销更小，因为不用拷贝fd\_set, 内核自己维护这个红黑树即可。2. 事件就绪时，不用循环遍历fd\_set来找到是哪个fd就绪，因为epoll在内核中维护了一个 就绪链表，只有有事件发生的fd会加入到这个链表，用户拿到的fd一定是有事件的fd。 另外，epoll通常搭配非阻塞io使用。
   2. 缺点：linux特有，无法跨平台。
   ![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/操作系统/多路复用/epoll.png)



muduo自身优先使用epoll实现io多路复用。

一些额外思考：

io多路复用本质上还是同步调用，可能涉及多次系统调用（比如epoll\_wait, epoll\_cntl），最近几年**io\_uring**比较火，各大公司的存储系统也基本切到io\_uring, 不知道有系统把io\_uring用到网络库没，io\_uring介入后，可以做异步（觉得异步编程难的，借助协程就可实现同步），且能明显降低系统调用次数。

# 2 Reactor设计模式

关于`Reactor`的详细解释参考：[ 这里](https://www.xiaolincoding.com/os/8_network_system/reactor.html#%E6%BC%94%E8%BF%9B " 这里")。这里简单解释下，本质上reactor就是一个事件监听器(reactor)，监听到事件后给予以分发处理（handler）。这里有两个变量：1. 监听器有多少个？ 2.处理器有多个（多少个线程来处理）? 所以会有多种设计方式, 单reactor/单线程，单reactor/多线程，多reactor/多线程等。一个典型的单reactor/单线程设计模式图如下：

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage_RSKho4xkN6.png)



单reactor通过io多路复用的接口等待有fd事件发生，针对每个fd，如果该fd有事件，则调用对应的handler。redis是这种）。

在muduo中，reactor对应的实现类是`EventLoop`，handler对应的实现类是`Channel`，而muduo是有多个`EventLoop`，每个`loop`有多个`channel`，对应关系如下图（详细分析见 [muduo源码分析2-事件循环（下)](https://www.wolai.com/64a1q1Q3cCT7Dbzm6Bz5Ax "muduo源码分析2-事件循环（下)"))

![](<https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgwhiteboard_exported_image%20(1)_OjywB-Xgii.png>)

明显muduo是典型的多reactor多线程模型，并遵从主从reactor设计，在主reactor中，只负责建立连接（Acceptor跑在这个reactor上），建立连接后，就从多个子reactor（也就是EventloopThreadPool)中选一个出来处理后续io，之后的处理都在这个线程中。换句话说：建立连接的线程和处理io的线程不在同一个线程。

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage_9DXIW6txYf.png)



# 3 可以借鉴的一些技巧

## 3.1 保证生命周期的`Channel` `tie`函数

`channel`的`tie`函数, 用于保证回调函数引用的对象，在真正执行回调时，一定没有析构，具体实现中，用到了`weak_ptr`。 个人的开发生活中，用到`weak_ptr`的机会是很少的，这里算是一个。详细如何使用，见[此处](https://www.wolai.com/ravenxrz/fgMUbiv6DPt5w4HU9RdPvE#rjxSiFLWaK7xs4qk9cAAbi "此处")。


## 3.2 incomplete type 使用unique\_ptr作为成员 ，其类析构函数要使用outline定义

标题有点长，用例子来解释：

```c++
/// TCP server, supports single-threaded and thread-pool models.
///
/// This is an interface class, so don't expose too much details.
class TcpServer : noncopyable
{
 public:
   ~TcpServer();  // force out-line dtor, for std::unique_ptr members.
 
  
  private:
    
   std::unique_ptr<Acceptor> acceptor_; // avoid revealing Acceptor
   };
```

muduo源码中TcpServer有一个unique\_ptr成员，其Acceptor类型在TcpServer.h文件中并没有实现，所以必须将TcpServer的析构函数用outline的方式实现（也就是定义在.cpp文件中）。这是因为std::unique\_ptr在析构时会调用`delete acceptor`, 此时必须要知道`acceptor`的类型定义，换成`shared_ptr`则不存在这个问题。详细见分析: [这里](https://www.wolai.com/ravenxrz/8vpH7tpYRdd8kxC4hn4XrM#fPUcszCbpHmm3yJTRFygMG "这里")。

## 3.3 检测running的线程是否符合预期

muduo保证每个`Eventloop`只能运行在一个线程中，为了强保证这点，EventLoop中有个函数`assertInLoopThread`，底层实现如下

```c++
  bool isInLoopThread() const { return threadId_ == CurrentThread::tid(); }

```

`threadId_`是构造`EventLoop`时初始化的。&#x20;

笔者在公司内项目就有函数要求调用上下文一定要在同一个线程中，不然就可能出现并发bug，笔者本身也踩过多次坑，如果有这个防护，至少能提前发现问题。

另一个使用场景为要求收发socket fd一定要在同一个线程，但是muduo的上层应用很可能在不同线程调用send, receive接口，所以可以做一个检测，如果检测到当前处于非预期线程，就主动切线程，如下函数：

```c++
void EventLoop::runInLoop(Functor cb)
{
  if (isInLoopThread())
  {
    cb();
  }
  else
  {
    queueInLoop(std::move(cb));
  }
}
```

## 3.4 事件wakeup机制

通常要做生产者-消费者模型，一端要通知另一端时，笔者通常会使用条件变量或者信号量的方式，但是在muduo中，使用了eventfd的方式来通知， 这部分分析见：wakeup。

通过eventfd方式，可以统一`Eventloop`被唤醒的方式。

## 3.5 定时器实现

muduo的定时器借助了timefd\_xxx 系统api，在内核的定时器之上构建自己的定时器，虽然精确不够，但是借助了fd + epoll实现了编程统一。这样整个系统中，有新链接、有io读写、有超时事件、有唤醒事件都可以完全统一到EventLoop中。

# 4 附录

## 4.1 select usage

```c++
#include <iostream>
#include <sys/select.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <unistd.h>
#include <string.h>
#include <vector>

#define PORT 8080
#define MAX_CLIENTS 100
#define BUFFER_SIZE 1024

int main() {
    int server_fd, new_socket;
    struct sockaddr_in address;
    int opt = 1;
    int addrlen = sizeof(address);
    char buffer[BUFFER_SIZE] = {0};

    // 创建套接字
    if ((server_fd = socket(AF_INET, SOCK_STREAM, 0)) == 0) {
        perror("socket failed");
        exit(EXIT_FAILURE);
    }

    // 设置套接字选项
    if (setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR | SO_REUSEPORT, &opt, sizeof(opt))) {
        perror("setsockopt");
        exit(EXIT_FAILURE);
    }
    address.sin_family = AF_INET;
    address.sin_addr.s_addr = INADDR_ANY;
    address.sin_port = htons(PORT);

    // 绑定套接字
    if (bind(server_fd, (struct sockaddr *)&address, sizeof(address)) < 0) {
        perror("bind failed");
        exit(EXIT_FAILURE);
    }
    if (listen(server_fd, 3) < 0) {
        perror("listen");
        exit(EXIT_FAILURE);
    }

    std::vector<int> client_sockets;
    fd_set readfds;

    while (true) {
        FD_ZERO(&readfds);
        FD_SET(server_fd, &readfds);
        int max_sd = server_fd;

        for (int sd : client_sockets) {
            FD_SET(sd, &readfds); // 准备监听fd set
            if (sd > max_sd)
                max_sd = sd;
        }

        // 等待事件, 同步等待
        int activity = select(max_sd + 1, &readfds, NULL, NULL, NULL);

        if ((activity < 0) && (errno != EINTR)) {
            perror("select error");
        }

        // 处理新连接
        if (FD_ISSET(server_fd, &readfds)) {
            if ((new_socket = accept(server_fd, (struct sockaddr *)&address, (socklen_t*)&addrlen)) < 0) {
                perror("accept");
                exit(EXIT_FAILURE);
            }
            std::cout << "New connection , socket fd is " << new_socket << std::endl;
            client_sockets.push_back(new_socket);
        }

        // 处理客户端数据
        for (int i = 0; i < client_sockets.size(); i++) {
            int sd = client_sockets[i];
            if (FD_ISSET(sd, &readfds)) { // 处理
                if (read(sd, buffer, BUFFER_SIZE) == 0) {
                    // 客户端断开连接
                    std::cout << "Client disconnected, socket fd is " << sd << std::endl;
                    close(sd);
                    client_sockets.erase(client_sockets.begin() + i);
                } else {
                    // 处理接收到的数据
                    std::cout << "Client " << sd << " sent: " << buffer << std::endl;
                    send(sd, buffer, strlen(buffer), 0);
                    memset(buffer, 0, BUFFER_SIZE);
                }
            }
        }
    }
    return 0;
}
```



## 4.2 epoll usage

```c++
#include <iostream>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <unistd.h>
#include <string.h>
#include <vector>
#include <errno.h>

#define PORT 8080
#define MAX_EVENTS 1024
#define BUFFER_SIZE 1024

int main() {
    int server_fd, new_socket;
    struct sockaddr_in address;
    int opt = 1;
    int addrlen = sizeof(address);
    char buffer[BUFFER_SIZE] = {0};

    // 创建套接字
    if ((server_fd = socket(AF_INET, SOCK_STREAM, 0)) == 0) {
        perror("socket failed");
        exit(EXIT_FAILURE);
    }

    // 设置套接字选项
    if (setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR | SO_REUSEPORT, &opt, sizeof(opt))) {
        perror("setsockopt");
        exit(EXIT_FAILURE);
    }
    address.sin_family = AF_INET;
    address.sin_addr.s_addr = INADDR_ANY;
    address.sin_port = htons(PORT);

    // 绑定套接字
    if (bind(server_fd, (struct sockaddr *)&address, sizeof(address)) < 0) {
        perror("bind failed");
        exit(EXIT_FAILURE);
    }
    if (listen(server_fd, 3) < 0) {
        perror("listen");
        exit(EXIT_FAILURE);
    }

    // 创建 epoll 实例
    int epfd = epoll_create1(0);
    if (epfd == -1) {
        perror("epoll_create1");
        exit(EXIT_FAILURE);
    }

    // 将监听套接字添加到 epoll 实例
    struct epoll_event event;
    event.events = EPOLLIN | EPOLLET; // 边缘触发
    event.data.fd = server_fd;
    if (epoll_ctl(epfd, EPOLL_CTL_ADD, server_fd, &event) == -1) {
        perror("epoll_ctl");
        exit(EXIT_FAILURE);
    }

    struct epoll_event events[MAX_EVENTS];
    while (true) {
        int n = epoll_wait(epfd, events, MAX_EVENTS, -1);
        for (int i = 0; i < n; i++) {
            int fd = events[i].data.fd;
            if (fd == server_fd) {
                // 处理新连接
                if ((new_socket = accept(server_fd, (struct sockaddr *)&address, (socklen_t*)&addrlen)) < 0) {
                    perror("accept");
                    continue;
                }
                std::cout << "New connection, socket fd is " << new_socket << std::endl;
                event.events = EPOLLIN | EPOLLET;
                event.data.fd = new_socket;
                if (epoll_ctl(epfd, EPOLL_CTL_ADD, new_socket, &event) == -1) {
                    perror("epoll_ctl");
                    close(new_socket);
                    continue;
                }
            } else {
                // 处理客户端数据
                int done = 0;
                while (!done) {
                    int count = read(fd, buffer, BUFFER_SIZE);
                    if (count == -1) {
                        if (errno != EAGAIN) {
                            perror("read");
                            done = 1;
                        }
                        break;
                    } else if (count == 0) {
                        done = 1;
                        break;
                    }
                    std::cout << "Client " << fd << " sent: " << buffer << std::endl;
                    send(fd, buffer, count, 0);
                    memset(buffer, 0, BUFFER_SIZE);
                }
                if (done) {
                    std::cout << "Client disconnected, socket fd is " << fd << std::endl;
                    epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
                    close(fd);
                }
            }
        }
    }
    close(server_fd);
    close(epfd);
    return 0;
}
```

## 4.3 epoll LT & ET的区别

LT模式（水平触发）：

- LT模式下，只要文件描述符（fd）还有数据可读，每次调用epoll\_wait都会返回该fd的事件，提醒用户程序去操作。

ET模式（边缘触发）：

- ET模式下，只有当数据第一次到达时才会触发事件，之后直到下次有新的数据流入之前，即使fd中还有剩余数据，也不会再触发事件。所以在ET模式下，read一个fd时，必须将它的buffer读光，即一直读到read的返回值小于请求值。
- epoll工作在ET模式时，必须使用**非阻塞套接字**，以避免由于一个文件的阻塞读/阻塞写操作把处理多个文件描述符的任务饿死。




