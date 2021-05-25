---
title: "Redis 学习 对象篇"
date: 2021-05-25 22:40:00
updated: 2021-05-26 01:25:30
description: "Redis 对象篇。"
mathjax: false
summary:
categories:
  - 学习
  - redis
  - 对象
tags:
  - redis
  - object
  - 对象
  - TODO
---

>   基于 [Redis 6.2.1](https://github.com/redis/redis/tree/6.2.1)

# Redis 对象

前文也有提到，`Redis` 中并没有直接使用底层的基本数据结构，比如简单动态字符串（`SDS`）、双端链表、字典、压缩列表、整数集合等。`Redis` 是基于这些基本的数据结构创建了一个对象系统，这个系统包含字符串对象（`string`）、列表对象（`list`）、哈希对象（`hash`）、集合对象（`set`）和有序集合对象（`sorted sort`）等。每种对象都用到了至少一种前面所介绍的数据结构。

通过这五种不同类型的对象，`Redis` 可以在执行命令前，根据对象的类型判断一个对象是否可以执行给定的命令。使用对象的另一个好处是，可以针对不同的使用场景，采用不同的底层基本数据结构实现对象，从而优化性能。

另外，`Redis` 的对象系统还实现了基于引用计数的内存回收机制，当程序不再使用某个对象的时候，会**自动释放这个对象所占用的内存**；`Redis` 还通过引用计数实现了对象共享机制，在适当的条件下，多个数据库键可以**共享同一个对象类节约内存**。

最后，`Redis` 的对象记录了访问时间信息，可以用来计算数据库键的空转时长，在服务器开启了`maxmemory`功能时，空转较大的键可能会优先被服务器删除。

## 对象的类型与编码

`Redis` 使用对象来表示数据库中的键和值，每当我们创建一个新的键值对时，至少会创建两个对象，一个对象用作键值对的键（键对象），另一个用作键值对的值（值对象）。

举个例子，一下`SET`命令在数据库中创建了一个新的键值对，其中键值对的键是一个包含了字符串值`“msg”`的对象，而键值对的值则是一个包含了字符串值`“hello world”`的对象。

```redis
redis > SET msg "hello world"
OK
```

`Redis` 中对象的结构如下：

```c
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj;
```

-   `type`：对象类型。
-   `encoding`：对象编码。
-   `lru`：对象最后一次被访问的时间。
-   `refcount`：引用计数。
-   `*ptr`：指向底层数据结构的指针。

### 类型

`Redis` 的对象类型共有**七**个。由对象的`type`属性记录。

-   `OBJ_STRING` 0   字符串(String)对象
-   `OBJ_LIST` 1     列表(List)对象
-   `OBJ_SET` 2      集合(Set)对象
-   `OBJ_ZSET` 3     有序集合(Sorted set)对象
-   `OBJ_HASH` 4     哈(Hash)希对象
-   `OBJ_MODULE` 5   模(Module)块对象
-   `OBJ_STREAM` 6   流(Stream)对象

>   对于`Redis`数据库保存的键值对来说，键总是一个字符串对象，而值则可以是字符串对象、列表对象、哈希对象、集合对象或者有序集合对象的其中一种，因此：
>
>   -   当我们称呼一个数据库键为“字符串键”时，我们指的是“这个数据库键所对应的值为字符串对象”；
>   -   当我们称呼一个键为“列表键”时，我们指的是“这个数据库键所对应的值为列表对象”。
>
>   `TYPE`命令的实现方式也与此类似，当我们对一个数据库键执行`TYPE`命令时，命令返回的结果为数据库键对应的值对象的类型，而不是键对象的类型：
>
>   ```c
>   redis> SET msg "hello world"
>   OK
>   redis> TYPE msg
>   "string"
>   redis> RPUSH numbers 1 3 5
>   (integer) 6
>   redis> TYPE numbers
>   "list"
>   // 其他对象同理，不赘述了。
>   ```

### 编码

对象的`ptr`指针指向对象的底层实现数据结构，而这些数据结构由对象的`encoding`属性决定。

`Redis` 的编码类型共有**十一**个。由对象的`encoding`属性记录。

>   下面的枚举就有些有点词不达意，，，

-   `OBJ_ENCODING_RAW` 0     `SDS`表示。
-   `OBJ_ENCODING_INT` 1     编码为整数（`long` 类型）
-   `OBJ_ENCODING_HT` 2      编码为字典
-   `OBJ_ENCODING_ZIPMAP` 3  编码为`zipmap`。⚠️这个在代码中从来没有用过。
-   `OBJ_ENCODING_LINKEDLIST` 4 不再使用：旧列表编码。
-   `OBJ_ENCODING_ZIPLIST` 5 编码为`ziplist`
-   `OBJ_ENCODING_INTSET` 6  编码为`intset`
-   `OBJ_ENCODING_SKIPLIST` 7  编码为`skiplist` （其实是跳表和字典）
-   `OBJ_ENCODING_EMBSTR` 8  embstr编码`SDS`
-   `OBJ_ENCODING_QUICKLIST` 9 编码为`ziplist`的链接列表
-   `OBJ_ENCODING_STREAM` 10 编码为列表包的基数树

### 不同类型对象的编码

不同类型采取了多种编码方式，下表列出了每种类型的对象可以使用的编码。

>   可通过`object.c` 中的`size_t objectComputeSize(robj **o*, size_t *sample_size*)` 查看
>   <details><summary>objectComputeSize 源码 展开/收起</summary>
>   ```c
>   size_t objectComputeSize(robj *o, size_t sample_size) {
>       sds ele, ele2;
>       dict *d;
>       dictIterator *di;
>       struct dictEntry *de;
>       size_t asize = 0, elesize = 0, samples = 0;
>   
>       if (o->type == OBJ_STRING) {
>           if(o->encoding == OBJ_ENCODING_INT) {
>               asize = sizeof(*o);
>           } else if(o->encoding == OBJ_ENCODING_RAW) {
>               asize = sdsZmallocSize(o->ptr)+sizeof(*o);
>           } else if(o->encoding == OBJ_ENCODING_EMBSTR) {
>               asize = sdslen(o->ptr)+2+sizeof(*o);
>           } else {
>               serverPanic("Unknown string encoding");
>           }
>       } else if (o->type == OBJ_LIST) {
>           if (o->encoding == OBJ_ENCODING_QUICKLIST) {
>               quicklist *ql = o->ptr;
>               quicklistNode *node = ql->head;
>               asize = sizeof(*o)+sizeof(quicklist);
>               do {
>                   elesize += sizeof(quicklistNode)+ziplistBlobLen(node->zl);
>                   samples++;
>               } while ((node = node->next) && samples < sample_size);
>               asize += (double)elesize/samples*ql->len;
>           } else if (o->encoding == OBJ_ENCODING_ZIPLIST) {
>               asize = sizeof(*o)+ziplistBlobLen(o->ptr);
>           } else {
>               serverPanic("Unknown list encoding");
>           }
>       } else if (o->type == OBJ_SET) {
>           if (o->encoding == OBJ_ENCODING_HT) {
>               d = o->ptr;
>               di = dictGetIterator(d);
>               asize = sizeof(*o)+sizeof(dict)+(sizeof(struct dictEntry*)*dictSlots(d));
>               while((de = dictNext(di)) != NULL && samples < sample_size) {
>                   ele = dictGetKey(de);
>                   elesize += sizeof(struct dictEntry) + sdsZmallocSize(ele);
>                   samples++;
>               }
>               dictReleaseIterator(di);
>               if (samples) asize += (double)elesize/samples*dictSize(d);
>           } else if (o->encoding == OBJ_ENCODING_INTSET) {
>               intset *is = o->ptr;
>               asize = sizeof(*o)+sizeof(*is)+is->encoding*is->length;
>           } else {
>               serverPanic("Unknown set encoding");
>           }
>       } else if (o->type == OBJ_ZSET) {
>           if (o->encoding == OBJ_ENCODING_ZIPLIST) {
>               asize = sizeof(*o)+(ziplistBlobLen(o->ptr));
>           } else if (o->encoding == OBJ_ENCODING_SKIPLIST) {
>               d = ((zset*)o->ptr)->dict;
>               zskiplist *zsl = ((zset*)o->ptr)->zsl;
>               zskiplistNode *znode = zsl->header->level[0].forward;
>               asize = sizeof(*o)+sizeof(zset)+sizeof(zskiplist)+sizeof(dict)+
>                       (sizeof(struct dictEntry*)*dictSlots(d))+
>                       zmalloc_size(zsl->header);
>               while(znode != NULL && samples < sample_size) {
>                   elesize += sdsZmallocSize(znode->ele);
>                   elesize += sizeof(struct dictEntry) + zmalloc_size(znode);
>                   samples++;
>                   znode = znode->level[0].forward;
>               }
>               if (samples) asize += (double)elesize/samples*dictSize(d);
>           } else {
>               serverPanic("Unknown sorted set encoding");
>           }
>       } else if (o->type == OBJ_HASH) {
>           if (o->encoding == OBJ_ENCODING_ZIPLIST) {
>               asize = sizeof(*o)+(ziplistBlobLen(o->ptr));
>           } else if (o->encoding == OBJ_ENCODING_HT) {
>               d = o->ptr;
>               di = dictGetIterator(d);
>               asize = sizeof(*o)+sizeof(dict)+(sizeof(struct dictEntry*)*dictSlots(d));
>               while((de = dictNext(di)) != NULL && samples < sample_size) {
>                   ele = dictGetKey(de);
>                   ele2 = dictGetVal(de);
>                   elesize += sdsZmallocSize(ele) + sdsZmallocSize(ele2);
>                   elesize += sizeof(struct dictEntry);
>                   samples++;
>               }
>               dictReleaseIterator(di);
>               if (samples) asize += (double)elesize/samples*dictSize(d);
>           } else {
>               serverPanic("Unknown hash encoding");
>           }
>       } else if (o->type == OBJ_STREAM) {
>           stream *s = o->ptr;
>           asize = sizeof(*o);
>           asize += streamRadixTreeMemoryUsage(s->rax);
>   
>           /* Now we have to add the listpacks. The last listpack is often non
>            * complete, so we estimate the size of the first N listpacks, and
>            * use the average to compute the size of the first N-1 listpacks, and
>            * finally add the real size of the last node. */
>           raxIterator ri;
>           raxStart(&ri,s->rax);
>           raxSeek(&ri,"^",NULL,0);
>           size_t lpsize = 0, samples = 0;
>           while(samples < sample_size && raxNext(&ri)) {
>               unsigned char *lp = ri.data;
>               lpsize += lpBytes(lp);
>               samples++;
>           }
>           if (s->rax->numele <= samples) {
>               asize += lpsize;
>           } else {
>               if (samples) lpsize /= samples; /* Compute the average. */
>               asize += lpsize * (s->rax->numele-1);
>               /* No need to check if seek succeeded, we enter this branch only
>                * if there are a few elements in the radix tree. */
>               raxSeek(&ri,"$",NULL,0);
>               raxNext(&ri);
>               asize += lpBytes(ri.data);
>           }
>           raxStop(&ri);
>       
>           /* Consumer groups also have a non trivial memory overhead if there
>            * are many consumers and many groups, let's count at least the
>            * overhead of the pending entries in the groups and consumers
>            * PELs. */
>           if (s->cgroups) {
>               raxStart(&ri,s->cgroups);
>               raxSeek(&ri,"^",NULL,0);
>               while(raxNext(&ri)) {
>                   streamCG *cg = ri.data;
>                   asize += sizeof(*cg);
>                   asize += streamRadixTreeMemoryUsage(cg->pel);
>                   asize += sizeof(streamNACK)*raxSize(cg->pel);
>       
>                   /* For each consumer we also need to add the basic data
>                    * structures and the PEL memory usage. */
>                   raxIterator cri;
>                   raxStart(&cri,cg->consumers);
>                   raxSeek(&cri,"^",NULL,0);
>                   while(raxNext(&cri)) {
>                       streamConsumer *consumer = cri.data;
>                       asize += sizeof(*consumer);
>                       asize += sdslen(consumer->name);
>                       asize += streamRadixTreeMemoryUsage(consumer->pel);
>                       /* Don't count NACKs again, they are shared with the
>                        * consumer group PEL. */
>                   }
>                   raxStop(&cri);
>               }
>               raxStop(&ri);
>           }
>       } else if (o->type == OBJ_MODULE) {
>           moduleValue *mv = o->ptr;
>           moduleType *mt = mv->type;
>           if (mt->mem_usage != NULL) {
>               asize = mt->mem_usage(mv->value);
>           } else {
>               asize = 0;
>           }
>       } else {
>           serverPanic("Unknown object type");
>       }
>       return asize;
>   }
>   ```
>   </details>

| 类型       | 编码                   | 对象                                                        |
| ---------- | ---------------------- | ----------------------------------------------------------- |
| OBJ_STRING | OBJ_ENCODING_INT       | 使用整数值实现的字符串对象                                  |
| OBJ_STRING | OBJ_ENCODING_RAW       | 使用SDS实现的字符串对抗                                     |
| OBJ_STRING | OBJ_ENCODING_EMBSTR    | 使用embstr编码的SDS实现的字符串对象                         |
| OBJ_LIST   | OBJ_ENCODING_QUICKLIST | 使用quicklist实现的列表对象                                 |
| OBJ_LIST   | OBJ_ENCODING_ZIPLIST   | 使用ziplist实现的列表对象                                   |
| OBJ_SET    | OBJ_ENCODING_HT        | 使用字典实现的集合对象                                      |
| OBJ_SET    | OBJ_ENCODING_INTSET    | 使用整数集合实现的集合对象                                  |
| OBJ_ZSET   | OBJ_ENCODING_SKIPLIST  | 使用跳表和字典实现的有序集合对象                            |
| OBJ_ZSET   | OBJ_ENCODING_ZIPLIST   | 使用ziplist实现的有序集合对象                               |
| OBJ_HASH   | OBJ_ENCODING_ZIPLIST   | 使用ziplist实现的哈希对象                                   |
| OBJ_HASH   | OBJ_ENCODING_HT        | 使用字典实现的哈希对象                                      |
| OBJ_STREAM | 无                     | <font color="red">TODO 这里还没弄清 应该是特殊的实现</font> |
| OBJ_MODULE | 无                     | <font color="red">TODO 这里还没弄清 应该是特殊的实现</font> |

>   使用`OBJECT ENCODING`命令可以查看一个数据库键的值对象的编码