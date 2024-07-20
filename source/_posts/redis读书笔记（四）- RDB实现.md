RDB 是redis持久化方式的一种。 本文介绍其实现原理，内容包括：

- redis 保存 RDB和 加载RDB的方式，SAVE和BGSAVE的实现原理
- RDB自动保存原理
- RDB文件的组成部分

<!--more-->

## RDB文件的创建和加载

RDB文件的创建有两种， SAVE和BGSAVE， 其中SAVE会阻塞进程，直到RDB创建完成。BGSAVE采用fork出子进程的方式来避免阻塞。实际上两种方式都会调用 `rdb.c/rdbSave`函数。

两种方式的调用伪代码：

![image-20240630150142909](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630150142909.png)

![image-20240630150247376](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630150247376.png)

> 个人比较关心 轮训处理请求和查看子进程退出信号的实现。
>
> 简单看了下，在 `serverCron` 函数中采用非阻塞的`wait3`来等待子进程结束。

redis会在启动时，查看是否存在rdb文件，如果存在则调用 `rdb.c/rbdLoad` 函数加载。

> 如果存在AOF文件，且服务器开启了AOF持久化功能，则优先采用AOF文件加载。


SAVE命令执行时，客户端的所有请求都会被拒绝。

BGSAVE命令执行时：

- 客户端的正常命令可以执行
- SAVE命令被阻塞，禁止 SAVE 命令和 BGSAVE 命令同时执行
- BGSAVE命令被执行，禁止多个BGSAVE同时执行
- BGREWRITEAOF被延后执行，等待 BGSAVE 执行完成后，再执行 BGREWRITEAOF
- 若果BGREWRITEAOF已经在执行，则BGSAVE会被拒绝

RDB文件被加载时，redis拒绝服务

## 自动RDB保存

用户可以通过 `save` 选项来设置多个保存条件， 只要满足其中一个就会自动保存。

如:

```
save 60 1
save 300 10
```

代表 60s 至少修改过一次就保存。 300s至少修改过10次就保存。

如果用户没有设置，redis的默认设置为：

```
save 900 1
sava 300 10
save 60 10000
```

自动保存的属性保存在 `redisServer` 中的 `saveparams` 数组中:

```cpp
struct redisServer {
  ...
  struct saveparam *saveparams;   /* Save points array for RDB */
  int saveparamslen;              /* Number of saving points */
  ...
};

// 服务器的保存条件（BGSAVE 自动执行的条件）
struct saveparam {
    // 多少秒之内
    time_t seconds;

    // 发生多少次修改
    int changes;
};
```

除了上述属性外， redisServer还维护一个dirty计数器，以及一个lastsave 时间戳。

```c
   // 自从上次 SAVE 执行以来，数据库被修改的次数
  long long dirty;                /* Changes to DB from the last save */
  // 最后一次完成 SAVE 的时间
  time_t lastsave;                /* Unix time of last successful save */
```

redis 每处理一次修改，就会递增dirty计数器。 

有了以上结构，就可以实现自动保存了。redis以100ms每次的频率执行 `serverCron` 函数，在其中的一项工作就是查看是否需要自动保存RDB。

```c
        /* If there is not a background saving/rewrite in progress check if
         * we have to save/rewrite now */
        // 既然没有 BGSAVE 或者 BGREWRITEAOF 在执行，那么检查是否需要执行它们

        // 遍历所有保存条件，看是否需要执行 BGSAVE 命令
         for (j = 0; j < server.saveparamslen; j++) {
            struct saveparam *sp = server.saveparams+j;

            /* Save if we reached the given amount of changes,
             * the given amount of seconds, and if the latest bgsave was
             * successful or if, in case of an error, at least
             * REDIS_BGSAVE_RETRY_DELAY seconds already elapsed. */
            // 检查是否有某个保存条件已经满足了
            if (server.dirty >= sp->changes &&
                server.unixtime-server.lastsave > sp->seconds &&
                (server.unixtime-server.lastbgsave_try >
                 REDIS_BGSAVE_RETRY_DELAY ||
                 server.lastbgsave_status == REDIS_OK))
            {
                redisLog(REDIS_NOTICE,"%d changes in %d seconds. Saving...",
                    sp->changes, (int)sp->seconds);
                // 执行 BGSAVE
                rdbSaveBackground(server.rdb_filename);
                break;
            }
         }
```

## RDB文件结构

redis RDB文件内容：

![image-20240630204117507](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630204117507.png)

其中大写的部分为常量。小写的部分的变量。

- REDIS: 5字节，REDIS字符串
- db_version: redis 版本，4字节
- databases，多个db的kv对数据。
- EOF， 结束位置
- checksum： 上面4部分的checksum

下面重点说说databases部分。

一个db的内容如下：

![image-20240630204716885](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630204716885.png)

- SELECTDB，代表一个操作吗，标识之后要读取的内容是一个db number
- db_number, 代表之后的kv属于哪个db
- key_value_paris: 保存了db的所有键值对，如果键值对有过期时间，过期时间一起保存。

key