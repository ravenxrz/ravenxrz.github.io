---
title: redis(3) - 数据库
categories: redis
date: 2024-06-27 20:48
---

本节介绍redis db相关的一些概念，包括redis db的key space，过期key处理， 数据库通知机制。

<!--more-->

一个redisServer有多个db:

```
struct redisServer {
    // 数据库
    redisDb *db;
    int dbnum;                      /* Total number of configured DBs */
};
```

客户端可通过 `SELECT` 命令切换db，在代码中，redisClient 有指向db的指针，通过改变该指针来切换:

```
typedef struct redisClient {
    // 当前正在使用的数据库
    redisDb *db;
};
```


# kv空间

在 `redisDB` 中存在一个dict，这是该db所有kv存储的地方:

```
typedef struct redisDb {

    // 数据库键空间，保存着数据库中的所有键值对
    dict *dict;                 /* The keyspace for this DB */
} redisDb;

```

## 读写操作的附加效果

- 更新 keyspace_hits or keyspace_misses
- 更新最新访问时间lru
- lazy删除已经过期的key
- 修改过key，更新dirty计数器，用于后续持久化

# 过期key

## 过期时间的设置

redis支持给key设定秒级别或者毫秒级别的过期时间，相关命令:

- EXPLIRE, PEXPIRE 设置过期时间
- EXPIREAT, PEXPIRARE 设置过期时间点
- TTL 查询过期时间
- PERSIST 移除过期时间

redis在 `redisDB` 中有一个 `expire` 字典，当用户给某个key设置过期时间时，会将该key添加进 `expire`字典，该字典的value为具体的过期时间点(timestamp):
```
typedef struct redisDb {

    // 数据库键空间，保存着数据库中的所有键值对
    dict *dict;                 /* The keyspace for this DB */

    // 键的过期时间，字典的键为键，字典的值为过期事件 UNIX 时间戳
    dict *expires;              /* Timeout of keys with a timeout set */
}
```

> TODO(zhangxingrui): redis 的ttl精度是如何保证的， 特别是毫秒级别精度，还是说只关注lazy delete就行？
>
> 看完本章后，redis的删除策略有两个，一个lazy deletion, 一个定期删除，所以精度和定期删除的精度一致。 定期删除由 redis内部 serverCron 周期决定。待分析。

## 删除过期键的策略

通常来说，过期键删除策略包含以下几种：

- 定时器删除，对每个过期键设定一个定时器，定时时间到后自动立即删除，这种策略对内存友好，但是定时器的实现非消耗cpu。
- lazy删除，过期后不立即删除，而是等着下一次访问顺手删除，对CPU友好，但是对内存不友好
- 定期删除，每隔一段时间，对整个db进行扫描，找出ttl到期的key，进行删除。 是定时删除和lazy删除的折中，但是参数难以制定，如扫描间隔，删除并发等。

**redis的删除策略为: lazy删除+定期删除。**

### **lazy删除**

、实现由: expireIfNeeded 函数实现。所有的读写操作会调用该函数。

