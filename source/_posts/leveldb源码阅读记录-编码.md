---
title: leveldb源码阅读记录-编码
categories: leveldb
abbrlink: 8f115521
date: 2020-10-12 09:46:54
tags:
---

## 0. 前言

在阅读本篇文章前，你应该能先了解什么是字节序，什么是小端序，什么是大端序。 

<!--more-->

## 1. 正文

leveldb中与编码相关的内容都在coding.h，coding.cc文件中。leveldb的编码都是采用小端序，编码一共分为固定长度编码和变长编码，且都分为32位和64位版本，我们先说固定长度编码。

## 2. 固定长度编码

```c++
// Lower-level versions of Put... that write directly into a character buffer
// REQUIRES: dst has enough space for the value being written

inline void EncodeFixed32(char* dst, uint32_t value) {
  uint8_t* const buffer = reinterpret_cast<uint8_t*>(dst);

  // Recent clang and gcc optimize this to a single mov / str instruction.
  buffer[0] = static_cast<uint8_t>(value);
  buffer[1] = static_cast<uint8_t>(value >> 8);
  buffer[2] = static_cast<uint8_t>(value >> 16);
  buffer[3] = static_cast<uint8_t>(value >> 24);
}

inline void EncodeFixed64(char* dst, uint64_t value) {
  uint8_t* const buffer = reinterpret_cast<uint8_t*>(dst);

  // Recent clang and gcc optimize this to a single mov / str instruction.
  buffer[0] = static_cast<uint8_t>(value);
  buffer[1] = static_cast<uint8_t>(value >> 8);
  buffer[2] = static_cast<uint8_t>(value >> 16);
  buffer[3] = static_cast<uint8_t>(value >> 24);
  buffer[4] = static_cast<uint8_t>(value >> 32);
  buffer[5] = static_cast<uint8_t>(value >> 40);
  buffer[6] = static_cast<uint8_t>(value >> 48);
  buffer[7] = static_cast<uint8_t>(value >> 56);
}
```

一句话总结，低位字节放在低位内存，高位字节放在高位内存，也即小端序。

解码过程：

```c++
// Lower-level versions of Get... that read directly from a character buffer
// without any bounds checking.

inline uint32_t DecodeFixed32(const char* ptr) {
  const uint8_t* const buffer = reinterpret_cast<const uint8_t*>(ptr);

  // Recent clang and gcc optimize this to a single mov / ldr instruction.
  return (static_cast<uint32_t>(buffer[0])) |
         (static_cast<uint32_t>(buffer[1]) << 8) |
         (static_cast<uint32_t>(buffer[2]) << 16) |
         (static_cast<uint32_t>(buffer[3]) << 24);
}

inline uint64_t DecodeFixed64(const char* ptr) {
  const uint8_t* const buffer = reinterpret_cast<const uint8_t*>(ptr);

  // Recent clang and gcc optimize this to a single mov / ldr instruction.
  return (static_cast<uint64_t>(buffer[0])) |
         (static_cast<uint64_t>(buffer[1]) << 8) |
         (static_cast<uint64_t>(buffer[2]) << 16) |
         (static_cast<uint64_t>(buffer[3]) << 24) |
         (static_cast<uint64_t>(buffer[4]) << 32) |
         (static_cast<uint64_t>(buffer[5]) << 40) |
         (static_cast<uint64_t>(buffer[6]) << 48) |
         (static_cast<uint64_t>(buffer[7]) << 56);
}
```

按字节取即可。

## 3. 变长编码

为什么需要变长编码？ 考虑一个场景，假设现在我们只用在存放一个int类型的1， 这么简单的一个数字，我们却需要4个字节（一般情况下），要是可以压缩就好了。变长编码就是用来解决这个问题的，使用变长编码后，只用一个字节就可以保存这个1. 

leveldb的变长编码也是采用小端序，同时在一个字节中，低7位表示的是实际数据，最高的1位用来表示，这个字节后序是否还有字节用来表示数字。举个例子：

假设要表示数字1000。 其二进制是111 1101000。 

首先取出低7位：1101000， 高位先补0凑成一个字节， 0110 1000。

取出高3位： 111， 高位补0凑成一个字节， 0000 0111.

采用小端序存储这个数字： 0110 1000 0000 0111。

因为这里用到了两个字节表示一个数字，所以在第一个字节的最高位需要设置成1，表示第二个字节也是用来表示“1000（一千）”这个数字的。最终编码结果为：

**1110 1000 0000 0111.**

解码过程：

首先，读取第一个字节， 1110 1000， 最高位为1，表示后续还有字节表示这个数字，还原最高位位1后（0110 1000），再取一个字节（0000 0111）， 因为0000 0111最高不为1，所以取字节结束。现在得到两个字节：

