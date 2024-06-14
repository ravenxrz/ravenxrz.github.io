---
title: redis源码-ziplist
categories: redis
date: 2024-06-09 11:58:54
---

## 前言

上文中介绍了redis内部的各个数据结构，由于ziplist 是为节约内存而设计，个人t比较感兴趣，本篇详细分析其源码。

<!--more-->

## ziplist基本结构

ziplist的基本结构图：

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img20240602195635.png)

- zlbytes: 4B, 记录整个ziplist占用的内存字节数。
- zltail: 4B, 记录ziplist尾节点距离ziplist的其实地址有多少字节。
- zllen: 2B, 当属性小于UIINT16_MAX时，代码节点数量。等于UINT16_MAX时，节点的真实数量需要遍历整个压缩列表才能算出。
- entry，集体存放元素的地方

## ziplist API

api如下:

```c
unsigned char *ziplistNew(void);
unsigned char *ziplistPush(unsigned char *zl, unsigned char *s, unsigned int slen, int where);
unsigned char *ziplistIndex(unsigned char *zl, int index);
unsigned char *ziplistNext(unsigned char *zl, unsigned char *p);
unsigned char *ziplistPrev(unsigned char *zl, unsigned char *p);
unsigned int ziplistGet(unsigned char *p, unsigned char **sval, unsigned int *slen, long long *lval);
unsigned char *ziplistInsert(unsigned char *zl, unsigned char *p, unsigned char *s, unsigned int slen);
unsigned char *ziplistDelete(unsigned char *zl, unsigned char **p);
unsigned char *ziplistDeleteRange(unsigned char *zl, unsigned int index, unsigned int num);
unsigned int ziplistCompare(unsigned char *p, unsigned char *s, unsigned int slen); unsigned char *ziplistFind(unsigned char *p, unsigned char *vstr, unsigned int vlen, unsigned int skip);
unsigned int ziplistLen(unsigned char *zl);
size_t ziplistBlobLen(unsigned char *zl);
```

下面结合各个api来分析源码：

### ziplistNew

ziplistNew用于新创一个zipList：

```c
unsigned char *ziplistNew(void) {

    // ZIPLIST_HEADER_SIZE 是 ziplist 表头的大小
    // 1 字节是表末端 ZIP_END 的大小
    unsigned int bytes = ZIPLIST_HEADER_SIZE+1;

    // 为表头和表末端分配空间
    unsigned char *zl = zmalloc(bytes);

    // 初始化表属性
    ZIPLIST_BYTES(zl) = intrev32ifbe(bytes);
    ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(ZIPLIST_HEADER_SIZE);
    ZIPLIST_LENGTH(zl) = 0;

    // 设置表末端
    zl[bytes-1] = ZIP_END;

    return zl;
}
```

zipListNew很简单，分配一片内存，大小为一个ziplist head + 1. 其中 ZIPLIST_HEADER_SIZE 为:

```c
#define ZIPLIST_HEADER_SIZE     (sizeof(uint32_t)*2+sizeof(uint16_t))
```

2个uint32_t为 zlbyte 和 zltail, 1个uint16_t 为zllen, 另外的一个字节为zlend, 固定设置为 ZIP_END(`#define ZIP_END 255 `)

来看各字段的初始化值:

- zlbytes: `ZIPLIST_BYTES(zl) = intrev32ifbe(bytes);` -> 当前内存大小，ziplist header size + 1 = 11 字节，一个空ziplist占用11字节
- zltail: `ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(ZIPLIST_HEADER_SIZE);` --> 表头大小即10字节。
- zllen: 0
- zlend: 0xFF

### zipListPush

zipListPush用于向zipList中插入一个节点, 一个节点(entry) 的结构如下:

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img20240602200243.png)

其中:

1. previous_entry_length: 前一个entry的长度，长度可以是1字节或者5字节。 如果value长度小于254， 则为1B， 否则为5B

