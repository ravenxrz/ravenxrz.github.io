---
title: Csapp-Cachelab题解
categories: Csapp
tags: cachelab
abbrlink: 153b500d
date: 2020-07-29 18:52:24
---

本次 lab, cachelab.

## 0. 说明

从这个lab开始，终于开始编写一些高级语言的代码了，而不像之前要去分析汇编。但是这并不意味这题目就简单了，实际上，cachelab耗费了我2天多的时间。ok，言归正传，这个lab的目的是什么呢？

cachelab帮助我们理解计算机存储体系中的重要组成部分--cache。 理解cache是如何组织的，如何工作的，又是如何影响我们的程序的性能的。

<!--more-->

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200726101259903.png" style="zoom: 33%;" />

cache站立于整个存储体系的上端（低于寄存器），其重要性不言而喻了。

题目说明：

cachelab只有2个题目。

1. 写一个cache工作的模拟器，给一段内存访问的trace file，根据trace file仿真，得到这段trace file的hit，miss，eviction数。
2. 编写cache友好的代码，具体是给3种给定大小的矩阵A，求A的转置，每种大小都要miss要求。

具体题目请参照cachelab的write up。

## 1. 第一部分--cache工作方式仿真

做完这个题目，可以让人理解cache的组织，工作方式，替换策略等。**先说一下题目，**本题主要可以分为以下几个部分：

1. trace file

题目会给出一些trace files， trace file的格式如下:

```
I 0400d7d4,8
 M 0421c7f0,4
 L 04f6b868,8
 S 7ff0005c8,8
```

语法为：

```
[space]operation address,size
```

The operation ﬁeld denotes the type of memory access: “I” denotes an instruction load, “L” a data load,  “S” a data store, and “M” a data modify (i.e., a data load followed by a data store). There is never a space before each “I”. There is always a space before each “M”, “L”, and “S”. The address ﬁeld speciﬁes a 64-bit hexadecimal memory address. The size ﬁeld speciﬁes the number of bytes accessed by the operation.

值得注意的是，我们只关注 data ，也就是 M,L,S开头的指令，不关注I开头的指令。解析时，可根据开头是否有空格来解析。

trace file就是我们要编写的代码的输入源，我们要做的就是根据这些trace file来仿真。

2. cache的组织方式

题目要求，最终的程序可以接受一些参数，改变cache的组织方式。 如：

```
./csim-ref -s 4 -E 1 -b 4 -t traces/yi.trace
```

代表：

- set的bit位数为: 4
- E=1，即一个set中只有一个cacheline
- b=4，即一个cache line的block位数为4
- trace file的路径为 traces/yi.trace

具体可使用的语法为：

```
Usage: ./csim-ref [-hv] -s <s> -E <E> -b <b> -t <tracefile>
• -h: Optional help ﬂag that prints usage info
• -v: Optional verbose ﬂag that displays trace info
• -s <s>: Number of set index bits (S = 2^s is the number of sets)
• -E <E>: Associativity (number of lines per set)
• -b <b>: Number of block bits (B = 2^b is the block size)
• -t <tracefile>: Name of the valgrind trace to replay
```

3. cacheline 的替换策略，采用LRU

ok，知道这些了，题目还给了我们一个标准答案，csim-ref, 我们自己最终的程序名为csim. 最终只要csim的输入输出和csim-ref相同即可。

**写程序之前，一定要分析好再写，所以先来分析下我们该怎么做：**

需要哪些背景知识：

cache的组织方式：

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200726102158585.png" style="zoom: 50%;" />

这张图，给出了cache的组织方式以及cacheline的read方式。

cache的组织方式非常直观，主要参数为 S, E, B. 这3个参数也决定了cache的相联关系：

1. E=1, 直接映射
2. E!=1,S!=1, E路S组相联
3. S=1, 全相联映射。（内存和Disk就是这种映射）

cacheline的read分为以下几步：

1. 根绝set bit，决定set索引
2. 比较该组的所有line，是否有匹配的tag。如有，并且valid有效，则hit，并根据b 决定数据偏移。否则，第3步。
3. load该cacheline到cache中，如果该组还有空块（valid=0),则加载到空块中，如果没有空块，则根据替换策略进行替换。

**总结下read(load)操作:**

1. hit, 能在当前cache中找到cacheline
2. miss，不能在cahce中找到cacheline，但是该组中有空cacheline。
3. eviction，不能再cache中找到cacheline，且该组中没有空cacheline，需要根据替换策略进行替换。

**现在在看下write(store)操作：**

cachelab采用的的是write-back+write-allocate方式。所以，一旦我们的store miss了，需要将该块load到cache中。分析可以下3中情况：

