______________________________________________________________________

## title: redis源码-ziplist categories: redis date: 2024-06-9 11:58:55

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
到此步，新节点的内存空间已经分配完毕，且将后续的元素往后移动完成。并更新了ziplist header tail offset(如果下一个节点不是表尾节点的话).

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

下面分析联机更新:

```c
/* When an entry is inserted, we need to set the prevlen field of the next
 * entry to equal the length of the inserted entry. It can occur that this
 * length cannot be encoded in 1 byte and the next entry needs to be grow
 * a bit larger to hold the 5-byte encoded prevlen. This can be done for free,
 * because this only happens when an entry is already being inserted (which
 * causes a realloc and memmove). However, encoding the prevlen may require
 * that this entry is grown as well. This effect may cascade throughout
 * the ziplist when there are consecutive entries with a size close to
 * ZIP_BIGLEN, so we need to check that the prevlen can be encoded in every
 * consecutive entry.
 *
 * 当将一个新节点添加到某个节点之前的时候，
 * 如果原节点的 header 空间不足以保存新节点的长度，
 * 那么就需要对原节点的 header 空间进行扩展（从 1 字节扩展到 5 字节）。
 *
 * 但是，当对原节点进行扩展之后，原节点的下一个节点的 prevlen 可能出现空间不足，
 * 这种情况在多个连续节点的长度都接近 ZIP_BIGLEN 时可能发生。
 *
 * 这个函数就用于检查并修复后续节点的空间问题。
 *
 * Note that this effect can also happen in reverse, where the bytes required
 * to encode the prevlen field can shrink. This effect is deliberately ignored,
 * because it can cause a "flapping" effect where a chain prevlen fields is
 * first grown and then shrunk again after consecutive inserts. Rather, the
 * field is allowed to stay larger than necessary, because a large prevlen
 * field implies the ziplist is holding large entries anyway.
 *
 * 反过来说，
 * 因为节点的长度变小而引起的连续缩小也是可能出现的，
 * 不过，为了避免扩展-缩小-扩展-缩小这样的情况反复出现（flapping，抖动），
 * 我们不处理这种情况，而是任由 prevlen 比所需的长度更长。
 
 * The pointer "p" points to the first entry that does NOT need to be
 * updated, i.e. consecutive fields MAY need an update. 
 *
 * 注意，程序的检查是针对 p 的后续节点，而不是 p 所指向的节点。
 * 因为节点 p 在传入之前已经完成了所需的空间扩展工作。
 *
 * T = O(N^2)
 */
static unsigned char *__ziplistCascadeUpdate(unsigned char *zl, unsigned char *p) {
    size_t curlen = intrev32ifbe(ZIPLIST_BYTES(zl)), rawlen, rawlensize;
    size_t offset, noffset, extra;
    unsigned char *np;
    zlentry cur, next;

    // T = O(N^2)
    while (p[0] != ZIP_END) {

        // 将 p 所指向的节点的信息保存到 cur 结构中
        cur = zipEntry(p);
        // 当前节点的长度
        rawlen = cur.headersize + cur.len;
        // 计算编码当前节点的长度所需的字节数
        // T = O(1)
        rawlensize = zipPrevEncodeLength(NULL,rawlen);

        /* Abort if there is no next entry. */
        // 如果已经没有后续空间需要更新了，跳出
        if (p[rawlen] == ZIP_END) break;

        // 取出后续节点的信息，保存到 next 结构中
        // T = O(1)
        next = zipEntry(p+rawlen);

        /* Abort when "prevlen" has not changed. */
        // 后续节点编码当前节点的空间已经足够，无须再进行任何处理，跳出
        // 可以证明，只要遇到一个空间足够的节点，
        // 那么这个节点之后的所有节点的空间都是足够的
        if (next.prevrawlen == rawlen) break;

        if (next.prevrawlensize < rawlensize) {

            /* The "prevlen" field of "next" needs more bytes to hold
             * the raw length of "cur". */
            // 执行到这里，表示 next 空间的大小不足以编码 cur 的长度
            // 所以程序需要对 next 节点的（header 部分）空间进行扩展

            // 记录 p 的偏移量
            offset = p-zl;
            // 计算需要增加的节点数量
            extra = rawlensize-next.prevrawlensize;
            // 扩展 zl 的大小
            // T = O(N)
            zl = ziplistResize(zl,curlen+extra);
            // 还原指针 p
            p = zl+offset;

            /* Current pointer and offset for next element. */
            // 记录下一节点的偏移量
            np = p+rawlen;
            noffset = np-zl;

            /* Update tail offset when next element is not the tail element. */
            // 当 next 节点不是表尾节点时，更新列表到表尾节点的偏移量
            // 
            // 不用更新的情况（next 为表尾节点）：
            //
            // |     | next |      ==>    |     | new next          |
            //       ^                          ^
            //       |                          |
            //     tail                        tail
            //
            // 需要更新的情况（next 不是表尾节点）：
            //
            // | next |     |   ==>     | new next          |     |
            //        ^                        ^
            //        |                        |
            //    old tail                 old tail
            // 
            // 更新之后：
            //
            // | new next          |     |
            //                     ^
            //                     |
            //                  new tail
            // T = O(1)
            if ((zl+intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))) != np) {
                ZIPLIST_TAIL_OFFSET(zl) =
                    intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+extra);
            }

            /* Move the tail to the back. */
            // 向后移动 cur 节点之后的数据，为 cur 的新 header 腾出空间
            //
            // 示例：
            //
            // | header | value |  ==>  | header |    | value |  ==>  | header      | value |
            //                                   |<-->|
            //                            为新 header 腾出的空间
            // T = O(N)
            memmove(np+rawlensize,
                np+next.prevrawlensize,
                curlen-noffset-next.prevrawlensize-1);
            // 将新的前一节点长度值编码进新的 next 节点的 header
            // T = O(1)
            zipPrevEncodeLength(np,rawlen);

            /* Advance the cursor */
            // 移动指针，继续处理下个节点
            p += rawlen;
            curlen += extra;
        } else {
            if (next.prevrawlensize > rawlensize) {
                /* This would result in shrinking, which we want to avoid.
                 * So, set "rawlen" in the available bytes. */
                // 执行到这里，说明 next 节点编码前置节点的 header 空间有 5 字节
                // 而编码 rawlen 只需要 1 字节
                // 但是程序不会对 next 进行缩小，
                // 所以这里只将 rawlen 写入 5 字节的 header 中就算了。
                // T = O(1)
                zipPrevEncodeLengthForceLarge(p+rawlen,rawlen);
            } else {
                // 运行到这里，
                // 说明 cur 节点的长度正好可以编码到 next 节点的 header 中
                // T = O(1)
                zipPrevEncodeLength(p+rawlen,rawlen);
            }

            /* Stop here, as the raw length of "next" has not changed. */
            break;
        }
    }

    return zl;
}

```

