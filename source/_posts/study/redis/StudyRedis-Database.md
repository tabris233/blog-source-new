---
title: "Redis 学习 数据库篇"
date: 2021-05-30 00:00:00
updated: 2021-06-06 12:48:12
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
  - TODO

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
>   <img src="https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/05/31/20210531235652.png" alt="image-20210531235648083" style="zoom: 67%;" />

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

<img src="https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/06/05/20210605225852.jpeg" alt="img" style="zoom: 67%;" />

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