1. hit, 能在当前cache中找到cacheline
2. miss，不能在cahce中找到cacheline，但是该组中有空cacheline。
3. eviction，不能在cache中找到cacheline，且该组中没有空cacheline，需要根据替换策略进行替换。

**最后看下modify操作：**

modify结合了load和store操作，分析可得以下3种情况：

1. hit(load) + hit(store), load命中，store也命中
2. miss(load)+hit(store), load miss，但是当前cache set中有空块，直接load到cache即可，后续store命中。
3. miss(load)+eviction(load)+hit(store),cache种没有该cacheline，且该组没有空块，最后store可命中

> 额外啰嗦一段：
>
> cacheline和block非常容易搞混，因为一会说将某个数据块 load到cache，一会又说将某个cacheline load到cache中。 
>
> 其实通常我们说将xx数据块load到cache中，这个数据块是包围在一个cacheline中的，cacheline除了这个数据块以外，还会包含一些元数据，如tag，valid，以及用于实现替换策略的辅助位等。
>
> **简单来说，cacheline包含block， 通常说的cacheline大小，都是说的cacheline内部的block的大小。如32字节，64字节等，都是说的cacheline内部的block的大小为32，64字节**

好了，分析到这里，基本上把整个流程走了一遍。剩下一些与主题无关代码，包括参数解析，trace file解析，可自行看以下代码（代码比较长，因为我加了详细注释，同时为了规范，加了一些“无用”代码，但看着应该没有压力）：

