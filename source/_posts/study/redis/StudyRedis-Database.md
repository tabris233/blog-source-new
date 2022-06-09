---
title: "Redis 学习 数据库篇"
date: 2021-05-30 00:00:00
updated: 2021-06-06 22:36:21
description: "Redis 数据篇。"
mathjax: false
summary:
categories:
  - 学习
  - redis
  - 数据库
tags:
  - redis
  - database
  - 数据库
---

> 基于  [Redis 6.2.1](https://github.com/redis/redis/tree/6.2.1)

# 服务器中的数据库

Redis 服务器将所有数据库保存到server.h/redisServer结构的db数组中，db数组的每一项都是server.h/redisDb结构，每个redisDb代表一个数据库：

```c
struct redisServer {
    // ...
    redisDb *db; // 数组，保存服务器的所有数据库
    // ...
}
```



>   说起来，四五百行的结构体，真是活久见啊。。。。
>
>   <img src="https://fastly.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/05/31/20210531235652.png" alt="image-20210531235648083" style="zoom: 67%;" />

在初始化服务器时，程序会根据服务器状态的dbnum属性来决定创建多少个数据库：

```c
struct redisServer {
    // ...
    int dbnum; // 服务器的数据库数量
    // ...
}
```

dbnum的属性值由服务器配置的database选项决定，缺省为16，所以Redis服务器默认会创建16个数据库。

```c
// redisServer
 ┌───────────┐
 │redisServer│
 ├───────────┤
 │    ...    │
 ├───────────┤     ┌──────┬───────┬───────┬─────┬───────┐
 │    db     │────►│db[0] │ db[1] │ db[2] │ ... │ db[15]│
 ├───────────┤     └──────┴───────┴───────┴─────┴───────┘
 │    ...    │
 ├───────────┤
 │   dbnum   │
 ├───────────┤
 │    ...    │
 └───────────┘
```

# 切换数据库

Redis客户端都有自己的目标数据库。在一个客户端没有主动切换数据库的时候，该客户端只会操作这个数据库的内容，同时这个数据库与其他数据库的内容是相互隔离的。

默认情况下Redis客户端的目标数据库为0号数据库，客户端可以使用`SELECT`命令主动切换目标数据库。Redis服务端会记录每个链接的目标数据库，也就是说，Redis可以同时连接处理多个不同目标数据库的客户端。

`server.h/client`

```c
typedef struct client {
    // ...
    redisDb *db;  // 客户端正在链接的数据库的指针
    // ...
} client;
```

`db.c`

```c
int selectDb(client *c, int id) {
    if (id < 0 || id >= server.dbnum)
        return C_ERR;
    c->db = &server.db[id];
    return C_OK;
}
// ...
void selectCommand(client *c) {
    int id;

    if (getIntFromObjectOrReply(c, c->argv[1], &id, NULL) != C_OK)
        return;

    if (server.cluster_enabled && id != 0) {
        addReplyError(c,"SELECT is not allowed in cluster mode");
        return;
    }
    if (selectDb(c,id) == C_ERR) {   // 切换具体的数据库id
        addReplyError(c,"DB index is out of range");
    } else {
        addReply(c,shared.ok);
    }
}
```

⚠️注意：默认情况下，客户端默认连接的是0号数据库，一般情况下只要用一个0号数据库就够了。但是如果是有使用多个数据库的诉求，还是要先显示的执行一个`SELECT`命令再执行别的命令。

>   这是自己简单学习的使用Redis，生产环境使用的还是公司自己搞的兼容Redis协议的自研数据库，记得没有`SELECT`命令，用的就是默认的一个数据库。。

# 数据库的键空间

Redis本身是一个键值对（key-value pair）数据库服务，服务中每一个数据库中采用了dict字典保存了数据库中的所有键值对，我们将这个字典称为键空间：

```c
typedef struct redisDb{
    // ...
    dict *dict;   // 数据库键空间，保存数据库中所有的键值对。
    // ...
}
```

键空间和用户所见的数据库是直接对应的：

-   键空间的键也就是数据库的键，每个键都是一个字符串对象。
-   键空间的值也就是数据库的值，每个值可以是字符串对象，列表对象、哈希表对象、集合对象和有序集合对象中的任意一种Redis对象。

如下图所示：

<img src="https://fastly.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/06/05/20210605225852.jpeg" alt="img" style="zoom: 67%;" />

而Redis中添加心焦也一样是在操作`redisDb`中的`dict`成员，支持基本dict操作，同时可以对其中每个键值的增删改查。

除基本的增删改查操作外，还有很多针对数据库本身的命令，也是通过对键空间的处理完成的。

比如说：

1.  用于清空整个数据库的FLUSHDB命令，就是通过删除键空间中的所有键值对来实现的。
2.  用于随机返回数据库中某个键的RANDOMKEY命令，就是通过在键空间中随机返回一个键来实现的。
3.  用于返回数据库键数量的DBSIZE命令，就是通过返回键空间中包含的键值对的数量来实现的。
4.  类似的命令还有EXISTS、RENAME、KEYS等，

这些命令都是通过对键空间进行操作来实现的。

当使用Redis命令对数据库进行读写时，服务器不仅会对键空间执行指定的读写操作，还会执行一些额外的维护操作，其中包括：

1.  在读取一个键之后（读操作和写操作都要对键进行读取），服务器会根据键是否存在来更新服务器的键空间命中（hit）次数或键空间不命中（miss）次数，这两个值可以在INFO stats命令的keyspace_hits属性和keyspace_misses属性中查看。
2.  在读取一个键之后，服务器会更新键的LRU（最后一次使用）时间，这个值可以用于计算键的闲置时间，使用OBJECTidletime命令可以查看键key的闲置时间。
3.  如果服务器在读取一个键时发现该键已经过期，那么服务器会先删除这个过期键，然后才执行余下的其他操作，本章稍后对过期键的讨论会详细说明这一点。
4.  如果有客户端使用WATCH命令监视了某个键，那么服务器在对被监视的键进行修改之后，会将这个键标记为脏（dirty），从而让事务程序注意到这个键已经被修改过。
5.  服务器每次修改一个键之后，都会对脏（dirty）键计数器的值增1，这个计数器会触发服务器的持久化以及复制操作。
6.  如果服务器开启了数据库通知功能，那么在对键进行修改之后，服务器将按配置发送相应的数据库通知。

# 设置键的生存时间或过期时间

通过`EXPIRE`命令或者`PEXPIRE`命令，客户端可以以秒或者毫秒精度为数据库中的某个键设置生存时间（Time To Live，TTL），在经过指定的秒数或者毫秒数之后，服务器就会自动删除生存时间为0的键：

```c
redis> SET key value
OK
redis> EXPIRE key 5
(integer) 1
redis> GET key // 5秒内
"value"
redis> GET key // 5秒外
(nil)
```

>   ⚠️注意：`SETEX`命令可以在设置一个字符串键的同时为键设置过期时间，因为这个命令是一个类型限定的命令（只能用于字符串键），所以本章不会对这个命令进行介绍，但`SETEX`命令设置过期时间的原理和本章介绍的`EXPIRE`命令设置过期时间的原理是完全一样的。

与`EXPIRE`命令和`PEXPIRE`命令类似，客户端可以通过`EXPIREAT`命令或`PEXPIREAT`命令，以秒或者毫秒精度给数据库中的某个键设置过期时间（expire time）。

过期时间是一个UNIX时间戳，当键的过期时间来临时，服务器就会自动从数据库中删除这个键：

```c
redis> SET key value
OK
redis> EXPIREAT key 1622906266
(integer) 1
redis> GET key // 1622906266之前
"value"
redis> GET key // 1622906266之前
(nil)
```

`TTL`命令和`PTTL`命令接受一个带有生存时间或者过期时间的键，返回这个键的剩余生存时间，也就是，返回距离这个键被服务器自动删除还有多长时间：

```c
redis> SET key value
OK
redis> EXPIRE key 1000
(integer) 1
redis> TTL key
(integer) 997
redis> SET another_key another_value
OK
> TIME
1622906549
913439
> EXPIREAT another_key 1622906749
1
> TTL another_key
177
```

## 设计过期时间

Reids有四个不同的命令可以用于设置键的生命时间（键可以存在多久）或过期时间（键什么时候会被删除）。

1.  `EXPIRE＜key＞＜ttl＞`命令用于将键$key$的生存时间设置为ttl秒。
2.  `PEXPIRE＜key＞＜ttl＞`命令用于将键$key$的生存时间设置为ttl毫秒。
3.  `EXPIREAT＜key＞＜timestamp＞`命令用于将键$key$的过期时间设置为$timestamp$所指定的秒数时间戳。
4.  `PEXPIREAT＜key＞＜timestamp＞`命令用于将键$key$的过期时间设置为$timestamp$所指定的毫秒数时间戳。

虽然有多种不同单位和不同形式的设置命令，但实际上EXPIRE、PEXPIRE、EXPIREAT三个命令都是使用PEXPIREAT命令来实现的：无论客户端执行的是以上四个命令中的哪一个，经过转换之后，最终的执行效果都是一样的。

这四个命令的具体实现在`expire.c`，都仅调用了`expireGenericCommand`，在`expireGenericCommand`中通过入参传入`basetime` 时间参数（本身是毫秒级时间戳），`unit`时间戳类型（妙计或毫秒级时间戳）来转化为实际操作的时间。最终具体逻辑是以毫秒级的时间戳设置过期时间。

```c
/* EXPIRE key seconds */
void expireCommand(client *c) {
    expireGenericCommand(c,mstime(),UNIT_SECONDS);
}

/* EXPIREAT key time */
void expireatCommand(client *c) {
    expireGenericCommand(c,0,UNIT_SECONDS);
}

/* PEXPIRE key milliseconds */
void pexpireCommand(client *c) {
    expireGenericCommand(c,mstime(),UNIT_MILLISECONDS);
}

/* PEXPIREAT key ms_time */
void pexpireatCommand(client *c) {
    expireGenericCommand(c,0,UNIT_MILLISECONDS);
}
```

`expireGenericCommand` 实现如下：

```c
/* This is the generic command implementation for EXPIRE, PEXPIRE, EXPIREAT
 * and PEXPIREAT. Because the command second argument may be relative or absolute
 * the "basetime" argument is used to signal what the base time is (either 0
 * for *AT variants of the command, or the current time for relative expires).
 *
 * unit is either UNIT_SECONDS or UNIT_MILLISECONDS, and is only used for
 * the argv[2] parameter. The basetime is always specified in milliseconds. */
void expireGenericCommand(client *c, long long basetime, int unit) {
    robj *key = c->argv[1], *param = c->argv[2];
    long long when; /* unix time in milliseconds when the key will expire. */

    if (getLongLongFromObjectOrReply(c, param, &when, NULL) != C_OK)
        return;
    int negative_when = when < 0;
    if (unit == UNIT_SECONDS) when *= 1000;                            // 如果是秒级时间戳就转化为毫秒级时间戳。
    when += basetime;                                                  // 加上当前时间，会把相对时间转化为绝对时间。
    if (((when < 0) && !negative_when) || ((when-basetime > 0) && negative_when)) {
        /* EXPIRE allows negative numbers, but we can at least detect an
         * overflow by either unit conversion or basetime addition. */
        addReplyErrorFormat(c, "invalid expire time in %s", c->cmd->name);
        return;
    }
    /* No key, return zero. */
    if (lookupKeyWrite(c->db,key) == NULL) {
        addReply(c,shared.czero);
        return;
    }

    if (checkAlreadyExpired(when)) {
        robj *aux;

        int deleted = server.lazyfree_lazy_expire ? dbAsyncDelete(c->db,key) :
                                                    dbSyncDelete(c->db,key);
        serverAssertWithInfo(c,key,deleted);
        server.dirty++;

        /* Replicate/AOF this as an explicit DEL or UNLINK. */
        aux = server.lazyfree_lazy_expire ? shared.unlink : shared.del;
        rewriteClientCommandVector(c,2,aux,key);
        signalModifiedKey(c,c->db,key);
        notifyKeyspaceEvent(NOTIFY_GENERIC,"del",key,c->db->id);
        addReply(c, shared.cone);
        return;
    } else {
        setExpire(c,c->db,key,when);
        addReply(c,shared.cone);
        signalModifiedKey(c,c->db,key);
        notifyKeyspaceEvent(NOTIFY_GENERIC,"expire",key,c->db->id);
        server.dirty++;
        return;
    }
}
```

## 保存过期时间

redisDb结构的expires字典保存了数据库中所有键的过期时间，我们称这个字典为过期字典：

过期字典的键是一个指针，这个指针指向键空间中的某个键对象（也即是某个数据库键）。

过期字典的值是一个long long类型的整数，这个整数保存了键所指向的数据库键的过期时间——一个毫秒精度的UNIX时间戳。

```c
typedef struct redisDb{
    // ...
    dict *expires;   // 过期字典，保存每个键的过期时间。
    // ...
}
```

### 通过`expires`字典可以方便对每个键计算并返回剩余生存时间。

`TTL`命令以秒为单位返回键的剩余生存时间，而`PTTL`命令则以毫秒为单位返回键的剩余生存时间。

### 同时也可以方便的判断一个键是否过期。

1.  如果一个键在没有`expires`中没有找到，则说明这个键没有过期时间。是永久存储的。
2.  如果可以找到，则与当前服务器时间戳对比。小于当前时间戳则过期，反之未过期。

>   ⚠️注意：
>
>   实现过期键判定的另一种方法是使用`TTL`命令或者`PTTL`命令，比如说，如果对某个键执行`TTL`命令，并且命令返回的值大于等于$0$，那么说明该键未过期。在实际中，`Redis`检查键是否过期的方法就是访问字典获取过期的时间戳，因为直接访问字典比执行一个命令稍微快一些

# 过期键的删除策略。

过期键的删除策略：

1.  定时删除：在设置键的过期时间的同时，创建一个定时器（timer），让定时器在键的过期时间来临时，立即执行对键的删除操作。
2.  惰性删除：放任键过期不管，但是每次从键空间中获取键时，都检查取得的键是否过期，如果过期的话，就删除该键；如果没有过期，就返回该键。
3.  定期删除：每隔一段时间，程序就对数据库进行一次检查，删除里面的过期键。至于要删除多少过期键，以及要检查多少个数据库，则由算法决定。

定期删除：每隔一段时间，程序就对数据库进行一次检查，删除里面的过期键。至于要删除多少过期键，以及要检查多少个数据库，则由算法决定。

## 定时删除

定时删除策略是对内存友好的，能够及时的将已过去数据的内存释放。

但定时删除的缺点是对CPU不友好，当一些键的过期时间比较集中的时候，删除这些过期键会占用相对较长的CPU时间，在内存不紧张但CPU紧张的场景，会提升机器的负载，影响到服务的性能（耗时和吞吐量）。

## 惰性删除

惰性删除是对CPU友好的，程序只会在取出键时才检查键的过期时间。这个策略不会在删除其他无关的键上花费任何CPU时间。

与定时删除相反，惰性删除对内存并不友好。当一个键已经过期，但是没有被访问到的时候，他的内存不会被释放，会一直保存。

在采取惰性删除策略时，如果数据库中存在很多很多过期键，又都没有被访问，那么就会一直保存在服务器中，这部分内存就被浪费了。服务器不会主动去释放他们，这种情况对于以来内存作为存储的Redis服务器来说，是不利的，随着时间的推移，这样的过期键越来越多，最终数据库的空间就不够用了，在未来某一个时就会过载。

## 定期删除

定时删除和惰性删除是对服务器两种资源（内存，CPU）偏重其中一个下所诞生的。

-   定时删除占用太多CPU时间，影响服务器的响应时间和吞吐量。
-   惰性删除浪费太多内存，有内存泄漏的危险。

定期删除策略是前两种策略的一种整合和折中：

-   定期删除策略每隔一段时间执行一次删除过期键操作，并通过限制删除操作制定的时长和频率来减少删除操作对CPU时间的影响。
-   除此之外，通过定期删除过期键，定期删除策略有效减少了因为过期键而带来的内存浪费。

定期删除策略的难点时确定删除操作执行的时长和频率：

-   如果删除操作执行的太频繁，或者执行时间太长，定期删除策略就会退化成定时删除策略，以至于将CPU时间过多地消耗在删除过期键上面。
-   如果删除操作执行得太少，或者执行时间太短，定期删除策略优惠和惰性删除策略一样，出现浪费内存的情况。

因此，如果采用定期删除策略，服务器必须根据情况，合理地设置删除操作的执行时长和执行频率。

# Redis服务器的过期键删除策略。

Redis服务器实际泗洪的是惰性删除和定期删除两种策略：通过配合这两种策略，服务器可以更好的平衡CPU和内存的消耗。

## 惰性删除策略的实现。

过期键的惰性删除策略由db.c/expireIfNeeded函数实现，所有读写数据库的Redis命令在执行之前都会调用expireIfNeeded函数对输入键进行检查：

-   如果输入键已经过期，那么expireIfNeeded函数将输入键从数据库中删除。
-   如果输入键未过期，那么expireIfNeeded函数不做动作。

expireIfNeeded函数就像一个过滤器，它可以在命令真正执行之前，过滤掉过期的输入键，从而避免命令接触到过期键。

另外，因为每个被访问的键都可能因为过期而被expireIfNeeded函数删除，所以每个命令的实现函数都必须能同时处理键存在以及键不存在这两种情况：

-   当键存在时，命令按照键存在的情况执行。
-   当键不存在或者键因为过期而被expireIfNeeded函数删除时，命令按照键不存在的情况执行。

## 定期删除策略的实现。

过期键的定期删除策略由`expire.c/activeExpireCycle`函数实现，每当Redis的服务器周期性操作`server.c/serverCron`函数执行时，`activeExpireCycle` 函数就会被`serverCron`中的`databsaesCron`调用。它在规定的时间内，多次便利服务器中的各个数据库，从数据库中的expires字典中随机检查一部分键的过期时间，并删除其中的过期键。

`activeExpireCycle`函数的工作模式可以总结如下：

-   函数每次运行时，都从一定数量的数据库中取出一定数量的随机键进行检查，并删除其中过期的键。
-   全局变量`current_db`会记录当前`activeExpireCycle`函数检查的进度，并在下一次`activeExpireCycle`函数调用时，接着上一次的进度进行处理。比如说，当前`activeExpireCycle`函数在便利$10$号数据库时返回了，那么下次`activeExpireCycle`函数执行时，将从$11$号数据库开始查找并删除过期键。
-   随着`activeExpireCycle`函数的不断执行，服务器中所有的数据库键都会被检查一遍，这是函数将`current_db`变量重置为$0$，然后再次开始新一轮的检查。

# AOF、RDB和复制功能对过期键的处理 

## AOF的处理

### AOF文件写入

当服务器以AOF持久化模式运行时，如果某个键已经过期，但它还没有被惰性删除或者定期删除，那么AOF文件不会因为这个过期键产生任何影响。

当过期键被惰性删除或者定期删除之后，程序会向AOF文件追加（append）一条DEL命令，来显式地记录该键已经被删除。

### AOF重写

和生成RDB文件时类似，在执行AOF重写的过程中，程序会对数据库中的键进行检查，已经过期的键不会被保存到重写后的AOF文件中。

## RDB的处理

## 生成RDB文件

在执行save命令或者BGSAVE命令创建一个新的RDB文件时，程序会对数据库中的键进行检查，已过期的键不会被保存到新创建的RDB文件中。

## 载入RDB文件

在启动Redis服务器时，如果服务器开启了RDB功能，那么服务器将对RDB文件进行载入：

-   如果服务器以主服务器模式运行，那么在载入RDB文件时，程序会对文件中保存的键进行检查，未过期的键将被载入到数据库中，而过期的键则会被忽略，所以过期的键对载入RDB文件的主服务器不会造成影响。

-   如果服务器以从服务器模式运行，那么在载入RDB文件时，文件中保存的所有键，无论是否过期，都会被载入到数据库中。

    >   不过，因为主从服务器在进行数据同步时，从服务器的数据库就会被清空，所以一般来讲，过期键对载入RDB文件的从服务器也不会产生影响。

## 复制

当服务器运行在复制模式下，从服务器的过期键删除动作由主服务器控制：

-   主服务器在删除一个过期键之后，会显式地向所有从服务器发送一个DEL命令，告知从服务器删除这个过期键。
-   从服务器在执行客户端发送的读命令式，碰到过期键也不会将过期键删除，而是继续像处理未过期的键一样来处理过期键。
-   从服务器只有接收到主服务器发过来的DEL命令，才会删除过期键。

通过主服务器来控制从服务器统一地删除过期键，可以保证主从服务器数据的一致性，也正是因为这个原因，当一个过期键仍然存在于主服务器的数据库时，这个过期键在从服务器的拷贝也会继续存在。

# 数据库通知

数据库通知时Redis 2.8版本新增加的功能，这个功能可以让客户端通过订阅给定的频道或者模式，来获知数据库中键的变化，以及数据库中命令的执行情况。

通知分为两种类型：

1.  键空间通知（key-space notification）：关注具体某个键执行了什么命令。

    ```c
    127.0.0.1:6379＞ SUBSCRIBE _ _keyspace@0_ _:message
    Reading messages... (press Ctrl-C to quit)
    1) "subscribe"  //订阅信息
    2) "__keyspace@0__:message"
    3) (integer) 1
    1) "message"    //执行SET命令
    2) "_ _keyspace@0_ _:message"
    3) "set"
    1) "message"    //执行EXPIRE命令
    2) "_ _keyspace@0_ _:message"
    3) "expire"
    1) "message"    //执行DEL命令
    2) "_ _keyspace@0_ _:message"
    3) "del"
    ```

    

2.  键事件通知（key-event notification）：关注某一个命令被那些键执行了。

    ```c
    127.0.0.1:6379＞ SUBSCRIBE _ _keyevent@0_ _:del
    Reading messages... (press Ctrl-C to quit)
    1) "subscribe"  //订阅信息
    2) "_ _keyevent@0_ _:del"
    3) (integer) 1
    1) "message"    //键key执行了DEL命令
    2) "_ _keyevent@0_ _:del"
    3) "key"
    1) "message"    //键number执行了DEL命令
    2) "_ _keyevent@0_ _:del"
    3) "number"
    1) "message"    //键message执行了DEL命令
    2) "_ _keyevent@0_ _:del"
    3) "message"
    ```

服务器配置的notify-keyspace-events选项决定了服务器所发送通知的类型：

-   想让服务器发送所有类型的键空间通知和键事件通知，可以将选项的值设置为AKE。
-   想让服务器发送所有类型的键空间通知，可以将选项的值设置为AK。
-   想让服务器发送所有类型的键事件通知，可以将选项的值设置为AE。
-   想让服务器只发送和字符串键有关的键空间通知，可以将选项的值设置为K$。
-   想让服务器只发送和列表键有关的键事件通知，可以将选项的值设置为El。

关于数据库通知功能的详细用法，以及notify-keyspace-events选项的更多设置，Redis的官方文档已经做了很详细的介绍，这里不再赘述。

## 发送通知

发送数据库通知的功能是由notify.c/notifyKeyspaceEvent函数实现的：

```c
void notifyKeyspaceEvent(int type,char *event,robj *key,int dbid);
```

函数的type参数是当前想要发送的通知的类型，程序会根据这个值来判断通知是否就是服务器配置notify-keyspace-events选项所选定的通知类型，从而决定是否发送通知。

event、keys和dbid分别是事件的名称、产生事件的键，以及产生事件的数据库号码，函数会根据type参数以及这三个参数来构建事件通知的内容，以及接收通知的频道名。

每当一个Redis命令需要发送数据库通知的时候，该命令的实现函数就会调用notify-KeyspaceEvent函数，并向函数传递传递该命令所引发的事件的相关信息。

```c

/* The API provided to the rest of the Redis core is a simple function:
 *
 * notifyKeyspaceEvent(char *event, robj *key, int dbid);
 *
 * 'event' is a C string representing the event name.
 * 'key' is a Redis object representing the key name.
 * 'dbid' is the database ID where the key lives.  */
void notifyKeyspaceEvent(int type, char *event, robj *key, int dbid) {
    sds chan;
    robj *chanobj, *eventobj;
    int len = -1;
    char buf[24];

    /* If any modules are interested in events, notify the module system now.
     * This bypasses the notifications configuration, but the module engine
     * will only call event subscribers if the event type matches the types
     * they are interested in. */
     moduleNotifyKeyspaceEvent(type, event, key, dbid);

    /* If notifications for this class of events are off, return ASAP. */
    if (!(server.notify_keyspace_events & type)) return;

    eventobj = createStringObject(event,strlen(event));

    /* __keyspace@<db>__:<key> <event> notifications. */
    if (server.notify_keyspace_events & NOTIFY_KEYSPACE) {
        chan = sdsnewlen("__keyspace@",11);
        len = ll2string(buf,sizeof(buf),dbid);
        chan = sdscatlen(chan, buf, len);
        chan = sdscatlen(chan, "__:", 3);
        chan = sdscatsds(chan, key->ptr);
        chanobj = createObject(OBJ_STRING, chan);
        pubsubPublishMessage(chanobj, eventobj);
        decrRefCount(chanobj);
    }

    /* __keyevent@<db>__:<event> <key> notifications. */
    if (server.notify_keyspace_events & NOTIFY_KEYEVENT) {
        chan = sdsnewlen("__keyevent@",11);
        if (len == -1) len = ll2string(buf,sizeof(buf),dbid);
        chan = sdscatlen(chan, buf, len);
        chan = sdscatlen(chan, "__:", 3);
        chan = sdscatsds(chan, eventobj->ptr);
        chanobj = createObject(OBJ_STRING, chan);
        pubsubPublishMessage(chanobj, key);
        decrRefCount(chanobj);
    }
    decrRefCount(eventobj);
}
```

# 总结

1.  Redis服务器的所有数据库都保存在redisServer.db数组中，而数据库的数量则由redisServer.dbnum属性保存。
2.  客户端通过修改目标数据库指针，让它指向redisServer.db数组中的不同元素来切换不同的数据库。
3.  数据库主要由dict和expires两个字典构成，其中dict字典负责保存键值对，而expires字典则负责保存键的过期时间。
4.  因为数据库由字典构成，所以对数据库的操作都是建立在字典操作之上的。
5.  数据库的键总是一个字符串对象，而值则可以是任意一种Redis对象类型，包括字符串对象、哈希表对象、集合对象、列表对象和有序集合对象，分别对应字符串键、哈希表键、集合键、列表键和有序集合键。
6.  expires字典的键指向数据库中的某个键，而值则记录了数据库键的过期时间，过期时间是一个以毫秒为单位的UNIX时间戳。
7.  Redis使用惰性删除和定期删除两种策略来删除过期的键：惰性删除策略只在碰到过期键时才进行删除操作，定期删除策略则每隔一段时间主动查找并删除过期键。
8.  执行SAVE命令或者BGSAVE命令所产生的新RDB文件不会包含已经过期的键。
9.  执行BGREWRITEAOF命令所产生的重写AOF文件不会包含已经过期的键。
10.  当一个过期键被删除之后，服务器会追加一条DEL命令到现有AOF文件的末尾，显式地删除过期键。
11.  当主服务器删除一个过期键之后，它会向所有从服务器发送一条DEL命令，显式地删除过期键。
12.  从服务器即使发现过期键也不会自作主张地删除它，而是等待主节点发来DEL命令，这种统一、中心化的过期键删除策略可以保证主从服务器数据的一致性。
13.  当Redis命令对数据库进行修改之后，服务器会根据配置向客户端发送数据库通知。