```c
int expireIfNeeded(redisDb *db, robj *key) {

    // 取出键的过期时间
    mstime_t when = getExpire(db,key);
    mstime_t now;

    // 没有过期时间
    if (when < 0) return 0; /* No expire for this key */

    /* Don't expire anything while loading. It will be done later. */
    // 如果服务器正在进行载入，那么不进行任何过期检查
    if (server.loading) return 0;

    /* If we are in the context of a Lua script, we claim that time is
     * blocked to when the Lua script started. This way a key can expire
     * only the first time it is accessed and not in the middle of the
     * script execution, making propagation to slaves / AOF consistent.
     * See issue #1525 on Github for more information. */
    now = server.lua_caller ? server.lua_time_start : mstime();

    /* If we are running in the context of a slave, return ASAP:
     * the slave key expiration is controlled by the master that will
     * send us synthesized DEL operations for expired keys.
     *
     * Still we try to return the right information to the caller, 
     * that is, 0 if we think the key should be still valid, 1 if
     * we think the key is expired at this time. */
    // 当服务器运行在 replication 模式时
    // 附属节点并不主动删除 key
    // 它只返回一个逻辑上正确的返回值
    // 真正的删除操作要等待主节点发来删除命令时才执行
    // 从而保证数据的同步
    if (server.masterhost != NULL) return now > when;

    // 运行到这里，表示键带有过期时间，并且服务器为主节点

    /* Return when this key has not expired */
    // 如果未过期，返回 0
    if (now <= when) return 0;

    /* Delete the key */
    server.stat_expiredkeys++;

    // 向 AOF 文件和附属节点传播过期信息
    propagateExpire(db,key);

    // 发送事件通知
    notifyKeyspaceEvent(REDIS_NOTIFY_EXPIRED,
        "expired",key,db->id);

    // 将过期键从数据库中删除
    return dbDelete(db,key);
}
```



### **定期删除（可以借鉴的实现）**

redis内部有定时器，每次 serverCron 函数被调用, event处理之前时， 调用`activeExpireCycle`:

