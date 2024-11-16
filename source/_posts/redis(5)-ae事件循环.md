---
title: redis(5)- ae事件循环
categories: redis
date: 2024-11-16 11:58:54
---

redis底层支持多种事件循环，本文仅看epoll类型的。

<!--more-->

## epoll介绍

> 下面摘抄自GPT

epoll 是 Linux 系统中用于高效处理大量文件描述符 I/O 事件的机制。它比传统的 `select` 和 `poll` 系统调用更高效，尤其是在处理大量连接时。epoll 的核心在于其事件驱动模型，它允许应用程序只关注感兴趣的事件，从而减少系统开销。

epoll 的使用方法主要包含三个系统调用：

**1. `epoll_create1(int flags)`:** 创建一个 epoll 实例，返回一个 epoll 文件描述符。`flags` 参数可以是 0 或 `EPOLL_CLOEXEC`。`EPOLL_CLOEXEC` 表示在 `fork()` 后子进程会自动关闭该 epoll 文件描述符。

```c
#include <sys/epoll.h>

int epfd = epoll_create1(0);  // 创建 epoll 实例
if (epfd == -1) {
    perror("epoll_create1");
    exit(1);
}
```

**2. `epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)`:** 管理 epoll 实例中的文件描述符。`epfd` 是 epoll 文件描述符；`op` 指定操作类型，可以是 `EPOLL_CTL_ADD`（添加）、`EPOLL_CTL_MOD`（修改）或 `EPOLL_CTL_DEL`（删除）；`fd` 是要操作的文件描述符；`event` 指向一个 `epoll_event` 结构体，定义了要监控的事件类型。

```c
struct epoll_event event;
event.events = EPOLLIN | EPOLLET; // 监听读事件，并使用边缘触发模式
event.data.fd = fd; // 要监控的文件描述符

if (epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &event) == -1) {
    perror("epoll_ctl: add");
    exit(1);
}
```

`epoll_event` 结构体定义如下：

```c

struct epoll_event {
    uint32_t events;  // 事件类型
    epoll_data_t data; // 用户数据
};
```

`events` 字段可以是以下事件类型的组合：

- `EPOLLIN`: 可读事件
- `EPOLLOUT`: 可写事件
- `EPOLLERR`: 错误事件
- `EPOLLHUP`: 悬挂事件
- `EPOLLRDHUP`: 对端关闭连接事件 (Linux 2.6.17+)
- `EPOLLET`: 边缘触发模式 (ET)
- `EPOLLONESHOT`: 一次性触发模式

**3. `epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout)`:** 等待 I/O 事件发生。`epfd` 是 epoll 文件描述符；`events` 指向一个 `epoll_event` 数组，用于存储发生的事件；`maxevents` 指定数组大小；`timeout` 指定超时时间（以毫秒为单位）。

```cpp
struct epoll_event events[MAX_EVENTS];
int nfds = epoll_wait(epfd, events, MAX_EVENTS, -1); // 阻塞等待事件
if (nfds == -1) {
    perror("epoll_wait");
    exit(1);
}

for (int i = 0; i < nfds; ++i) {
    if (events[i].events & EPOLLIN) {
        // 处理可读事件
    }
    if (events[i].events & EPOLLOUT) {
        // 处理可写事件
    }
    // ... 处理其他事件
}
```

**边缘触发 (ET) 和水平触发 (LT):**

- **水平触发 (LT):** 默认模式。只要文件描述符处于就绪状态，`epoll_wait` 就会返回事件。即使你没有处理完所有数据，下次调用 `epoll_wait` 仍然会返回事件。
- **边缘触发 (ET):** 只有当文件描述符状态发生变化时，`epoll_wait` 才会返回事件。例如，只有当新的数据到达时，才会返回可读事件。使用 ET 模式需要特别小心，必须确保每次都读取所有可用的数据，否则可能会错过事件。 通常需要配合非阻塞 I/O 使用。

## redis源码分析