处理比较常见，循环逐渐往后遍历，如果发现后一个节点的prevLen不足以容纳本节点的编码长度 `next.prevrawlensize < rawlensize` 就将后一个节点扩容。值得说的有两点：
1. 级联更新仅处理expand的场景，即某个节点插入/删除后引起后面节点size变大的场景，对于引起后面节点size变小的场景不予处理。也就是说prevLen变成5B后，不会shrink到1B, 是为了避免频繁抖动引起的内存分配。对应代码:
```c
if (next.prevrawlensize > rawlensize) {
    /* This would result in shrinking, which we want to avoid.
     * So, set "rawlen" in the available bytes. */
    // 执行到这里，说明 next 节点编码前置节点的 header 空间有 5 字节
    // 而编码 rawlen 只需要 1 字节
    // 但是程序不会对 next 进行缩小，
    // 所以这里只将 rawlen 写入 5 字节的 header 中就算了。
    // T = O(1)
    zipPrevEncodeLengthForceLarge(p+rawlen,rawlen);
}
```
2. 该算法的时间复杂度为O(n^2), 遍历后面所有entry为O(N)，每个entery memove到后面的位置又为一次O(N)

> 笔者认为每个entry都要resize一把是非常大的浪费，既然ziplist中的entry数不会过多（过多会换为另一种数据结构，比如lst)，完全可以首先遍历一遍ziplist,预计算出最后要expand的size，再一把resize，最后移动一次并更新各entryy的header就可以了。