01110 1000 0000 0111

按照小端序还原：

**0000 0111 0110 1000（二进制） ==> 1000（十进制）**

知道了原理，来看代码吧。

变长编码函数签名：

```c++
// Lower-level versions of Put... that write directly into a character buffer
// and return a pointer just past the last byte written.
// REQUIRES: dst has enough space for the value being written
char* EncodeVarint32(char* dst, uint32_t value);
char* EncodeVarint64(char* dst, uint64_t value);
```

实现

```c++
char* EncodeVarint32(char* dst, uint32_t v) {
  // Operate on characters as unsigneds
  uint8_t* ptr = reinterpret_cast<uint8_t*>(dst);
  static const int B = 128;
  if (v < (1 << 7)) {
    *(ptr++) = v;
  } else if (v < (1 << 14)) {
    *(ptr++) = v | B;
    *(ptr++) = v >> 7;
  } else if (v < (1 << 21)) {
    *(ptr++) = v | B;
    *(ptr++) = (v >> 7) | B;
    *(ptr++) = v >> 14;
  } else if (v < (1 << 28)) {
    *(ptr++) = v | B;
    *(ptr++) = (v >> 7) | B;
    *(ptr++) = (v >> 14) | B;
    *(ptr++) = v >> 21;
  } else {
    *(ptr++) = v | B;
    *(ptr++) = (v >> 7) | B;
    *(ptr++) = (v >> 14) | B;
    *(ptr++) = (v >> 21) | B;
    *(ptr++) = v >> 28;
  }
  return reinterpret_cast<char*>(ptr);
}

```

按照前面讲的例子，这段代码不难理解，变量B用来设置一个字节的最高位，表示后续是否还有字节。

```c++
char* EncodeVarint64(char* dst, uint64_t v) {
  static const int B = 128;
  uint8_t* ptr = reinterpret_cast<uint8_t*>(dst);
  while (v >= B) {
    *(ptr++) = v | B;
    v >>= 7;
  }
  *(ptr++) = static_cast<uint8_t>(v);
  return reinterpret_cast<char*>(ptr);
}
```

64位是32位的循环版本，两个代码表示同一个意思。

解码：

```c++
// Standard Get... routines parse a value from the beginning of a Slice
// and advance the slice past the parsed value.
bool GetVarint32(Slice* input, uint32_t* value);
bool GetVarint64(Slice* input, uint64_t* value);

//----------最终会走到下面这个函数---------------

const char* GetVarint32PtrFallback(const char* p, const char* limit,
                                   uint32_t* value) {
  uint32_t result = 0;
  for (uint32_t shift = 0; shift <= 28 && p < limit; shift += 7) {
    uint32_t byte = *(reinterpret_cast<const uint8_t*>(p));
    p++;
    if (byte & 128) {	// 后续还有字节
      // More bytes are present
      result |= ((byte & 127) << shift);
    } else {		// 后续无字节，到了最后的字节
      result |= (byte << shift);
      *value = result;
      return reinterpret_cast<const char*>(p);
    }
  }
  return nullptr;
}
```

为了方便用户使用，leveldb还封住了一个Put函数：

```c++
void PutVarint32(std::string* dst, uint32_t v) {
  char buf[5];
  char* ptr = EncodeVarint32(buf, v);
  dst->append(buf, ptr - buf);
}
```

首先申请了一个5字节的buf，用来存储编码后的结果。为什么是5个字节？因为当32位的数值过大，比如 0xffffffff， 那么变长编码肯定需要5个字节才能表示。后面就很简单那了，执行编码，将编码结果保存在dst中。

## 4. PutLengthPrefixedSlice & GetLengthPrefixedSlice

Put函数：

```c++
void PutLengthPrefixedSlice(std::string* dst, const Slice& value) {
  PutVarint32(dst, value.size());
  dst->append(value.data(), value.size());
}
```

对于一个字符串，这里首先将它的长度放到dst中，然后再将数值放入。 

![](https://pic.downk.cc/item/5f8444821cd1bbb86b066dd9.png)

Get函数：

```c++
bool GetLengthPrefixedSlice(Slice* input, Slice* result) {
  uint32_t len;
  if (GetVarint32(input, &len) && input->size() >= len) {
    *result = Slice(input->data(), len);
    input->remove_prefix(len);
    return true;
  } else {
    return false;
  }
}
```

这段代码的意思相当于将编码后的string中的长度信息去除，提出出原始string。

## 5. 总结

本文分析了leveldb中与编码相关的知识，包含了固定长度编码与变长编码的原理与实现。这些内容虽然看似与leveldb无关，但是leveldb内部存储中却无时无刻不用到这些编码，所以我们现在都是在为后续看懂leveldb做铺垫。