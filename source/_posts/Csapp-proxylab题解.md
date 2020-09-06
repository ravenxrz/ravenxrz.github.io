---
title: Csapp-Proxylab题解
categories: Csapp
tags: proxylab
abbrlink: 5d2f135a
date: 2020-09-06 12:27:25
---
本次lab Proxylab。 也是csapp的最后一个lab。

## 1.实验目的

proxylab, 从最终目的来看，我们实现的是一个代理服务器。proxy接收来client的请求，转发给server，并从server中取得数据后，返回给client。另外，proxy从server中取回数据后还需要做cache，这样，当另一个client请求相同的数据时，就不用从server端请求数据，降低了server的压力。

再看看一些细节的要求，要求支持多线程请求，多线程涉及一些同步的问题，所以做的时候一定要考虑哪些是互斥资源，用什么方式同步等。

<!--more-->

## 2.个人实现

下面是整个处理流程图：

![image-20200906094312362](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200906094312362.png)

client向proxy请求数据，proxy将请求放入到一个任务队列中，交由子线程处理。线程负责参数解析，首先看cache中是否有满足的数据，有则直接返回给client，没有则向server请求，server返回数据后，线程将数据插入到cache中，并返回给client。

ok，整个处理流程知道后就可以开始写代码了。

**下文不会给出所有代码，具体代码已贴github:**

