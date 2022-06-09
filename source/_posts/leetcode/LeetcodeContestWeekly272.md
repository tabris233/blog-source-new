---
title: "LeetCode 第272场周赛"
date: 2021-12-19 11:19:23
updated: 2021-12-19 11:40:23
description: ["LeetCode 第272场周赛。纯手速场，我中间还蹲了个大号。。"]
toc: true
author: tabris
img: https://fastly.jsdelivr.net/gh/tabris233/cdn-assets/PicGo20211219110235.png
katex: true
summary:
categories:
    - leetcode
tags:
    - 周赛
    - 双周赛

---

![image-20210829114205849](https://fastly.jsdelivr.net/gh/tabris233/cdn-assets/PicGo20211219110235.png)



[5956. 找出数组中的第一个回文字符串](https://leetcode-cn.com/contest/weekly-contest-272/problems/find-first-palindromic-string-in-the-array/)

回文串判定不在赘述，遍历下字符串数组找到第一个回文串返回即可。

```cpp
class Solution {
public:
    bool check(string word) {
        int n = word.length();
        for (int i=0, j=n-1; i<j; i++, j--) {
            if (word[i] != word[j]) return false;
        }
        return true;
    }
    string firstPalindrome(vector<string>& words) {
        for (auto word: words) {
            if (check(word)) return word;
        }

        return "";
    }
};
```

[5957. 向字符串添加空格](https://leetcode-cn.com/contest/weekly-contest-272/problems/adding-spaces-to-a-string/)

简单模拟题，整一个空字符串，不断追加即可，

遍历原字符串，索引在`spaces`中的时候就先加一个空格。

考虑spaces数组是有序的，维护一个`spaces`的下标，一个个比对就好。

```cpp
class Solution {
public:
    string addSpaces(string s, vector<int>& spaces) {
        string ans = "";
        int p = 0;
        for (int i=0; i<s.length(); i++) {
            if (p < spaces.size() && i == spaces[p]) {
                ans += " ";
                p++;
            }
            ans += s[i];
        }

        return ans;
    }
};
```

[5958. 股票平滑下跌阶段的数目](https://leetcode-cn.com/contest/weekly-contest-272/problems/number-of-smooth-descent-periods-of-a-stock/)

对于第`i`天的股票价格，如果有连续`k`天下降为$1$，对结果的贡献就是`k`。

eg: $[3,2,1]$ 中第$3$天的价格是$1$，那么多了的平滑下降阶段就有$[3,2,1]，[2,1]，[1]$，$3$个。

```cpp
class Solution {
public:
    long long getDescentPeriods(vector<int>& prices) {
        long long ans = 1;
        long long sum = 1;
        for(int i=1; i<prices.size(); i++) {
            if (prices[i] == prices[i-1]-1) sum ++;
            else sum = 1;
            ans += sum;
        }

        return ans;
    }
};
```

[5959. 使数组 K 递增的最少操作次数](https://leetcode-cn.com/contest/weekly-contest-272/problems/minimum-operations-to-make-the-array-k-increasing/)

首先要意识到，k把原数组分成了k个不相干的子数组。

子数组要成为连续不下降序列，那就可以求他的连续不下降子序列的值，然后再改其他值就行了。

```cpp
class Solution {
public:
    int lis (vector<int>& a) {
        int n = a.size();
        vector<int> b;

        b.push_back(a[0]);
        for(int i=1; i<a.size(); i++) {
            if (a[i] > b.back()) {
                b.push_back(a[i]);
                continue;
            }

            auto p = upper_bound(b.begin(), b.end(), a[i]);
            if (p == b.end()) {
                b.push_back(a[i]);
            } else {
                *p = a[i];
            }
        }

        // for(auto x: a) printf(" %d", x); puts("");
        // for(auto x: b) printf(" %d", x); puts("");

        return b.size();
    }

    int kIncreasing(vector<int>& arr, int k) {
        vector<vector<int> > a(k);

        for (int i=0; i<arr.size(); i++) {
            a[i%k].push_back(arr[i]);
        }

        int ans = 0;

        for (auto b: a) {
            ans += b.size() - lis(b);
        }

        return ans;
    }
};
```