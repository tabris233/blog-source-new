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

#### 字符串对象的命令

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

#### 编码转换

当列表对象可以同时满足以下两个条件时， 列表对象使用 `ziplist` 编码：

1.  列表对象保存的所有字符串元素的长度都小于 `64` 字节；
2.  列表对象保存的元素数量小于 `512` 个；

不能满足这两个条件的列表对象需要使用 `linkedlist` 编码。

⚠️注意：以上两个条件的上限值是可以修改的， 具体请看配置文件中关于 `list-max-ziplist-value` 选项和 `list-max-ziplist-entries` 选项的说明。

#### 列表对象的命令

>   仅列举常用命令。

| 命令    | `ziplist` 编码的实现方法                                     | `linkedlist` 编码的实现方法                                  |
| :------ | :----------------------------------------------------------- | :----------------------------------------------------------- |
| LPUSH   | 调用 `ziplistPush` 函数， 将新元素推入到压缩列表的表头。     | 调用 `listAddNodeHead` 函数， 将新元素推入到双端链表的表头。 |
| RPUSH   | 调用 `ziplistPush` 函数， 将新元素推入到压缩列表的表尾。     | 调用 `listAddNodeTail` 函数， 将新元素推入到双端链表的表尾。 |
| LPOP    | 调用 `ziplistIndex` 函数定位压缩列表的表头节点， 在向用户返回节点所保存的元素之后， 调用`ziplistDelete` 函数删除表头节点。 | 调用 `listFirst` 函数定位双端链表的表头节点， 在向用户返回节点所保存的元素之后， 调用 `listDelNode` 函数删除表头节点。 |
| RPOP    | 调用 `ziplistIndex` 函数定位压缩列表的表尾节点， 在向用户返回节点所保存的元素之后， 调用`ziplistDelete` 函数删除表尾节点。 | 调用 `listLast` 函数定位双端链表的表尾节点， 在向用户返回节点所保存的元素之后， 调用 `listDelNode` 函数删除表尾节点。 |
| LINDEX  | 调用 `ziplistIndex` 函数定位压缩列表中的指定节点， 然后返回节点所保存的元素。 | 调用 `listIndex` 函数定位双端链表中的指定节点， 然后返回节点所保存的元素。 |
| LLEN    | 调用 `ziplistLen` 函数返回压缩列表的长度。                   | 调用 `listLength` 函数返回双端链表的长度。                   |
| LINSERT | 插入新节点到压缩列表的表头或者表尾时， 使用`ziplistPush` 函数； 插入新节点到压缩列表的其他位置时， 使用 `ziplistInsert` 函数。 | 调用 `listInsertNode` 函数， 将新节点插入到双端链表的指定位置。 |
| LREM    | 遍历压缩列表节点， 并调用 `ziplistDelete` 函数删除包含了给定元素的节点。 | 遍历双端链表节点， 并调用 `listDelNode` 函数删除包含了给定元素的节点。 |
| LTRIM   | 调用 `ziplistDeleteRange` 函数， 删除压缩列表中所有不在指定索引范围内的节点。 | 遍历双端链表节点， 并调用 `listDelNode` 函数删除链表中所有不在指定索引范围内的节点。 |
| LSET    | 调用 `ziplistDelete` 函数， 先删除压缩列表指定索引上的现有节点， 然后调用 `ziplistInsert` 函数， 将一个包含给定元素的新节点插入到相同索引上面。 | 调用 `listIndex` 函数， 定位到双端链表指定索引上的节点， 然后通过赋值操作更新节点的值。 |

### 哈希对象

哈希对象的编码可以是 `ziplist` 或者 `hashtable` 。

`ziplist` 编码的哈希对象使用压缩列表作为底层实现， 每当有新的键值对要加入到哈希对象时， 程序会先将保存了键的压缩列表节点推入到压缩列表表尾， 然后再将保存了值的压缩列表节点推入到压缩列表表尾， 因此：

-   保存了同一键值对的两个节点总是紧挨在一起， 保存键的节点在前， 保存值的节点在后；
-   先添加到哈希对象中的键值对会被放在压缩列表的表头方向， 而后来添加到哈希对象中的键值对会被放在压缩列表的表尾方向。