[proxy.c](https://github.com/raven-books-source-codes/csapp_lab/blob/master/proxylab/proxylab-handout/proxy.c)

### 1. 主线程工作

先看看主线程做的工作，主线程的工作相对简单，除了一些必要的初始化工作外，只需要开启socket监听，一旦有新连接就将该连接的文件描述符(fd)放入到任务队列中：

```c

int main(int argc, char const *argv[])
{
    int listenfd, connfd;
    char proxy_port[10];
    char client_hostname[BUF_SIZE];
    char client_port[10];
    socklen_t clientlen;
    struct sockaddr_storage clientaddr;
    
    /* init */
    log_set_level(LOG_INFO);
    sbuf_init(&sbuf, WORKER_NUM * 3);
    cache_init(&mycache);
    create_thread_workers(WORKER_NUM);
    
    /* parse port from cmd */
    if (argc != 2) {
        fprintf(stderr, "usage: %s <port>\n", argv[0]);
        log_error("need specify proxy port\n");
        exit(1);
    } else {
        strcpy(proxy_port, argv[1]);
    }
    
    /* start listen */
    listenfd = Open_listenfd(proxy_port);
    while (1) {
        clientlen = sizeof(clientaddr);
        connfd = Accept(listenfd, (SA *) &clientaddr, &clientlen);
        Getnameinfo((SA *) &clientaddr, clientlen, client_hostname, BUF_SIZE,
                    client_port, BUF_SIZE, 0);
        log_info("Accepted connection from (%s, %s)\n", client_hostname,
                 client_port);
        /* put task into task queue */
        sbuf_insert(&sbuf, connfd);
    }
    
    /* free resource  */
    cache_deinit(&mycache);
    return 0;
}
```

### 2. 子线程工作

子线程从任务队列中获取一个“任务“，然后doit

```c
static void *thread_routinue(void *vargs)
{
    int connfd;
    pthread_detach(pthread_self());
    
    while (1) {
        connfd = sbuf_remove(&sbuf);
        doit(connfd);
        close(connfd);
    }
    return NULL;
}
```

doit就复杂些了，因为基本包含了所有工作：

1. 负责获取client请求
2. 解析client请求
3. 转换成可以发送给server的请求
4. 获取数据，可能从cache中获取，也可能从server中获取

```c
static void doit(int fd)
{
    char src_uri[BUF_SIZE], dest_uri[BUF_SIZE], src_version[BUF_SIZE],
            dest_version[BUF_SIZE], src_headers[BUF_SIZE], dest_headers[BUF_SIZE],
            hostname[BUF_SIZE], server_port[BUF_SIZE];
    char *buf = (char *) malloc(sizeof(char) * BUF_SIZE);
    char *request_to_server = (char *) malloc(sizeof(char) * BUF_SIZE);
    if (!buf || !request_to_server) {
        log_error(
                "allocate memory for request which is sending to server faild\n");
        return;
    }
    
    /* init */
    memset(src_uri, 0, BUF_SIZE);
    memset(src_version, 0, BUF_SIZE);
    memset(src_headers, 0, BUF_SIZE);
    memset(hostname, 0, BUF_SIZE);
    memset(buf, 0, BUF_SIZE);
    memset(request_to_server, 0, BUF_SIZE);
    memset(server_port, 0, BUF_SIZE);
    
    /* parse request from client */
    if (parse_client_request(fd, src_uri, src_version, src_headers, hostname,
                             server_port)) {
        log_error("parse client request error\n");
        return;
    }
    log_info("request: %s\n", src_uri);
    log_info("http version: %s\n", src_version);
    log_info("headers: %s\n", src_headers);
    log_info("hostname: %s\n", hostname);
    
    /* convert uri , headers, version to the uri headers version that will be
     * send to server */
    /* append HOSTNAME first */
    sprintf(buf, "Host: %s\r\n", hostname);
    strcat(src_headers, buf);
    if (convert_request(src_uri, src_headers, src_version, dest_uri,
                        dest_headers, dest_version)) {
        log_error("convert request failed\n");
        return;
    }
    
    /* constrcut request sending to server */
    sprintf(buf, "GET %s %s\r\n", dest_uri, dest_version);
    strcat(request_to_server, buf);
    strcat(request_to_server, dest_headers);
    strcat(request_to_server, "\r\n"); /* end of requst */
    log_info("coneverted request to server:\n%s\n", request_to_server);
    
    /* fetch data  */
    /* is data in cache? */
    cache_item *ci;
    ci = cache_get(&mycache, dest_uri);
    int read_len = -1;
    if (ci == NULL) {
        /* fetch from server*/
        if ((read_len = fetch_data_from_server(request_to_server,
                                               strlen(request_to_server) + 1,
                                               hostname, server_port, buf)) == -1) {
            log_error("fetch data from server failed\n");
            return;
        }
        log_info("fetch data from server sucess %d\n", read_len);
        
        /* save to cache */
        log_debug("put data into cache\n");
        cache_put(&mycache, dest_uri, buf, read_len);
    } else {
        /* fetch from cache */
        log_debug("fetch data from cache\n");
        read_len = ci->effect_size;
        memcpy(buf, ci->data, read_len);
    }
    
    /* send data to client */
    int write_len = -1;
    if ((write_len = send_data_to_client(fd, buf, read_len)) == -1) {
        log_error("send data to client failed\n");
        return;
    }
    log_info("send data to client sucess %d\n", write_len);
    
    /* free resource */
    free(request_to_server);
    free(buf);
}
```

整个流程就是这样，处理起来不算来。下面我们主要看看整个流程中设计到的一些互斥资源。

1. sbuf，互斥队列，这个是csapp提供了，我们可以直接用，内部采用“生产者-消费者”的思想+信号量的实现。
2. cache，典型的”读写锁“实现，个人这里采用的是写优先的读写锁信号量实现。

### 3. cache的实现

那下面就说一下cache的实现吧。

文中定义了cache的最大size和一个object的最大size。为了简单起见，我们按照max object size来分块，然后采用LRU算法来替换cache block。

首先看看用到的数据结构:

```c
typedef struct{
   char key[100] ;     /* use request uri as key */
   char data[MAX_OBJECT_SIZE];
   size_t effect_size;      /* effect size */
   unsigned long long age;      /* use to implement LRU */
}cache_item;

typedef struct{
    cache_item *ci;
    size_t ci_num;     /* cache item pair_num */
}cache;
```

cache_item定义了一个cache block的数据及元数据。

cache则由多个cache_item组成了。

然后是要用到的一些函数：

```c
int cache_init(cache *cp);
/* cache_put */
int cache_put(cache *cp, char *key, char *data, size_t size);
/* cache_get */
cache_item *cache_get(cache *cp, char *key);
int cache_deinit(cache *cp);
```

接口很简单，就get、put。

先看看简单的init和deinit：

```c
/**
 * cache init
 * @param cp
 * @return 0 sucess
 *         -1 failed
 */
int cache_init(cache *cp)
{
	if(!cp) return -1;
    /* cache_get cache item pair_num */
    const int ci_num = MAX_CACHE_SIZE / MAX_OBJECT_SIZE;
    cp->ci = (cache_item *) malloc(sizeof(cache_item) * ci_num);
    cp->ci_num = ci_num;
    /* init every cache block */
    for (size_t i = 0; i < cp->ci_num; i++) {
        cp->ci[i].effect_size = 0;
    }
    
    /* semaphore init */
    sem_init(&rcounter_mutex, 0, 1);
    sem_init(&res_mutex, 0, 1);
    sem_init(&w_mutex, 0, 1);
    return 0;
}

/**
 * deinit
 * @param cp
 * @return
 */
int cache_deinit(cache *cp)
{
    if (!cp) return -1;
    
    for (size_t i = 0; i < cp->ci_num; i++) {
        free(&cp->ci[i]);
    }
    return 0;
}
```

说get和put之前，先说说读写锁-写优先的操作。

1. 写-写互斥
2. 写-读互斥
3. 读-读共享
4. 读加锁时，后续有读线程又有写线程时，写线程优先。

看PV实现：

```c
int read_counter = 0;		// 记录读者数
sem_t rc_mutex;				// 读者数的操作锁
sem_t res_mutex;			// 互斥资源锁
sem_t w_mutex;				// 实现写优先

// 写者
while(1){
	P(&w_mutex);				// 占用w_mutex, 阻塞读者
	P(&res_mutex);
	do write
	V(&res_mutex);
	V(&w_mutex);
}

// 读着
while(1){
	P(&w_mutex);
	P(&rc_mutex);
	read_counter++;
	if(read_counter == 1)
		P(&res_mutex);
	V(&rc_mutex);
	V(&w_mutex);
	do read
	P(&rc_mutex);
	read_counter--;
	if(read_counter == 0)
		V(&res_mutex);
	V(&rc_mutex);
}
```

ok, 有了以上基础知识，写cache_get和cache_put就会很简单了。

```c
/**
 * put data into cache
 * @param cp
 * @param key
 * @param data
 * @param size
 * @return  0 sucess
 */
int cache_put(cache *cp, char *key, char *data, size_t size)
{
    static unsigned long long age_counter = 0;
    int empty_block_idx = -1;
    cache_item *ci;
    
    /* write start */
    P(&w_mutex);
    P(&res_mutex);
    
    /* increment age counter */
    age_counter++;
    /* first try to find an empty block */
    for (size_t i = 0; i < cp->ci_num; i++) {
        if (cp->ci[i].effect_size == 0) {
            empty_block_idx = i;
            break;
        }
    }
    
    /* need evict ?*/
    if (empty_block_idx == -1) {
        empty_block_idx = evict(cp);
    }
    
    ci = &cp->ci[empty_block_idx];
    memcpy(ci->key, key, strlen(key));
    memcpy(ci->data, data, size);
    ci->effect_size = size;
    ci->age = age_counter;
    
    /* write end*/
    V(&res_mutex);
    V(&w_mutex);
    return 0;
}


/**
 * get data from cache in terms of key
 * @param cp
 * @param key
 * @return
 *         NULL not found
 */
cache_item *cache_get(cache *cp, char *key)
{
    cache_item *ci = NULL;
    
    /* read start*/
    P(&w_mutex);
    P(&rcounter_mutex);
    rcounter++;
    if (rcounter == 1)
        P(&res_mutex);
    V(&rcounter_mutex);
    V(&w_mutex);
    
    for (size_t i = 0; i < cp->ci_num; i++) {
        if (cp->ci[i].effect_size != 0 &&
            !strcmp(cp->ci[i].key, key)) {
            ci = &cp->ci[i];
            break;
        }
    }
    
    P(&rcounter_mutex);
    rcounter--;
    if (rcounter == 0)
        V(&res_mutex);
    V(&rcounter_mutex);
    return ci;
}
```

### 4. cache实现的额外想法

其实cache最开始我是准备用Log-Strucutre的实现，但是发现实现起来很复杂，而且csapp的测试跑不了太久，Log-Sturcutre的优点没法充分利用。下面简单说一下思路。Log-Strucutre的特点就是Only-Append. 题目的要求是缓存从server返回的数据，而这些数据的大小不一，如果按照上文的实现，**必然会导致很多内部碎片**。而采用only-append的思想，则不存在内部碎片的问题，如下图：

![image-20200906101718082](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200906101718082.png)

但是这样又会存在一个新问题，那就是cache空间是有限的，我们最后总会evict掉某些块，**那就很容产生外部碎片问题，**这该怎么解决呢？开启一个compact线程就好了，compact线程可以周期性的对整个内存空间做压缩处理，去掉外部碎片问题。compact的触发点可以有两个：

1. 周期性触发
2. 当前cache已满

但是每次compact可能也会成为cache的读写性能瓶颈。 所以现在又该怎么办？

可以考虑**“分页思想”，**操作系统里面最经典的知识之一，要想提高系统的内存利用率，降低外部碎片和内部碎片，那就将内存分页，将每个取回的file进行分页,然后插入到“虚拟内存页”，不过这样需要维护页表了。实现难度较大，所以我也没做了，实在没时间继续耗在csapp上，不过还是想将这个思路写在这里，说不定未来我就回来写了呢。

## 3. 总结

这个实验涉及的知识较多，网络编程（c语言的socket一言难尽啊，特别写过python和java后。。。），多线程并发，同步信号量，cache替换策略，每一个都是重要的知识点。所以也还是挺有收获的。

当然了，这也是csapp的最后一个lab，也就意味着我的csapp之旅告一段落了，但是csapp这本书是真的值得每个程序员多看几遍的，里面的每个知识点都相当重要。