---
title: "【TODO】Redis 学习 基础数据结构篇 之 跳表(skiplist)"
date: 2021-05-04 20:10:58
description: ["Redis 基础数据结构 - 跳表。"]
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
  - skiplist
  - 跳表
---

[TOC]

> 基于 [Redis 6.2.1](https://github.com/redis/redis/tree/6.2.1)
>
> 参考资料：
>
> [https://www.jianshu.com/p/9d8296562806](https://www.jianshu.com/p/9d8296562806)

# 跳表

跳表（skiplist）是一种有序的数据结构，它通过在每个节点中维护多个指向其他节点的指针，从而达到快速访问的目的。

跳跃表支持平均$O(log N)$, 最坏$O(N)$复杂度的节点查找，还可以通过顺序性操作来批量处理节点。

## 跳表的数据结构

跳表既然是叫什么什么表，那么本质上还是链表。

```graphviz
digraph link {
  rankdir = LR;  //让图片横过来
  node[shape = record]; //record形状是专门用来做类似”结构体“的东西的
  1[label = "{1|}"];//每个的'|'都是一列

  2[label = "{2|}"];
  3[label = "{3|}"];
  4[label = "{4|}"];
  5[label = "{5|}"];
  6[label = "{6|}"];
  7[label = "{7|}"];
  8[label = "{8|NULL}"];
  1->2:w;
  2->3:w;
  3->4:w;
  4->5:w;
  5->6:w;
  6->7:w;
  7->8:w;
}
```

对于一个单链表只能依次遍历，但跳表存储的是有序的节点。

众所周知，利用有序这个单调性是可以二分的，因此可以优化查找操作。二分需要不断的折半，也就是找当前操作区间的正中间的元素，递归下去直到这个区间长度变为1。其实二叉查找树（BST，Binary Search Tree）就是这么玩儿的。像下图这样：

```graphviz
digraph G{
  graph [ordering="out"];
  4 -> 2;
  4 -> 6;
  2 -> 1;
  2 -> 3;
  6 -> 5;
  6 -> 7;
}
```

这里看起来链表和二叉树差别很大，链表相当于只能存二叉树叶子结点的内容，对于非叶子结点无能为力了。

但办法还是有的，上图是一个无重复结点的二叉查找树。稍微换个样子就可以了。如下图：

```graphviz
digraph G{
  graph [ordering="out"];
  l11[label = "1"];
  l21[label = "1"];
  l31[label = "1"];
  l41[label = "1"];
  l25[label = "5"];
  l33[label = "3"];
  l35[label = "5"];
  l42[label = "2"];
  l43[label = "3"];
  l44[label = "4"];
  l45[label = "5"];
  l46[label = "6"];
  l47[label = "7"];

  l11->l21;
  l11->l25;
  l21->l31;
  l21->l33;
  l25->l35;
  l25->l47;
  l31->l41;
  l31->l42;
  l33->l43;
  l33->l44;
  l35->l45;
  l35->l46;

}
```

当该二叉查找树维护的序列只在叶子时就差不多一样了。

跳表结构和该二叉查找树一样采用冗余节点存储二分查找中间值的时候，就能够实现$O(log N)$的查找效率了。

现在可以看下跳表的真正模样了。

```graphviz
digraph link {
    rankdir = LR;  //让图片横过来

    node[shape = record];//record形状是专门用来做类似”结构体“的东西的
    1[label = "{1|}"];//每个的'|'都是一列
    2[label = "{2|}"];
    3[label = "{3|}"];
    4[label = "{4|}"];
    5[label = "{5|}"];
    6[label = "{6|}"];
    7[label = "{7|}"];
    8[label = "{8|NULL}"];
    1->2:w;
    2->3:w;
    3->4:w;
    4->5:w;
    5->6:w;
    6->7:w;
    7->8:w;
}
```

## 跳表的增删改查

## 跳表的优势

# Redis 中跳表的实现