```c
/**
 * @author: raven
 * @date 2020-7-26
 * @description: 模拟仿真计算机cache的 load modify load过程
 * @issue:
 * 1.没有考虑任何复杂度问题
 * 2.函数没有做安全性检查,比如判定null等
 * 3.main函数中使用到的内存引用没有free
 */
#include "cachelab.h"

#define _GNU_SOURCE         // getline函数不属于c标准, 需要开启GNU扩展

#include <stdio.h>
#include <stdlib.h>
#include <getopt.h>
#include <ctype.h>
#include <string.h>

/*--------------------------前置定义-----------------------*/
#define FILE_NAME_LEN 100
#define OS_PTR_LEN 64
typedef unsigned long long address64_t;     // 64-bit address

#define DEBUG_HIT(verbose) do{if(verbose) printf("hit\n");}while(0)
#define DEBUG_MISS(verbose) do{if(verbose) printf("miss\n");}while(0)
#define DEBUG_EVICT(verbose) do{if(verbose) printf("evict\n");}while(0)

/*--------------------------数据结构定义-----------------------*/
/**
 * 参数
 */
typedef struct param {
    int v;                     // verbose
    int s;                     // set bits 数
    int E;                     // E-way
    int b;                     // block offset bits
    char t[FILE_NAME_LEN];     // trace file path
} param;

/**
 * cache line 结构
 */
typedef struct cache_line {
    int valid;                  // 有效位
    int tag;                    // tag
//    int block_data;             // 存储load到cache的数据，但是由于是模拟，实际这个filed是没用的
    int age;                    // age，用于实现LRU替换策略
} cache_line;

/**
 *  trace item结构： 存储trace file中的一行数据（不包括I开头的行)
 *  op_mode: 操作模式： L(load), M(modify), S(store)
 */
typedef enum op_mode {
    L = 0,                           // load
    M = 1,                           // modify
    S = 2                            // store
} op_mode;

typedef struct trace_item {
    op_mode mode;                 // 操作模式
    address64_t addr;                 // 地址
//    unsigned int access_size;       // 访问的内存单元数(byte为单位), unsigned int 可以 typedef，但是没想到好名字
} trace_item;

/**
 * set: cache中的set， 一个set可以由多个line组成
 * cache: 模拟的cache table，cache由多个set组成
 */
typedef struct set {
    cache_line *lines;
} set;

typedef struct cache {
    set *sets;
} cache;

/*--------------------------全局变量定义----------------------*/
char *usage = "Usage: ./csim-ref [-hv] -s <num> -E <num> -b <num> -t <file>\n"
              "Options:\n"
              "  -h         Print this help message.\n"
              "  -v         Optional verbose flag.\n"
              "  -s <num>   Number of set index bits.\n"
              "  -E <num>   Number of lines per set.\n"
              "  -b <num>   Number of block offset bits.\n"
              "  -t <file>  Trace file.\n";             // example 略...
// 结果集合
unsigned int hits, miss, evicts;

/*--------------------------操作function定义----------------------*/
/**
 * 解析输入参数
 * @param argc
 * @param argv
 * @param p  解析结果放置在p引用的内存中
 * @return 0 success parse
 *         -1 failed
 */
int parse_input_params(int argc, char *argv[], param *p)
{
    int opt;
    
    while ((opt = getopt(argc, argv, "hvs:E:b:t:")) != -1) {
        switch (opt) {
            case 'v':
                p->v = 1;
                break;
            case 's':
                p->s = atoi(optarg);
                break;
            case 'E':
                p->E = atoi(optarg);
                break;
            case 'b':
                p->b = atoi(optarg);
                break;
            case 't':
                sprintf(p->t, "%s", optarg);
                break;
            case 'h':
            case '?':
            default:
                printf("%s\n", usage);
                return -1;
        }
    }
    return 0;
}


/**
 * 根据字符c_mode,返回对应的op_mode枚举类型
 * @param c_mode
 * @return
 */
op_mode parse_op_mode(char c_mode)
{
    switch (c_mode) {
        case 'L':
            return L;
        case 'M':
            return M;
        case 'S':
            return S;
        default:
            perror("can't get op mode: invalid char!");
            exit(-1);
    }
}

/**
 * 解析trace file
 * @param trace_item_ptr 解析结构放置的地方
 * @param trace_item_num trace数组长度
 * @param path 解析文件路径
 * @return  有效的trace item数量
 *          -1 解析失败
 */
size_t parse_trace_file(trace_item **trace_item_ptr, size_t *trace_item_num, const char *path)
{
    FILE *file = NULL;
    char *line = NULL;
    trace_item *trace_items = *trace_item_ptr;
    size_t len = 0;
    if (!(file = fopen(path, "r"))) {
        perror("trace file not exit");
        return -1;
    }
    
    // read entry line by line
    int i = 0;
    while (getline(&line, &len, file) != -1) {
        if (line[0] == 'I')  // I 指令不需要解析
            continue;
        
        if (*trace_item_num <= i) {
            // line 过多,超过trace_item_num, 需要重新分配
            int new_num = *trace_item_num;
            do {                    // 保证new_num 一定要比i大
                new_num *= 2;
            } while (new_num <= i);
            
            trace_items = realloc(trace_items, new_num * sizeof(trace_item));
            if (!trace_items) {
                printf("realloc memory for trace item failed\n");
                exit(-1);
            }
            *trace_item_ptr = trace_items;
            // initialize  new alloc memory
            memset(trace_items + (*trace_item_num), 0, (new_num - *trace_item_num) * sizeof(trace_item));
            // update item_num (包含了无效的项)
            *trace_item_num = (*trace_item_num) << 1;
        }
        // mode
        trace_items[i].mode = parse_op_mode(line[1]);
        
        // address
        char addr[OS_PTR_LEN + 1];
        int j = 3;      // 代表line中"address"在line中的索引位置
        int offset = 0;
        // 跳过所有
        while (!isdigit(line[j]) && isspace(line[j]))
            ++j;
        offset = j;
        // 设置address
        while (line[j] != ',') {
            addr[j - offset] = line[j];
            j++;
        }
        addr[j - offset] = '\0';
        trace_items[i].addr = strtol(addr, NULL, 16);
        
        // byte_nums note: 实际仿真中,这个字段是没用的
//        char buf[10];
//        j++;     // 更新至第一个byte num索引索引位置
//        offset = j;
//        while (line[j] != '\n') {
//            buf[j - offset] = line[j];
//            j++;
//        }
//        buf[j] = '\0';
//        trace_items[i].access_size = atoi(buf);
        
        // update idx
        i++;
    }
    
    // 删除无用项
    *trace_item_num = i;
    *trace_item_ptr = (trace_item *) realloc(*trace_item_ptr, sizeof(trace_item) * (*trace_item_num));
    
    fclose(file);
    return 0;
}

/**
 * 初始化系统cache
 * @param sys_cache
 */
void init_cache(cache *sys_cache, param *p)
{
    const int S = 1 << p->s;
    const int E = p->E;
//    const int B = 1 << p->b;
    
    set *sets = (set *) malloc(sizeof(set) * S);
    for (size_t i = 0; i < S; i++) {
        cache_line *cls = (cache_line *) malloc(sizeof(cache_line) * E);
        // initialize
        memset(cls, 0, sizeof(cache_line) * E);
        sets[i].lines = cls;
    }
    sys_cache->sets = sets;
}

/**
 * check是否命中
 * @param cache_set 需要检测的set
 * @param p    系统参数
 * @param tag  地址中的tag
 * @param cl_idx   如果命中,cl_idx表示当前命中的cacheline在set的中位置(从0开始索引)
 *                 如果未命中,cl_idx = -1
 */
void check_if_hit(set *cache_set, param *p, int tag, size_t *cl_idx)
{
    const int E = p->E;
    for (size_t i = 0; i < E; i++) {
        if (!cache_set->lines[i].valid) continue;
        if (cache_set->lines[i].tag == tag) {
            *cl_idx = i;
            return;
        }
    }
    *cl_idx = -1;
}
//TODO: check_if_hit和check_has_non_block可以做成一个函数,不过这里不考虑效率问题,所以就这样分了
/**
 * 检验 这组set中,是否还有空block
 * @param cache_set  待检验的set
 * @param p  系统参数
 * @param cl_idx  如果有空块,则cl_idx=找到的第一个空块
 *                如果没有空块,cl_idx=-1
 */
void check_has_empty_block(set *cache_set, param *p, size_t *cl_idx)
{
    const int E = p->E;
    for (size_t i = 0; i < E; i++) {
        if (!cache_set->lines[i].valid) {
            *cl_idx = i;
            return;
        }
    }
    *cl_idx = -1;
}

/**
 * 采用LRU算法找到需要evict的cache line索引
 * Note: 本函数假设当前set中没有空块
 * @param cache_set 需要检查的set
 * @param p     系统参数
 * @param cl_idx cl_idx = 找到的cache line在set中的索引
 */
void find_evict_cache_line(set *cache_set, param *p, size_t *cl_idx)
{
    const int E = p->E;
    int min_age_idx = 0;
    for (size_t i = 1; i < E; i++) {
        if (cache_set->lines[min_age_idx].age > cache_set->lines[i].age) {
            min_age_idx = i;
        }
    }
    *cl_idx = min_age_idx;
}

/**
 * load 操作
 * @param cache_set
 * @param tag
 * @param age
 */
void do_load(set *cache_set, param *p, int tag, int age)
{
    const int verbose = p->v;
    // 首先看能否hit
    size_t cl_idx;
    check_if_hit(cache_set, p, tag, &cl_idx);
    if (cl_idx != -1) // 命中
    {
        hits++;
        cache_line *cl = &cache_set->lines[cl_idx];
        cl->tag = tag;
        cl->age = age;
        DEBUG_HIT(verbose);
        return;
    }
    
    // 未命中,查看是否有空块
    check_has_empty_block(cache_set, p, &cl_idx);
    if (cl_idx != -1) // 有空块
    {
        miss++;
        cache_line *cl = &cache_set->lines[cl_idx];
        cl->valid = 1;
        cl->tag = tag;
        cl->age = age;
        DEBUG_MISS(verbose);
        return;
    }
    
    // 未命中,没有空块
    find_evict_cache_line(cache_set, p, &cl_idx);
    if (cl_idx != -1) // ecvit
    {
        miss++;
        DEBUG_MISS(verbose);
        cache_line *cl = &cache_set->lines[cl_idx];
        cl->tag = tag;
        cl->age = age;
        evicts++;
        DEBUG_EVICT(verbose);
        return;
    }
}

void do_store(set *cache_set, param *p, int tag, int age)
{
    const int verbose = p->v;
    size_t cl_idx;
    // 是否hit
    check_if_hit(cache_set, p, tag, &cl_idx);
    if (cl_idx != -1) // 命中
    {
        hits++;
        cache_set->lines[cl_idx].age = age;
        DEBUG_HIT(verbose);
        return;
    }
    
    // 是否有空块
    check_has_empty_block(cache_set, p, &cl_idx);
    if (cl_idx != -1) // 空块
    {
        miss++;
        cache_set->lines[cl_idx].valid = 1;
        cache_set->lines[cl_idx].tag = tag;
        cache_set->lines[cl_idx].age = age;
        DEBUG_MISS(verbose);
        return;
    }
    
    // evcit
    find_evict_cache_line(cache_set, p, &cl_idx);
    if (cl_idx != -1) // evcit
    {
        miss++;
        DEBUG_MISS(verbose);
        cache_set->lines[cl_idx].tag = tag;
        cache_set->lines[cl_idx].age = age;
        evicts++;
        DEBUG_HIT(verbose);
        return;
    }
    
}

void do_modify(set *cache_set, param *p, int tag, int age)
{
    const int verbose = p->v;
    size_t cl_idx = -1;
    // hit?
    check_if_hit(cache_set, p, tag, &cl_idx);
    if (cl_idx != -1) {
        // hit , 之后的store也会hit
        hits += 2;
        cache_set->lines[cl_idx].age = age;
        DEBUG_HIT(verbose);
        DEBUG_HIT(verbose);
        return;
    }
    
    // 空块
    check_has_empty_block(cache_set, p, &cl_idx);
    if (cl_idx != -1) {
        miss++;
        DEBUG_MISS(verbose);
        // 首先load
        cache_set->lines[cl_idx].valid = 1;
        cache_set->lines[cl_idx].tag = tag;
        cache_set->lines[cl_idx].age = age;
        // 再store
        hits++;
        DEBUG_HIT(verbose);
        return;
    }
    
    // evict
    find_evict_cache_line(cache_set, p, &cl_idx);
    if (cl_idx != -1) {
        miss++;
        DEBUG_MISS(verbose);
        // evict
        DEBUG_EVICT(verbose);
        evicts++;
        cache_set->lines[cl_idx].tag = tag;
        cache_set->lines[cl_idx].age = age;
        // store
        hits++;
        DEBUG_HIT(verbose);
        return;
    }
}

/**
 * 模拟仿真
 * @param sys_cache
 * @param trace_item
 * @param item_num
 */
void simulate(cache *sys_cache, param *p, trace_item *trace_items, const unsigned int item_num)
{
    const int s = p->s;
    const int b = p->b;
//    const int E = p->E;
    const int tag_bit = OS_PTR_LEN - s - b;
    
    // 构造set tag mask
    // set mask
    address64_t set_mask = 1;
    for (size_t i = 0; i < s - 1; i++) {
        set_mask = set_mask << 1 | 1;
    }
    // tag mask
    address64_t tag_mask = 1;
    for (size_t i = 0; i < tag_bit - 1; i++) {
        tag_mask = tag_mask << 1 | 1;
    }
    
    for (size_t i = 0; i < item_num; i++) {
        trace_item item = trace_items[i];
        address64_t addr = item.addr;
        op_mode mode = item.mode;
//        int access_size = item.access_size;     // useless
        
        // step1: 得到set 和 tag
        int set_idx = addr >> b & set_mask;
        int tag = addr >> (b + s);
        
        // step2: 根据操作码来决定具体实施什么样的操作
        switch (mode) {
            case L:
                do_load(&sys_cache->sets[set_idx], p, tag, i);
                break;
            case M:
                do_modify(&sys_cache->sets[set_idx], p, tag, i);
                break;
            case S:
                do_store(&sys_cache->sets[set_idx], p, tag, i);
                break;
        }
    }
}

int main(int argc, char *argv[])
{
    // 系统输入参数
    param sys_param;
    // 解析trace file
    size_t trace_item_len = 10;
    trace_item *trace_items = (trace_item *) malloc(trace_item_len * sizeof(trace_item));
    // 系统cache
    cache sys_cache;
    int ret;
    
    // 解析参数
    ret = parse_input_params(argc, argv, &sys_param);
    if (ret) {
        printf("parse input params failed\n");
        printf("%s\n", usage);
        return -1;
    }
    
    // 解析trace file
    ret = parse_trace_file(&trace_items, &trace_item_len, sys_param.t);
    if (ret) {
        printf("parse trace file failed");
        return -1;
    }
    
    // 初始化系统cache
    init_cache(&sys_cache, &sys_param);
    
    // 开始仿真
    simulate(&sys_cache, &sys_param, trace_items, trace_item_len);
    
    // 打印输出
    printSummary(hits, miss, evicts);
    return 0;
}
```

