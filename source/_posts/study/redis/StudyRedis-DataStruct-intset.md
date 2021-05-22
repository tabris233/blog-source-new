---
title: "Redis 学习 基础数据结构篇 之 整数集合(intset)"
date: 2021-05-22 17:27:23
description: "Redis 基础数据结构 - 整数集合。"
katex: true
summary:
categories:
  - 学习
  - redis
tags:
  - redis
  - intset
  - 整数集合
---

[TOC]

> 基于 [Redis 6.2.1]([redis/redis at 6.2.1 (github.com)](https://github.com/redis/redis/tree/6.2.1))
>
> 图片截取自《Redis设计与实现》

# 整数集合（intset）

整数集合（intset）是`Set`底层实现之一，当集合只包含整数值元素，且这个集合的元素数量不多时，`Redis`就会使用整数集合作为集合的底层实现。

## 整数集合的实现

整数集合比较简单，简单来说就是用整数数存储整数是在来维护集合。十几年了也没更新过。

整数集合的结构体定义在`intset.h`

```c
typedef struct intset {
    uint32_t encoding;
    uint32_t length;
    int8_t contents[];
} intset;
```

-   `encoding` 编码方式：这里有三种`INTSET_ENC_INT16`、`INTSET_ENC_INT32`、`INTSET_ENC_INT64`，分表表示contents中存储的元素的实际类型，所以整数集合真正存储的只有`int16_t`、`int32_t`、`int64_t`这三种类型。

-   `length` 集合的元素数量。

-   `contents`保存元素的数组，虽然定义是`int8_t`但真正的类型要看`encoding`的值，这里只是方便内存申请用了`int8_t`，而为了快速查找，这里进行安元素大小顺序存储，为了后面查找是可以二分查找。

    <details><summary>查找函数实现 展开/收起</summary>
    ```c
    /* Search for the position of "value". Return 1 when the value was found and
     * sets "pos" to the position of the value within the intset. Return 0 when
     * the value is not present in the intset and sets "pos" to the position
     * where "value" can be inserted. */
    static uint8_t intsetSearch(intset *is, int64_t value, uint32_t *pos) {
        int min = 0, max = intrev32ifbe(is->length)-1, mid = -1;
        int64_t cur = -1;
    
        /* The value can never be found when the set is empty */
        if (intrev32ifbe(is->length) == 0) {
            if (pos) *pos = 0;
            return 0;
        } else {
            /* Check for the case where we know we cannot find the value,
             * but do know the insert position. */
            if (value > _intsetGet(is,max)) {
                if (pos) *pos = intrev32ifbe(is->length);
                return 0;
            } else if (value < _intsetGet(is,0)) {
                if (pos) *pos = 0;
                return 0;
            }
        }
        
        while(max >= min) {
            mid = ((unsigned int)min + (unsigned int)max) >> 1;
            cur = _intsetGet(is,mid);
            if (value > cur) {
                min = mid+1;
            } else if (value < cur) {
                max = mid-1;
            } else {
                break;
            }
        }
        
        if (value == cur) {
            if (pos) *pos = mid;
            return 1;
        } else {
            if (pos) *pos = min;
            return 0;
        }
    }
    ```
    </details>

---

这里⚠️注意，整数集合只能存储同一类型的值，不会同时存`int64_t`和`int16_t`。比如这个例子：

```
   ┌────┬───────┬────────────────────────────┐
   │ 1  │   2   │   -2675256175807981027     │
   └────┴───────┴────────────────────────────┘
```

前两个数字只需要`int16_t`就可以存下了， 只有第三个数字需要`int64_t`，但当集合中插入第三个数字的时候就会进行升级，把所有的数字转换成`int64_t`存储。

## 整数集合的升级

每当我们要将一个新元素添加到整数集合里面，并且新元素的类型比整数集合现有所有元素的类型都要长时，整数集合需要先进行升级（upgrade），然后才能将新元素添加到整数集合里面。升级整数集合并添加新元素共分为三步进行：

1.  根据新元素的类型，扩展整数集合底层数组的空间大小，并为新元素分配空间。
2.  将底层数组现有的所有元素都转换成与新元素相同的类型，并将类型转换后的元素放置到正确的位上，而且在放置元素的过程中，需要继续维持底层数组的有序性质不变。
3.  将新元素添加到底层数组里面。

---

举个例子，最开始的一个整数集合为

![image-20210522193314271](https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/05/22/20210522193314.png)

目前只需要 `int16_t` 就可以存下了。对应的表示为。

![image-20210522193403366](https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/05/22/20210522193403.png)

现在要将65535 加入集合，显然`int64_t` 不够用了，需要`int32_t`来存储，这是需要申请新的内存为`sizeof(int32_t) * 4` = 128位，于是重新分配的内存如下

![image-20210522193609210](https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/05/22/20210522194353.png)

 然后开始逐渐从后至前转换每一个元素。

![image-20210522194813874](https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/05/22/20210522194813.png)

![image-20210522194841128](https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/05/22/20210522194841.png)

![image-20210522194857158](https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/05/22/20210522194857.png)

![image-20210522194936185](https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/05/22/20210522194936.png)

就这样完成了升级的过程。

### 升级的好处。

1.  **提升灵活性**: 对于静态类型语言，为了避免类型错误，通常不会将两种不用类型的值放到同一个数据结构里面。整数集合可以自动升级底层数据类型，不会出现类型错误。
2.  **节约内存**： 当存储的数值域有限的时候使用较大的类型对于空间是一种浪费。整数集合只有在必要的情况下才使用更大的数据类型。

---

⚠️注意整数集合不支持降级操作，比如上述例子，如果65535被删除了后面再添加的也是小于32768的数时编码类型仍然为`INTSET_ENC_INT32`。

## 整数集合的API

| 函数          | 作用                           | 时间复杂度                                      |
| ------------- | ------------------------------ | ----------------------------------------------- |
| intsetNew     | 创建一个新的整数集合           | $O(1)$                                          |
| intsetAdd     | 将给定的元素添加到整数集合里面 | $O(N)$                                          |
| intsetRemove  | 从整数集合中移除给定元素       | $O(N)$                                          |
| intsetFind    | 检查给定值是否存在于整数集合   | $O(logN)$<br />因为底层数组有序，采用了二分查找 |
| intsetRandom  | 从整数集合中随机返回一个元素   | $O(1)$                                          |
| intsetGet     | 取出底层数组再给定索引上的元素 | $O(1)$                                          |
| intsetLen     | 返回证书集合包含的元素个数     | $O(1)$                                          |
| intsetBlobLen | 返回证书集合占用的内存字节数   | $O(1)$                                          |



## 回顾

-   整数集合是集合键的底层实现之一
-   整数集合的底层实现为数组，这个数组以有序、无重复的方式保存集合元素，在有需要时，程序会根据新添加元素的类型，改变这个数组的类型。
-   升级操作为整数集合带来了操作上的灵活性，并且尽可能的节约了内存。
-   整数集合只支持升级操作，不支持降级操作。