```c
// ziplist 编码
      ┌─────────┬────────┬───────┬────────┬───────┬───────┬────┬──────────┬──────────────┬───────┐
      │ zlbytes │ zltail │ zllen │ "name" │ "Tom" │ "age" │ 25 │ "career" │ "Programmer" │ zlend │
      └─────────┴────────┴───────┴────────┴───────┴───────┴────┴──────────┴──────────────┴───────┘
                                     ▲        ▲       ▲      ▲       ▲            ▲  
                                     │        │       │      │       │            │
                                     │        │       │      │       │            │
                                    key     value    key   value    key         value
                                     │        │       │      │       │            │
                                     └   ─────┘       └─   ──┘       └───────    ─┘
                                     第一个键值对      第二个键值对        第三个键值对
```

`hashtable` 编码的哈希对象使用字典作为底层实现， 哈希对象中的每个键值对都使用一个字典键值对来保存：

-   字典的每个键都是一个字符串对象， 对象中保存了键值对的键；
-   字典的每个值都是一个字符串对象， 对象中保存了键值对的值。

```c
// hashtable 编码
  ┌────────────┐
  │    dict    │     ┌────────────┐
  ├────────────┤  ┌─►│StringObject│
  │StringObject│  │  │     25     │
  │   "age "   ├──┘  └────────────┘
  ├────────────┤     ┌────────────┐
  │StringObject├────►│StringObject│
  │  "career"  │     │"Programmer"│
  ├────────────┤     └────────────┘
  │StringObject├──┐  ┌────────────┐
  │   "name"   │  └─►│StringObject│
  └────────────┘     │   "Tom"    │
                     └────────────┘
```

#### 编码转换

当哈希对象可以同时满足以下两个条件时， 哈希对象使用 `ziplist` 编码：

1.  哈希对象保存的所有键值对的键和值的字符串长度都小于 `64` 字节；
2.  哈希对象保存的键值对数量小于 `512` 个；

不能满足这两个条件的哈希对象需要使用 `hashtable` 编码。

⚠️注意：这两个条件的上限值是可以修改的， 具体请看配置文件中关于 `hash-max-ziplist-value` 选项和 `hash-max-ziplist-entries` 选项的说明。

#### 哈希命令的实现

>   仅列举常用命令。

| 命令    | `ziplist` 编码实现方法                                       | `hashtable` 编码的实现方法                                   |
| :------ | :----------------------------------------------------------- | :----------------------------------------------------------- |
| HSET    | 首先调用 `ziplistPush` 函数， 将键推入到压缩列表的表尾， 然后再次调用 `ziplistPush` 函数， 将值推入到压缩列表的表尾。 | 调用 `dictAdd` 函数， 将新节点添加到字典里面。               |
| HGET    | 首先调用 `ziplistFind` 函数， 在压缩列表中查找指定键所对应的节点， 然后调用 `ziplistNext` 函数， 将指针移动到键节点旁边的值节点， 最后返回值节点。 | 调用 `dictFind` 函数， 在字典中查找给定键， 然后调用`dictGetVal` 函数， 返回该键所对应的值。 |
| HEXISTS | 调用 `ziplistFind` 函数， 在压缩列表中查找指定键所对应的节点， 如果找到的话说明键值对存在， 没找到的话就说明键值对不存在。 | 调用 `dictFind` 函数， 在字典中查找给定键， 如果找到的话说明键值对存在， 没找到的话就说明键值对不存在。 |
| HDEL    | 调用 `ziplistFind` 函数， 在压缩列表中查找指定键所对应的节点， 然后将相应的键节点、 以及键节点旁边的值节点都删除掉。 | 调用 `dictDelete` 函数， 将指定键所对应的键值对从字典中删除掉。 |
| HLEN    | 调用 `ziplistLen` 函数， 取得压缩列表包含节点的总数量， 将这个数量除以 `2` ， 得出的结果就是压缩列表保存的键值对的数量。 | 调用 `dictSize` 函数， 返回字典包含的键值对数量， 这个数量就是哈希对象包含的键值对数量。 |
| HGETALL | 遍历整个压缩列表， 用 `ziplistGet` 函数返回所有键和值（都是节点）。 | 遍历整个字典， 用 `dictGetKey` 函数返回字典的键， 用`dictGetVal` 函数返回字典的值。 |



### 集合对象

集合对象的编码可以是 `intset` 或者 `hashtable` 。

---

`intset` 编码的集合对象使用整数集合作为底层实现， 集合对象包含的所有元素都被保存在整数集合里面。