## 2. 第二部分--编写cache友好代码

本题能加强学生对cache的认识，编写cahce友好的代码。原题很简单，就是给一个矩阵A，求其转置。但是有一些额外说明，具体请读writeup。下面是本题的答案要求：

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200729141941145.png)

m，代表miss次数。

本题核心点：

1. 加强理解直接相联映射
2. **理解blocking（分块）机制带来的时间局部性提升**

重点：**blocking机制**，一定要理解blocking，请参照： http://csapp.cs.cmu.edu/public/waside/waside-blocking.pdf

在正式解题前，先说下系统的一些参数，最重要的就是cache的组织方式了：

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200729142729751.png)

s=5, S=32, 即32组

E=1， 即每组一个cacheline

b=5，B=32， 即一个cacheline的block大小为32字节（后面简称cacheline大小为32字节，实际说的是cacheline内部的block大小）



blocking机制运用到矩阵转置来，即将A分成多个行条带，A行条带扫描。B分为多个块，每个块由多个条带组成，每个B条带按照列扫描。如图：

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200729144043920.png)

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200729144439379.png)

### 1. 32x32

一个cacheline 32字节， 一个int 4字节， 则一个cacheline可以放8个int， 矩阵为32x32， 则矩阵一行需要4个cacheline。

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200729143603293.png)

