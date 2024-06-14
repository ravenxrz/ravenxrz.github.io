---
title: redis读书笔记（二）-redis对象和类型编码结构
categories: redis
date: 2024-06-18 11:58:54
---


# 1. redisObj

redis 在实现kv数据库时，并不直接使用前一章提到的数据结构，而是在其上封装了一个redisObject:

<!--more-->

```c
/* A redis object, that is a type able to hold a string / list / set */

/* The actual Redis Object */
/*
 * Redis 对象
 */
#define REDIS_LRU_BITS 24
#define REDIS_LRU_CLOCK_MAX ((1<<REDIS_LRU_BITS)-1) /* Max value of obj->lru */
#define REDIS_LRU_CLOCK_RESOLUTION 1000 /* LRU clock resolution in ms */
typedef struct redisObject {

    // 类型
    unsigned type:4;

    // 编码
    unsigned encoding:4;

    // 对象最后一次被访问的时间
    unsigned lru:REDIS_LRU_BITS; /* lru time (relative to server.lruclock) */

    // 引用计数
    int refcount;

    // 指向实际值的指针
    void *ptr;

} robj;
```

## 1. type

type 4bits, 包含如下:

- REDIS_STRING
- REDIS_LIST
- REDIS_HASH
- REDIS_SET
- REDIS_ZSET

> kv中，key总是字符串type，redisObject中的type总是说value的type
> 笔者注:实际上3bits已经够了，可能是多预留?

## 2. encoding

ptr指向实际的对象，而具体的对象类型，由encoding决定：

```c
#define REDIS_ENCODING_RAW 0     /* Raw representation, SDS */
#define REDIS_ENCODING_INT 1     /* Encoded as integer, long type interger */
#define REDIS_ENCODING_HT 2      /* Encoded as hash table */
#define REDIS_ENCODING_ZIPMAP 3  /* Encoded as zipmap */
#define REDIS_ENCODING_LINKEDLIST 4 /* Encoded as regular linked list */
#define REDIS_ENCODING_ZIPLIST 5 /* Encoded as ziplist */
#define REDIS_ENCODING_INTSET 6  /* Encoded as intset */
#define REDIS_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
#define REDIS_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
```

### 不同type可对应的底层encoding

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img20240602204047.png)

> 使用 OBJECT ENCODING \[key\] 查看编码类型

# 2. 字符串对象

字符串对象的编码可以是 int, raw 或者 embstr

- int, 如果通过SET命令存储的value是一个long类型可保存的值，那么直接保存，在使用的时候，直接通过强转 `ptr` 得到该值。
- raw, 如果存储的value长度大于32字节，那么使用SDS保存。
- embstr, 如果存储的value长度小于等于32字节，那么字符串对象使用embstr保存

**embstr和raw相比，少一次内存分配。 raw对象会分配redisObj和ptr指向的SDS对象，共计2次。embstr是一次性分配redisObj和ptr指向的SDS对象.**

> 笔者注: 和cpp中shared_ptr一样，new方式和make_shared的区别。

> 浮点型也是字符串内省，但是可以执行一些数字运算。只不过在计算时，首先将字符串转为数字，计算后，又将数字转为字符串。

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img20240602210135.png)

## 编码转换条件

- int -> embstr/raw: 只要出现不是数字，或者长度超过一定阈值(TODO(zhangxingrui): 阈值是多少？)
- embsr -> raw： 长度超过32位，或者原本是embstr，但是对embstr对象做了修改, 如:

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img20240602210656.png)

## 字符串命令的实现

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img20240602210820.png)

# 3. 列表对象

列表对象(type: REDIS_LIST), 底层实现有 ziplist 和 linkedlist 。

## 转换条件

在以下条件时使用 ziplist:

- 列表保存的字符串元素的长度都小于64 字节；
- 列表对象保存的元素数量小于512个。

> 上述两个条件都可以修改ia。 see list-max-ziplist-value 和 list-max-zilist-entries

## list命令实现

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img20240614204907.png)

# 4. hash table对象

hash table 的编码可以是ziplist和 hashtable

hashtable实现hash table对象很好理解。 这里说下 ziplist 是如何实现hashtable的。 对于每次插入hashtable的元素，每次都将kv分别依次插入到ziplist的表尾，所以ziplist实现的hashtable的同一个kv对总是挨在一起。

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img20240614205310.png)

## 转换条件

和list的ziplist一样，满足:

- 列表保存的字符串元素的长度都小于64 字节；
- 列表对象保存的元素数量小于512个。

> 上述两个条件都可以修改ia。 see hash-max-ziplist-value 和 hash-max-zilist-entries

## hashtable命令的实现

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img20240614205440.png)

# 5. 集合对象

集合对象的编码可以是intset或者hashtable.

## 转换条件

使用intset的条件:

- 集合所有保存的对象都是整数
- 集合对象保存的元素数据量不超过512个

> 第二个条件是可以修改的， see set-max-intset-entries

