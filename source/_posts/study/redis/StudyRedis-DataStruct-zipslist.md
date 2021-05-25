---
title: "Redis 学习 基础数据结构篇 之 压缩列表(ziplist)"
date: 2021-05-22 20:05:48
updated: 2021-05-25 01:13:23
description: "Redis 基础数据结构 - 压缩列表(ziplist)。"
katex: true
summary:
categories:
  - 学习
  - redis
  - 基础数据结构
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



压缩列表是  `Redis` 为了节约内存而开发的，是由一系列特殊编码的连续内存快组成的顺序行（sequential）数据结构，是一个特殊编码的双向链表。一个压缩列表可以包含任意过个节点（entry），每个节点可以保存一个字节数组或者一个整数值。

## 压缩列表结构

`ziplist.c` 文件顶部的注释内容，解释了`ziplist`的结构。

`ziplist`本身声明为一个`unsigned char *`类型的字符串，总体布局如下：

`<zlbytes> <zltail> <zllen> <entry> <entry> ... <entry> <zlend>`

-   `zlbytes`：`uint32_t`类型。`zlbytes`是一个无符号整型，用于保存ziplist占用的字节数，包括`zlbytes`字段本身的四个字节。需要存储该值，以便能够调整整个结构的大小，而无需先遍历它。
-   `zltail`：`uint32_t`类型。`zltail`是列表中最后一个条目的偏移量。这允许在列表的另一端进行弹出操作，而无需完全遍历。
-   `zllen`：`uint16_t`类型。`zllen`是条目数。当条目数超过$2 ^ {16}-2$时，此值将设置为$2 ^ {16}-1$，我们需要遍历整个列表以了解其包含多少项。
-   `entry`：`entry`是实际的存储条目。
-   `zlend`：`uint8_t`类型。`zlend`是代表`ziplist`末尾的特殊条目。编码为等于$255$的单个字节。没有其他普通条目以设置为$255$的字节开始。

---

### 创建新的压缩列表

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

## 压缩列表节点的结构

具体节点的结构，源码中注释已经说的很明白了。