现在的问题是如何确定条带的长度。 

我们知道cpu一次read，都会load一个cache line到cache中。一个cacheline是32字节。如何将条带设置低于32字节，比如12字节，那么32字节中会有20字节无法利用，浪费了一半的cache。如果大于32字节，那扫描一个条带就会触发至少2次load。那是不是设置成32字节就好了？也不是，我们还要考虑A，B的load，store交替过程中，会造成cacheline的overlap（冲突不命中）。为了让你理解这个概念，先看以下面这道题：

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200729145230952.png" alt="image-20200729145230952" style="zoom:50%;" />

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200729145241517.png" alt="image-20200729145241517" style="zoom:50%;" />

现在回到我们的题目，我们想要cacheline的利用率高，又不想发生太多cacheline的冲突不命中。观察到在32x32的矩阵中，**每8行重复一个cache空间。**如果一个条带为32字节，即8个矩阵元素，刚好对应了一个cache line大小。所以我们可以设置， A的行扫描条带为8个元素， B的列扫描条带为8个元素，那一个块的宽度呢？显然一个cacheline是32字节，刚好也是8个元素，宽度为8肯定利用率高。

最终决定的参数：

1. A行条带 8元素
2. B中一个块的大小为8x8

**另外，miss数计算会在之后给出。**

