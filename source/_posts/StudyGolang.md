---
title: StudyGolang
date: 2019-11-08 15:33:11
update: 2021-08-08 11:46:11
description: ["Golang"]
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
  - 编程语言
tags:
  - Golang
---

中文：在线入门指引
http://go-tour-zh.appspot.com/basics/1
中文：文档首页
https://go-zh.org/doc/
中文首页
https://go-zh.org/
英文首页
https://golang.org/

[Go 中文主页](https://go-zh.org/)

先把这个教程看完 - [Go 指南](https://tour.go-zh.org/)

<!-- more -->

然后就去写代码

完毕

------

# go 语言常用的一些代码 (刷题

### 自定义排序

[【Go语言】基本类型排序和 slice 排序](https://itimetraveler.github.io/2016/09/07/【Go语言】基本类型排序和 slice 排序/)

```go
package main

import (
    "fmt"
    "sort"
)

type Person struct {
    Name string
    Age  int
}

// 按照 Person.Age 从大到小排序
type PersonSlice [] Person

func (a PersonSlice) Len() int {    	 // 重写 Len() 方法
    return len(a)
}
func (a PersonSlice) Swap(i, j int){     // 重写 Swap() 方法
    a[i], a[j] = a[j], a[i]
}
func (a PersonSlice) Less(i, j int) bool {    // 重写 Less() 方法， 从大到小排序
    return a[j].Age < a[i].Age
}

func main() {
    people := [] Person{
        {"zhang san", 12},
        {"li si", 30},
        {"wang wu", 52},
        {"zhao liu", 26},
    }

    fmt.Println(people)

    sort.Sort(PersonSlice(people))    // 按照 Age 的逆序排序
    fmt.Println(people)

    sort.Sort(sort.Reverse(PersonSlice(people)))    // 按照 Age 的升序排序
    fmt.Println(people)

}
```

### 比较大小

```go
func max(a, b int) int {
	if a > b {
		return a
	} else {
		return b
	}
}

func min(a, b int) int {
	if a < b {
		return a
	} else {
		return b
	}
}
```

### 模拟C++的Vector

```go
/**
 * 一维
 * 初始长度为空
 */
ans := make([]int, 0)      // 相当于C++ vector<int> ans;
ans = append(ans, 23)      // 相当于C++ ans.push_back(23);

/**
 * 二维切片
 * LeetCode 5280
 */
func groupThePeople(groupSizes []int) [][]int {
    ans := make([][]int, 0)            // 相当于C++ vector<vector<int> > ans;

    cnt := make([][]int, 500)          // 相当于C++ vector<int> cnt[500];

    for i, v := range groupSizes {
        cnt[v] = append(cnt[v], i)     // 相当于C++ cnt[v].push_back(i);
        if len(cnt[v]) == v {
            ans = append(ans, cnt[v])  // 相当于C++ ans.push_back(cnt[v]);
            cnt[v] = make([]int, 0)    // 相当于C++ cnt.clear();
        }
    }
    return ans
}
```


--------

# 刷题板子


## LCS

```go
    int dp[1005][1005];
    int lcs(string word1, string word2) {
        int ls1 = word1.size(), ls2 = word2.size();
        for(int i = 1; i<=ls1; i++) for(int j = 1; j<=ls2; j++) {
            if (word1[i-1] == word2[j-1]) {
                dp[i][j] = dp[i-1][j-1] + 1;
            } else {
                dp[i][j] = max(dp[i-1][j], dp[i][j-1]);
            }
        }
        return dp[ls1][ls2];
    }
```

## 二进制枚举

```go
// 双层二进制枚举
            for(int j = 1; j < (1 << n); j += 1)
                for(int x = j; x; x = (x - 1) & j)  // 这里有个优化
```

## 并查集

```go
// 并查集
type unionSet struct {
    fa []int
    // length_ []int
}

// func (u *unionSet) length(x int) int {
// return u.length_[u.find(x)]
// }

func NewUnionSet(n int) unionSet {
    fa := make([]int, n)
    for i := range fa {
        fa[i] = i
    }
    // length_ := make([]int, n)
    // for i := range length_ {
    //     length_[i] = 1
    // }
    return unionSet{fa}
}

func (u *unionSet) Find(x int) int {
    if x != u.fa[x] {
        u.fa[x] = u.Find(u.fa[x])
    }
    return u.fa[x]
}

func (u *unionSet) Join(x, y int) {
    fx, fy := u.Find(x), u.Find(y)

    if fx != fy {
        u.fa[fy]=fx
        // if u.length(fx) > u.length(fy) {
        //     u.fa[fy]=fx
        // } else {
        //     u.fa[fx]=fy
        // }
        // u.length_[fx], u.length_[fy] = u.length(fx) + u.length(fy), u.length(fx) + u.length(fy)
    }
}
```

## 自定义结构体+排序

```go
// 自定义结构体+排序
type box struct {
    num int
    size int
}

type boxSlice []box
func(t boxSlice) Len() int {return len(t)}
func(t boxSlice) Less(i, j int) bool {return t[i].size > t[j].size}
func(t boxSlice) Swap(i, j int) {t[i], t[j] = t[j], t[i]}

// eg:
    a := []box
    sort.Sort(boxSlice(a))
```

## LIS

```go
// lis
lis := []int{}
for _, v := range arr {
  p := sort.SearchInts(lis, v+1)  // 二分查找 第一个 >= v+1 的位置
                                  // v 最长上升子序列。
                                  // v+1 最长不下降子序列。
  if p < len(lis) {
    lis[p] = v
  } else {
    lis = append(lis, v)
  }
}
```

## BIT(树状数组)

```go
type BIT struct {
    sum [100009]int
}

func lowbit(x int) int {
    return x&-x
}

func (bit *BIT) up(i, v int) {
    for ;i<=100000; i+=lowbit(i) {
        bit.sum[i]+=v
    }
}

func (bit *BIT) get(i int) int {
    ans := 0
    for ; i>0; i-=lowbit(i) {
        ans += bit.sum[i]
    }
    return ans
}
```

## 离散化

```go
func lisanhua(nums []int) []int {
	temp := make([]int, len(nums))  // 这里的cap 要提前申请好。
	copy(temp, nums)

	sort.Ints(temp)
	lt := 0
	for i := range temp {
		if i > 0 && temp[i] == temp[i-1] {
			continue
		}

		temp[lt] = temp[i]
		lt++
	}

	for i := range nums {
		nums[i] = sort.SearchInts(temp, nums[i]) + 1
	}

	return nums
}
```