>   **源码注释（翻译）**
>
>   *ZIPLIST条目*
>    *===============*
>
>    *ziplist中的每个条目都以包含两个信息的元数据作为前缀。首先，存储前一个条目的长度，以便能够从后到前遍历列表。第二，提供条目编码。它表示条目类型，整数或字符串，对于字符串，还表示字符串有效负载的长度。因此，完整的条目存储如下：*
>
>    `<prevlen> <encoding> <entry-data>`
>
>    *有时，编码代表条目本身，就像后面将要看到的小整数一样。在这种情况下，<entry-data>部分丢失了，我们可以这样：*
>
>    `<prevlen> <encoding>`
>
>    *上一个条目的长度<prevlen>以以下方式进行编码：如果此长度小于254个字节，则它将仅消耗一个字节来表示该长度，这是一个不带符号的8位整数。当长度大于或等于254时，它将占用5个字节。第一个字节设置为254（FE），以指示随后的值更大。剩余的4个字节将前一个条目的长度作为值。*
>
>    *因此，实际上，条目是通过以下方式编码的：*
>
>    `<prevlen from 0 to 253> <encoding> <entry>`
>
>    *或者，如果上一个条目长度大于253个字节，则使用以下编码：*
>
>   ` 0xFE <4字节无符号小端字节序 prevlen> <encoding> <entry>` 
>
>    *条目的编码字段(<encoding>)取决于条目的内容。当条目是字符串时，编码第一个字节的前2位将保存用于存储字符串长度的编码类型，然后是字符串的实际长度。当条目是整数时，前2位都设置为1。接下来的2位用于指定在此标头之后将存储哪种整数。不同类型和编码的概述如下。第一个字节始终足以确定条目的类型。*
>
>    `| 00pppppp |` - 1个字节
>         *长度小于或等于63个字节（6位）的字符串值。 “ pppppp”表示无符号的6位长度。*
>    `| 01pppppp | qqqqqqqq |` - 2个字节
>         *长度小于或等于16383字节（14位）的字符串值。*
>         *重要说明：14位数字采用大端字节序存储。*
>    `| 10______ | qqqqqqqq | rrrrrrrr | ssssssss | tttttttt |` - 5个字节
>         *长度大于或等于16384个字节的字符串值。第一个字节之后的仅4个字节代表最大2 ^ 32-1的长度。第一个字节的低6位未使用，设置为零。*
>         *重要说明：32位数字存储在big endian中。*
>    `| 11000000 |` - 3个字节
>         *整数编码为int16_t（2个字节）。*
>    `| 11010000 |` - 5个字节
>         *整数编码为int32_t（4个字节）。*
>    `| 11100000 |` - 9个字节
>         *整数编码为int64_t（8字节）。*
>    `| 11110000 |` - 4字节
>         *整数编码为24位带符号（3个字节）。*
>    `| 11111110 |` - 2个字节
>         *整数编码为8位带符号（1个字节）。*
>   `| 1111xxxx |` -（xxxx在0001和1101之间）立即4位整数。
>         *0到12之间的无符号整数。编码值实际上是1到13，因为不能使用0000和1111，因此应从编码的4位值中减去1以获得正确的值。*
>    `| 11111111 |` - ziplist*的结尾标识*。
>
>    *像ziplist标头一样，所有整数都以小端字节序表示，即使此代码在大端系统中编译也是如此。*
>
>   ---
>
>   *实际ZIP列表的示例*
>   *==========================*
>
>    *以下是一个ziplist，其中包含表示字符串“ 2”和“ 5”的两个元素。它由15个字节组成，我们在视觉上将其分为几部分：*
>
>   ```c
>     [0f 00 00 00] [0c 00 00 00] [02 00] [00 f3] [02 f6] [ff]
>           |             |          |       |       |     |
>        zlbytes        zltail    entries   "2"     "5"   end
>   ```
>
>    *前4个字节代表数字15，即整个ziplist组成的字节数。*
>
>   *接下来的4个字节是找到最后一个ziplist条目的偏移量，即12，实际上，最后一个条目“ 5”位于ziplist内的偏移量12处。*
>
>   *下一个16位整数表示ziplist中的元素数，由于其中只有两个元素，因此其值为2。*
>
>   *最后，“ 00 f3”是代表数字2的第一个条目。它由前一个条目长度（由于这是我们的第一个条目）而为零和与编码| 1111xxxx |相对应的字节F3组成。 xxxx在0001和1101之间。我们需要删除“ F”个高阶位1111，并从“ 3”中减去1，因此条目值为“ 2”。*
>
>   *下一个条目的前缀为02，因为第一个条目正好由两个字节组成。条目本身F6的编码方式与第一个条目完全相同，并且6-1 = 5，因此该条目的值为5。*
>
>   *最后，特殊条目FF向ziplist的结尾发出信号。*
>
>    *在上面的字符串中添加另一个值为“ Hello World”的元素，使我们可以显示ziplist如何编码小字符串。我们将仅显示条目本身的十六进制转储。想象一下上面的ziplist中存储“ 5”的条目后面的字节：*
>
>   ```c
>    [02] [0b] [48 65 6c 6c 6f 20 57 6f 72 6c 64]
>   ```
>
>    *第一个字节02是上一个条目的长度。下一个字节表示模式| 00pppppp |中的编码这表示该条目是长度为<pppppp>的字符串，因此0B表示其后为11个字节的字符串。从第三个字节（48）到最后一个字节（64），只有“ Hello World”的ASCII字符。*

---

访问`ziplist` 节点的辅助结构：

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

-   `prevrawlensize`： 编码 prevrawlen 所需的字节大小。字符串的时候可能是1、可能是2、也可能是5。
-   `prevrawlen`：前置节点的长度
-   `lensize`：用于编码此条目类型/len的字节。例如，字符串具有1、2或5个字节头。整数始终使用单个字节。
-   `len`：用于表示实际节点的字节。对于字符串，这只是字符串长度。而对于整数，则为1、2、3、4、8或0（立即4位），具体取决于数字范围。
-   `headersize`：当前节点 header 的大小等于 prevrawlensize + lensize
-   `encoding`：当前节点值所使用的编码类型
-   `p`：指向当前节点的指针

⚠️注意：再次申明下，这个结构体知识便于处理实际节点的**辅助**结构。实际存储的只有`<prevrawlen> <len> <data>`。

## 连锁更新

每个节点的`prevlen`都记录了上一个节点的长度；

-   如果上一个节点的长度小于254字节，那么`prevlen`需要1字节保存长度值。
-   如果上一个节点的长度大于等于254字节，那么`prevlen`需要5字节保存长度值。



这时如果考虑一个极端情况，一个压缩列表有连续多个长度介于250字节到253字节的节点`e1`到`eN`。

```
zlbytes zltail zllen e1 e2 e3 ... eN zlend
```

因为`e1`到`e[N-1]`的节点长度都是小雨254的的，所以`e2`到`eN`记录前一个节点的`prevlen`只需要1个字节。