### zipListDelete

再看删除是如何操作的:

```c
/* Delete a single entry from the ziplist, pointed to by *p.
 * Also update *p in place, to be able to iterate over the
 * ziplist, while deleting entries. 
 *
 * 从 zl 中删除 *p 所指向的节点，
 * 并且原地更新 *p 所指向的位置，使得可以在迭代列表的过程中对节点进行删除。
 *
 * T = O(N^2)
 */
unsigned char *ziplistDelete(unsigned char *zl, unsigned char **p) {

    // 因为 __ziplistDelete 时会对 zl 进行内存重分配
    // 而内存充分配可能会改变 zl 的内存地址
    // 所以这里需要记录到达 *p 的偏移量
    // 这样在删除节点之后就可以通过偏移量来将 *p 还原到正确的位置
    size_t offset = *p-zl;
    zl = __ziplistDelete(zl,*p,1);

    /* Store pointer to current element in p, because ziplistDelete will
     * do a realloc which might result in a different "zl"-pointer.
     * When the delete direction is back to front, we might delete the last
     * entry and end up with "p" pointing to ZIP_END, so check this. */
    *p = zl+offset;

    return zl;
}
```
> 从接口参数来看，p是个二级指针，因为删除过程，ziplist会做realloc，整个ziplist的内存地址可能都变了，所以p作为ziplist中的一个entry地址，也会跟着变，传入二级指针方便更新。

整个操作很简单，记录当前entry到头之间的offset，做删除，然后根据offset还原。看 `__ziplistDelete`:

```c

/* Delete "num" entries, starting at "p". Returns pointer to the ziplist. 
 *
 * 从位置 p 开始，连续删除 num 个节点。
 *
 * 函数的返回值为处理删除操作之后的 ziplist 。
 *
 * T = O(N^2)
 */
static unsigned char *__ziplistDelete(unsigned char *zl, unsigned char *p, unsigned int num) {
    unsigned int i, totlen, deleted = 0;
    size_t offset;
    int nextdiff = 0;
    zlentry first, tail;

    // 计算被删除节点总共占用的内存字节数
    // 以及被删除节点的总个数
    // T = O(N)
    first = zipEntry(p);
    for (i = 0; p[0] != ZIP_END && i < num; i++) {
        p += zipRawEntryLength(p);
        deleted++;
    }

    // totlen 是所有被删除节点总共占用的内存字节数
    totlen = p-first.p;
    if (totlen > 0) {
        if (p[0] != ZIP_END) {

            // 执行这里，表示被删除节点之后仍然有节点存在

            /* Storing `prevrawlen` in this entry may increase or decrease the
             * number of bytes required compare to the current `prevrawlen`.
             * There always is room to store this, because it was previously
             * stored by an entry that is now being deleted. */
            // 因为位于被删除范围之后的第一个节点的 header 部分的大小
            // 可能容纳不了新的前置节点，所以需要计算新旧前置节点之间的字节数差
            // T = O(1)
            nextdiff = zipPrevLenByteDiff(p,first.prevrawlen);
            // 如果有需要的话，将指针 p 后退 nextdiff 字节，为新 header 空出空间
            p -= nextdiff;
            // 将 first 的前置节点的长度编码至 p 中
            // T = O(1)
            zipPrevEncodeLength(p,first.prevrawlen);

            /* Update offset for tail */
            // 更新到达表尾的偏移量
            // T = O(1)
            ZIPLIST_TAIL_OFFSET(zl) =
                intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))-totlen);

            /* When the tail contains more than one entry, we need to take
             * "nextdiff" in account as well. Otherwise, a change in the
             * size of prevlen doesn't have an effect on the *tail* offset. */
            // 如果被删除节点之后，有多于一个节点
            // 那么程序需要将 nextdiff 记录的字节数也计算到表尾偏移量中
            // 这样才能让表尾偏移量正确对齐表尾节点
            // T = O(1)
            tail = zipEntry(p);
            if (p[tail.headersize+tail.len] != ZIP_END) {
                ZIPLIST_TAIL_OFFSET(zl) =
                   intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+nextdiff);
            }

            /* Move tail to the front of the ziplist */
            // 从表尾向表头移动数据，覆盖被删除节点的数据
            // T = O(N)
            memmove(first.p,p,
                intrev32ifbe(ZIPLIST_BYTES(zl))-(p-zl)-1);
        } else {

            // 执行这里，表示被删除节点之后已经没有其他节点了

            /* The entire tail was deleted. No need to move memory. */
            // T = O(1)
            ZIPLIST_TAIL_OFFSET(zl) =
                intrev32ifbe((first.p-zl)-first.prevrawlen);
        }

        /* Resize and update length */
        // 缩小并更新 ziplist 的长度
        offset = first.p-zl;
        zl = ziplistResize(zl, intrev32ifbe(ZIPLIST_BYTES(zl))-totlen+nextdiff);
        ZIPLIST_INCR_LENGTH(zl,-deleted);
        p = zl+offset;

        /* When nextdiff != 0, the raw length of the next entry has changed, so
         * we need to cascade the update throughout the ziplist */
        // 如果 p 所指向的节点的大小已经变更，那么进行级联更新
        // 检查 p 之后的所有节点是否符合 ziplist 的编码要求
        // T = O(N^2)
        if (nextdiff != 0)
            zl = __ziplistCascadeUpdate(zl,p);
    }

    return zl;
}

```