1. encoding: 如果编码为字符串，根据字符串的长度可分为:

   1. 长度小于等于(2^6 - 1)， 编码长度为1B. 对应编码值:

   ```c
   #define ZIP_STR_06B (0 << 6)
   buf[0] = ZIP_STR_06B | rawlen;
   ```

   2. 长度小于等于(2^14 - 1), 编码长度为2B. 对应编码值:

   ```c
   #define ZIP_STR_14B (1 << 6)
   buf[0] = ZIP_STR_14B | ((rawlen >> 8) & 0x3f);
   buf[1] = rawlen & 0xff;
   ```

   3. 否则，编码长度为5B。对应编码值：

   ```c
    buf[0] = ZIP_STR_32B;
    buf[1] = (rawlen >> 24) & 0xff;
    buf[2] = (rawlen >> 16) & 0xff;
    buf[3] = (rawlen >> 8) & 0xff;
    buf[4] = rawlen & 0xff;
   ```

   如果可以转成整数编码，则为1B. 各值编码规则为:

   1. 对于 0 ~ 12的数字，encoding 为: `ZIP_INT_IMM_MIN+value`, 其中，`ZIP_INT_IMM_MIN`为 `#define ZIP_INT_IMM_MIN 0xf1    /* 11110001 */`
   1. 对于int8_t，encoding 为 `#define ZIP_INT_8B 0xfe`
   1. 对于int16_t， encoding为 `#define ZIP_INT_16B (0xc0 | 0<<4)`
   1. 对于int24_t， encoding为 `#define ZIP_INT_24B (0xc0 | 3<<4)`
   1. 对于int32_t， encoding为 `#define ZIP_INT_32B (0xc0 | 1<<4)`
   1. 对于int64_t， encoding为 `#define ZIP_INT_64B (0xc0 | 2<<4)`

   可以看到encoding不仅代码了本entry的编码类型，同时也包含了具体的entry length

!\[\](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img20240609220002.png =400x)

ok, 有了这些基础知识，看下源码：

```c

unsigned char *ziplistPush(unsigned char *zl, unsigned char *s, unsigned int slen, int where) {

    // 根据 where 参数的值，决定将值推入到表头还是表尾
    unsigned char *p;
    p = (where == ZIPLIST_HEAD) ? ZIPLIST_ENTRY_HEAD(zl) : ZIPLIST_ENTRY_END(zl);

    // 返回添加新值后的 ziplist
    // T = O(N^2)
    return __ziplistInsert(zl,p,s,slen);
}

```

这里我们先分析向表尾插入, 所以结合 ZIPLIST_ENTRY_END 宏分析:

```cpp
#define ZIPLIST_ENTRY_END(zl)   ((zl)+intrev32ifbe(ZIPLIST_BYTES(zl))-1)
```

p 指向zlend 。 然后调用  `__ziplistInsert`:

```c
static unsigned char *__ziplistInsert(unsigned char *zl, unsigned char *p, unsigned char *s, unsigned int slen) {
    // 记录当前 ziplist 的长度
    size_t curlen = intrev32ifbe(ZIPLIST_BYTES(zl)), reqlen, prevlen = 0;
    size_t offset;
    int nextdiff = 0;
    unsigned char encoding = 0;
    long long value = 123456789; /* initialized to avoid warning. Using a value
                                    that is easy to see if for some reason
                                    we use it uninitialized. */
    zlentry entry, tail;

    /* Find out prevlen for the entry that is inserted. */
    if (p[0] != ZIP_END) {
        // 如果 p[0] 不指向列表末端，说明列表非空，并且 p 正指向列表的其中一个节点
        // 那么取出 p 所指向节点的信息，并将它保存到 entry 结构中
        // 然后用 prevlen 变量记录前置节点的长度
        // （当插入新节点之后 p 所指向的节点就成了新节点的前置节点）
        // T = O(1)
        entry = zipEntry(p);
        prevlen = entry.prevrawlen;
    } else {
        // 如果 p 指向表尾末端，那么程序需要检查列表是否为：
        // 1)如果 ptail 也指向 ZIP_END ，那么列表为空；
        // 2)如果列表不为空，那么 ptail 将指向列表的最后一个节点。
        unsigned char *ptail = ZIPLIST_ENTRY_TAIL(zl);
        if (ptail[0] != ZIP_END) {
            // 表尾节点为新节点的前置节点

            // 取出表尾节点的长度
            // T = O(1)
            prevlen = zipRawEntryLength(ptail);
        }
    }

    /* See if the entry can be encoded */
    // 尝试看能否将输入字符串转换为整数，如果成功的话：
    // 1)value 将保存转换后的整数值
    // 2)encoding 则保存适用于 value 的编码方式
    // 无论使用什么编码， reqlen 都保存节点值的长度
    // T = O(N)
    if (zipTryEncoding(s,slen,&value,&encoding)) {
        /* 'encoding' is set to the appropriate integer encoding */
        reqlen = zipIntSize(encoding);
    } else {
        /* 'encoding' is untouched, however zipEncodeLength will use the
         * string length to figure out how to encode it. */
        reqlen = slen;
    }
    /* We need space for both the length of the previous entry and
     * the length of the payload. */
    // 计算编码前置节点的长度所需的大小
    // T = O(1)
    reqlen += zipPrevEncodeLength(NULL,prevlen);
    // 计算编码当前节点值所需的大小
    // T = O(1)
    reqlen += zipEncodeLength(NULL,encoding,slen);

    /* When the insert position is not equal to the tail, we need to
     * make sure that the next entry can hold this entry's length in
     * its prevlen field. */
    // 只要新节点不是被添加到列表末端，
    // 那么程序就需要检查看 p 所指向的节点（的 header）能否编码新节点的长度。
    // nextdiff 保存了新旧编码之间的字节大小差，如果这个值大于 0 
    // 那么说明需要对 p 所指向的节点（的 header ）进行扩展
    // T = O(1)
    nextdiff = (p[0] != ZIP_END) ? zipPrevLenByteDiff(p,reqlen) : 0;

    /* Store offset because a realloc may change the address of zl. */
    // 因为重分配空间可能会改变 zl 的地址
    // 所以在分配之前，需要记录 zl 到 p 的偏移量，然后在分配之后依靠偏移量还原 p 
    offset = p-zl;
    // curlen 是 ziplist 原来的长度
    // reqlen 是整个新节点的长度
    // nextdiff 是新节点的后继节点扩展 header 的长度（要么 0 字节，要么 4 个字节）
    // T = O(N)
    zl = ziplistResize(zl,curlen+reqlen+nextdiff);
    p = zl+offset;

    /* Apply memory move when necessary and update tail offset. */
    if (p[0] != ZIP_END) {
        // 新元素之后还有节点，因为新元素的加入，需要对这些原有节点进行调整

        /* Subtract one because of the ZIP_END bytes */
        // 移动现有元素，为新元素的插入空间腾出位置
        // T = O(N)
        memmove(p+reqlen,p-nextdiff,curlen-offset-1+nextdiff);

        /* Encode this entry's raw length in the next entry. */
        // 将新节点的长度编码至后置节点
        // p+reqlen 定位到后置节点
        // reqlen 是新节点的长度
        // T = O(1)
        zipPrevEncodeLength(p+reqlen,reqlen);

        /* Update offset for tail */
        // 更新到达表尾的偏移量，将新节点的长度也算上
        ZIPLIST_TAIL_OFFSET(zl) =
            intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+reqlen);

        /* When the tail contains more than one entry, we need to take
         * "nextdiff" in account as well. Otherwise, a change in the
         * size of prevlen doesn't have an effect on the *tail* offset. */
        // 如果新节点的后面有多于一个节点
        // 那么程序需要将 nextdiff 记录的字节数也计算到表尾偏移中
        // 这样才能让表尾偏移量正确对齐表尾节点
        // T = O(1)
        tail = zipEntry(p+reqlen);
        if (p[reqlen+tail.headersize+tail.len] != ZIP_END) {
            ZIPLIST_TAIL_OFFSET(zl) =
                intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+nextdiff);
        }
    } else {
        /* This element will be the new tail. */
        // 新元素是新的表尾节点
        ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(p-zl);
    }

    /* When nextdiff != 0, the raw length of the next entry has changed, so
     * we need to cascade the update throughout the ziplist */
    // 当 nextdiff != 0 时，新节点的后继节点的（header 部分）长度已经被改变，
    // 所以需要级联地更新后续的节点
    if (nextdiff != 0) {
        offset = p-zl;
        // T  = O(N^2)
        zl = __ziplistCascadeUpdate(zl,p+reqlen);
        p = zl+offset;
    }

    /* Write the entry */
    // 一切搞定，将前置节点的长度写入新节点的 header
    p += zipPrevEncodeLength(p,prevlen);
    // 将节点值的长度写入新节点的 header
    p += zipEncodeLength(p,encoding,slen);
    // 写入节点值
    if (ZIP_IS_STR(encoding)) {
        // T = O(N)
        memcpy(p,s,slen);
    } else {
        // T = O(1)
        zipSaveInteger(p,value,encoding);
    }

    // 更新列表的节点数量计数器
    // T = O(1)
    ZIPLIST_INCR_LENGTH(zl,1);

    return zl;
}

```