```c
// intset 编码
┌────────────────┐
│     intset     │
├────────────────┤
│    encoding    │
│INTSET_ENC_INT16│
├────────────────┤
│     length     │
│       3        │
├────────────────┤        ┌───┬───┬───┐
│    contents    ├───────►│ 1 │ 3 │ 5 │
└────────────────┘        └───┴───┴───┘
```

---

`hashtable` 编码的集合对象使用字典作为底层实现， 字典的每个键都是一个字符串对象， 每个字符串对象包含了一个集合元素， 而字典的值则全部被设置为 `NULL` 。

```c
// hashtable 编码
  ┌────────────┐
  │    dict    │
  ├────────────┤
  │StringObject├──────► NULL
  │   "age"    │
  ├────────────┤
  │StringObject├──────► NULL
  │  "career"  │
  ├────────────┤
  │StringObject├──────► NULL
  │   "name"   │
  └────────────┘
```

#### 编码的转换

当集合对象可以同时满足以下两个条件时， 对象使用 `intset` 编码：

1.  集合对象保存的所有元素都是整数值；
2.  集合对象保存的元素数量不超过 `512` 个；

不能满足这两个条件的集合对象需要使用 `hashtable` 编码。

⚠️注意：第二个条件的上限值是可以修改的， 具体请看配置文件中关于 `set-max-intset-entries` 选项的说明。

#### 集合命令的实现

>   仅列举常用命令。

| 命令        | `intset` 编码的实现方法                                      | `hashtable` 编码的实现方法                                   |
| :---------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| SADD        | 调用 `intsetAdd` 函数， 将所有新元素添加到整数集合里面。     | 调用 `dictAdd` ， 以新元素为键， `NULL` 为值， 将键值对添加到字典里面。 |
| SCARD       | 调用 `intsetLen` 函数， 返回整数集合所包含的元素数量， 这个数量就是集合对象所包含的元素数量。 | 调用 `dictSize` 函数， 返回字典所包含的键值对数量， 这个数量就是集合对象所包含的元素数量。 |
| SISMEMBER   | 调用 `intsetFind` 函数， 在整数集合中查找给定的元素， 如果找到了说明元素存在于集合， 没找到则说明元素不存在于集合。 | 调用 `dictFind` 函数， 在字典的键中查找给定的元素， 如果找到了说明元素存在于集合， 没找到则说明元素不存在于集合。 |
| SMEMBERS    | 遍历整个整数集合， 使用 `intsetGet` 函数返回集合元素。       | 遍历整个字典， 使用 `dictGetKey` 函数返回字典的键作为集合元素。 |
| SRANDMEMBER | 调用 `intsetRandom` 函数， 从整数集合中随机返回一个元素。    | 调用 `dictGetRandomKey` 函数， 从字典中随机返回一个字典键。  |
| SPOP        | 调用 `intsetRandom` 函数， 从整数集合中随机取出一个元素， 在将这个随机元素返回给客户端之后， 调用 `intsetRemove` 函数， 将随机元素从整数集合中删除掉。 | 调用 `dictGetRandomKey` 函数， 从字典中随机取出一个字典键， 在将这个随机字典键的值返回给客户端之后， 调用`dictDelete` 函数， 从字典中删除随机字典键所对应的键值对。 |
| SREM        | 调用 `intsetRemove` 函数， 从整数集合中删除所有给定的元素。  | 调用 `dictDelete` 函数， 从字典中删除所有键为给定元素的键值对。 |

### 有序集合对象

有序集合的编码可以是 `ziplist` 或者 `skiplist` 。

---

`ziplist` 编码的有序集合对象使用压缩列表作为底层实现， 每个集合元素使用两个紧挨在一起的压缩列表节点来保存， 第一个节点保存元素的成员（member）， 而第二个元素则保存元素的分值（score）。（这里和`ziplist`表示`HASH`对象有些类似）

压缩列表内的集合元素按分值从小到大进行排序， 分值较小的元素被放置在靠近表头的方向， 而分值较大的元素则被放置在靠近表尾的方向。