```c
	/* Try to expire a few timed out keys. The algorithm used is adaptive and
 * will use few CPU cycles if there are few expiring keys, otherwise
 * it will get more aggressive to avoid that too much memory is used by
 * keys that can be removed from the keyspace.
 *
 * 函数尝试删除数据库中已经过期的键。
 * 当带有过期时间的键比较少时，函数运行得比较保守，
 * 如果带有过期时间的键比较多，那么函数会以更积极的方式来删除过期键，
 * 从而可能地释放被过期键占用的内存。
 *
 * No more than REDIS_DBCRON_DBS_PER_CALL databases are tested at every
 * iteration.
 *
 * 每次循环中被测试的数据库数目不会超过 REDIS_DBCRON_DBS_PER_CALL 。
 *
 * This kind of call is used when Redis detects that timelimit_exit is
 * true, so there is more work to do, and we do it more incrementally from
 * the beforeSleep() function of the event loop.
 *
 * 如果 timelimit_exit 为真，那么说明还有更多删除工作要做，
 * 那么在 beforeSleep() 函数调用时，程序会再次执行这个函数。
 *
 * Expire cycle type:
 *
 * 过期循环的类型：
 *
 * If type is ACTIVE_EXPIRE_CYCLE_FAST the function will try to run a
 * "fast" expire cycle that takes no longer than EXPIRE_FAST_CYCLE_DURATION
 * microseconds, and is not repeated again before the same amount of time.
 *
 * 如果循环的类型为 ACTIVE_EXPIRE_CYCLE_FAST ，
 * 那么函数会以“快速过期”模式执行，
 * 执行的时间不会长过 EXPIRE_FAST_CYCLE_DURATION 毫秒，
 * 并且在 EXPIRE_FAST_CYCLE_DURATION 毫秒之内不会再重新执行。
 *
 * If type is ACTIVE_EXPIRE_CYCLE_SLOW, that normal expire cycle is
 * executed, where the time limit is a percentage of the REDIS_HZ period
 * as specified by the REDIS_EXPIRELOOKUPS_TIME_PERC define. 
 *
 * 如果循环的类型为 ACTIVE_EXPIRE_CYCLE_SLOW ，
 * 那么函数会以“正常过期”模式执行，
 * 函数的执行时限为 REDIS_HS 常量的一个百分比，
 * 这个百分比由 REDIS_EXPIRELOOKUPS_TIME_PERC 定义。
 */

void activeExpireCycle(int type) {
    /* This function has some global state in order to continue the work
     * incrementally across calls. */
    // 静态变量，用来累积函数连续执行时的数据
    static unsigned int current_db = 0; /* Last DB tested. */
    static int timelimit_exit = 0;      /* Time limit hit in previous call? */
    static long long last_fast_cycle = 0; /* When last fast cycle ran. */

    unsigned int j, iteration = 0;
    // 默认每次处理的数据库数量
    unsigned int dbs_per_call = REDIS_DBCRON_DBS_PER_CALL;
    // 函数开始的时间
    long long start = ustime(), timelimit;

    // 快速模式
    if (type == ACTIVE_EXPIRE_CYCLE_FAST) {
        /* Don't start a fast cycle if the previous cycle did not exited
         * for time limt. Also don't repeat a fast cycle for the same period
         * as the fast cycle total duration itself. */
        // 如果上次函数没有触发 timelimit_exit ，那么不执行处理
        if (!timelimit_exit) return;
        // 如果距离上次执行未够一定时间，那么不执行处理
        if (start < last_fast_cycle + ACTIVE_EXPIRE_CYCLE_FAST_DURATION*2) return;
        // 运行到这里，说明执行快速处理，记录当前时间
        last_fast_cycle = start;
    }

    /* We usually should test REDIS_DBCRON_DBS_PER_CALL per iteration, with
     * two exceptions:
     *
     * 一般情况下，函数只处理 REDIS_DBCRON_DBS_PER_CALL 个数据库，
     * 除非：
     *
     * 1) Don't test more DBs than we have.
     *    当前数据库的数量小于 REDIS_DBCRON_DBS_PER_CALL
     * 2) If last time we hit the time limit, we want to scan all DBs
     * in this iteration, as there is work to do in some DB and we don't want
     * expired keys to use memory for too much time. 
     *     如果上次处理遇到了时间上限，那么这次需要对所有数据库进行扫描，
     *     这可以避免过多的过期键占用空间
     */
    if (dbs_per_call > server.dbnum || timelimit_exit)
        dbs_per_call = server.dbnum;

    /* We can use at max ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC percentage of CPU time
     * per iteration. Since this function gets called with a frequency of
     * server.hz times per second, the following is the max amount of
     * microseconds we can spend in this function. */
    // 函数处理的微秒时间上限
    // ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC 默认为 25 ，也即是 25 % 的 CPU 时间
    timelimit = 1000000*ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC/server.hz/100;
    timelimit_exit = 0;
    if (timelimit <= 0) timelimit = 1;

    // 如果是运行在快速模式之下
    // 那么最多只能运行 FAST_DURATION 微秒 
    // 默认值为 1000 （微秒）
    if (type == ACTIVE_EXPIRE_CYCLE_FAST)
        timelimit = ACTIVE_EXPIRE_CYCLE_FAST_DURATION; /* in microseconds. */

    // 遍历数据库
    for (j = 0; j < dbs_per_call; j++) {
        int expired;
        // 指向要处理的数据库
        redisDb *db = server.db+(current_db % server.dbnum);

        /* Increment the DB now so we are sure if we run out of time
         * in the current DB we'll restart from the next. This allows to
         * distribute the time evenly across DBs. */
        // 为 DB 计数器加一，如果进入 do 循环之后因为超时而跳出
        // 那么下次会直接从下个 DB 开始处理
        current_db++;

        /* Continue to expire if at the end of the cycle more than 25%
         * of the keys were expired. */
        do {
            unsigned long num, slots;
            long long now, ttl_sum;
            int ttl_samples;

            /* If there is nothing to expire try next DB ASAP. */
            // 获取数据库中带过期时间的键的数量
            // 如果该数量为 0 ，直接跳过这个数据库
            if ((num = dictSize(db->expires)) == 0) {
                db->avg_ttl = 0;
                break;
            }
            // 获取数据库中键值对的数量
            slots = dictSlots(db->expires);
            // 当前时间
            now = mstime();

            /* When there are less than 1% filled slots getting random
             * keys is expensive, so stop here waiting for better times...
             * The dictionary will be resized asap. */
            // 这个数据库的使用率低于 1% ，扫描起来太费力了（大部分都会 MISS）
            // 跳过，等待字典收缩程序运行
            if (num && slots > DICT_HT_INITIAL_SIZE &&
                (num*100/slots < 1)) break;

            /* The main collection cycle. Sample random keys among keys
             * with an expire set, checking for expired ones. 
             *
             * 样本计数器
             */
            // 已处理过期键计数器
            expired = 0;
            // 键的总 TTL 计数器
            ttl_sum = 0;
            // 总共处理的键计数器
            ttl_samples = 0;

            // 每次最多只能检查 LOOKUPS_PER_LOOP 个键
            if (num > ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP)
                num = ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP;

            // 开始遍历数据库
            while (num--) {
                dictEntry *de;
                long long ttl;

                // 从 expires 中随机取出一个带过期时间的键
                if ((de = dictGetRandomKey(db->expires)) == NULL) break;
                // 计算 TTL
                ttl = dictGetSignedIntegerVal(de)-now;
                // 如果键已经过期，那么删除它，并将 expired 计数器增一
                if (activeExpireCycleTryExpire(db,de,now)) expired++;
                if (ttl < 0) ttl = 0;
                // 累积键的 TTL
                ttl_sum += ttl;
                // 累积处理键的个数
                ttl_samples++;
            }

            /* Update the average TTL stats for this database. */
            // 为这个数据库更新平均 TTL 统计数据
            if (ttl_samples) {
                // 计算当前平均值
                long long avg_ttl = ttl_sum/ttl_samples;
                
                // 如果这是第一次设置数据库平均 TTL ，那么进行初始化
                if (db->avg_ttl == 0) db->avg_ttl = avg_ttl;
                /* Smooth the value averaging with the previous one. */
                // 取数据库的上次平均 TTL 和今次平均 TTL 的平均值
                db->avg_ttl = (db->avg_ttl+avg_ttl)/2;
            }

            /* We can't block forever here even if there are many keys to
             * expire. So after a given amount of milliseconds return to the
             * caller waiting for the other active expire cycle. */
            // 我们不能用太长时间处理过期键，
            // 所以这个函数执行一定时间之后就要返回

            // 更新遍历次数
            iteration++;

            // 每遍历 16 次执行一次
            if ((iteration & 0xf) == 0 && /* check once every 16 iterations. */
                (ustime()-start) > timelimit)
            {
                // 如果遍历次数正好是 16 的倍数
                // 并且遍历的时间超过了 timelimit
                // 那么断开 timelimit_exit
                timelimit_exit = 1;
            }

            // 已经超时了，返回
            if (timelimit_exit) return;

            /* We don't repeat the cycle if there are less than 25% of keys
             * found expired in the current DB. */
            // 如果已删除的过期键占当前总数据库带过期时间的键数量的 25 %
            // 那么不再遍历
        } while (expired > ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP/4);
    }
}


```

