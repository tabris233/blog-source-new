---
title: "LeetCode 第51场双周赛"
date: 2021-05-01 23:54:58
description: ["LeetCode 第51场双周赛。。 菜死了啊"]
toc: true
author: tabris
# 图片推荐使用图床(腾讯云、七牛云、又拍云等)来做图片的路径.如:http://xxx.com/xxx.jpg
img: https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/05/01/20210501233929.png
# 如果top值为true,则会是首页推荐文章
top: false
# 如果要对文章设置阅读验证密码的话,就可以在设置password的值,该值必须是用SHA256加密后的密码,防止被他人识破
password:
# 本文章是否开启mathjax，且需要在主题的_config.yml文件中也需要开启才行
mathjax: false
summary:
categories:
    - leetcode
tags:
    - 周赛
    - 双周赛
---

![](https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/05/01/20210501233929.png)

GG

# 5730. 将所有数字用字符替换

简单模拟题目

```cpp
class Solution {
public:
    string replaceDigits(string s) {
        for(int i=0;i<s.size(); i+=2) {
            s[i+1] = s[i]+(s[i+1]-'0');
        }
        return s;
    }
};
```

# 5731. 座位预约管理系统

数据结构题， 维护一个set就可以了

```cpp
class SeatManager {
    set<int> s;
public:
    SeatManager(int n) {
        s.clear();
        for(int i=1;i<=n;i++) s.insert(i);
    }

    int reserve() {
        int mn = *s.begin();
        s.erase(mn);
        return mn;
    }

    void unreserve(int x) {
        s.insert(x);
    }
};
```

# 5732. 减小和重新排列数组后的最大元素

开始想都没想就搞二分了。。。搞了半天

后面发现只要拍个序 模拟一下就好了， 如果这个一个减上一个超过1， 就变成上一个的值+1.

没看到题目说 第一个 必须为 1  wa了一发。

```cpp
class Solution {
public:
    int maximumElementAfterDecrementingAndRearranging(vector<int>& arr) {
        int n = arr.size();
        sort(arr.begin(), arr.end());
        arr[0]=1;
        for(int i=1;i<arr.size();i++) {
            if(arr[i] > arr[i-1]+1) {
                arr[i] = arr[i-1]+1;
            }
        }

        return arr[n-1];
    }
};
```

# 5733. 最近的房间

这个题本身也不难， 数据和查询两个数组按照size从大到小拍个序就好了。 搞个set维护roomId。 set自带的二分查找下就行了。  因为是绝对值就再维护个负值的set(后面看了其他大佬的代码发现迭代器减一下就行了)。

不太会用set， 不会写c++的匿名函数，， 后面判断逻辑又少了， GG。

```cpp
bool cmp(vector<int>& a, vector<int>& b) {
    return a[1] > b[1];
}

class Solution {
public:
    vector<int> closestRoom(vector<vector<int>>& rooms, vector<vector<int>>& q) {
        vector<int> ans(q.size());

        for(int i = 0; i<q.size(); i++) {
            q[i].push_back(i);
            ans[i] = -1;
        }
        sort(rooms.begin(), rooms.end(), cmp);
        sort(q.begin(), q.end(), cmp);

        int p = 0,as,asf;
        set<int> s, sf;

        for(auto& qq: q) {
            for(;p<rooms.size() && rooms[p][1] >= qq[1]; p++) {
                s.insert(rooms[p][0]);
                sf.insert(-rooms[p][0]);
            }

            as = asf = -1;
            auto it = s.lower_bound(qq[0]);
            if(it!=s.end()) {
                as = (*it) - qq[0];
                ans[qq[2]] = (*it);
            }

            auto it2 = sf.lower_bound(-qq[0]);
            if(it2 != sf.end()) {
                asf = (*it2) - (-qq[0]);
                if(ans[qq[2]] == -1 || asf <= as) ans[qq[2]] = -(*it2);
            }
        }

        return ans;
    }
};
```