```c
// ziplist 编码
      ┌─────────┬────────┬───────┬──────────┬───────┬──────────┬─────┬─────────┬─────┬───────┐
      │ zlbytes │ zltail │ zllen │ "banana" │  5.0  │ "cherry" │ 6.0 │ "apple" │ 8.5 │ zlend │
      └─────────┴────────┴───────┴──────────┴───────┴──────────┴─────┴─────────┴─────┴───────┘
                                     ▲          ▲        ▲        ▲       ▲        ▲  
                                     │          │        │        │       │        │
                                     │          │        │        │       │        │
                                    key       value     key     value    key     value
                                     │          │        │        │       │        │
                                     └───   ────┘        └─     ──┘       └──    ──┘
                                     分值最少的元素       分值排第二的元素     分值最大的元素
```

---

`skiplist` 编码的有序集合对象使用 `zset` 结构作为底层实现， 一个 `zset` 结构同时包含一个字典和一个跳跃表：

```c
typedef struct zset {
    dict *dict;
    zskiplist *zsl;
} zset;
```

`zset`中的`zsl`成员是一个跳表，按照`scrore`从小到大保存了所有的集合元素，跳表节点的`object`属性保存了元素的成员，`score`属性保存了元素的分值。通过这个跳表，程序可以对有序集合进行范围操作，比如`ZRANGE`，`ZRANK`等。

`zset`中的`dict`成员为有序集合存储了从`member`到`score`的映射。通过这个字典，程序可以用$O(1)$的复杂度查找给定成员的分值，`ZSCORE`命令就是如此实现的，同时很多其他有序集合的实现也是通过它来实现的。

 值得一提的是， 虽然 `zset` 结构同时使用跳表和字典来保存有序集合元素， 但这两种数据结构都会通过指针来共享相同元素的成员和分值， 所以同时使用跳表和字典来保存集合元素不会产生任何重复成员或者分值， 也不会因此而浪费额外的内存。

>   为什么有序集合要用两种底层数据结构实现呢？
>
>   A：如果有序集合采用跳表或字典单独实现，那么性能会降低很多。
>
>   -   如果只使用跳表
>       1.  查询特定成员的分数值需要遍历整个跳表，复杂度为$O(N)$。 用字典之后复杂度降为$O(1)$
>   -   如果只使用字典
>       1.  字典中不会按score顺序存储元素的，那么对于`ZRANK`，`ZRANGE`等需要根据`score`来查找的命令时需要对字典排序。复杂度变成$O(Nlog(N))$.

```c
// zset 编码
                                              ┌────────┐
                                        ┌────►│ dictht │
                                        │     ├────────┤             ┌────────────┐
                                        │     │  ...   │             │StringObject├──────► 5.0
                        ┌───────┐       │     ├────────┤             │ "banana"   │
                 ┌─────►│  dict │       │     │ table  ├────────────►├────────────┤
                 │      ├───────┤       │     ├────────┤             │StringObject├──────► 6.0
                 │      │  ...  │       │     │  ...   │             │  "cherry" │
                 │      ├───────┤       │     └────────┘             ├────────────┤
                 │      │ ht[0] ├───────┘                            │StringObject├──────► 8.5
                 │      ├───────┤                                    │  "apple"   │
                 │      │  ...  │                                    └────────────┘
┌────────────┐   │      └───────┘
│    zset    │   │
├────────────┤   │
│    dict    ├───┘
├────────────┤
│    zslle   ├───┐
└────────────┘   │                       ┌─────┐
                 │      ┌────────┐       │ L32 ├─────────► NULL
                 └─────►│ header ├──────►├─────┤
                        ├────────┤       │ ... │
                        │  tail  ├──┐    ├─────┤                                                            ┌──────────────┐
                        ├────────┤  │    │ L5  ├───────────────────────────────────────────────────────────►│      L5      ├─────► NULL
                        │ level  │  │    ├─────┤       ┌──────────────┐                                     ├──────────────┤
                        │   5    │  │    │ L4  ├──────►│      L4      ├────────────────────────────────────►│      L4      ├─────► NULL
                        ├────────┤  │    ├─────┤       ├──────────────┤                                     ├──────────────┤
                        │ length │  │    │ L3  ├──────►│      L3      ├────────────────────────────────────►│      L3      ├─────► NULL
                        │   3    │  │    ├─────┤       ├──────────────┤          ┌──────────────┐           ├──────────────┤
                        └────────┘  │    │ L2  ├──────►│      L2      ├─────────►│      L2      ├──────────►│      L2      ├─────► NULL
                                    │    ├─────┤       ├──────────────┤          ├──────────────┤           ├──────────────┤
                                    │    │ L1  ├──────►│      L1      ├─────────►│      L1      ├──────────►│      L1      ├─────► NULL
                                    │    └─────┘       ├──────────────┤          ├──────────────┤           ├──────────────┤
                                    │            ┌─────┤      BW      │◄─────────┤      BW      │◄──────────┤      BW      │
                                    │            ▼     ├──────────────┤          ├──────────────┤           ├──────────────┤
                                    │           NULL   │      5.0     │          │      6.0     │           │      8.5     │
                                    │                  ├──────────────┤          ├──────────────┤    ┌─────►├──────────────┤
                                    │                  │ StringObject │          │ StringObject │    │      │ StringObject │
                                    │                  │   "banana"   │          │   "cherry"   │    │      │   "apple"    │
                                    │                  └──────────────┘          └──────────────┘    │      └──────────────┘
                                    │                                                                │
                                    │                                                                │
                                    │                                                                │
                                    └────────────────────────────────────────────────────────────────┘
```