- ACTIVE_EXPIRE_CYCLE_FAST 类型在 beforesleep 中调用
- ACTIVE_EXPIRE_CYCLE_SLOW 类型在 ServerCron 中调用

- FAST类型在`2*ACTIVE_EXPIRE_CYCLE_FAST_DURATION` 间隔内不会多次执行。且FAST类型的扫描时间最长为`ACTIVE_EXPIRE_CYCLE_FAST_DURATION(1s)`
- SLOW类型的扫描时间最长为 25% 的cpu时间: ` timelimit = 1000000*ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC/server.hz/100;`
- 扫描的dbnum由当前最大的serverDB和上一轮是否扫描超时决定，如果上一轮扫描超时，则执行全表扫描。
- 循环扫描每一个expire db
  - 如果一个expire db（db是dict的结构）的hash loading factor低于1%， 则直接跳过该db（可能是有太多无效的扫描？TODO:zhangxingrui 不确定redis是如何扫描的）
  - 对于一个db，一次循环扫描次数最多不超过`ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP 20 ` 次（但是如果expire的key数量过多，会有多次循环`while (expired > ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP/4`）
  - 每扫描16次，检查当前扫描耗时，如果超过timelimt, 则break

> 值得借鉴的有： 分模式处理，不同case用不同模式，扫描过期时间的处理，防止阻塞。
>
> TODO(zhangxingrui) 当前尚不清楚redis的事件处理和前台io的之间的关系，待看。 