这时，如果讲一个长度大于等于254字节的新节点 `new` 设置为压缩列表的头节点，即`e1`节点的前置节点。

```
zlbytes zltail zllen new e1 e2 e3 ... eN zlend
```

因为`e1`的`prevlen`只有1个字节，不能保存`new`的长度。所以需要重新分配空间为5个字节长。

同样的道理，`e2`的`pewvlen`也只有1个字节，也需要重新分配空间，

`e3`、...、`eN`都需要重新分配空间了。

Redis将这种现象称为“连锁更新”（cascade update）。

<details><summary>连锁更新 源码 展开/收起</summary>
```c
/* When an entry is inserted, we need to set the prevlen field of the next
 * entry to equal the length of the inserted entry. It can occur that this
 * length cannot be encoded in 1 byte and the next entry needs to be grow
 * a bit larger to hold the 5-byte encoded prevlen. This can be done for free,
 * because this only happens when an entry is already being inserted (which
 * causes a realloc and memmove). However, encoding the prevlen may require
 * that this entry is grown as well. This effect may cascade throughout
 * the ziplist when there are consecutive entries with a size close to
 * ZIP_BIG_PREVLEN, so we need to check that the prevlen can be encoded in
 * every consecutive entry.
 *
 * Note that this effect can also happen in reverse, where the bytes required
 * to encode the prevlen field can shrink. This effect is deliberately ignored,
 * because it can cause a "flapping" effect where a chain prevlen fields is
 * first grown and then shrunk again after consecutive inserts. Rather, the
 * field is allowed to stay larger than necessary, because a large prevlen
 * field implies the ziplist is holding large entries anyway.
 *
 * The pointer "p" points to the first entry that does NOT need to be
 * updated, i.e. consecutive fields MAY need an update. */
unsigned char *__ziplistCascadeUpdate(unsigned char *zl, unsigned char *p) {
    zlentry cur;
    size_t prevlen, prevlensize, prevoffset; /* Informat of the last changed entry. */
    size_t firstentrylen; /* Used to handle insert at head. */
    size_t rawlen, curlen = intrev32ifbe(ZIPLIST_BYTES(zl));
    size_t extra = 0, cnt = 0, offset;
    size_t delta = 4; /* Extra bytes needed to update a entry's prevlen (5-1). */
    unsigned char *tail = zl + intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl));

    /* Empty ziplist */
    if (p[0] == ZIP_END) return zl;
    
    zipEntry(p, &cur); /* no need for "safe" variant since the input pointer was validated by the function that returned it. */
    firstentrylen = prevlen = cur.headersize + cur.len;
    prevlensize = zipStorePrevEntryLength(NULL, prevlen);
    prevoffset = p - zl;
    p += prevlen;
    
    /* Iterate ziplist to find out how many extra bytes do we need to update it. */
    while (p[0] != ZIP_END) {
        assert(zipEntrySafe(zl, curlen, p, &cur, 0));
    
        /* Abort when "prevlen" has not changed. */
        if (cur.prevrawlen == prevlen) break;
    
        /* Abort when entry's "prevlensize" is big enough. */
        if (cur.prevrawlensize >= prevlensize) {
            if (cur.prevrawlensize == prevlensize) {
                zipStorePrevEntryLength(p, prevlen);
            } else {
                /* This would result in shrinking, which we want to avoid.
                 * So, set "prevlen" in the available bytes. */
                zipStorePrevEntryLengthLarge(p, prevlen);
            }
            break;
        }
    
        /* cur.prevrawlen means cur is the former head entry. */
        assert(cur.prevrawlen == 0 || cur.prevrawlen + delta == prevlen);
    
        /* Update prev entry's info and advance the cursor. */
        rawlen = cur.headersize + cur.len;
        prevlen = rawlen + delta;
        prevlensize = zipStorePrevEntryLength(NULL, prevlen);
        prevoffset = p - zl;
        p += rawlen;
        extra += delta;
        cnt++;
    }
    
    /* Extra bytes is zero all update has been done(or no need to update). */
    if (extra == 0) return zl;
    
    /* Update tail offset after loop. */
    if (tail == zl + prevoffset) {
        /* When the the last entry we need to update is also the tail, update tail offset
         * unless this is the only entry that was updated (so the tail offset didn't change). */
        if (extra - delta != 0) {
            ZIPLIST_TAIL_OFFSET(zl) =
                intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+extra-delta);
        }
    } else {
        /* Update the tail offset in cases where the last entry we updated is not the tail. */
        ZIPLIST_TAIL_OFFSET(zl) =
            intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+extra);
    }
    
    /* Now "p" points at the first unchanged byte in original ziplist,
     * move data after that to new ziplist. */
    offset = p - zl;
    zl = ziplistResize(zl, curlen + extra);
    p = zl + offset;
    memmove(p + extra, p, curlen - offset - 1);
    p += extra;
    
    /* Iterate all entries that need to be updated tail to head. */
    while (cnt) {
        zipEntry(zl + prevoffset, &cur); /* no need for "safe" variant since we already iterated on all these entries above. */
        rawlen = cur.headersize + cur.len;
        /* Move entry to tail and reset prevlen. */
        memmove(p - (rawlen - cur.prevrawlensize),
                zl + prevoffset + cur.prevrawlensize,
                rawlen - cur.prevrawlensize);
        p -= (rawlen + delta);
        if (cur.prevrawlen == 0) {
            /* "cur" is the previous head entry, update its prevlen with firstentrylen. */
            zipStorePrevEntryLength(p, firstentrylen);
        } else {
            /* An entry's prevlen can only increment 4 bytes. */
            zipStorePrevEntryLength(p, cur.prevrawlen+delta);
        }
        /* Foward to previous entry. */
        prevoffset -= cur.prevrawlen;
        cnt--;
    }
    return zl;
}
```
</details>