#### 编码的转换

当有序集合对象可以同时满足以下两个条件时， 对象使用 `ziplist` 编码：

1.  有序集合保存的元素数量小于 `128` 个；
2.  有序集合保存的所有元素成员的长度都小于 `64` 字节；

不能满足以上两个条件的有序集合对象将使用 `skiplist` 编码。

⚠️注意：以上两个条件的上限值是可以修改的， 具体请看配置文件中关于 `zset-max-ziplist-entries` 选项和 `zset-max-ziplist-value` 选项的说明。

#### 有序集合命令的实现

>   仅列举常用命令。

| 命令      | `ziplist` 编码的实现方法                                     | `zset` 编码的实现方法                                        |
| :-------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| ZADD      | 调用 `ziplistInsert` 函数， 将成员和分值作为两个节点分别插入到压缩列表。 | 先调用 `zslInsert` 函数， 将新元素添加到跳跃表， 然后调用 `dictAdd` 函数， 将新元素关联到字典。 |
| ZCARD     | 调用 `ziplistLen` 函数， 获得压缩列表包含节点的数量， 将这个数量除以 `2` 得出集合元素的数量。 | 访问跳跃表数据结构的 `length` 属性， 直接返回集合元素的数量。 |
| ZCOUNT    | 遍历压缩列表， 统计分值在给定范围内的节点的数量。            | 遍历跳跃表， 统计分值在给定范围内的节点的数量。              |
| ZRANGE    | 从表头向表尾遍历压缩列表， 返回给定索引范围内的所有元素。    | 从表头向表尾遍历跳跃表， 返回给定索引范围内的所有元素。      |
| ZREVRANGE | 从表尾向表头遍历压缩列表， 返回给定索引范围内的所有元素。    | 从表尾向表头遍历跳跃表， 返回给定索引范围内的所有元素。      |
| ZRANK     | 从表头向表尾遍历压缩列表， 查找给定的成员， 沿途记录经过节点的数量， 当找到给定成员之后， 途经节点的数量就是该成员所对应元素的排名。 | 从表头向表尾遍历跳跃表， 查找给定的成员， 沿途记录经过节点的数量， 当找到给定成员之后， 途经节点的数量就是该成员所对应元素的排名。 |
| ZREVRANK  | 从表尾向表头遍历压缩列表， 查找给定的成员， 沿途记录经过节点的数量， 当找到给定成员之后， 途经节点的数量就是该成员所对应元素的排名。 | 从表尾向表头遍历跳跃表， 查找给定的成员， 沿途记录经过节点的数量， 当找到给定成员之后， 途经节点的数量就是该成员所对应元素的排名。 |
| ZREM      | 遍历压缩列表， 删除所有包含给定成员的节点， 以及被删除成员节点旁边的分值节点。 | 遍历跳跃表， 删除所有包含了给定成员的跳跃表节点。 并在字典中解除被删除元素的成员和分值的关联。 |
| ZSCORE    | 遍历压缩列表， 查找包含了给定成员的节点， 然后取出成员节点旁边的分值节点保存的元素分值。 | 直接从字典中取出给定成员的分值。                             |

## 类型检查和命令多态

`Redis` 中用于操作键的命令基本上可以分为两种类型。

1.  可以对任何类型的键执行

    >   比如说 `DEL` 命令、 `EXPIRE` 命令、 `RENAME` 命令、 `TYPE` 命令、 `OBJECT` 命令， 等等。