看完insert流程，这个流程就是insert的逆操作，比较简单，需要注意的是:
1. 即使删除n个entry，也只需要resize一次，因为预先遍历了一次ziplist的n个要删除的节点的size，得到 `total_len`.
> 所以为什么级联更新时不采用先遍历再reisze?
2. 删除后，也可能造成级联更新:

```c
/* When nextdiff != 0, the raw length of the next entry has changed, so
 * we need to cascade the update throughout the ziplist */
// 如果 p 所指向的节点的大小已经变更，那么进行级联更新
// 检查 p 之后的所有节点是否符合 ziplist 的编码要求
// T = O(N^2)
if (nextdiff != 0)
    zl = __ziplistCascadeUpdate(zl,p);
```

### 其它

其他函数比较直观简单，不做分析。

## 总结

ziplist是redis为节约内存而设计的，通过**整数编码（这个还蛮节约内存的, 值得借鉴）**，连续内存分配（这个既节约了内存，又更cache friendly，提升访问速度）确实一定程度减低了内存占用，但是源码中充斥着 `ziplistResize`, 带来内存重分配，还有**级联更新的风险（这可能带来时延毛刺）**,个人觉得不是个好的设计。


最后给一个网上的ziplist perf test:

> THROUGHPUT:
> Running the throughput test (heavily pipelined, doing 1M pushes to 1M keys 20 times):
> Linked lists average 1M inserts in 5726 msec, or 175k inserts/sec.
> Zip lists average 1M inserts in 6756 msec, or 148k inserts/sec.
> Bottom line: zip lists provide 4x as much memory for data, at a cost of 20% throughput. Totally worth it for lists of small items.
> 来源: https://groups.google.com/g/redis-db/c/hb2AD5c33Cw
>
> 看起来ziplist相比链表用了4倍内存压缩，但仅减低了20%的吞吐，比我预期的好。但从新版本已经将ziplist切换成listpack来看，ziplist逐渐成为历史:
>
> ziplist, Redis <= 6.2, a space-efficient encoding used for small lists.
> listpack, Redis >= 7.0, a space-efficient encoding used for small lists.
>
> 来源：https://redis.io/docs/latest/commands/object-encoding/