先看epoll创建调用链路：

![image-20241116185454042](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20241116185454042.png)

具体初始化代码在`aeCreateEventLoop`

![image-20241116185606116](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20241116185606116.png)

`aeCreateEventLoop` 支持传入一个 `setsize` 参数，该参数限制了最大可注册的事件个数。**默认为 maxclients(10000) + CONFIG_FDSET_INCR(128)**

![image-20241116185826442](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20241116185826442.png)

这里的eventLoop定义：

```cpp
/* State of an event based program */
typedef struct aeEventLoop {
    int maxfd;   /* highest file descriptor currently registered */
    int setsize; /* max number of file descriptors tracked */
    long long timeEventNextId;
    time_t lastTime;     /* Used to detect system clock skew */
    aeFileEvent *events; /* Registered events */
    aeFiredEvent *fired; /* Fired events */
    aeTimeEvent *timeEventHead;
    int stop;
    void *apidata; /* This is used for polling API specific data */
    aeBeforeSleepProc *beforesleep;
} aeEventLoop;

/* File event structure */
typedef struct aeFileEvent {
    int mask; /* one of AE_(READABLE|WRITABLE) */
    aeFileProc *rfileProc;
    aeFileProc *wfileProc;
    void *clientData;
} aeFileEvent;

/* Time event structure */
typedef struct aeTimeEvent {
    long long id; /* time event identifier. */
    long when_sec; /* seconds */
    long when_ms; /* milliseconds */
    aeTimeProc *timeProc;
    aeEventFinalizerProc *finalizerProc;
    void *clientData;
    struct aeTimeEvent *next;
} aeTimeEvent;

/* A fired event */
typedef struct aeFiredEvent {
    int fd;
    int mask;
} aeFiredEvent;

```

主体逻辑分配event array，初始化些变量，然后调用`aeApiCreate` :

![image-20241116190047751](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20241116190047751.png)

> 这里传给epoll_create的1024没啥用。查了下manual：
>
>     epoll_create()  creates  a new epoll(7) instance.  Since Linux 2.6.8, the size argument is ignored, but must be greater than
>     zero;
>     In  the  initial epoll_create() implementation, the size argument informed the kernel of the number of file descriptors that
>     the caller expected to add to the epoll instance.  The kernel used this information as a hint for the  amount  of  space  to
>     initially  allocate  in  internal data structures describing events.  (If necessary, the kernel would allocate more space if
>     the caller's usage exceeded the hint given in size.)  Nowadays, this hint is no  longer  required  (the  kernel  dynamically
>     sizes  the  required data structures without needing the hint), but size must still be greater than zero, in order to ensure
>     backward compatibility when new epoll applications are run on older kernels.

看了怎么创建的，再看下怎么使用。使用 `epoll_ctl` 搜索：

![image-20241116190451087](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20241116190451087.png)

用的地方还挺多的，这里就分析`initServer`

先看`aeCrerateFileEvent`的定义:

```cpp
int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
        aeFileProc *proc, void *clientData)
{
    if (fd >= eventLoop->setsize) {
        errno = ERANGE;
        return AE_ERR;
    }
    aeFileEvent *fe = &eventLoop->events[fd];

    if (aeApiAddEvent(eventLoop, fd, mask) == -1)
        return AE_ERR;
    fe->mask |= mask;
    if (mask & AE_READABLE) fe->rfileProc = proc;
    if (mask & AE_WRITABLE) fe->wfileProc = proc;
    fe->clientData = clientData;
    if (fd > eventLoop->maxfd)
        eventLoop->maxfd = fd;
    return AE_OK;
}

static int aeApiAddEvent(aeEventLoop *eventLoop, int fd, int mask) {
    aeApiState *state = eventLoop->apidata;
    struct epoll_event ee = {0}; /* avoid valgrind warning */
    /* If the fd was already monitored for some event, we need a MOD
     * operation. Otherwise we need an ADD operation. */
    int op = eventLoop->events[fd].mask == AE_NONE ?
            EPOLL_CTL_ADD : EPOLL_CTL_MOD;

    ee.events = 0;
    mask |= eventLoop->events[fd].mask; /* Merge old events */
    if (mask & AE_READABLE) ee.events |= EPOLLIN;
    if (mask & AE_WRITABLE) ee.events |= EPOLLOUT;
    ee.data.fd = fd;
    if (epoll_ctl(state->epfd,op,fd,&ee) == -1) return -1;
    return 0;
}

```