如果新插入, 则prevLen为0; 否则找到表尾最后一个节点，获取其长度：

```c
// 如果 p 指向表尾末端，那么程序需要检查列表是否为：
// 1)如果 ptail 也指向 ZIP_END ，那么列表为空；
// 2)如果列表不为空，那么 ptail 将指向列表的最后一个节点。
unsigned char *ptail = ZIPLIST_ENTRY_TAIL(zl);
if (ptail[0] != ZIP_END) {
    // 表尾节点为新节点的前置节点

    // 取出表尾节点的长度
    // T = O(1)
    prevlen = zipRawEntryLength(ptail);
}
```

接着尝试整数编码:

```c

/* See if the entry can be encoded */
// 尝试看能否将输入字符串转换为整数，如果成功的话：
// 1)value 将保存转换后的整数值
// 2)encoding 则保存适用于 value 的编码方式
// 无论使用什么编码， reqlen 都保存节点值的长度
// T = O(N)
if (zipTryEncoding(s,slen,&value,&encoding)) {
    /* 'encoding' is set to the appropriate integer encoding */
    reqlen = zipIntSize(encoding);
} else {
    /* 'encoding' is untouched, however zipEncodeLength will use the
     * string length to figure out how to encode it. */
    reqlen = slen;
}

```

先看传入的值是否能编码为整数，`zipTryEncoding`：

```c
static int zipTryEncoding(unsigned char *entry, unsigned int entrylen, long long *v, unsigned char *encoding) {
    long long value;

    // 忽略太长或太短的字符串
    if (entrylen >= 32 || entrylen == 0) return 0;

    // 尝试转换
    // T = O(N)
    if (string2ll((char*)entry,entrylen,&value)) {
        /* Great, the string can be encoded. Check what's the smallest
         * of our encoding types that can hold this value. */
        // 转换成功，以从小到大的顺序检查适合值 value 的编码方式
        if (value >= 0 && value <= 12) {
            *encoding = ZIP_INT_IMM_MIN+value;
        } else if (value >= INT8_MIN && value <= INT8_MAX) {
            *encoding = ZIP_INT_8B;
        } else if (value >= INT16_MIN && value <= INT16_MAX) {
            *encoding = ZIP_INT_16B;
        } else if (value >= INT24_MIN && value <= INT24_MAX) {
            *encoding = ZIP_INT_24B;
        } else if (value >= INT32_MIN && value <= INT32_MAX) {
            *encoding = ZIP_INT_32B;
        } else {
            *encoding = ZIP_INT_64B;
        }

        // 记录值到指针
        *v = value;

        // 返回转换成功标识
        return 1;
    }

    // 转换失败
    return 0;
}
```