```c
void trans32_v1(int A[32][32], int B[32][32])
{
#define LEN 32
#define BSIZE 8
    // NOTE: 这里采用4循环是为了更好理解“blocking”机制，其实采用3个循环就能做，具体是融合j和k
    int i, j, k,q;

    // 所有循环视角从B矩阵出发
    for (i = 0; i < LEN; i += BSIZE) // 1. 以block行为单位，扫描整个矩阵
    {
        for (j = 0; j < LEN; j += BSIZE) // 2. 以一个block为单位，扫描一个block行
        {
            for (k = j; k < j + BSIZE; k++) // 3. 以一个条带为单位，扫描一个块
            {
                for (q = 0; q < BSIZE; q++) // 4. 以最小粒度为单位， 扫描一个条带
                {
                    B[i + q][k] = A[k][i + q];
                }
            }
        }
    }
#undef LEN
#undef BSIZE
}
```

这样写，miss数是300多，而满分要求miss数<300. 继续分析，什么导致了多余的miss？

cachelab的writeup中，有这样一句话：

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200729150322560.png)

对角线？ 是的，对角线上的元素，会发生冲突不命中问题。究其根本，在于我们对A进行了反复读。如何解决？把A的数据放到寄存器就行了。再次回到write u中：

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/5f212d3314195aa594512d18.png)



题目要求我们最多不能定义12个局部变量，上述代码仅仅4个。我们还有8个变量没用，最终的代码：

```c
/**
 * @brief block size 8
 * miss计算:
 * BSIZE 8， 总共分为了16块， 假设块编号从0开始，由左向右，由上向下。
 * 则左边第一列块中的每一块（块编号为0,4,8,12)miss数为(9+7+7)：
 * 1.第1个9代表 load A的第一个条带(1个miss)+ store B的第一个条带（8个miss）
 * 2.第2个7代表 load A的剩余7列带来的miss
 * 3.第3个7代表，store B的剩余7列带来的miss
 * 具体可参见文件： m32n32-23miss
 * 所以这4个块总miss为：4*(9+7+7) = 92
 * 
 * 上述说完了最靠左边的一列block，还剩3列block，3*4 = 12个block
 * 这些block的共性就是 load A条带不会和Store A条带冲突。
 * 所以扫描完一个block会造成的miss为8+8：
 * 1. 第一个8代表，store B的第一个条带（8个miss）
 * 2. 第二个8代表， load 8个A条带带来的miss
 * 所以这12个块总miss为： 12*(8+8) = 192
 * 具体可参见文件：m32n32-16miss
 * 
 * 另外还有3个固定的额外开销，具体对应哪个cache miss未知，猜测为初始化循环变量带来的（不过又觉得应该不是，如果你知道，还请告诉我）
 * 具体可参见文件：m32n32-extra-miss
 * 
 * 所以最终的miss为:
 * 92+192+3 = 287
 * @param A 
 * @param B 
 */
char trans32_v1_desc[] = "trans32_v1_desc";
void trans32_v1(int A[32][32], int B[32][32])
{
#define LEN 32
#define BSIZE 8
    int i, j, k;
    int t0, t1, t2, t3, t4, t5, t6, t7;

    // 所有循环视角从B矩阵出发
    for (i = 0; i < LEN; i += BSIZE) // 1. 以block行为单位，扫描整个矩阵
    {
        for (j = 0; j < LEN; j += BSIZE) // 2. 以一个block为单位，扫描一个block行
        {
            for (k = j; k < j + BSIZE; k++) // 3. 以一个条带为单位，扫描一个块
            {

                // 避免对角线的 A B矩阵cacheline竞争
                t0 = A[k][i + 0];
                t1 = A[k][i + 1];
                t2 = A[k][i + 2];
                t3 = A[k][i + 3];
                t4 = A[k][i + 4];
                t5 = A[k][i + 5];
                t6 = A[k][i + 6];
                t7 = A[k][i + 7];

                B[i + 0][k] = t0;
                B[i + 1][k] = t1;
                B[i + 2][k] = t2;
                B[i + 3][k] = t3;
                B[i + 4][k] = t4;
                B[i + 5][k] = t5;
                B[i + 6][k] = t6;
                B[i + 7][k] = t7;

                // 第二种写法,但是存在对角线竞争
                // for (q = 0; q < BSIZE; q++) // 4. 以最小粒度为单位， 扫描一个条带
                // {
                //     B[i + q][k] = A[k][i + q];
                // }
            }
        }
    }
#undef LEN
#undef BSIZE
}
```