传入fd和对应mask，以及事件响应后的proc，和一个context。然后调用`aeApiAddEvent`，底层就是`epoll_ctl`来为fd添加event。

ok,回到`initServer`：

![image-20241116190550527](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20241116190550527.png)

第一个`for`循环是最重要的。不过这里的`ipfd数组`哪来的?

该数组定义在`redisServer`结构体:

```cpp
    int ipfd[CONFIG_BINDADDR_MAX]; /* TCP socket file descriptors */
    int ipfd_count;             /* Used slots in ipfd[] */
```

CONFIG_BINDADDR_MAX默认16.

ipfd的初始化在:

![image-20241116190935768](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20241116190935768.png)

这个函数的功能注释如下:

```cpp

/* Initialize a set of file descriptors to listen to the specified 'port'
 * binding the addresses specified in the Redis server configuration.
 *
 * The listening file descriptors are stored in the integer array 'fds'
 * and their number is set in '*count'.
 *
 * The addresses to bind are specified in the global server.bindaddr array
 * and their number is server.bindaddr_count. If the server configuration
 * contains no specific addresses to bind, this function will try to
 * bind * (all addresses) for both the IPv4 and IPv6 protocols.
 *
 * On success the function returns C_OK.
 *
 * On error the function returns C_ERR. For the function to be on
 * error, at least one of the server.bindaddr addresses was
 * impossible to bind, or no bind addresses were specified in the server
 * configuration but the function is not able to bind * for at least
 * one of the IPv4 or IPv6 protocols. */
int listenToPort(int port, int *fds, int *count) {

```

翻译下：根据配置的`server.bindaddr`和传入的port，配置一堆fd来监听这些地址+port。如果没有配置`server.bindaddr`，则默认绑定到所有ip。

所以回到前面的for循环，`aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE, acceptTcpHandler,NULL)`一行，默认添加了16个fd的可读事件，回调函数为`acceptTcpHandler`。

回忆上文`aeCreateFileEvent`实现中的一行`aeFileEvent *fe = &eventLoop->events[fd]`, 其实这里还蛮取巧的，因为linux（类unix系统应该都一样）中每个进程分配fd都是从当前可用的最小的开始分配，除开默认每个进程都有的0,1,2. 那么就是从3开始分配。所以这里直接将fd用作index来索引events了。不过其他流程注册的fd有多大？？？还好events默认大小也挺大的(1w+). 

for循环完成后，又为socket和pipe也加了可读的事件。

ok，常规事件添加完成，redis还有超时事件。

![image-20241116191820476](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20241116191820476.png)

实现上是个单链表, 新添加的超时事件插入到超时链表头。（笔者：那每次都从扫描整表？？？）

```c
long long aeCreateTimeEvent(aeEventLoop *eventLoop, long long milliseconds,
        aeTimeProc *proc, void *clientData,
        aeEventFinalizerProc *finalizerProc)
{
    long long id = eventLoop->timeEventNextId++;
    aeTimeEvent *te;

    te = zmalloc(sizeof(*te));
    if (te == NULL) return AE_ERR;
    te->id = id;
    aeAddMillisecondsToNow(milliseconds,&te->when_sec,&te->when_ms);
    te->timeProc = proc;
    te->finalizerProc = finalizerProc;
    te->clientData = clientData;
    te->next = eventLoop->timeEventHead;
    eventLoop->timeEventHead = te;
    return id;
}


/* Time event structure */
typedef struct aeTimeEvent {
    long long id; /* time event identifier. */
    long when_sec; /* seconds */
    long when_ms; /* milliseconds */
    aeTimeProc *timeProc;
    aeEventFinalizerProc *finalizerProc;
    void *clientData;
    struct aeTimeEvent *next;
} aeTimeEvent;

```