<br />

除了添加节点，删除节点也可能产生连锁更新，比如在上一个例子中的压缩列表基础上删除`new`。

因为连锁更新在最坏情况下需要对压缩列表执行N次空间重分配操作，而每次空间重分配的最坏复杂度为$O(N)$，所以连锁更新的最坏复杂度为$O(N^2)$。

要注意的是，尽管连锁更新的复杂度较高，但它真正造成性能问题的几率是很低的：

-   首先，压缩列表里要恰好有多个连续的、长度介于250字节至253字节之间的节点，连锁更新才有可能被引发，在实际中，这种情况并不多见；
-   其次，即使出现连锁更新，但只要被更新的节点数量不多，就不会对性能造成任何影响：比如说，对三五个节点进行连锁更新是绝对不会影响性能的；

因为以上原因，`ziplistPush`等命令的平均复杂度仅为$O(N)$，在实际中，我们可以放心地使用这些函数，而不必担心连锁更新会影响压缩列表的性能。

---

## API

| 函数               | 作用                                                         | 算法复杂度                                                   |
| ------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| ziplistNew         | 创建一个新的压缩列表                                         | $O(1)$                                                       |
| ziplistPush        | 创建一个包含给定值的新节点,并将这个新节点添加到压缩列表的表头或者表尾 | 平均$O(N)$，最坏$O(N^2)$                                     |
| ziplistInsert      | 将包含给定值的新节点插入到给定节点之后                       | 平均$O(N)$，最坏$O(N^2)$                                     |
| ziplistindex       | 返回压缩列表给定索引上的节点                                 | $O(N)$                                                       |
| ziplistfind        | 在压缩列表中查找并返回包含了给定值的节点                     | 因为节点的值可能是一个字节数组，所以检查节点值和给定值是否相同的复杂度为$O(N)$，而查找整个列表的复杂度$O(N^2)$。 |
| ziplistNext        | 返回给定节点的下一个节点                                     | $O(1)$                                                       |
| ziplistPrev        | 返回给定节点的前一个节点                                     | $O(1)$                                                       |
| ziplistGet         | 获取给定节点所保存的值                                       | $O(1)$                                                       |
| ziplistDelete      | 从压缩列表中删除给定的节点                                   | 平均$O(N)$，最坏$O(N^2)$                                     |
| ziplistDeleteRange | 删除压缩列表在给定索引上的连续多个节点                       | 平均$O(N)$，最坏$O(N^2)$                                     |
| ziplistBlobLen     | 返回压缩列表目前占用的内存字节数                             | $O(1)$                                                       |
| ziplistLen         | 返回压缩列表目前包含的节点数量                               | 返回压缩列表目前包含的节点数量，节点数量小于65535时为$O(1)$，大于65535时为$O(N)$ |



# 总结

1.  压缩列表是一种为节约内存而开发的顺序型数据结构，本质就是一个双向链表。
2.  压缩列表被用作列表键和哈希键的底层实现之一。
3.  压缩列表可以包含多个节点，每个节点可以保存一个字节数组或者整数值。
4.  添加新节点到压缩列表，或者从压缩列表中删除节点，可能会引发连锁更新操作，但这种操作出现的几率并不高。