**最终的miss数为287**，在代码的注释部分，给出了miss的详细计算方式。

### 2. 61x67

64x64的最后来说，先把简单的说了来。

61和67看着不规则比较难，但是题目要求很松，只要低于2000miss即可。

依然是上面的算法，更改BSIZE, 经测试BSIZE=16就能满足要求。

```c
void trans6761_v1(int A[67][61], int B[61][67])
{
    // 从A矩阵视角
#define M 67
#define N 61
#define BSIZE 16
    int i, j, k, q;

    // 所有循环视角从B矩阵出发
    for (i = 0; i < N; i += BSIZE) // 1. 以block行为单位，扫描整个矩阵
    {
        for (j = 0; j < M; j += BSIZE) // 2. 以一个block为单位，扫描一个block行
        {
            for (k = j; k < j + BSIZE; k++) // 3. 以一个条带为单位，扫描一个块
            {
                for (q = 0; q < BSIZE; q++) // 4. 以最小粒度为单位， 扫描一个条带
                {
                    if (k < M && i + q < N)
                    {
                        B[i + q][k] = A[k][i + q];
                    }
                    else
                    {
                        break;
                    }
                }
            }
        }
    }
#undef N
#undef M
#undef BSIZE
}
```

**最终的miss为1817**

### 3. 64x64

64x64则难很多了，如果依然照搬上面的算法，最终的miss应该是1800多，离满分1300还差很远。来分析下原因：

1. 如果BSIZE 依然取8会发生什么问题？

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/5f212d5b14195aa594514cfb.png)

上图给出了此时的矩阵内部cacheline分布，有个很严重的问题在于一个cache空间，在矩阵中仅4行就重复了。那如果条带继续为8， B列向扫描一个条带，后4个元素就会替换前4个元素所在的cacheline。造成过多**冲突不命中**。

2. 那，如果将BSIZE设置为4呢？

实际上，采用同样的算法，BSIZE=4,的确是最优的了。但是设置为4又会带来什么缺点？**cacheline的利用率仅有一半**。 一个cacheline 32字节，可装载8个元素，但实际只是用了4个元素。

3. 那，如何保住利用率的同时，减少冲突不命中？

要保住利用率，BSIZE应该是8的整数倍。但是冲突不命中如何解决？

以BSIZE=8为例：

A条带长度为8，我们将一个A条带的前4列正常填入B的前4行，而**A条带的后4列填入到其他地方，**再等待某个时机，将这些临时填充的数据归还到正确位置即可。示意图：

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/5f212d6b14195aa59451559a.png)

绿色代表正常填入区间。

灰色代表本应该填入，但是不填入，转而填入到红色区域。

红色区域代表临时填充区间。

接着，在某个时机：

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/5f212d7d14195aa594515d89.png)

几个问题：

1. 为什么红色区域要横着填？

   因为要考虑cache友好性，如果竖着填，那将跨越多个cacheline。

2. 为什么要选择第0行，第4 5 6 7列 作为 第0列，第4 5 6 7行的临时区间？

   这只是个示意图，实际上我选择了3种方案，最终方案不是这样映射的，贴这个图是为了方便理解。

3. 某个时机？具体是什么时机？

   将要填写第0行，第4,5,6,7列数据时，首先先移动数据到正确的位置（第0列，第4,5,6,7行），然后才填写第0行，第4，5,6,7列。

4. 最靠右的块如何处理？它已经没有 右边的空间 作为缓冲了。

   单独处理。具体之后会说。

#### 最终方案

最终我选择方案为： BSIZE=8，临时填充区间示意图如下：

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/5f212d8c14195aa5945163a2.png)

为什么会这样选？如果像之前示意图那样选，依然会存在很多冲突映射，甚至不如不做映射。仔细分析了下trace file,手动模拟了cache line的load store过程，选择的这样的临时填充映射。

**最后再说说扫描到最后一个8x8的方格时如何处理**，由于最后的方格右边已经没有空间做缓冲，那先考虑一个简单的，直接对A和B矩阵的最后8x8方格做一一映射。显然后4x8会把和前4x8会产生冲突不命中。

这样做的miss数应该是1500多。虽然比最开始的算法1800多，还是好不少，但依然没法满分。没办法，继续考虑。

