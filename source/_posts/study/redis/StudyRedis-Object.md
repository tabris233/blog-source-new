---
title: "Redis 学习 对象篇"
date: 2021-05-25 22:40:00
updated: 2021-05-27 01:33:41
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
> 基于 [Redis 6.2.1](https://github.com/redis/redis/tree/6.2.1)

# Redis 对象

前文也有提到，`Redis` 中并没有直接使用底层的基本数据结构，比如简单动态字符串（`SDS`）、双端链表、字典、压缩列表、整数集合等。`Redis` 是基于这些基本的数据结构创建了一个对象系统，这个系统包含字符串对象（`string`）、列表对象（`list`）、哈希对象（`hash`）、集合对象（`set`）和有序集合对象（`sorted sort`）等。每种对象都用到了至少一种前面所介绍的数据结构。

通过这五种不同类型的对象，`Redis` 可以在执行命令前，根据对象的类型判断一个对象是否可以执行给定的命令。使用对象的另一个好处是，可以针对不同的使用场景，采用不同的底层基本数据结构实现对象，从而优化性能。

另外，`Redis` 的对象系统还实现了基于引用计数的内存回收机制，当程序不再使用某个对象的时候，会**自动释放这个对象所占用的内存**；`Redis` 还通过引用计数实现了对象共享机制，在适当的条件下，多个数据库键可以**共享同一个对象类节约内存**。

最后，`Redis` 的对象记录了访问时间信息，可以用来计算数据库键的空转时长，在服务器开启了 `maxmemory` 功能时，空转较大的键可能会优先被服务器删除。

## 对象的类型与编码

`Redis` 使用对象来表示数据库中的键和值，每当我们创建一个新的键值对时，至少会创建两个对象，一个对象用作键值对的键（键对象），另一个用作键值对的值（值对象）。

举个例子，一下 `SET` 命令在数据库中创建了一个新的键值对，其中键值对的键是一个包含了字符串值 `“msg”` 的对象，而键值对的值则是一个包含了字符串值 `“hello world”` 的对象。

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

- `type`：对象类型。
- `encoding`：对象编码。
- `lru`：对象最后一次被访问的时间。
- `refcount`：引用计数。
- `*ptr`：指向底层数据结构的指针。

### 类型

`Redis` 的对象类型共有**七**个。由对象的 `type` 属性记录。

- `OBJ_STRING` 0   字符串(String)对象
- `OBJ_LIST` 1     列表(List)对象
- `OBJ_SET` 2      集合(Set)对象
- `OBJ_ZSET` 3     有序集合(Sorted set)对象
- `OBJ_HASH` 4     哈(Hash)希对象
- `OBJ_MODULE` 5   模(Module)块对象
- `OBJ_STREAM` 6   流(Stream)对象

> 对于 `Redis` 数据库保存的键值对来说，键总是一个字符串对象，而值则可以是字符串对象、列表对象、哈希对象、集合对象或者有序集合对象的其中一种，因此：
>
> - 当我们称呼一个数据库键为“字符串键”时，我们指的是“这个数据库键所对应的值为字符串对象”；
> - 当我们称呼一个键为“列表键”时，我们指的是“这个数据库键所对应的值为列表对象”。
>
> `TYPE` 命令的实现方式也与此类似，当我们对一个数据库键执行 `TYPE` 命令时，命令返回的结果为数据库键对应的值对象的类型，而不是键对象的类型：
>
> ```c
> redis> SET msg "hello world"
> OK
> redis> TYPE msg
> "string"
> redis> RPUSH numbers 1 3 5
> (integer) 6
> redis> TYPE numbers
> "list"
> // 其他对象同理，不赘述了。
> ```

### 编码

对象的 `ptr` 指针指向对象的底层实现数据结构，而这些数据结构由对象的 `encoding` 属性决定。

`Redis` 的编码类型共有**十一**个。由对象的 `encoding` 属性记录。

> 下面的枚举就有些有点词不达意，，，

- `OBJ_ENCODING_RAW` 0     `SDS` 表示。
- `OBJ_ENCODING_INT` 1     编码为整数（`long` 类型）
- `OBJ_ENCODING_HT` 2      编码为字典
- `OBJ_ENCODING_ZIPMAP` 3  编码为 `zipmap`。⚠️ 这个在代码中从来没有用过。
- `OBJ_ENCODING_LINKEDLIST` 4 不再使用：旧列表编码。
- `OBJ_ENCODING_ZIPLIST` 5 编码为 `ziplist`
- `OBJ_ENCODING_INTSET` 6  编码为 `intset`
- `OBJ_ENCODING_SKIPLIST` 7  编码为 `skiplist` （其实是跳表和字典）
- `OBJ_ENCODING_EMBSTR` 8  embstr 编码 `SDS`
- `OBJ_ENCODING_QUICKLIST` 9 编码为 `ziplist` 的链接列表
- `OBJ_ENCODING_STREAM` 10 编码为列表包的基数树

### 不同类型对象的编码

不同类型采取了多种编码方式，下表列出了每种类型的对象可以使用的编码。

> 可通过 `object.c` 中的 `size_t objectComputeSize(robj **o*, size_t *sample_size*)` 查看
>
> <details><summary>objectComputeSize 源码 展开/收起</summary>
>   ```c
>   size_t objectComputeSize(robj *o, size_t sample_size) {
>       sds ele, ele2;
>       dict *d;
>       dictIterator *di;
>       struct dictEntry *de;
>       size_t asize = 0, elesize = 0, samples = 0;
>
>   if (o->type == OBJ_STRING) {
>       if(o->encoding == OBJ_ENCODING_INT) {
>           asize = sizeof(*o);
>       } else if(o->encoding == OBJ_ENCODING_RAW) {
>           asize = sdsZmallocSize(o->ptr)+sizeof(*o);
>       } else if(o->encoding == OBJ_ENCODING_EMBSTR) {
>           asize = sdslen(o->ptr)+2+sizeof(*o);
>       } else {
>           serverPanic("Unknown string encoding");
>       }
>   } else if (o->type == OBJ_LIST) {
>       if (o->encoding == OBJ_ENCODING_QUICKLIST) {
>           quicklist *ql = o->ptr;
>           quicklistNode *node = ql->head;
>           asize = sizeof(*o)+sizeof(quicklist);
>           do {
>               elesize += sizeof(quicklistNode)+ziplistBlobLen(node->zl);
>               samples++;
>           } while ((node = node->next) && samples < sample_size);
>           asize += (double)elesize/samples*ql->len;
>       } else if (o->encoding == OBJ_ENCODING_ZIPLIST) {
>           asize = sizeof(*o)+ziplistBlobLen(o->ptr);
>       } else {
>           serverPanic("Unknown list encoding");
>       }
>   } else if (o->type == OBJ_SET) {
>       if (o->encoding == OBJ_ENCODING_HT) {
>           d = o->ptr;
>           di = dictGetIterator(d);
>           asize = sizeof(*o)+sizeof(dict)+(sizeof(struct dictEntry*)*dictSlots(d));
>           while((de = dictNext(di)) != NULL && samples < sample_size) {
>               ele = dictGetKey(de);
>               elesize += sizeof(struct dictEntry) + sdsZmallocSize(ele);
>               samples++;
>           }
>           dictReleaseIterator(di);
>           if (samples) asize += (double)elesize/samples*dictSize(d);
>       } else if (o->encoding == OBJ_ENCODING_INTSET) {
>           intset *is = o->ptr;
>           asize = sizeof(*o)+sizeof(*is)+is->encoding*is->length;
>       } else {
>           serverPanic("Unknown set encoding");
>       }
>   } else if (o->type == OBJ_ZSET) {
>       if (o->encoding == OBJ_ENCODING_ZIPLIST) {
>           asize = sizeof(*o)+(ziplistBlobLen(o->ptr));
>       } else if (o->encoding == OBJ_ENCODING_SKIPLIST) {
>           d = ((zset*)o->ptr)->dict;
>           zskiplist *zsl = ((zset*)o->ptr)->zsl;
>           zskiplistNode *znode = zsl->header->level[0].forward;
>           asize = sizeof(*o)+sizeof(zset)+sizeof(zskiplist)+sizeof(dict)+
>                   (sizeof(struct dictEntry*)*dictSlots(d))+
>                   zmalloc_size(zsl->header);
>           while(znode != NULL && samples < sample_size) {
>               elesize += sdsZmallocSize(znode->ele);
>               elesize += sizeof(struct dictEntry) + zmalloc_size(znode);
>               samples++;
>               znode = znode->level[0].forward;
>           }
>           if (samples) asize += (double)elesize/samples*dictSize(d);
>       } else {
>           serverPanic("Unknown sorted set encoding");
>       }
>   } else if (o->type == OBJ_HASH) {
>       if (o->encoding == OBJ_ENCODING_ZIPLIST) {
>           asize = sizeof(*o)+(ziplistBlobLen(o->ptr));
>       } else if (o->encoding == OBJ_ENCODING_HT) {
>           d = o->ptr;
>           di = dictGetIterator(d);
>           asize = sizeof(*o)+sizeof(dict)+(sizeof(struct dictEntry*)*dictSlots(d));
>           while((de = dictNext(di)) != NULL && samples < sample_size) {
>               ele = dictGetKey(de);
>               ele2 = dictGetVal(de);
>               elesize += sdsZmallocSize(ele) + sdsZmallocSize(ele2);
>               elesize += sizeof(struct dictEntry);
>               samples++;
>           }
>           dictReleaseIterator(di);
>           if (samples) asize += (double)elesize/samples*dictSize(d);
>       } else {
>           serverPanic("Unknown hash encoding");
>       }
>   } else if (o->type == OBJ_STREAM) {
>       stream *s = o->ptr;
>       asize = sizeof(*o);
>       asize += streamRadixTreeMemoryUsage(s->rax);
>
>       /* Now we have to add the listpacks. The last listpack is often non
>        * complete, so we estimate the size of the first N listpacks, and
>        * use the average to compute the size of the first N-1 listpacks, and
>        * finally add the real size of the last node. */
>       raxIterator ri;
>       raxStart(&ri,s->rax);
>       raxSeek(&ri,"^",NULL,0);
>       size_t lpsize = 0, samples = 0;
>       while(samples < sample_size && raxNext(&ri)) {
>           unsigned char *lp = ri.data;
>           lpsize += lpBytes(lp);
>           samples++;
>       }
>       if (s->rax->numele <= samples) {
>           asize += lpsize;
>       } else {
>           if (samples) lpsize /= samples; /* Compute the average. */
>           asize += lpsize * (s->rax->numele-1);
>           /* No need to check if seek succeeded, we enter this branch only
>            * if there are a few elements in the radix tree. */
>           raxSeek(&ri,"$",NULL,0);
>           raxNext(&ri);
>           asize += lpBytes(ri.data);
>       }
>       raxStop(&ri);
>   
>       /* Consumer groups also have a non trivial memory overhead if there
>        * are many consumers and many groups, let's count at least the
>        * overhead of the pending entries in the groups and consumers
>        * PELs. */
>       if (s->cgroups) {
>           raxStart(&ri,s->cgroups);
>           raxSeek(&ri,"^",NULL,0);
>           while(raxNext(&ri)) {
>               streamCG *cg = ri.data;
>               asize += sizeof(*cg);
>               asize += streamRadixTreeMemoryUsage(cg->pel);
>               asize += sizeof(streamNACK)*raxSize(cg->pel);
>   
>               /* For each consumer we also need to add the basic data
>                * structures and the PEL memory usage. */
>               raxIterator cri;
>               raxStart(&cri,cg->consumers);
>               raxSeek(&cri,"^",NULL,0);
>               while(raxNext(&cri)) {
>                   streamConsumer *consumer = cri.data;
>                   asize += sizeof(*consumer);
>                   asize += sdslen(consumer->name);
>                   asize += streamRadixTreeMemoryUsage(consumer->pel);
>                   /* Don't count NACKs again, they are shared with the
>                    * consumer group PEL. */
>               }
>               raxStop(&cri);
>           }
>           raxStop(&ri);
>       }
>   } else if (o->type == OBJ_MODULE) {
>       moduleValue *mv = o->ptr;
>       moduleType *mt = mv->type;
>       if (mt->mem_usage != NULL) {
>           asize = mt->mem_usage(mv->value);
>       } else {
>           asize = 0;
>       }
>   } else {
>       serverPanic("Unknown object type");
>   }
>   return asize;
> }
> ```
> </details>

| 类型       | 编码                   | 对象                                                        |
| ---------- | ---------------------- | ----------------------------------------------------------- |
| OBJ_STRING | OBJ_ENCODING_INT       | 使用整数值实现的字符串对象                                  |
| OBJ_STRING | OBJ_ENCODING_RAW       | 使用 SDS 实现的字符串对抗                                   |
| OBJ_STRING | OBJ_ENCODING_EMBSTR    | 使用 embstr 编码的 SDS 实现的字符串对象                     |
| OBJ_LIST   | OBJ_ENCODING_QUICKLIST | 使用 quicklist 实现的列表对象                               |
| OBJ_LIST   | OBJ_ENCODING_ZIPLIST   | 使用 ziplist 实现的列表对象                                 |
| OBJ_SET    | OBJ_ENCODING_HT        | 使用字典实现的集合对象                                      |
| OBJ_SET    | OBJ_ENCODING_INTSET    | 使用整数集合实现的集合对象                                  |
| OBJ_ZSET   | OBJ_ENCODING_SKIPLIST  | 使用跳表和字典实现的有序集合对象                            |
| OBJ_ZSET   | OBJ_ENCODING_ZIPLIST   | 使用 ziplist 实现的有序集合对象                             |
| OBJ_HASH   | OBJ_ENCODING_ZIPLIST   | 使用 ziplist 实现的哈希对象                                 |
| OBJ_HASH   | OBJ_ENCODING_HT        | 使用字典实现的哈希对象                                      |
| OBJ_STREAM | 无                     | <font color="red">TODO 这里还没弄清 应该是特殊的实现</font> |
| OBJ_MODULE | 无                     | <font color="red">TODO 这里还没弄清 应该是特殊的实现</font> |

> 使用 `OBJECT ENCODING` 命令可以查看一个数据库键的值对象的编码

## 对象的编码

### 字符串对象

字符串对象的编码可以是 `INT`、`RAW` 或者 `EMBSTR`。

如果一个字符串对象保存的是整数，同时可以用 `long` 类型表示，那么该字符串对象的整数值会保存为 `long` 类型。编码设置为 `int`。否则会当错字符串处理。

> 值的一说的是，浮点数都是被当作字符串处理的。
>
> 在执行某些操作，需要当作浮点数处理的时候。Redis 也提供的单独的命令，如 `INCRBYFLOAT` 等。

如果一个字符串对象保存的是字符串，并且这个字符串值的长度**大于 32** 字节，那么字符串对象的值将使用一个简单动态字符串（`SDS`）存储。编码设置为 `raw`。

如果一个字符串对象保存的是字符串，并且这个字符串值的长度**小于等于 32** 字节，那么字符串对象的值将使用 `embstr` 编码的方式存储。编码设置为 `embstr`。


#### `embstr` 编码

`embstr` 编码是专门用于保存短字符串的一个优化的编码方式，这种编码和 `raw` 一样，都是用 `redisObject` 结构和 `sdshdr` 结构来表示字符串，但是 `raw` 会调用两次内存分配函数分别穿件 `redisObject` 结构和 `sdshdr` 结构，而 `embstr` 编码则通过调用一次内存分配函数来分配一块连续的空间，空间中一次包含 `redisObject` 和 `sdshdr` 两个结构。

```plaintext
   ┌────────────────────────────────────────┬───────────────────────────┐
   │                redisObject             │          sdshdr           │
   ├──────┬──────────┬─────┬─────┬──────────┼─────┬───────┬───────┬─────┤
   │ type │ encoding │ ptr │ lru │ refcount │ len │ alloc │ flags │ buf │
   └──────┴──────────┴─────┴─────┴──────────┴─────┴───────┴───────┴─────┘
                    embstr 编码创建的内存块结构
```

下面分别看下 `embstr` 和 `raw` 编码的字符串对象的创建函数：

```c
/* embstr 编码
 * Create a string object with encoding OBJ_ENCODING_EMBSTR, that is
 * an object where the sds string is actually an unmodifiable string
 * allocated in the same chunk as the object itself. */
robj *createEmbeddedStringObject(const char *ptr, size_t len) {
    robj *o = zmalloc(sizeof(robj)+sizeof(struct sdshdr8)+len+1);   // 只分配了一次内存
                                                                    // ⚠️注意：这里值对象的内存是固定的，所以值不能修改。
    struct sdshdr8 *sh = (void*)(o+1);

    o->type = OBJ_STRING;
    o->encoding = OBJ_ENCODING_EMBSTR;
    o->ptr = sh+1;
    o->refcount = 1;
    if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
        o->lru = (LFUGetTimeInMinutes()<<8) | LFU_INIT_VAL;
    } else {
        o->lru = LRU_CLOCK();
    }

    sh->len = len;
    sh->alloc = len;
    sh->flags = SDS_TYPE_8;
    if (ptr == SDS_NOINIT)
        sh->buf[len] = '\0';
    else if (ptr) {
        memcpy(sh->buf,ptr,len);
        sh->buf[len] = '\0';
    } else {
        memset(sh->buf,0,len+1);
    }
    return o;
}

// raw 编码
/* Create a string object with encoding OBJ_ENCODING_RAW, that is a plain
 * string object where o->ptr points to a proper sds string. */
robj *createRawStringObject(const char *ptr, size_t len) {
    return createObject(OBJ_STRING, sdsnewlen(ptr,len));  // 第一次分配内存给sds。
}
robj *createObject(int type, void *ptr) {
    robj *o = zmalloc(sizeof(*o));                        // 第二次分配内存给redisObject
    o->type = type;
    o->encoding = OBJ_ENCODING_RAW;
    o->ptr = ptr;
    o->refcount = 1;

    /* Set the LRU to the current lruclock (minutes resolution), or
     * alternatively the LFU counter. */
    if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
        o->lru = (LFUGetTimeInMinutes()<<8) | LFU_INIT_VAL;
    } else {
        o->lru = LRU_CLOCK();
    }
    return o;
}
```

embstr 编码的字符串对象在执行命令时，产生的效果和 raw 编码的字符串对象执行命令时产生的效果是相同的，但使用 embstr 编码的字符串对象来保存短字符串值有以下好处：

- embstr 编码将创建字符串对象所需的内存分配次数从 raw 编码的两次降低为一次。
- 释放 embstr 编码的字符串对象只需要调用一次内存释放函数，而释放 raw 编码的字符串对象需要调用两次内存释放函数。
- 因为 embstr 编码的字符串对象的所有数据都保存在一块连续的内存里面，所以这种编码的字符串对象比起 raw 编码的字符串对象能够更好地利用缓存带来的优势。

⚠️ 注意：这里值对象的内存是固定的，同时 Redis 本身没有编写 `embstr` 编码的字符串的修改程序，所以它是只读的。

---

Redis 的字符串对象的众多命令中，可能会出现当前编码不满足的情况。

比如 `int` 编码下的字符串对象被执行 `INCR` 命令都是正常的，但如果被执行 `APPEND` 命令，就会出现问题。整数从语义上也没有追加的操作。类似这种情况会将该字符串转化为 `raw` 编码进行处理。（⚠️ 注意：这种编码转换行为，即使处理后的编码**小于等于 32** 也不会采用 `embstr` 编码。）

```
> set int 1
OK
> OBJECT encoding int
int
> APPEND int 5
2
> get int
15
> OBJECT encoding int
raw
```

`embstr` 类型的转换也是只能转成 `raw`。如上面所说，`embstr` 是只读的，所以对 embstr 编码的字符串对象执行任何修改命令时，程序会先将对象的编码从 embstr 转换成 raw，然后再执行修改命令。

### 字符串对象的命令

| 命令        | `int` 编码的实现方法                                                                                           | `embstr` 编码的实现方法                                                                                                                                                            | `raw` 编码的实现方法                                                                                                                                                             |
| ----------- | -------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| SET         | 使用 `int` 编码保存值。                                                                                        | 使用 `embstr` 编码保存值。                                                                                                                                                         | 使用 `raw` 编码保存值。                                                                                                                                                          |
| GET         | 拷贝对象所保存的整数值， 将这个拷贝转换成字符串值， 然后向客户端返回这个字符串值。                             | 直接向客户端返回字符串值。                                                                                                                                                         | 直接向客户端返回字符串值。                                                                                                                                                       |
| APPEND      | 将对象转换成 `raw` 编码， 然后按 `raw` 编码的方式执行此操作。                                                  | 将对象转换成 `raw` 编码， 然后按 `raw` 编码的方式执行此操作。                                                                                                                      | 调用 `sdscatlen` 函数， 将给定字符串追加到现有字符串的末尾。                                                                                                                     |
| INCRBYFLOAT | 取出整数值并将其转换成 `longdouble` 类型的浮点数， 对这个浮点数进行加法计算， 然后将得出的浮点数结果保存起来。 | 取出字符串值并尝试将其转换成 `long double` 类型的浮点数， 对这个浮点数进行加法计算， 然后将得出的浮点数结果保存起来。 如果字符串值不能被转换成浮点数， 那么向客户端返回一个错误。 | 取出字符串值并尝试将其转换成 `longdouble` 类型的浮点数， 对这个浮点数进行加法计算， 然后将得出的浮点数结果保存起来。 如果字符串值不能被转换成浮点数， 那么向客户端返回一个错误。 |
| INCRBY      | 对整数值进行加法计算， 得出的计算结果会作为整数被保存起来。                                                    | `embstr` 编码不能执行此命令， 向客户端返回一个错误。                                                                                                                               | `raw` 编码不能执行此命令， 向客户端返回一个错误。                                                                                                                                |
| DECRBY      | 对整数值进行减法计算， 得出的计算结果会作为整数被保存起来。                                                    | `embstr` 编码不能执行此命令， 向客户端返回一个错误。                                                                                                                               | `raw` 编码不能执行此命令， 向客户端返回一个错误。                                                                                                                                |
| STRLEN      | 拷贝对象所保存的整数值， 将这个拷贝转换成字符串值， 计算并返回这个字符串值的长度。                             | 调用 `sdslen` 函数， 返回字符串的长度。                                                                                                                                            | 调用 `sdslen` 函数， 返回字符串的长度。                                                                                                                                          |
| SETRANGE    | 将对象转换成 `raw` 编码， 然后按 `raw` 编码的方式执行此命令。                                                  | 将对象转换成 `raw` 编码， 然后按 `raw` 编码的方式执行此命令。                                                                                                                      | 将字符串特定索引上的值设置为给定的字符。                                                                                                                                         |
| GETRANGE    | 拷贝对象所保存的整数值， 将这个拷贝转换成字符串值， 然后取出并返回字符串指定索引上的字符。                     | 直接取出并返回字符串指定索引上的字符。                                                                                                                                             | 直接取出并返回字符串指定索引上的字符。                                                                                                                                           |

### 列表对象


列表对象的编码可以是 `ziplist` 或者 `linkedlist` 。

`ziplist` 编码的列表对象使用压缩列表作为底层实现， 每个压缩列表节点（entry）保存了一个列表元素。

另一方面， `linkedlist` 编码的列表对象使用双端链表作为底层实现， 每个双端链表节点（node）都保存了一个字符串对象， 而每个字符串对象都保存了一个列表元素。

### 哈希对象


### 集合对象

### 有序集合对象


## 类型检查和命令多态


## 内存回收


## 对象共享


## 对象的空转时长


## 总结