好，到此，eventloop的创建的添加都说完了。再看看触发，触发的关键字是 `epoll_wait`

![image-20241116192107455](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20241116192107455.png)

都是通过`aeProcessEvents`函数处理，一个来自main, 一个是些rdb和aof的处理。这里关心main函数的:

```cpp
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        aeProcessEvents(eventLoop, AE_ALL_EVENTS);
    }
}
```

每次处理event前，有个before处理(这个不关心)。 看`aeProcessEvents`

函数注释：

```cpp
/* Process every pending time event, then every pending file event
 * (that may be registered by time event callbacks just processed).
 * Without special flags the function sleeps until some file event
 * fires, or when the next time event occurs (if any).
 *
 * If flags is 0, the function does nothing and returns.
 * if flags has AE_ALL_EVENTS set, all the kind of events are processed.
 * if flags has AE_FILE_EVENTS set, file events are processed.
 * if flags has AE_TIME_EVENTS set, time events are processed.
 * if flags has AE_DONT_WAIT set the function returns ASAP until all
 * the events that's possible to process without to wait are processed.
 *
 * The function returns the number of events processed. */
int aeProcessEvents(aeEventLoop *eventLoop, int flags)

```

先处理time event，再出来 file event。 默认flag是`AE_ALL_EVENTS`，处理所有事件。

先从Timer中（单链表）找到最近的要过期的event。如果有，则计算该event距离当前的时间，并保存在tvp中, 给后面的epoll_wait用。

![image-20241116192914384](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20241116192914384.png)

![image-20241116193059075](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20241116193059075.png)

![image-20241116193115970](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20241116193115970.png)

这里的aeSearchNearestTimer会扫描全list，找到最近超时的。（只能说当前redis只有一个超时时间，即serverCron，不然这个全扫有点夸张了）

如果tvp不为空，说明有time event，如果已经过期，则该超时时间被设置为0，epoll_wait立即返回。否则等待到超时。如果evp为空，说明没有超时事件（笔者注：可以在wait的时候又注册新的超时事件吗？）, 则无限等待。

epoll_wait返回后，说明有要处理的event，把他们记在fired数组中。

回到上层，做handler处理。

```cpp
        numevents = aeApiPoll(eventLoop, tvp);
        for (j = 0; j < numevents; j++) {
            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
            int mask = eventLoop->fired[j].mask;
            int fd = eventLoop->fired[j].fd;
            int rfired = 0;

	    /* note the fe->mask & mask & ... code: maybe an already processed
             * event removed an element that fired and we still didn't
             * processed, so we check if the event is still valid. */
            if (fe->mask & mask & AE_READABLE) {
                rfired = 1;
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);  
            }
            if (fe->mask & mask & AE_WRITABLE) {
                if (!rfired || fe->wfileProc != fe->rfileProc)
                    fe->wfileProc(eventLoop,fd,fe->clientData,mask);
            }
            processed++;
        }

```

最后处理Time event:

```cpp
    /* Check time events */
    if (flags & AE_TIME_EVENTS)
        processed += processTimeEvents(eventLoop);

```

整个`processTimeEvents` 是一个while循环，遍历超时链表（又是一次全遍历）:

![image-20241116193947553](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20241116193947553.png)

找到超时的，然后timeProc回调，如果返回了`AE_NOMORE(-1)`，则然后标成可删除的，等到 **下次处理时再删除** (为什么要等下次epoll调度？还是我的理解有误？或许是当前redis的唯一定时任务是常驻后台的，不用关心这个问题)。

![image-20241116194207785](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20241116194207785.png)

好，分析到这里，总体来说，还是蛮简单的。