2.  只能对特定类型的键执行

    >    比如说：
    >
    >   -   `SET` 、 `GET` 、 `APPEND` 、 `STRLEN` 等命令只能对字符串键执行；
    >   -   `HDEL` 、 `HSET` 、 `HGET` 、 `HLEN` 等命令只能对哈希键执行；
    >   -   `RPUSH` 、 `LPOP` 、 `LINSERT` 、 `LLEN` 等命令只能对列表键执行；
    >   -   `SADD` 、 `SPOP` 、 `SINTER` 、 `SCARD` 等命令只能对集合键执行；
    >   -   `ZADD` 、 `ZCARD` 、 `ZRANK` 、 `ZSCORE` 等命令只能对有序集合键执行；



#### 类型检查的实现

为了确保只有指定类型的键可以执行某些特定的命令， 在执行一个类型特定的命令之前， Redis 会先检查输入键的类型是否正确， 然后再决定是否执行给定的命令。

类型特定命令所进行的类型检查是通过 `redisObject` 结构的 `type` 属性来实现的：

-   在执行一个类型特定命令之前， 服务器会先检查输入数据库键的值对象是否为执行命令所需的类型， 如果是的话， 服务器就对键执行指定的命令；
-   否则， 服务器将拒绝执行命令， 并向客户端返回一个类型错误。

举个例子， 对于 `LLEN` 命令来说：

-   在执行 `LLEN` 命令之前， 服务器会先检查输入数据库键的值对象是否为列表类型， 也即是， 检查值对象 `redisObject` 结构 `type` 属性的值是否为 `REDIS_LIST` ， 如果是的话， 服务器就对键执行 `LLEN` 命令；
-   否则的话， 服务器就拒绝执行命令并向客户端返回一个类型错误；

下图展示了这一类型检查过程。

<img src="https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/05/29/20210529213124.png" alt="image-20210529213124492" style="zoom:50%;" />

其他类型特定命令的类型检查过程也和这里展示的 `LLEN` 命令的类型检查过程类似。

#### 多态命令的实现



Redis 除了会根据值对象的类型来判断键是否能够执行指定命令之外， 还会根据值对象的编码方式， 选择正确的命令实现代码来执行命令。

举个例子， 在前面介绍列表对象的编码时我们说过， 列表对象有 `ziplist` 和 `linkedlist` 两种编码可用， 其中前者使用压缩列表 API 来实现列表命令， 而后者则使用双端链表 API 来实现列表命令。

现在， 考虑这样一个情况， 如果我们对一个键执行 LLEN 命令， 那么服务器除了要确保执行命令的是列表键之外， 还需要根据键的值对象所使用的编码来选择正确的 LLEN 命令实现：

-   如果列表对象的编码为 `ziplist` ， 那么说明列表对象的实现为压缩列表， 程序将使用 `ziplistLen` 函数来返回列表的长度；
-   如果列表对象的编码为 `linkedlist` ， 那么说明列表对象的实现为双端链表， 程序将使用 `listLength` 函数来返回双端链表的长度；

