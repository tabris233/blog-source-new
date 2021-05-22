---
title: "【TODO】Redis 学习 基础数据结构篇 之 压缩列表(ziplist)"
date: 2021-05-22 20:05:48
updated: 2021-05-22 20:05:48
description: "Redis 基础数据结构 - 压缩列表(ziplist)。"
katex: true
summary:
categories:
  - 学习
  - redis
tags:
  - redis
  - ziplist
  - 压缩列表
---

[TOC]

>   基于 [Redis 6.2.1](https://github.com/redis/redis/tree/6.2.1)
>
>   图片截取自《Redis设计与实现》

# 压缩列表（ziplist）

压缩列表（ziplist）是列表键和哈希键的底层实现之一。

当一个列表键只包含**少量**列表项，并且每个列表项要么就是**小整数值**，要么就是长度比较短的字符串，那么`Redis` 就会使用压缩列表来做列表键的底层实现

当一个哈希键只包含**少量**键值对，并且每个键值对的键和值要么就是**小整数值**，要么就是**长度比较短**的字符串，那么`Redis` 就会使用压缩列表来做哈希键的底层实现

## 压缩列表的构成

压缩列表是  `Redis` 为了节约内存而开发的，是由一系列特殊编码的连续内存快组成的顺序行（sequential）数据结构，是一个特殊编码的双向链表。一个压缩列表可以包含任意过个节点（entry），每个节点可以保存一个字节数组或者一个整数值。

`ziplist.c` 文件顶部的注释内容，解释了`ziplist`的结构。

`ziplist`本身声明为一个`unsigned char *`类型的字符串，总体布局如下：

`<zlbytes> <zltail> <zllen> <entry> <entry> ... <entry> <zlend>`

-   `zlbytes`：`uint32_t`类型。`zlbytes`是一个无符号整型，用于保存ziplist占用的字节数，包括`zlbytes`字段本身的四个字节。需要存储该值，以便能够调整整个结构的大小，而无需先遍历它。
-   `zltail`：`uint32_t`类型。`zltail`是列表中最后一个条目的偏移量。这允许在列表的另一端进行弹出操作，而无需完全遍历。
-   `zllen`：`uint16_t`类型。`zllen`是条目数。当条目数超过$2 ^ {16}-2$时，此值将设置为$2 ^ {16}-1$，我们需要遍历整个列表以了解其包含多少项。
-   `entry`：`entry`是实际的存储条目。
-   `zlend`：`uint8_t`类型。`zlend`是代表`ziplist`末尾的特殊条目。编码为等于$255$的单个字节。没有其他普通条目以设置为$255$的字节开始。

---

创建一个新的空的`ziplist`源码如下：

```c
/* The size of a ziplist header: two 32 bit integers for the total
 * bytes count and last item offset. One 16 bit integer for the number
 * of items field. */
#define ZIPLIST_HEADER_SIZE     (sizeof(uint32_t)*2+sizeof(uint16_t))

/* Size of the "end of ziplist" entry. Just one byte. */
#define ZIPLIST_END_SIZE        (sizeof(uint8_t))

...

/* Create a new empty ziplist. */
unsigned char *ziplistNew(void) {
    unsigned int bytes = ZIPLIST_HEADER_SIZE+ZIPLIST_END_SIZE;
    unsigned char *zl = zmalloc(bytes);
    ZIPLIST_BYTES(zl) = intrev32ifbe(bytes);
    ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(ZIPLIST_HEADER_SIZE);
    ZIPLIST_LENGTH(zl) = 0;
    zl[bytes-1] = ZIP_END;
    return zl;
}
```

从代码块第13行可以看出，`ziplist`除`entry`的内容被定义为头部（`ZIPLIST_HEADER_SIZE`对应`<zlbytes> <zltail> <zllen>`）和尾部（`ZIPLIST_END_SIZE`对应`zlend`），申请这么大的内存就好了，。

---

`ziplist` 节点的结构：

```c
/* We use this function to receive information about a ziplist entry.
 * Note that this is not how the data is actually encoded, is just what we
 * get filled by a function in order to operate more easily. */
typedef struct zlentry {
    unsigned int prevrawlensize; /* Bytes used to encode the previous entry len*/
    unsigned int prevrawlen;     /* Previous entry len. */
    unsigned int lensize;        /* Bytes used to encode this entry type/len.
                                    For example strings have a 1, 2 or 5 bytes
                                    header. Integers always use a single byte.*/
    unsigned int len;            /* Bytes used to represent the actual entry.
                                    For strings this is just the string length
                                    while for integers it is 1, 2, 3, 4, 8 or
                                    0 (for 4 bit immediate) depending on the
                                    number range. */
    unsigned int headersize;     /* prevrawlensize + lensize. */
    unsigned char encoding;      /* Set to ZIP_STR_* or ZIP_INT_* depending on
                                    the entry encoding. However for 4 bits
                                    immediate integers this can assume a range
                                    of values and must be range-checked. */
    unsigned char *p;            /* Pointer to the very start of the entry, that
                                    is, this points to prev-entry-len field. */
} zlentry;
```

-   `prevrawlensize`： 编码 prevrawlen 所需的字节大小
-   `prevrawlen`：前置节点的长度
-   `lensize`：编码 len 所需的字节大小
-   `len`：当前节点值的长度
-   `headersize`：当前节点 header 的大小等于 prevrawlensize + lensize
-   `encoding`：当前节点值所使用的编码类型
-   `p`：指向当前节点的指针



