额外补充下为什么要检查dict的hash loading factor， 只有大于1%的才继续检查。是因为`dictGetRandomKey`的实现，我们知道redis中的hash table实现采用链式hash, 通过一次random找到hash slot的链表，再通过一次hash确定链表中的一个node。如果hash loading factor 过小，可能要多次循环才能找到一个有值的链表。

```c
dictEntry *dictGetRandomKey(dict *d)
{
    dictEntry *he, *orighe;
    unsigned int h;
    int listlen, listele;

    // 字典为空
    if (dictSize(d) == 0) return NULL;
    ...
	{
        // T = O(N)
        do {
            h = random() & d->ht[0].sizemask;
            he = d->ht[0].table[h];
        } while(he == NULL);
    }

    /* Now we found a non empty bucket, but it is a linked
     * list and we need to get a random element from the list.
     * The only sane way to do so is counting the elements and
     * select a random index. */
    // 目前 he 已经指向一个非空的节点链表
    // 程序将从这个链表随机返回一个节点
    listlen = 0;
    orighe = he;
    // 计算节点数量, T = O(1)
    while(he) {
        he = he->next;
        listlen++;
    }
    // 取模，得出随机节点的索引
    listele = random() % listlen;
    he = orighe;
    // 按索引查找节点
    // T = O(1)
    while(listele--) he = he->next;

    // 返回随机节点
    return he;
}
```

## RDB、AOF和复制对过期键的处理

1. RDB， 在BGSAVE或SAVE命令执行时，排除掉过期键

2. AOF

   1. AOF载入
      1. 如果以主模式AOF加载时，不会加载过期键
      2. 如果以从模式加载，会把所有键加载，包括过期键。在主从同步时，从数据库会被清空，所以一般没有问题（TODO：zhangxingrui，那还是有问题，如果先来从备机读，怎么搞？还是说代理会检查同步进度）

   2. AOF写入
      1. 如果过期键还没有实际被删除，即lazy deletion或者定期删除。 则AOF不受影响。 当真正被删除时，AOF会像AOF文件追加一条DEL命令。
   3. AOF重写
      1. 和RDB一样，会检测过期键，不会将过期键写入到新的AOF文件中。

3. 复制场景

   1. 从服务器不会主动删除过期键，要等到主服务器发送DEL命令才删除。所以一定会有一个gap，主上面已经没有过期key，从上面还有。此时从主上读key读不到，从从服务器上读能读到。



# 数据库通知

redis从2.8之后支持两种subsribe：

1. 针对某个key的订阅，监控某个key的操作。这被称为key-space-notification
2. 针对某个命令的订阅，监控某个命令的操作。 这被称为key-event-notification

> 具体启用哪些notification，查看 notify-keyspace-event option

## 发送通知

发送通知由notify.c的 notifyKeyspaceEvent函数实现：

