---
title: Redis 学习 基础数据结构篇 之SDS
date: 2021-04-22 21:47:23
description: ["redis 基础数据结构 - 简单动态字符串。redis 3.2 版本更新了 sds 2.0， 本文简单总结下sds1.0，再总结下sds2.0."]
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
  - 学习
  - redis
  - 基础数据结构
tags:
  - redis
  - sds
  - 简单动态字符串
---

[TOC]

> 这篇文章基本是抄来的。 好不要脸啊我。
>
> 带有空还是要自己整理一下。
>
> 基于 [Redis 6.2.1]([redis/redis at 6.2.1 (github.com)](https://github.com/redis/redis/tree/6.2.1))

# SDS

> Redis 没有直接使用 C 语言传统的字符串表示（以空字符结尾的字符数组，以下简称 C 字符串）， 而是自己构建了一种名为简单动态字符串（simple dynamic string，SDS）的抽象类型， 并将 SDS 用作 Redis 的默认字符串表示。
> Redis 中除了字符串字面量，当作常量使用的地方用了C语言传统字符串。 其他可以想见的涉及到字符串的东西都是用SDS实现的。

> redis 3.2 版本更新了 sds 2.0， 本文简单总结下sds1.0，再总结下sds2.0.

## SDS 1.0

> SDS 1.0 内容copy 自 《Redis 设计与实现（第二版）》

### sds1.0 定义

每个 sds.h/sdshdr 结构表示一个 SDS 值：

```c
struct sdshdr {

    // 记录 buf 数组中已使用字节的数量
    // 等于 SDS 所保存字符串的长度
    int len;

    // 记录 buf 数组中未使用字节的数量
    int free;

    // 字节数组，用于保存字符串
    char buf[];

};
```

图 2-1 展示了一个 SDS 示例：

- `free` 属性的值为 `0` ， 表示这个 SDS 没有分配任何未使用空间。
- `len` 属性的值为 `5` ， 表示这个 SDS 保存了一个五字节长的字符串。
- `buf` 属性是一个 `char` 类型的数组， 数组的前五个字节分别保存了 `'R'` 、 `'e'` 、 `'d'` 、 `'i'` 、 `'s'` 五个字符， 而最后一个字节则保存了空字符 `'\0'` 。

![](https://box.kancloud.cn/2015-09-13_55f50d7faffa3.png)

SDS 遵循 C 字符串以空字符结尾的惯例， 保存空字符的 `1` 字节空间不计算在 SDS 的 `len` 属性里面， 并且为空字符分配额外的 `1` 字节空间， 以及添加空字符到字符串末尾等操作都是由 SDS 函数自动完成的， 所以这个空字符对于 SDS 的使用者来说是完全透明的。

遵循空字符结尾这一惯例的好处是， SDS 可以直接重用一部分 C 字符串函数库里面的函数。

举个例子， 如果我们有一个指向图 2-1 所示 SDS 的指针 `s` ， 那么我们可以直接使用 `stdio.h/printf` 函数， 通过执行以下语句：

```
printf("%s", s->buf);
```

来打印出 SDS 保存的字符串值 `"Redis"` ， 而无须为 SDS 编写专门的打印函数。

图 2-2 展示了另一个 SDS 示例:

- 这个 SDS 和之前展示的 SDS 一样， 都保存了字符串值 `"Redis"` 。
- 这个 SDS 和之前展示的 SDS 的区别在于， 这个 SDS 为 `buf` 数组分配了五字节未使用空间， 所以它的 `free` 属性的值为 `5` （图中使用五个空格来表示五字节的未使用空间）。

![](https://box.kancloud.cn/2015-09-13_55f50d86a66ae.png)

### sds 1.0 与C字符串的区别

根据传统， C 语言使用长度为 `N+1` 的字符数组来表示长度为 `N` 的字符串， 并且字符数组的最后一个元素总是空字符 `'\0'` 。

比如说， 图 2-3 就展示了一个值为 `"Redis"` 的 C 字符串：

![](https://box.kancloud.cn/2015-09-13_55f50e077afb3.png)

C 语言使用的这种简单的字符串表示方式， 并不能满足 Redis 对字符串在安全性、效率、以及功能方面的要求， 本节接下来的内容将详细对比 C 字符串和 SDS 之间的区别， 并说明 SDS 比 C 字符串更适用于 Redis 的原因。

#### 常数复杂度获取字符串长度

因为 C 字符串并不记录自身的长度信息， 所以为了获取一个 C 字符串的长度， 程序必须遍历整个字符串， 对遇到的每个字符进行计数， 直到遇到代表字符串结尾的空字符为止， 这个操作的复杂度为 ![O(N)](https://box.kancloud.cn/2015-09-13_55f50e14dc650.png) 。

举个例子， 图 2-4 展示了程序计算一个 C 字符串长度的过程。

![](https://box.kancloud.cn/2015-09-13_55f50e169b803.png)

![](https://box.kancloud.cn/2015-09-13_55f50e1ce00fa.png)

![](https://box.kancloud.cn/2015-09-13_55f50e1e8455f.png)

![](https://box.kancloud.cn/2015-09-13_55f50e1f6223a.png)

![](https://box.kancloud.cn/2015-09-13_55f50e20dedbf.png)

![](https://box.kancloud.cn/2015-09-13_55f50e21b34d8.png)

和 C 字符串不同， 因为 SDS 在 `len` 属性中记录了 SDS 本身的长度， 所以获取一个 SDS 长度的复杂度仅为 ![O(1)](https://box.kancloud.cn/2015-09-13_55f50e22eca8b.png) 。

举个例子， 对于图 2-5 所示的 SDS 来说， 程序只要访问 SDS 的 `len` 属性， 就可以立即知道 SDS 的长度为 `5` 字节：

![](https://box.kancloud.cn/2015-09-13_55f50e23d7963.png)

又比如说， 对于图 2-6 展示的 SDS 来说， 程序只要访问 SDS 的 `len` 属性， 就可以立即知道 SDS 的长度为 `11` 字节。

![](https://box.kancloud.cn/2015-09-13_55f50e24c2ded.png)

设置和更新 SDS 长度的工作是由 SDS 的 API 在执行时自动完成的， 使用 SDS 无须进行任何手动修改长度的工作。

通过使用 SDS 而不是 C 字符串， Redis 将获取字符串长度所需的复杂度从 ![O(N)](https://box.kancloud.cn/2015-09-13_55f50e14dc650.png) 降低到了 ![O(1)](https://box.kancloud.cn/2015-09-13_55f50e22eca8b.png) ， 这确保了获取字符串长度的工作不会成为 Redis 的性能瓶颈。

比如说， 因为字符串键在底层使用 SDS 来实现， 所以即使我们对一个非常长的字符串键反复执行 STRLEN 命令， 也不会对系统性能造成任何影响， 因为 STRLEN 命令的复杂度仅为 ![O(1)](https://box.kancloud.cn/2015-09-13_55f50e22eca8b.png) 。

#### 杜绝缓冲区溢出

除了获取字符串长度的复杂度高之外， C 字符串不记录自身长度带来的另一个问题是容易造成缓冲区溢出（buffer overflow）。

举个例子， `<string.h>/strcat` 函数可以将 `src` 字符串中的内容拼接到 `dest` 字符串的末尾：

```
char *strcat(char *dest, const char *src);
```

因为 C 字符串不记录自身的长度， 所以 `strcat` 假定用户在执行这个函数时， 已经为 `dest` 分配了足够多的内存， 可以容纳 `src` 字符串中的所有内容， 而一旦这个假定不成立时， 就会产生缓冲区溢出。

举个例子， 假设程序里有两个在内存中紧邻着的 C 字符串 `s1` 和 `s2` ， 其中 `s1` 保存了字符串 `"Redis"` ， 而 `s2` 则保存了字符串 `"MongoDB"`， 如图 2-7 所示。

![](https://box.kancloud.cn/2015-09-13_55f50e28c1620.png)

如果一个程序员决定通过执行：

```
strcat(s1, " Cluster");
```

将 `s1` 的内容修改为 `"Redis Cluster"` ， 但粗心的他却忘了在执行 `strcat` 之前为 `s1` 分配足够的空间， 那么在 `strcat` 函数执行之后， `s1` 的数据将溢出到 `s2` 所在的空间中， 导致 `s2` 保存的内容被意外地修改， 如图 2-8 所示。

![](https://box.kancloud.cn/2015-09-13_55f50e29e04a2.png)

与 C 字符串不同， SDS 的空间分配策略完全杜绝了发生缓冲区溢出的可能性： 当 SDS API 需要对 SDS 进行修改时， API 会先检查 SDS 的空间是否满足修改所需的要求， 如果不满足的话， API 会自动将 SDS 的空间扩展至执行修改所需的大小， 然后才执行实际的修改操作， 所以使用 SDS 既不需要手动修改 SDS 的空间大小， 也不会出现前面所说的缓冲区溢出问题。

举个例子， SDS 的 API 里面也有一个用于执行拼接操作的 `sdscat` 函数， 它可以将一个 C 字符串拼接到给定 SDS 所保存的字符串的后面， 但是在执行拼接操作之前， `sdscat` 会先检查给定 SDS 的空间是否足够， 如果不够的话， `sdscat` 就会先扩展 SDS 的空间， 然后才执行拼接操作。

比如说， 如果我们执行：

```
sdscat(s, " Cluster");
```

其中 SDS 值 `s` 如图 2-9 所示， 那么 `sdscat` 将在执行拼接操作之前检查 `s` 的长度是否足够， 在发现 `s` 目前的空间不足以拼接 `" Cluster"`之后， `sdscat` 就会先扩展 `s` 的空间， 然后才执行拼接 `" Cluster"` 的操作， 拼接操作完成之后的 SDS 如图 2-10 所示。

![](https://box.kancloud.cn/2015-09-13_55f50e30446df.png)

![](https://box.kancloud.cn/2015-09-13_55f50e315f5ac.png)

注意图 2-10 所示的 SDS ： `sdscat` 不仅对这个 SDS 进行了拼接操作， 它还为 SDS 分配了 `13` 字节的未使用空间， 并且拼接之后的字符串也正好是 `13` 字节长， 这种现象既不是 bug 也不是巧合， 它和 SDS 的空间分配策略有关， 接下来的小节将对这一策略进行说明。

#### 减少修改字符串时带来的内存重分配次数

正如前两个小节所说， 因为 C 字符串并不记录自身的长度， 所以对于一个包含了 `N` 个字符的 C 字符串来说， 这个 C 字符串的底层实现总是一个 `N+1` 个字符长的数组（额外的一个字符空间用于保存空字符）。

因为 C 字符串的长度和底层数组的长度之间存在着这种关联性， 所以每次增长或者缩短一个 C 字符串， 程序都总要对保存这个 C 字符串的数组进行一次内存重分配操作：

- 如果程序执行的是增长字符串的操作， 比如拼接操作（append）， 那么在执行这个操作之前， 程序需要先通过内存重分配来扩展底层数组的空间大小 —— 如果忘了这一步就会产生缓冲区溢出。
- 如果程序执行的是缩短字符串的操作， 比如截断操作（trim）， 那么在执行这个操作之后， 程序需要通过内存重分配来释放字符串不再使用的那部分空间 —— 如果忘了这一步就会产生内存泄漏。

举个例子， 如果我们持有一个值为 `"Redis"` 的 C 字符串 `s` ， 那么为了将 `s` 的值改为 `"Redis Cluster"` ， 在执行：

```
strcat(s, " Cluster");
```

之前， 我们需要先使用内存重分配操作， 扩展 `s` 的空间。

之后， 如果我们又打算将 `s` 的值从 `"Redis Cluster"` 改为 `"Redis Cluster Tutorial"` ， 那么在执行：

```
strcat(s, " Tutorial");
```

之前， 我们需要再次使用内存重分配扩展 `s` 的空间， 诸如此类。

因为内存重分配涉及复杂的算法， 并且可能需要执行系统调用， 所以它通常是一个比较耗时的操作：

- 在一般程序中， 如果修改字符串长度的情况不太常出现， 那么每次修改都执行一次内存重分配是可以接受的。
- 但是 Redis 作为数据库， 经常被用于速度要求严苛、数据被频繁修改的场合， 如果每次修改字符串的长度都需要执行一次内存重分配的话， 那么光是执行内存重分配的时间就会占去修改字符串所用时间的一大部分， 如果这种修改频繁地发生的话， 可能还会对性能造成影响。

为了避免 C 字符串的这种缺陷， SDS 通过未使用空间解除了字符串长度和底层数组长度之间的关联： 在 SDS 中， `buf` 数组的长度不一定就是字符数量加一， 数组里面可以包含未使用的字节， 而这些字节的数量就由 SDS 的 `free` 属性记录。

通过未使用空间， SDS 实现了空间预分配和惰性空间释放两种优化策略。

##### 空间预分配

空间预分配用于优化 SDS 的字符串增长操作： 当 SDS 的 API 对一个 SDS 进行修改， 并且需要对 SDS 进行空间扩展的时候， 程序不仅会为 SDS 分配修改所必须要的空间， 还会为 SDS 分配额外的未使用空间。

其中， 额外分配的未使用空间数量由以下公式决定：

- 如果对 SDS 进行修改之后， SDS 的长度（也即是 `len` 属性的值）将小于 `1 MB` ， 那么程序分配和 `len` 属性同样大小的未使用空间， 这时 SDS `len` 属性的值将和 `free` 属性的值相同。 举个例子， 如果进行修改之后， SDS 的 `len` 将变成 `13` 字节， 那么程序也会分配`13` 字节的未使用空间， SDS 的 `buf` 数组的实际长度将变成 `13 + 13 + 1 = 27` 字节（额外的一字节用于保存空字符）。
- 如果对 SDS 进行修改之后， SDS 的长度将大于等于 `1 MB` ， 那么程序会分配 `1 MB` 的未使用空间。 举个例子， 如果进行修改之后， SDS 的 `len` 将变成 `30 MB` ， 那么程序会分配 `1 MB` 的未使用空间， SDS 的 `buf` 数组的实际长度将为 `30 MB + 1 MB + 1 byte` 。

通过空间预分配策略， Redis 可以减少连续执行字符串增长操作所需的内存重分配次数。

举个例子， 对于图 2-11 所示的 SDS 值 `s` 来说， 如果我们执行：

```
sdscat(s, " Cluster");
```

那么 `sdscat` 将执行一次内存重分配操作， 将 SDS 的长度修改为 `13` 字节， 并将 SDS 的未使用空间同样修改为 `13` 字节， 如图 2-12 所示。

![](https://box.kancloud.cn/2015-09-13_55f50e329713a.png)

![](https://box.kancloud.cn/2015-09-13_55f50e33eb3a4.png)

如果这时， 我们再次对 `s` 执行：

```
sdscat(s, " Tutorial");
```

那么这次 `sdscat` 将不需要执行内存重分配： 因为未使用空间里面的 `13` 字节足以保存 `9` 字节的 `" Tutorial"` ， 执行 `sdscat` 之后的 SDS 如图 2-13 所示。

![](https://box.kancloud.cn/2015-09-13_55f50e3578644.png)

在扩展 SDS 空间之前， SDS API 会先检查未使用空间是否足够， 如果足够的话， API 就会直接使用未使用空间， 而无须执行内存重分配。

通过这种预分配策略， SDS 将连续增长 `N` 次字符串所需的内存重分配次数从必定 `N` 次降低为最多 `N` 次。

##### 惰性空间释放

惰性空间释放用于优化 SDS 的字符串缩短操作： 当 SDS 的 API 需要缩短 SDS 保存的字符串时， 程序并不立即使用内存重分配来回收缩短后多出来的字节， 而是使用 `free` 属性将这些字节的数量记录起来， 并等待将来使用。

举个例子， `sdstrim` 函数接受一个 SDS 和一个 C 字符串作为参数， 从 SDS 左右两端分别移除所有在 C 字符串中出现过的字符。

比如对于图 2-14 所示的 SDS 值 `s` 来说， 执行：

```
sdstrim(s, "XY");   // 移除 SDS 字符串中的所有 'X' 和 'Y'
```

会将 SDS 修改成图 2-15 所示的样子。

![](https://box.kancloud.cn/2015-09-13_55f50e3698506.png)

![](https://box.kancloud.cn/2015-09-13_55f50e3856487.png)

注意执行 `sdstrim` 之后的 SDS 并没有释放多出来的 `8` 字节空间， 而是将这 `8` 字节空间作为未使用空间保留在了 SDS 里面， 如果将来要对 SDS 进行增长操作的话， 这些未使用空间就可能会派上用场。

举个例子， 如果现在对 `s` 执行：

```
sdscat(s, " Redis");
```

那么完成这次 `sdscat` 操作将不需要执行内存重分配： 因为 SDS 里面预留的 `8` 字节空间已经足以拼接 `6` 个字节长的 `" Redis"` ， 如图 2-16 所示。

![](https://box.kancloud.cn/2015-09-13_55f50e39ce035.png)

通过惰性空间释放策略， SDS 避免了缩短字符串时所需的内存重分配操作， 并为将来可能有的增长操作提供了优化。

与此同时， SDS 也提供了相应的 API ， 让我们可以在有需要时， 真正地释放 SDS 里面的未使用空间， 所以不用担心惰性空间释放策略会造成内存浪费。

#### 二进制安全

C 字符串中的字符必须符合某种编码（比如 ASCII）， 并且除了字符串的末尾之外， 字符串里面不能包含空字符， 否则最先被程序读入的空字符将被误认为是字符串结尾 —— 这些限制使得 C 字符串只能保存文本数据， 而不能保存像图片、音频、视频、压缩文件这样的二进制数据。

举个例子， 如果有一种使用空字符来分割多个单词的特殊数据格式， 如图 2-17 所示， 那么这种格式就不能使用 C 字符串来保存， 因为 C 字符串所用的函数只会识别出其中的 `"Redis"` ， 而忽略之后的 `"Cluster"` 。

![](https://box.kancloud.cn/2015-09-13_55f50e3b04bab.png)

虽然数据库一般用于保存文本数据， 但使用数据库来保存二进制数据的场景也不少见， 因此， 为了确保 Redis 可以适用于各种不同的使用场景， SDS 的 API 都是二进制安全的（binary-safe）： 所有 SDS API 都会以处理二进制的方式来处理 SDS 存放在 `buf` 数组里的数据， 程序不会对其中的数据做任何限制、过滤、或者假设 —— 数据在写入时是什么样的， 它被读取时就是什么样。

这也是我们将 SDS 的 `buf` 属性称为字节数组的原因 —— Redis 不是用这个数组来保存字符， 而是用它来保存一系列二进制数据。

比如说， 使用 SDS 来保存之前提到的特殊数据格式就没有任何问题， 因为 SDS 使用 `len` 属性的值而不是空字符来判断字符串是否结束， 如图 2-18 所示。

![](https://box.kancloud.cn/2015-09-13_55f50e3cdec5b.png)

通过使用二进制安全的 SDS ， 而不是 C 字符串， 使得 Redis 不仅可以保存文本数据， 还可以保存任意格式的二进制数据。

#### 兼容部分 C 字符串函数

虽然 SDS 的 API 都是二进制安全的， 但它们一样遵循 C 字符串以空字符结尾的惯例： 这些 API 总会将 SDS 保存的数据的末尾设置为空字符， 并且总会在为 `buf` 数组分配空间时多分配一个字节来容纳这个空字符， 这是为了让那些保存文本数据的 SDS 可以重用一部分 `<string.h>`库定义的函数。

![](https://box.kancloud.cn/2015-09-13_55f50e3e47994.png)

举个例子， 如图 2-19 所示， 如果我们有一个保存文本数据的 SDS 值 `sds` ， 那么我们就可以重用 `<string.h>/strcasecmp` 函数， 使用它来对比 SDS 保存的字符串和另一个 C 字符串：

```
strcasecmp(sds->buf, "hello world");
```

这样 Redis 就不用自己专门去写一个函数来对比 SDS 值和 C 字符串值了。

与此类似， 我们还可以将一个保存文本数据的 SDS 作为 `strcat` 函数的第二个参数， 将 SDS 保存的字符串追加到一个 C 字符串的后面：

```
strcat(c_string, sds->buf);
```

这样 Redis 就不用专门编写一个将 SDS 字符串追加到 C 字符串之后的函数了。

通过遵循 C 字符串以空字符结尾的惯例， SDS 可以在有需要时重用 `<string.h>` 函数库， 从而避免了不必要的代码重复。

### 总结

表 2-1 对 C 字符串和 SDS 之间的区别进行了总结。

---

表 2-1 C 字符串和 SDS 之间的区别

| C 字符串                                                                        | SDS                                                                          |
| ---------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| 获取字符串长度的复杂度为 ![O(N)](https://box.kancloud.cn/2015-09-13_55f50e14dc650.png) 。 | 获取字符串长度的复杂度为 ![O(1)](https://box.kancloud.cn/2015-09-13_55f50e22eca8b.png) 。 |
| API 是不安全的，可能会造成缓冲区溢出。                                                        | API 是安全的，不会造成缓冲区溢出。                                                          |
| 修改字符串长度 `N` 次必然需要执行 `N` 次内存重分配。                                              | 修改字符串长度 `N` 次最多需要执行 `N` 次内存重分配。(极大成都降低了频繁内存分配导致的开销)                          |
| 只能保存文本数据。                                                                    | 可以保存文本或者二进制数据。                                                               |
| 可以使用所有 `<string.h>` 库中的函数。                                                   | 可以使用一部分 `<string.h>` 库中的函数。                                                  |

### sds 1.0 主要API

表 2-2 SDS 的主要操作 API

| 函数            | 作用                                                      | 时间复杂度                                                                                                |
| ------------- | ------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `sdsnew`      | 创建一个包含给定 C 字符串的 SDS 。                                   | ![O(N)](https://box.kancloud.cn/2015-09-13_55f50e98b76b4.png) ， `N` 为给定 C 字符串的长度。                    |
| `sdsempty`    | 创建一个不包含任何内容的空 SDS 。                                     | ![O(1)](https://box.kancloud.cn/2015-09-13_55f50e99de6fb.png)                                        |
| `sdsfree`     | 释放给定的 SDS 。                                             | ![O(1)](https://box.kancloud.cn/2015-09-13_55f50e99de6fb.png)                                        |
| `sdslen`      | 返回 SDS 的已使用空间字节数。                                       | 这个值可以通过读取 SDS 的 `len` 属性来直接获得， 复杂度为 ![O(1)](https://box.kancloud.cn/2015-09-13_55f50e99de6fb.png) 。  |
| `sdsavail`    | 返回 SDS 的未使用空间字节数。                                       | 这个值可以通过读取 SDS 的 `free` 属性来直接获得， 复杂度为 ![O(1)](https://box.kancloud.cn/2015-09-13_55f50e99de6fb.png) 。 |
| `sdsdup`      | 创建一个给定 SDS 的副本（copy）。                                   | ![O(N)](https://box.kancloud.cn/2015-09-13_55f50e98b76b4.png) ， `N` 为给定 SDS 的长度。                     |
| `sdsclear`    | 清空 SDS 保存的字符串内容。                                        | 因为惰性空间释放策略，复杂度为 ![O(1)](https://box.kancloud.cn/2015-09-13_55f50e99de6fb.png) 。                      |
| `sdscat`      | 将给定 C 字符串拼接到 SDS 字符串的末尾。                                | ![O(N)](https://box.kancloud.cn/2015-09-13_55f50e98b76b4.png) ， `N` 为被拼接 C 字符串的长度。                   |
| `sdscatsds`   | 将给定 SDS 字符串拼接到另一个 SDS 字符串的末尾。                           | ![O(N)](https://box.kancloud.cn/2015-09-13_55f50e98b76b4.png) ， `N` 为被拼接 SDS 字符串的长度。                 |
| `sdscpy`      | 将给定的 C 字符串复制到 SDS 里面， 覆盖 SDS 原有的字符串。                    | ![O(N)](https://box.kancloud.cn/2015-09-13_55f50e98b76b4.png) ， `N` 为被复制 C 字符串的长度。                   |
| `sdsgrowzero` | 用空字符将 SDS 扩展至给定长度。                                      | ![O(N)](https://box.kancloud.cn/2015-09-13_55f50e98b76b4.png) ， `N` 为扩展新增的字节数。                       |
| `sdsrange`    | 保留 SDS 给定区间内的数据， 不在区间内的数据会被覆盖或清除。                       | ![O(N)](https://box.kancloud.cn/2015-09-13_55f50e98b76b4.png) ， `N` 为被保留数据的字节数。                      |
| `sdstrim`     | 接受一个 SDS 和一个 C 字符串作为参数， 从 SDS 左右两端分别移除所有在 C 字符串中出现过的字符。 | ![O(M*N)](https://box.kancloud.cn/2015-09-13_55f50eae65cb0.png) ， `M` 为 SDS 的长度， `N` 为给定 C 字符串的长度。   |
| `sdscmp`      | 对比两个 SDS 字符串是否相同。                                       | ![O(N)](https://box.kancloud.cn/2015-09-13_55f50e98b76b4.png) ， `N` 为两个 SDS 中较短的那个 SDS 的长度。          |

## SDS2.0

> 2.0 原由
>
> https://github.com/antirez/redis/issues/757
> 问题就在len和free的使用类型上，使用的是unsigned int
>
> - unsigned int，也就是32位，最多只能记录4GB大小，超过4G的大小将无法使用
> - 内存损耗，如果字符串都非常短，可能16位就能记录，那么32位就损耗了2位
>
> 简单的换len和free的类型是不行的，仔细观察上面两个缺点，发现他们其实是有点互斥的成分在里面。
> 一个嫌弃是unsigned int过小，一个是嫌弃unsigned int过大
>
> 然后这个PR就提出了自适应的sdshdr
> [https://github.com/antirez/redis/pull/2509](https://github.com/antirez/redis/pull/2509)也就是SDS2.0的实现

### SDS 2.0 定义

```c
typedef char *sds;

/* Note: sdshdr5 is never used, we just access the flags byte directly.
 * However is here to document the layout of type 5 SDS strings. */
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```

- `len`：字符串的长度，不包含`'\0'`，`'\0'`由自己维护，用户不需要关心；

- `alloc`：申请的内存空间大小；

- `flags`：类型，当前仅使用低三位，后续可能会有扩展，用于判断当前sds对应的header的类型；

- `buf`：变长部分，存储数据；

2.0 根据长度的不同，分成了5种， 分别是5位，8位，16位，32位，64位。

其中sdshdr5较为特殊，没有len和alloc，flags的3位表示类型，剩下的5位表示长度，最多只能表示$2^5 = 32$长度的字符串。

2.0版本与1.0版本本质功能上差别不大。

- 最大的区别就是`header`的定义，分成5类记录长度，而不是之前的`int len`，是为了节省内存（抠的着实是有些细了）。

- 其次在非32位环境下支持超过4GB大小内存的能力。

## 总结

比起 C 字符串， SDS 具有以下优点：

1. 常数复杂度获取字符串长度。
2. 杜绝缓冲区溢出。
3. 减少修改字符串长度时所需的内存重分配次数。
4. 二进制安全。
5. 兼容部分 C 字符串函数。