借用面向对象方面的术语来说， 我们可以认为 LLEN 命令是多态（[polymorphism](http://en.wikipedia.org/wiki/Polymorphism_(computer_science))）的： 只要执行 LLEN 命令的是列表键， 那么无论值对象使用的是 `ziplist` 编码还是 `linkedlist` 编码， 命令都可以正常执行。

下图展示了 LLEN 命令从类型检查到根据编码选择实现函数的整个执行过程， 其他类型特定命令的执行过程也是类似的。

<img src="https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/05/29/20210529213308.png" alt="image-20210529213308665" style="zoom:50%;" />

实际上， 我们可以将 DEL 、 EXPIRE 、 TYPE 等命令也称为多态命令， 因为无论输入的键是什么类型， 这些命令都可以正确地执行。

DEL 、 EXPIRE 等命令和 LLEN 等命令的区别在于， 前者是基于类型的多态 —— 一个命令可以同时用于处理多种不同类型的键， 而后者是基于编码的多态 —— 一个命令可以同时用于处理多种不同编码。

## 内存回收

>   Redis的内存回收类似智能指针维护的那种，通过引用计数实现。

因为 C 语言并不具备自动的内存回收功能， 所以 Redis 在自己的对象系统中构建了一个引用计数（[reference counting](http://en.wikipedia.org/wiki/Reference_counting)）技术实现的内存回收机制， 通过这一机制， 程序可以通过跟踪对象的引用计数信息， 在适当的时候自动释放对象并进行内存回收。

每个对象的引用计数信息由 `redisObject` 结构的 `refcount` 属性记录：

```c
typedef struct redisObject {
    // ...
    // 引用计数
    int refcount;
    // ...
} robj;
```

对象的引用计数信息会随着对象的使用状态而不断变化：

-   在创建一个新对象时， 引用计数的值会被初始化为 `1` ；
-   当对象被一个新程序使用时， 它的引用计数值会被增一；
-   当对象不再被一个程序使用时， 它的引用计数值会被减一；
-   当对象的引用计数值变为 `0` 时， 对象所占用的内存会被释放。

------

修改对象引用计数的 API

| 函数            | 作用                                                         |
| :-------------- | :----------------------------------------------------------- |
| `incrRefCount`  | 将对象的引用计数值增一。                                     |
| `decrRefCount`  | 将对象的引用计数值减一， 当对象的引用计数值等于 `0` 时， 释放对象。 |
| `resetRefCount` | 将对象的引用计数值设置为 `0` ， 但并不释放对象， 这个函数通常在需要重新设置对象的引用计数值时使用。 |

------

对象的整个生命周期可以划分为创建对象、操作对象、释放对象三个阶段。

作为例子， 以下代码展示了一个字符串对象从创建到释放的整个过程：

```c
// 创建一个字符串对象 s ，对象的引用计数为 1
robj *s = createStringObject(...)

// 对象 s 执行各种操作 ...

// 将对象 s 的引用计数减一，使得对象的引用计数变为 0
// 导致对象 s 被释放
decrRefCount(s)
```

其他不同类型的对象也会经历类似的过程。

## 对象共享

除了用于实现引用计数内存回收机制之外， 对象的引用计数属性还带有对象共享的作用。

举个例子， 假设键 A 创建了一个包含整数值 `100` 的字符串对象作为值对象， 如下图所示。

<img src="https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/05/29/20210529212403.png" alt="image-20210529212235826" style="zoom:50%;" />

如果这时键 B 也要创建一个同样保存了整数值 `100` 的字符串对象作为值对象， 那么服务器有以下两种做法：

1.  为键 B 新创建一个包含整数值 `100` 的字符串对象；
2.  让键 A 和键 B 共享同一个字符串对象；

以上两种方法很明显是第二种方法更节约内存。

在 Redis 中， 让多个键共享同一个值对象需要执行以下两个步骤：

1.  将数据库键的值指针指向一个现有的值对象；
2.  将被共享的值对象的引用计数增一。

举个例子， 下图就展示了包含整数值 `100` 的字符串对象同时被键 A 和键 B 共享之后的样子， 可以看到， 除了对象的引用计数从之前的 `1` 变成了 `2` 之外， 其他属性都没有变化。

<img src="https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/05/29/20210529212430.png" alt="image-20210529212430327" style="zoom:50%;" />

共享对象机制对于节约内存非常有帮助， 数据库中保存的相同值对象越多， 对象共享机制就能节约越多的内存。

比如说， 假设数据库中保存了整数值 `100` 的键不只有键 A 和键 B 两个， 而是有一百个， 那么服务器只需要用一个字符串对象的内存就可以保存原本需要使用一百个字符串对象的内存才能保存的数据。

目前来说， Redis 会在初始化服务器时， 创建一万个字符串对象， 这些对象包含了从 `0` 到 `9999` 的所有整数值， 当服务器需要用到值为 `0`到 `9999` 的字符串对象时， 服务器就会使用这些共享对象， 而不是新创建对象。

注意

创建共享字符串对象的数量可以通过修改 `server.h/REDIS_SHARED_INTEGERS` 常量来修改。

举个例子， 如果我们创建一个值为 `100` 的键 `A` ， 并使用 OBJECT REFCOUNT 命令查看键 `A` 的值对象的引用计数， 我们会发现值对象的引用计数为 `2` ：

```c
redis> SET A 100
OK

redis> OBJECT REFCOUNT A
(integer) 2
```

引用这个值对象的两个程序分别是持有这个值对象的服务器程序， 以及共享这个值对象的键 `A` ， 如下图所示。

<img src="https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/05/29/20210529212508.png" alt="image-20210529212505012" style="zoom:50%;" />

如果这时我们再创建一个值为 `100` 的键 `B` ， 那么键 `B` 也会指向包含整数值 `100` 的共享对象， 使得共享对象的引用计数值变为 `3` ：

```c
redis> SET B 100
OK

redis> OBJECT REFCOUNT A
(integer) 3

redis> OBJECT REFCOUNT B
(integer) 3
```

下图展示了共享值对象的三个程序。

<img src="https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/05/29/20210529212531.png" alt="image-20210529212531180" style="zoom:50%;" />

另外， 这些共享对象不单单只有字符串键可以使用， 那些在数据结构中嵌套了字符串对象的对象（`linkedlist` 编码的列表对象、 `hashtable` 编码的哈希对象、 `hashtable` 编码的集合对象、以及 `zset` 编码的有序集合对象）都可以使用这些共享对象。

为什么 Redis 不共享包含字符串的对象？

当服务器考虑将一个共享对象设置为键的值对象时， 程序需要先检查给定的共享对象和键想创建的目标对象是否完全相同， 只有在共享对象和目标对象完全相同的情况下， 程序才会将共享对象用作键的值对象， 而一个共享对象保存的值越复杂， 验证共享对象和目标对象是否相同所需的复杂度就会越高， 消耗的 CPU 时间也会越多：

-   如果共享对象是保存整数值的字符串对象， 那么验证操作的复杂度为 $O(1)$ ；
-   如果共享对象是保存字符串值的字符串对象， 那么验证操作的复杂度为 $O(N)$ ；
-   如果共享对象是包含了多个值（或者对象的）对象， 比如列表对象或者哈希对象， 那么验证操作的复杂度将会是 $O(N^2)$  。

因此， 尽管共享更复杂的对象可以节约更多的内存， 但受到 CPU 时间的限制， Redis 只对包含整数值的字符串对象进行共享。

## 对象的空转时长

除了前面介绍过的 `type` 、 `encoding` 、 `ptr` 和 `refcount` 四个属性之外， `redisObject` 结构包含的最后一个属性为 `lru` 属性， 该属性记录了对象最后一次被命令程序访问的时间：

```c
typedef struct redisObject {
    // ...
    unsigned lru:22;
    // ...
} robj;
```

`OBJECT IDLETIME` 命令可以打印出给定键的空转时长， 这一空转时长就是通过将当前时间减去键的值对象的 `lru` 时间计算得出的：

```c
redis> SET msg "hello world"
OK

# 等待一小段时间
redis> OBJECT IDLETIME msg
(integer) 20

# 等待一阵子
redis> OBJECT IDLETIME msg
(integer) 180

# 访问 msg 键的值
redis> GET msg
"hello world"

# 键处于活跃状态，空转时长为 0
redis> OBJECT IDLETIME msg
(integer) 0
```

⚠️注意:

`OBJECT IDLETIME` 命令的实现是特殊的， 这个命令在访问键的值对象时， 不会修改值对象的 `lru` 属性。

除了可以被 OBJECT IDLETIME 命令打印出来之外， 键的空转时长还有另外一项作用： 如果服务器打开了 `maxmemory` 选项， 并且服务器用于回收内存的算法为 `volatile-lru` 或者 `allkeys-lru` ， 那么当服务器占用的内存数超过了 `maxmemory` 选项所设置的上限值时， 空转时长较高的那部分键会优先被服务器释放， 从而回收内存。

配置文件的 `maxmemory` 选项和 `maxmemory-policy` 选项的说明介绍了关于这方面的更多信息。

## 总结

-   Redis 数据库中的每个键值对的键和值都是一个对象。
-   Redis 共有字符串、列表、哈希、集合、有序集合五种类型的对象， 每种类型的对象至少都有两种或以上的编码方式， 不同的编码可以在不同的使用场景上优化对象的使用效率。
-   服务器在执行某些命令之前， 会先检查给定键的类型能否执行指定的命令， 而检查一个键的类型就是检查键的值对象的类型。
-   Redis 的对象系统带有引用计数实现的内存回收机制， 当一个对象不再被使用时， 该对象所占用的内存就会被自动释放。
-   Redis 会共享值为 `0` 到 `9999` 的字符串对象。
-   对象会记录自己的最后一次被访问的时间， 这个时间可以用于计算对象的空转时间。