```c
/* The API provided to the rest of the Redis core is a simple function:
 *
 * notifyKeyspaceEvent(char *event, robj *key, int dbid);
 *
 * 'event' is a C string representing the event name.
 *
 * event 参数是一个字符串表示的事件名
 *
 * 'key' is a Redis object representing the key name.
 *
 * key 参数是一个 Redis 对象表示的键名
 *
 * 'dbid' is the database ID where the key lives.  
 *
 * dbid 参数为键所在的数据库
 */

void notifyKeyspaceEvent(int type, char *event, robj *key, int dbid) {
    sds chan;
    robj *chanobj, *eventobj;
    int len = -1;
    char buf[24];

    /* If notifications for this class of events are off, return ASAP. */
    // 如果服务器配置为不发送 type 类型的通知，那么直接返回
    if (!(server.notify_keyspace_events & type)) return;

    // 事件的名字
    eventobj = createStringObject(event,strlen(event));

    /* __keyspace@<db>__:<key> <event> notifications. */
    // 发送键空间通知
    if (server.notify_keyspace_events & REDIS_NOTIFY_KEYSPACE) {

        // 构建频道对象
        chan = sdsnewlen("__keyspace@",11);
        len = ll2string(buf,sizeof(buf),dbid);
        chan = sdscatlen(chan, buf, len);
        chan = sdscatlen(chan, "__:", 3);
        chan = sdscatsds(chan, key->ptr);

        chanobj = createObject(REDIS_STRING, chan);

        // 通过 publish 命令发送通知
        pubsubPublishMessage(chanobj, eventobj);

        // 释放频道对象
        decrRefCount(chanobj);
    }

    /* __keyevente@<db>__:<event> <key> notifications. */
    // 发送键事件通知
    if (server.notify_keyspace_events & REDIS_NOTIFY_KEYEVENT) {

        // 构建频道对象
        chan = sdsnewlen("__keyevent@",11);
        // 如果在前面发送键空间通知的时候计算了 len ，那么它就不会是 -1
        // 这可以避免计算两次 buf 的长度
        if (len == -1) len = ll2string(buf,sizeof(buf),dbid);
        chan = sdscatlen(chan, buf, len);
        chan = sdscatlen(chan, "__:", 3);
        chan = sdscatsds(chan, eventobj->ptr);

        chanobj = createObject(REDIS_STRING, chan);

        // 通过 publish 命令发送通知
        pubsubPublishMessage(chanobj, key);

        // 释放频道对象
        decrRefCount(chanobj);
    }

    // 释放事件对象
    decrRefCount(eventobj);
}
```

- 检查是否启用了 notify-keyspace-evenets
- 然后看当前的event是哪种类型，就发哪种event notification

## 调用处

上述函数再各个数据结构的op中会有调用，如：

![image-20240627083633824](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img/image-20240627083633824.png)

如string的set command中：

```c
void setGenericCommand(redisClient *c, int flags, robj *key, robj *val, robj *expire, int unit, robj *ok_reply, robj *abort_reply) {
...

    // 发送事件通知
    notifyKeyspaceEvent(REDIS_NOTIFY_STRING,"set",key,c->db->id);

    // 发送事件通知
    if (expire) notifyKeyspaceEvent(REDIS_NOTIFY_GENERIC,
        "expire",key,c->db->id);

...
}

```

第一个发送是 string 类型， set event

第二个发送时， 通用类型， expire event



# 补充： serverCron

TODO:zhangxingrui, 继续看书分析。事件处理应该是redis的核心代码。也是个人最关心的一些代码。

在redis的main函数中，调用 `    aeMain(server.el);`， 为redis的事件处理loop:

```
void aeMain(aeEventLoop *eventLoop) {

    eventLoop->stop = 0;

    while (!eventLoop->stop) {

        // 如果有需要在事件处理前执行的函数，那么运行它
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);

        // 开始处理事件
        aeProcessEvents(eventLoop, AE_ALL_EVENTS);
    }
}

```

关注 `aeProcessEvents(eventLoop, AE_ALL_EVENTS);`

# 本节总结

1. redis Server db数组和redis Client的操作db
2. 一个db的key存在一个dict中，此外还有一个expire dict用于存放EXPIRE的key
3. 删除有两个策略，一个lazy删除，一个定期删除。
4. 介绍了RDB、AOF和复制（主从）场景下的过期键处理
5. 从服务器对过期key的处理方式
6. redis对数据库做了修改后的通知方式