## 集合实现

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img20240614205727.png)

# 6. 有序集合对象

有序集合底层采用ziplist或skiplist实现。

ziplist实现时，成员member和score紧挨在一起，并且按照score排序。

skiplist时，用zset结构表示:

```c
/*
 * 有序集合
 */
typedef struct zset {

    // 字典，键为成员，值为分值
    // 用于支持 O(1) 复杂度的按成员取分值操作
    dict *dict;

    // 跳跃表，按分值排序成员
    // 用于支持平均复杂度为 O(log N) 的按分值定位成员操作
    // 以及范围操作
    zskiplist *zsl;

} zset;
```

实际用同时用了两个数据结构， dict和zskiplist，一个用于O(1)查询，一个用于排序。

## 编码转换

ziplist条件:

- 有序集合保存的元素数量小于128
- 有序集合保存的所有元素成员的长度都小于64字节

> 上述条件可修改, see zset-max-ziplist-entries 和 zset-max-ziplist-value

## 有序集合实现

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img20240614210523.png)

# 思考

上述内容很简单，唯一让我思考的地方在于某个类型从一个数据结构转换到另一个数据结构时的开销，毕竟数据转移+内存分配时耗时的。 在实际使用上就可能出现毛刺。 得看看源码的实现。下面结合list实现的ziplist -> linkedlist 的源码分析：

分析函数 `listTypePush`:

```c
void listTypePush(robj *subject, robj *value, int where) {

    /* Check if we need to convert the ziplist */
    // 是否需要转换编码？
    listTypeTryConversion(subject,value);

    if (subject->encoding == REDIS_ENCODING_ZIPLIST &&
        ziplistLen(subject->ptr) >= server.list_max_ziplist_entries)
            listTypeConvert(subject,REDIS_ENCODING_LINKEDLIST);
...
}
```

函数`listTypeTryConversion`:

```
void listTypeTryConversion(robj *subject, robj *value) {

    // 确保 subject 为 ZIPLIST 编码
    if (subject->encoding != REDIS_ENCODING_ZIPLIST) return;

    if (sdsEncodedObject(value) &&// 如果是raw_str或者emstr 
        // 看字符串是否过长
        sdslen(value->ptr) > server.list_max_ziplist_value)
            // 将编码转换为双端链表
            listTypeConvert(subject,REDIS_ENCODING_LINKEDLIST);
}

```

这里可以看到：

1. 如果已经不是ziplist编码，直接返回

2. 检查当前sds对象的长度是否比 `list_max_zilist_value` 还长，如果满足则转换

   ```
   list_max_ziplist_value -> #define REDIS_LIST_MAX_ZIPLIST_VALUE 64
   ```

另一个条件是:

```cpp
if (subject->encoding == REDIS_ENCODING_ZIPLIST &&
        ziplistLen(subject->ptr) >= server.list_max_ziplist_entries)
            listTypeConvert(subject,REDIS_ENCODING_LINKEDLIST);

#define REDIS_LIST_MAX_ZIPLIST_ENTRIES 512
```

如果整个ziplist的个数大于 `list_max_ziplist_entries` ， 那么则转换。

再来分析 `listTypeConvert`:

```c
void listTypeConvert(robj *subject, int enc) {

    listTypeIterator *li;

    listTypeEntry entry;

    redisAssertWithInfo(NULL,subject,subject->type == REDIS_LIST);

    // 转换成双端链表
    if (enc == REDIS_ENCODING_LINKEDLIST) {

        list *l = listCreate();

        listSetFreeMethod(l,decrRefCountVoid);

        /* listTypeGet returns a robj with incremented refcount */
        // 遍历 ziplist ，并将里面的值全部添加到双端链表中
        li = listTypeInitIterator(subject,0,REDIS_TAIL);
        while (listTypeNext(li,&entry)) listAddNodeTail(l,listTypeGet(&entry));
        listTypeReleaseIterator(li);

        // 更新编码
        subject->encoding = REDIS_ENCODING_LINKEDLIST;

        // 释放原来的 ziplist
        zfree(subject->ptr);

        // 更新对象值指针
        subject->ptr = l;

    } else {
        redisPanic("Unsupported list conversion");
    }
}
```

很明显，一个迭代器+while循环，将原ziplist -> linkedlist。 所以没什么特别的优化。



# 对象共享

redis对整数0-9999之间的整数对象做了内存池，后续用引用技术共享。 这10k个对象是在redis服务初始化时就创建的。

> 使用 OBJECT REFCOUNT obj_name 来查询一个对象的引用计数

> redis只对整数对象共享，字符串共享不太可行，因为字符串共享需要比较整个字符串，cpu开销过大。

# lru字段

在redisObject中，lru字段保存这该对象上次访问的时间戳，用于后续缓存剔除。

> 使用 OBJECT IDLETIME obj_name 查询一个对象的空闲时间， 该时间为 当前时间 - lru记录的值





