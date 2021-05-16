---
title: "Redis 学习 基础数据结构篇 之 dict"
date: 2021-04-29 22:35:23
description: ["redis 基础数据结构 - 字典。"]
toc: true
author: tabris

# 图片推荐使用图床(腾讯云、七牛云、又拍云等)来做图片的路径.如:http://xxx.com/xxx.jpg
img:

# 如果top值为true,则会是首页推荐文章
top: false

# 如果要对文章设置阅读验证密码的话,就可以在设置password的值,该值必须是用SHA256加密后的密码,防止被他人识破
password:

# 本文章是否开启mathjax，且需要在主题的_config.yml文件中也需要开启才行
mathjax: false
summary:
categories:
  - redis
tags:
  - redis
  - dict
  - rehash
  - 渐进式 rehesh
---

[TOC]

> 基于 [Redis 6.2.1]([redis/redis at 6.2.1 (github.com)](https://github.com/redis/redis/tree/6.2.1))

# dict（字典）

> 字典， 又称符号表（symbol table）、关联数组（associative array）或者映射（map）， 是一种用于保存键值对（key-value pair）的抽象数据结构。

        在字典中，一个键（key）和一个值（value）关联，关联上的键和值被称为键值对

        很多语言都提供了字典的实现，如C++ STL中的map，python中的dict，Go中的map等等。C语言中并为提供字典实现，因此Redis自行实现了字典。

        Redis中很多地方用到了字典，Redis的数据库，HASH类型等。

## dict的实现

`src/dict.h` 中定义了字典

```c
// hashtable 哈希表结构
typedef struct dictht {
    dictEntry **table;      // dictEntry 二维指针， 被当成指针数组用的。
    unsigned long size;     // 哈希表容量大小
    unsigned long sizemask; // 等于size-1， 方便位运算的
    unsigned long used;     // 哈希表使用的大小
} dictht;
```

        `table` 属性是一个数组， 数组中的每个元素都是一个指向 `dict.h/dictEntry` 结构的指针， 每个 `dictEntry` 结构保存着一个键值对。

        `size` 属性记录了哈希表的大小， 也即是 `table` 数组的大小， 而 `used` 属性则记录了哈希表目前已有节点（键值对）的数量。

        `sizemask` 属性的值总是等于 `size - 1` ， 这个属性和哈希值一起决定一个键应该被放到 `table` 数组的哪个索引上面。

        采用的hash算法在`src/siphash.c` 中， 采用的hash算法 是 `SipHash 1-2` 本文并不展开介绍`SipHash 1-2`原理。

```c
// 哈希表节点， 也就是一个键值对
typedef struct dictEntry {
    void *key;              // 键
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;                    // 值
    struct dictEntry *next; // next指针， 用来开链解决冲突的。
} dictEntry;
```

        `key` 属性保存着键值对中的键， 而 `v` 属性则保存着键值对中的值， 其中键值对的值可以是一个指针， 或者是一个 `uint64_t` 整数， 又或者是一个 `int64_t` 整数。

        `next` 属性是指向另一个哈希表节点的指针， 这个指针可以将多个哈希值相同的键值对连接在一次， 以此来解决键冲突（collision）的问题。

举个例子， 图 4-2 就展示了如何通过 `next` 指针， 将两个索引值相同的键 `k1` 和 `k0` 连接在一起。

```c
// 字典 结构
typedef struct dict {
    dictType *type;  // 字典类型 （用来支持不同类型的，暂时不用泰太过关注）
    void *privdata;  // 私有数据
    dictht ht[2];    // 哈希表  这里有两个，ht[1] 是rehash的时候才用的
    long rehashidx;  // 是否rehash标记 /没有rehash的时候等于-1， rehash的时候为接下来要rehash的哈希表下标
    int16_t pauserehash; // 暂停rehash标记 /* If >0 rehashing is paused (<0 indicates coding error) */
} dict;
```

`type` 属性和 `privdata` 属性是针对不同类型的键值对， 为创建多态字典而设置的：

- `type` 属性是一个指向 `dictType` 结构的指针， 每个 `dictType` 结构保存了一簇用于操作特定类型键值对的函数， Redis 会为用途不同的字典设置不同的类型特定函数。

- 而 `privdata` 属性则保存了需要传给那些类型特定函数的可选参数。

```c
typedef struct dictType {
  // 计算哈希值的函数
  unsigned int (*hashFunction)(const void *key);
  // 复制键的函数
  void *(*keyDup)(void *privdata, const void *key);
  // 复制值的函数
  void *(*valDup)(void *privdata, const void *obj);
  // 对比键的函数
  int (*keyCompare)(void *privdata, const void *key1, const void *key2);
  // 销毁键的函数
  void (*keyDestructor)(void *privdata, void *key);
  // 销毁值的函数
  void (*valDestructor)(void *privdata, void *obj);
} dictType;
```

        `ht` 属性是一个包含两个项的数组， 数组中的每个项都是一个 `dictht` 哈希表， 一般情况下， 字典只使用 `ht[0]` 哈希表， `ht[1]` 哈希表只会在对 `ht[0]` 哈希表进行 rehash 时使用。

        除了 `ht[1]` 之外， 另一个和 rehash 有关的属性就是 `rehashidx` ： 它记录了 rehash 目前的进度， 如果目前没有在进行 rehash ， 那么它的值为 `-1` 。

下图是一个完整的字典结构（普通状态下，没有进行rehash）：

![](https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/05/01/20210501184352.png)

## hash 算法

TODO 感觉没啥好说的了。

### hash冲突解决（链地址法）

TODO 这个感觉也没啥好说的，太基础了。

## rehash 和 渐进式 rehash

### rehash

        随着频繁的增删操作， 哈希表可能不够用了或者空间使用率变低了，这时候就需要进行扩容或缩容，也就是Redis的rehash过程。

        rehash过程很好理解，当哈希表需要扩容或缩容的时候，改变哈希表的大小，对所有元素重新 hash 后放到新的位置。

        以下图为例， 当字典使用率过低时，会进行缩容。按照used个数确定合适的容量生成一个新的。这时候将旧的哈希表（h[0]）中的数据重新hash放到新的哈希表中（h[1]）。

![](https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/05/01/20210501184231.jpeg)

        同理当容量不足的时候，字典会进行扩容。容量大小会翻一倍（乘2）。也是一样生成一个新的。把旧的哈希表（h[0]）中的数据重新hash放到新的哈希表中（h[1]）。

![IMG_E3060FCFC999-1.jpeg](https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/05/01/20210501222136.jpeg)

        ⚠️注意， 如上图所示，当给一个容量为4的哈希表添加键值对，然后再删一个，然后再添加一个。。。这样重复的删除再添加。 会一直rehash下去，每次都要进行大量的内存操作和数据结构的维护处理，想想Redis本身单进程的，这得慢成什么样子。 所以 Redis 在处理扩容缩容时有这样的策略：

        一般情况下 当`used/size >= 1` 时, 进行**扩容**。 容量缩到大于 used 的第一个大小。

        正在进行高负载命令时（行 BGSAVE 命令或BGREWRITEAOF 命令）， 当`used/size > 5`时才进行扩容，容量缩到大于 used 的第一个大小。

        当`used/size <= 1/10` 时进行**缩容**。容量缩到大于 used 的第一个大小。

扩容策略源码：

```c
/* Expand the hash table if needed */
static int _dictExpandIfNeeded(dict *d)
{
    /* Incremental rehashing already in progress. Return. */
    if (dictIsRehashing(d)) return DICT_OK;

    /* If the hash table is empty expand it to the initial size. */
    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);

    /* If we reached the 1:1 ratio, and we are allowed to resize the hash
     * table (global setting) or we should avoid it but the ratio between
     * elements/buckets is over the "safe" threshold, we resize doubling
     * the number of buckets. */
    if (d->ht[0].used >= d->ht[0].size &&
        (dict_can_resize ||
         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio) &&
        dictTypeExpandAllowed(d))
    {
        return dictExpand(d, d->ht[0].used + 1);
    }
    return DICT_OK;
}
```

### 渐进式rehash

        有了上面rehash的，字典可以应对容量的变化， 但如果字典已经存储了十万，百万，甚至千万量级的键值对，一次rehash的过程会持续很久，这个时候因为Redis单进程处理会阻塞很久，业务服务就会卡住。

        因此Redis实现了渐进式的rehash，将rehash的操作分散到字典多次的增删改查操作中。每一次只操作少量的哈希表的槽位，将整体rehash操作均摊。

        从字典的结构中可以看到它有两个哈希表。正常情况下直接使用的`ht[0]`, 渐进式的rehash过程就是逐步将键值对迁移到`ht[1]`中。在rehash的过程中进行操作时字典会先从`ht[0]`找，如果找不到则在`ht[1]`中找。 如下面`dictFind`函数：

```c
dictEntry *dictFind(dict *d, const void *key)
{
    dictEntry *he;
    uint64_t h, idx, table;

    if (dictSize(d) == 0) return NULL; /* dict is empty */
    if (dictIsRehashing(d)) _dictRehashStep(d);
    h = dictHashKey(d, key);
    for (table = 0; table <= 1; table++) {
        idx = h & d->ht[table].sizemask;
        he = d->ht[table].table[idx];
        while(he) {
            if (key==he->key || dictCompareKeys(d, key, he->key))
                return he;
            he = he->next;
        }
        if (!dictIsRehashing(d)) return NULL;
    }
    return NULL;
}
```

渐进式rehash主要时限制了两个，

1. 最多可迁移的槽的个数`n`。

2. 最多可遍历的空槽的个数`n*10`

如果进行了这么多的遍历或rehash操作，函数则退出，如果rehash已经结束返回`0`，没有结束返回`1`。

最后看下rehash的完整函数吧：

```c
/* Performs N steps of incremental rehashing. Returns 1 if there are still
 * keys to move from the old to the new hash table, otherwise 0 is returned.
 *
 * Note that a rehashing step consists in moving a bucket (that may have more
 * than one key as we use chaining) from the old to the new hash table, however
 * since part of the hash table may be composed of empty spaces, it is not
 * guaranteed that this function will rehash even a single bucket, since it
 * will visit at max N*10 empty buckets in total, otherwise the amount of
 * work it does would be unbound and the function may block for a long time. */
int dictRehash(dict *d, int n) {
    int empty_visits = n*10; /* Max number of empty buckets to visit. */
    if (!dictIsRehashing(d)) return 0;

    while(n-- && d->ht[0].used != 0) {
        dictEntry *de, *nextde;

        /* Note that rehashidx can't overflow as we are sure there are more
         * elements because ht[0].used != 0 */
        assert(d->ht[0].size > (unsigned long)d->rehashidx);
        while(d->ht[0].table[d->rehashidx] == NULL) {
            d->rehashidx++;
            if (--empty_visits == 0) return 1;
        }
        de = d->ht[0].table[d->rehashidx];
        /* Move all the keys in this bucket from the old to the new hash HT */
        while(de) {
            uint64_t h;

            nextde = de->next;
            /* Get the index in the new hash table */
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;
            d->ht[0].used--;
            d->ht[1].used++;
            de = nextde;
        }
        d->ht[0].table[d->rehashidx] = NULL;
        d->rehashidx++;
    }

    /* Check if we already rehashed the whole table... */
    if (d->ht[0].used == 0) {
        zfree(d->ht[0].table);
        d->ht[0] = d->ht[1];
        _dictReset(&d->ht[1]);
        d->rehashidx = -1;
        return 0;
    }

    /* More to rehash... */
    return 1;
}
```

## dict重要API

| 函数                 | 作用                                      | 时间复杂度                                                                            |
| ------------------ | --------------------------------------- | -------------------------------------------------------------------------------- |
| `dictCreate`       | 创建一个新的字典。                               | ![O(1)](https://box.kancloud.cn/2015-09-13_55f5139a8e88e.png)                    |
| `dictAdd`          | 将给定的键值对添加到字典里面。                         | ![O(1)](https://box.kancloud.cn/2015-09-13_55f5139a8e88e.png)                    |
| `dictReplace`      | 将给定的键值对添加到字典里面， 如果键已经存在于字典，那么用新值取代原有的值。 | ![O(1)](https://box.kancloud.cn/2015-09-13_55f5139a8e88e.png)                    |
| `dictFetchValue`   | 返回给定键的值。                                | ![O(1)](https://box.kancloud.cn/2015-09-13_55f5139a8e88e.png)                    |
| `dictGetRandomKey` | 从字典中随机返回一个键值对。                          | ![O(1)](https://box.kancloud.cn/2015-09-13_55f5139a8e88e.png)                    |
| `dictDelete`       | 从字典中删除给定键所对应的键值对。                       | ![O(1)](https://box.kancloud.cn/2015-09-13_55f5139a8e88e.png)                    |
| `dictRelease`      | 释放给定字典，以及字典中包含的所有键值对。                   | ![O(N)](https://box.kancloud.cn/2015-09-13_55f513a53880b.png) ， `N` 为字典包含的键值对数量。 |

# 总结

字典中最重要的实现是 rehash 及其渐进式的方式。