**既然是后4x8和前4x8冲突了，那把两次赋值过程分开不就好了吗？**

是的，基于这样的思想，我改了代码，嗯，不错，**这次跑的结果为1190**。总算挤近1300了。

别忘了，依然要解决重复加载A条带带来的冲突不命中问题（即，多定义局部变量）。

最终代码：

```c
void trans64_v1(int A[64][64], int B[64][64])
{
#define LEN 64
    int i, j, k;
    int t0, t1, t2, t3, t4, t5, t6, t7;

    // 所有循环视角从B矩阵出发
    for (i = 0; i < LEN; i += 8) // 1. 以block行为单位，扫描整个矩阵
    {
        for (j = 0; j < LEN; j += 8) // 2. 以一个block为单位，扫描一个block行
        {
            for (k = j; k < j + 8; k++) // 3. 以一个条带为单位，扫描一个块
            {

                t0 = A[k][i + 0];
                t1 = A[k][i + 1];
                t2 = A[k][i + 2];
                t3 = A[k][i + 3];
                t4 = A[k][i + 4];
                t5 = A[k][i + 5];
                t6 = A[k][i + 6];
                t7 = A[k][i + 7];

                if (j == 0)
                {
                    // 第一块
                    // 前4个正常放置
                    B[i + 0][k] = t0;
                    B[i + 1][k] = t1;
                    B[i + 2][k] = t2;
                    B[i + 3][k] = t3;

                    // 后4个需要单独处理
                    B[i + k % 4][k - k % 4 + 8] = t4;
                    B[i + k % 4][k - k % 4 + 9] = t5;
                    B[i + k % 4][k - k % 4 + 10] = t6;
                    B[i + k % 4][k - k % 4 + 11] = t7;
                }
                else
                {
                    // 非第一块，首先需要将元素搬迁到正确的位置
                    if (k == j) // 第一次进入该块时，搬迁
                    {
                        for (size_t a = i; a < i + 4; a++)
                        {
                            for (size_t b = j; b < j + 4; b++) // 一个条带
                            {
                                // 两点关于一条直线对称
                                B[b + i - j + 4][a - i + j - 8] = B[a][b];
                                B[b + i - j + 4][a - i + j - 4] = B[a][b + 4];
                            }
                        }
                    }

                    // 然后，填入本block正确的元素
                    if (j != LEN - 8)
                    { 
                        // 中间块处理方式和第一块处理方式相同
                        // 前4个正常放置
                        B[i + 0][k] = t0;
                        B[i + 1][k] = t1;
                        B[i + 2][k] = t2;
                        B[i + 3][k] = t3;

                        // 后4个需要单独处理
                        B[i + k % 4][k - k % 4 + 8] = t4;
                        B[i + k % 4][k - k % 4 + 9] = t5;
                        B[i + k % 4][k - k % 4 + 10] = t6;
                        B[i + k % 4][k - k % 4 + 11] = t7;
                    }
                    else
                    {
                        // 最后一块单独处理
                        // 只处理8x8的上 4x8方块
                        B[i + 0][k] = t0;
                        B[i + 1][k] = t1;
                        B[i + 2][k] = t2;
                        B[i + 3][k] = t3;
                    }
                }
            }
        }
        // 补齐最后一块的下4行条带， 即最后的8x8的下4x8方块
        for (size_t a = 56; a < 64; a++)
        {
            t4 = A[a][i + 4];
            t5 = A[a][i + 5];
            t6 = A[a][i + 6];
            t7 = A[a][i + 7];

            B[i + 4][a] = t4;
            B[i + 5][a] = t5;
            B[i + 6][a] = t6;
            B[i + 7][a] = t7;
        }
    }
#undef LEN
}
```

如果你眼尖，会发现一个问题，那就是局部变量用了13个。好吧，确实是，but，我们是可以解决的：

```c
for (size_t a = i; a < i + 4; a++)
{
    for (size_t b = j; b < j + 4; b++) // 一个条带
    {
        // 两点关于一条直线对称
        B[b + i - j + 4][a - i + j - 8] = B[a][b];
        B[b + i - j + 4][a - i + j - 4] = B[a][b + 4];
    }
}
```

把这个循环展开即可。

## 3. 总结

呼，加上今天的文章，cachelab一共花了3天的时间，不过是完全值得的，能够深入理解经常碰到的cacheline的本质，理清了很多概念上的东西。 cache友好的代码确实很难想也很难写，以我目前的能力，写代码时还考虑不到这么底层，考虑一些时间、空间局部性就快极限了，每一行代码都去想cache line实在耗费很多精力，不得不说，还有很多路要走啊。

最近也要忙项目了，下一个lab不知道什么时候再做。但一定要把csapp啃完！

