---
title: "LeetCode 第241场周赛"
date: 2021-05-16 12:00:00
description: ["LeetCode 第241场周赛。。 菜死了啊"]
toc: true
author: tabris
img: https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/05/16/20210516113536.png
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

![image-20210516113536256](https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/05/16/20210516113536.png)

GG

# 5759. 找出所有子集的异或总和再求和

模拟题目

二进制枚举所有集合 计算每个集合的异或和， 最后加起来就好

```cpp
class Solution {
public:
    int subsetXORSum(vector<int>& nums) {
        int n = nums.size();
        int ans = 0;
        for(int i=1;i<(1<<n);i++) {
            int tmp = 0;
            for(int j=0;j<n;j++) {
                if(i>>j&1) tmp ^= nums[j];
            }
            ans += tmp;
        }
        return ans;
    }
};
```

# 5760. 构成交替字符串需要的最小交换次数

模拟就好， 我这里写的太麻烦了。

先判断下`1` 和`0`的个数是否是差值不大于1的. 否则无解。

然后在分情况讨论下 看和`1010101` 或者 `0101010` 的差异个数， 除 2 就是了。

```cpp
class Solution {
public:
    int minSwaps(string s) {
        // 111000
        // 101010
        int a0 = 0, a1 = 0;
        string ss;
        for(auto c: s) {
            if(c == '0') {
                a0 ++;
            } else {
                a1 ++;
            }
        }
        if(abs(a0-a1)>1) return -1;
        else if(a0 == a1){
            int ans1 = 0, ans2 = 0;
            //. 0101010   1010101
            for(int i=0;i<s.size();i++) {
                if(i&1) {
                    if(s[i] != '1') ans1++;
                    if(s[i] != '0') ans2++;
                } else {
                    if(s[i] != '0') ans1++;
                    if(s[i] != '1') ans2++;
                }
            }
            // printf("%s | %d %d\n", s.c_str(), ans1, ans2);
            return min(ans1, ans2)/2;
        } else {
            if(a0 > a1) {
                int ans1 = 0;
                //. 0101010
                for(int i=0;i<s.size();i++) {
                    if(i&1) {
                        if(s[i] != '1') ans1++;
                    } else {
                        if(s[i] != '0') ans1++;
                    }
                }
                return ans1/2;
            } else {
                int ans2 = 0;
                //. 1010101
                for(int i=0;i<s.size();i++) {
                    if(i&1) {
                        if(s[i] != '0') ans2++;
                    } else {
                        if(s[i] != '1') ans2++;
                    }
                }
                return ans2/2;
            }
        }
    }
};
```

# 5761. 找出和为指定值的下标对

数据结构实现题，这里观察下数据范围， 应该容易想到可以遍历`nums1` 同时将 `nums2`中的元素放入`map`中。



```cpp
class FindSumPairs {
    int ans;
    vector<int> a1, a2;
    map<int, int> mp;
public:
    FindSumPairs(vector<int>& nums1, vector<int>& nums2) {
        a1 = nums1;
        a2 = nums2;
        for(auto num: nums2) {
            mp[num] ++;
        }
    }

    void add(int index, int val) {
        mp[a2[index]] --;
        a2[index] += val;
        mp[a2[index]] ++;
    }

    int count(int tot) {
        int ans = 0;
        for(auto a: a1) {
            if(tot-a>0 && mp.count(tot-a)) {
                ans += mp[tot-a];
            }
        }
        return ans;
    }
};

/**
 * Your FindSumPairs object will be instantiated and called as such:
 * FindSumPairs* obj = new FindSumPairs(nums1, nums2);
 * obj->add(index,val);
 * int param_2 = obj->count(tot);
 */
```

# 5762. 恰有 K 根木棍可以看到的排列数目

这个题最开始想偏了， 当成了 LIS 的个数， 想了半天不回做， 后来改了回来。

首先打了个表， 看下结果的规律， 然后就写了， 这里和杨辉三角差不多。

```cpp
class Solution {
public:
    static const int MOD = 1e9+7;
    void test() {
        for(int i=1;i<=10;i++) {
            vector<int> a(i);
            vector<int> dp(i);
            vector<int> ans(i+1);
            for(int j=0;j<i;j++) a[j]=j;
            do{
                int sum = 1, mx = a[0];
                for(int j=1;j<i;j++) {
                    if(a[j]>mx) {
                        sum++;
                        mx = a[j];
                    }
                }
                ans[sum]++;
            }while(next_permutation(a.begin(), a.end()));

            for(auto x: ans) printf("%d ", x); puts("");
        }
    }
    // int f[1001][1001];
    int g[1001][1001];
    void init() {
        g[1][1] = 1;
        for(int i=2;i<=1000;i++) {
            for(int j=1;j<=i;j++) {
                // f[i][j] = (f[i-1][j] + f[i-1][j-1])%MOD;
                g[i][j] = (0LL+g[i-1][j-1] + 1LL*(i-1)*g[i-1][j]%MOD)%MOD;
            }
            // for(int j=1;j<=i;j++) printf("%d ", f[i][j]);puts("");
            // for(int j=1;j<=i;j++) printf("%d ", g[i][j]);puts("");
        }

    }
    int rearrangeSticks(int n, int k) {
        // test();
        init();
        return g[n][k];
    }
};
```