从上述代码可分析出：

1. 对于数字的字符串形式大于等于32，或者直接为空的，不编码。
1. 对于 0 ~ 12的数字，encoding 为: `ZIP_INT_IMM_MIN+value`, 其中，`ZIP_INT_IMM_MIN`为 `#define ZIP_INT_IMM_MIN 0xf1    /* 11110001 */`
1. 对于int8_t，encoding 为 `#define ZIP_INT_8B 0xfe`
1. 对于int16_t， encoding为 `#define ZIP_INT_16B (0xc0 | 0<<4)`
1. 对于int24_t， encoding为 `#define ZIP_INT_24B (0xc0 | 3<<4)`
1. 对于int32_t， encoding为 `#define ZIP_INT_32B (0xc0 | 1<<4)`
1. 对于int64_t， encoding为 `#define ZIP_INT_64B (0xc0 | 2<<4)`

如果编码成功，调用 `zipIntSize` 换算 reqLen:

```c

reqlen = zipIntSize(encoding);
----
static unsigned int zipIntSize(unsigned char encoding) {

    switch(encoding) {
    case ZIP_INT_8B:  return 1;
    case ZIP_INT_16B: return 2;
    case ZIP_INT_24B: return 3;
    case ZIP_INT_32B: return 4;
    case ZIP_INT_64B: return 8;
    default: return 0; /* 4 bit immediate */
    }

    assert(NULL);
    return 0;
}

```

如果编码失败，则 reqLen直接等于传入的slen，即raw string len.

计算前置节点的**编码**长度：

```c
reqlen += zipPrevEncodeLength(NULL,prevlen);
```

对于空列表， prevLen 要么等于1，要等等于5，根据ZIP_BIGLEN的关系决定。

```c
// -- func: zipPrevEncodeLength
// 仅返回编码 len 所需的字节数量
if (p == NULL) {
    return (len < ZIP_BIGLEN) ? 1 : sizeof(len)+1;

#define ZIP_BIGLEN 254
```

再计算encoding的编码len， 对于整数编码，encoding 编码len总是1B（前面那些magic number 都是1字节），对于非整数编码: 1. 如果传入slen\<=(2^4 - 1), 则编码长度为1B； 如果\<=(2^14 - 1), 编码长度为2B； 其他情况，都是5B。

```c
static unsigned int zipEncodeLength(unsigned char *p, unsigned char encoding, unsigned int rawlen) {
    unsigned char len = 1, buf[5];

    // 编码字符串
    if (ZIP_IS_STR(encoding)) {
        /* Although encoding is given it may not be set for strings,
         * so we determine it here using the raw length. */
        if (rawlen <= 0x3f) {  // 1B
            if (!p) return len;
            buf[0] = ZIP_STR_06B | rawlen;
        } else if (rawlen <= 0x3fff) { // 2B
            len += 1;
            if (!p) return len;
            buf[0] = ZIP_STR_14B | ((rawlen >> 8) & 0x3f);
            buf[1] = rawlen & 0xff;
        } else { // 5B
            len += 4;
            if (!p) return len;
            buf[0] = ZIP_STR_32B;
            buf[1] = (rawlen >> 24) & 0xff;
            buf[2] = (rawlen >> 16) & 0xff;
            buf[3] = (rawlen >> 8) & 0xff;
            buf[4] = rawlen & 0xff;
        }

    // 编码整数
    } else {
        /* Implies integer encoding, so length is always 1. */
        if (!p) return len;
        buf[0] = encoding;
    }

    /* Store this length at p */
    // 将编码后的长度写入 p 
    memcpy(p,buf,len);

    // 返回编码所需的字节数
    return len;
}

```

**完成prevlen和encoding编码len的计算后，本entry的所有信息长度都已经计算出，开始为本节点分配内存**:

```c
// reqlen = cur entry encoded size = prevLen encoded size + encoding encoded size + string encoded size
zl = ziplistResize(zl,curlen+reqlen+nextdiff);
```

先忽略 nextdiff, 假定它是0即可。 curlen为ziplist之前的size， reqlen为当前要插入的节点的size（包括它的一些头信息，如encoding，prevlen，和实际的size，slen或者整数编码后的size)。

```c
static unsigned char *ziplistResize(unsigned char *zl, unsigned int len) {

    // 用 zrealloc ，扩展时不改变现有元素
    zl = zrealloc(zl,len);

    // 更新 bytes 属性
    ZIPLIST_BYTES(zl) = intrev32ifbe(len);

    // 重新设置表末端
    zl[len-1] = ZIP_END;

    return zl;
}
```

> 这里有点恐怖，每次插入一个entry，居然需要realloc一次，暂不知道redis内存分配是否有做优化。

然后插入 `entry`:

1. 首先写入 prevLen:

   ```c

   p += zipPrevEncodeLength(p,prevlen);
   --
   static unsigned int zipPrevEncodeLength(unsigned char *p, unsigned int len) {
       ... 
       {

           // 1 字节
           if (len < ZIP_BIGLEN) {
               p[0] = len;
               return 1;

           // 5 字节
           } else {
               // 添加 5 字节长度标识
               p[0] = ZIP_BIGLEN;
               // 写入编码
               memcpy(p+1,&len,sizeof(len));
               // 如果有必要的话，进行大小端转换
               memrev32ifbe(p+1);
               // 返回编码长度
               return 1+sizeof(len);
           }
       }
   }

   ```

   1.1 如果prevLen的编码长度小于`ZIP_BIGLEN(254)`, 则一字节足够编码

   1.2 否则，用5字节编码prevLen，其中第一个字节为ZIP_BIGLEN, 剩余四字节为实际长度。

1. 接着编码 `encoding`：

```c
// 将节点值的长度写入新节点的 header
p += zipEncodeLength(p,encoding,slen);
```

用前面说的规则，将encoding编码到p。

3. 最后编码value

```c
if (ZIP_IS_STR(encoding)) {
    // T = O(N)
    memcpy(p,s,slen);
} else {
    // T = O(1)
    zipSaveInteger(p,value,encoding);
}
```

并更新 ziplist header中zlen:

```c
// 更新列表的节点数量计数器
// T = O(1)
ZIPLIST_INCR_LENGTH(zl,1);
```

#### 级联更新

前面介绍了向表尾的插入逻辑，现在分析向表中间插入一个节点:

1. 先获取 prevLen 信息:

```c
/* Find out prevlen for the entry that is inserted. */
if (p[0] != ZIP_END) {
    // 如果 p[0] 不指向列表末端，说明列表非空，并且 p 正指向列表的其中一个节点
    // 那么取出 p 所指向节点的信息，并将它保存到 entry 结构中
    // 然后用 prevlen 变量记录前置节点的长度
    // （当插入新节点之后 p 所指向的节点就成了新节点的前置节点）
    // T = O(1)
    entry = zipEntry(p);
    prevlen = entry.prevrawlen;
 }
```

这里的zipEnry:

```c
/*
 * 保存 ziplist 节点信息的结构
 */
typedef struct zlentry {

    // prevrawlen ：前置节点的长度
    // prevrawlensize ：编码 prevrawlen 所需的字节大小
    unsigned int prevrawlensize, prevrawlen;

    // len ：当前节点值的长度
    // lensize ：编码 len 所需的字节大小
    unsigned int lensize, len;

    // 当前节点 header 的大小
    // 等于 prevrawlensize + lensize
    unsigned int headersize;

    // 当前节点值所使用的编码类型
    unsigned char encoding;

    // 指向当前节点的指针
    unsigned char *p;

} zlentry;

/* Return a struct with all information about an entry. 
 *
 * 将 p 所指向的列表节点的信息全部保存到 zlentry 中，并返回该 zlentry 。
 *
 * T = O(1)
 */
static zlentry zipEntry(unsigned char *p) {
    zlentry e;

    // e.prevrawlensize 保存着编码前一个节点的长度所需的字节数
    // e.prevrawlen 保存着前一个节点的长度
    // T = O(1)
    ZIP_DECODE_PREVLEN(p, e.prevrawlensize, e.prevrawlen);

    // p + e.prevrawlensize 将指针移动到列表节点本身
    // e.encoding 保存着节点值的编码类型
    // e.lensize 保存着编码节点值长度所需的字节数
    // e.len 保存着节点值的长度
    // T = O(1)
    ZIP_DECODE_LENGTH(p + e.prevrawlensize, e.encoding, e.lensize, e.len);

    // 计算头结点的字节数
    e.headersize = e.prevrawlensize + e.lensize;

    // 记录指针
    e.p = p;

    return e;
}

```

2. 计算 `nextdiff`:

```c
/* When the insert position is not equal to the tail, we need to
 * make sure that the next entry can hold this entry's length in
 * its prevlen field. */
// 只要新节点不是被添加到列表末端，
// 那么程序就需要检查看 p 所指向的节点（的 header）能否编码新节点的长度。
// nextdiff 保存了新旧编码之间的字节大小差，如果这个值大于 0 
// 那么说明需要对 p 所指向的节点（的 header ）进行扩展
// T = O(1)
nextdiff = (p[0] != ZIP_END) ? zipPrevLenByteDiff(p,reqlen) : 0;
```

> 值得一提的是，如果nextdiff不为0，再为插入节点分配内存的时候，也要加上nextdiff的size:

```c
// curlen 是 ziplist 原来的长度
// reqlen 是整个新节点的长度
// nextdiff 是新节点的后继节点扩展 header 的长度（要么 0 字节，要么 4 个字节）
// T = O(N)
zl = ziplistResize(zl,curlen+reqlen+nextdiff);
```

接着往后移动p所指向的entry, 更新后一个entry的prevLen, 更新整个ziplist的offset

```c
// 新元素之后还有节点，因为新元素的加入，需要对这些原有节点进行调整

/* Subtract one because of the ZIP_END bytes */
// 移动现有元素，为新元素的插入空间腾出位置
// T = O(N)
memmove(p+reqlen,p-nextdiff,curlen-offset-1+nextdiff);

/* Encode this entry's raw length in the next entry. */
// 将新节点的长度编码至后置节点
// p+reqlen 定位到后置节点
// reqlen 是新节点的长度
// T = O(1)
zipPrevEncodeLength(p+reqlen,reqlen);

/* Update offset for tail */
// 更新到达表尾的偏移量，将新节点的长度也算上
ZIPLIST_TAIL_OFFSET(zl) =
    intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+reqlen);

/* When the tail contains more than one entry, we need to take
 * "nextdiff" in account as well. Otherwise, a change in the
 * size of prevlen doesn't have an effect on the *tail* offset. */
// 如果新节点的后面有多于一个节点
// 那么程序需要将 nextdiff 记录的字节数也计算到表尾偏移量中
// 这样才能让表尾偏移量正确对齐表尾节点
// T = O(1)
tail = zipEntry(p+reqlen);
if (p[reqlen+tail.headersize+tail.len] != ZIP_END) {
    ZIPLIST_TAIL_OFFSET(zl) =
        intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+nextdiff);
}
```

如果`nextdiff`不为空，说明更新了插入节点的后一个节点的prevlen, 这样可能带来级联更新:

```c
/* When nextdiff != 0, the raw length of the next entry has changed, so
 * we need to cascade the update throughout the ziplist */
// 当 nextdiff != 0 时，新节点的后继节点的（header 部分）长度已经被改变，
// 所以需要级联地更新后续的节点
if (nextdiff != 0) {
    offset = p-zl;
    // T  = O(N^2)
    zl = __ziplistCascadeUpdate(zl,p+reqlen);
    p = zl+offset;
}
```
