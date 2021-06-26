---
title: "LeetCode 第55场双周赛"
date: 2021-06-26 23:54:09
Updated: 2021-06-26 23:54:09
description: ["LeetCode 第55场双周赛。。 翻车啊啊啊啊啊"]
toc: true
author: tabris
# 图片推荐使用图床(腾讯云、七牛云、又拍云等)来做图片的路径.如:http://xxx.com/xxx.jpg
img: https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/05/16/20210516114449.png
# 如果top值为true,则会是首页推荐文章
top: false
# 如果要对文章设置阅读验证密码的话,就可以在设置password的值,该值必须是用SHA256加密后的密码,防止被他人识破
password:
# 本文章是否开启mathjax，且需要在主题的_config.yml文件中也需要开启才行
katex: true
summary:
categories:
    - leetcode
tags:
    - 周赛
    - 双周赛

---

![image-20210626235353301](https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/06/26/20210626235353.png)

GG

# [5780. 删除一个元素使数组严格递增](https://leetcode-cn.com/contest/biweekly-contest-55/problems/remove-one-element-to-make-the-array-strictly-increasing/)

暴力一下，

枚举每个元素删除后的数据是不是严格递增的， 数据范围是$1000$. 平凡才$1e6$没问题的。

```cpp
class Solution {
public:
    
    bool f(vector<int>& a) {
        for(int i=1;i<a.size();i++) {
            if(a[i] <= a[i-1]) return false;
        }
        return true;
    }    
    bool canBeIncreasing(vector<int>& nums) {
        int n = nums.size();
        vector<int> a;
        for(int i = 0;i<n;i++) {
            a.clear();
            for(int j=0;j<n;j++) {
                if(i == j) continue;
                a.push_back(nums[j]);
            }
            if(f(a)) return true;
        }
            
        return false;
    }
};
```



# [5781. 删除一个字符串中所有出现的给定子字符串](https://leetcode-cn.com/contest/biweekly-contest-55/problems/remove-all-occurrences-of-a-substring/)

开始还想要不要有啥骚操作， 后面觉得暴力没啥问题，然后暴力就过了，，

```cpp
class Solution {
public:
    string removeOccurrences(string s, string part) {
        int lp = part.size();
        string ss;
        int found;
        while((found = s.find(part)) != string::npos) {
            ss.clear();
            for(int i = 0; i<s.size(); i++) {
                if(found <= i && i< found + lp) continue;
                ss+=s[i];
            }
            s=ss;
        }
        return s;
    }
};
```



# [5782. 最大子序列交替和](https://leetcode-cn.com/contest/biweekly-contest-55/problems/maximum-alternating-subsequence-sum/)

简单dp？吧，

**dp[0]** 表示以长度为**偶数**的子序列的最大值。

**dp[1]** 表示以长度为**奇数**的子序列的最大值。

遍历每个数字，计算当前这个数字为奇数或者偶数的最大值， 奇数就是dp[0]减去它，偶数就是d[1]加上它。

```cpp
class Solution {
public:
    long long maxAlternatingSum(vector<int>& nums) {
        vector<long long> dp(2, 0), dp2(2, 0);
        dp[0] = dp2[0] = -1e9;
        for(auto a: nums) {
            dp2[1] = max(dp[0] - a, dp[1]);
            dp2[0] = max(dp[1] + a, dp[0]);
            
            dp = dp2;
        }
        return max(dp[0], dp2[1]);
    }
};
```



# [5783. 设计电影租借系统](https://leetcode-cn.com/contest/biweekly-contest-55/problems/design-movie-rental-system/)

模拟题

我是用了一个 map[(shop, movie)] = price，其中price的政府表示借出还是没借出。借出为负，没借出为正。

用一个priority_queue<tuple<price，shop，movie >> 的小顶堆维护借出的。

用n个priority_queue<tuple<price，shop>>的小顶堆维护没借出这个电影的商店。

```cpp
class MovieRentingSystem {
public:
    map<tuple<int,int>, int> vis;  // <shop, movie> 没借出+， 借出-;
    priority_queue<tuple<int,int,int>, vector<tuple<int,int,int>>, greater<tuple<int,int,int>>> jc; // price，shop，movie 
    priority_queue<tuple<int,int>, vector<tuple<int,int>>, greater<tuple<int,int>>> wj[300005]; // 【movie】 price，shop
    
    MovieRentingSystem(int n, vector<vector<int>>& entries) {
        for(auto &entry: entries) {
            int shop = entry[0], movie = entry[1], price = entry[2];
            
            vis[tuple(shop, movie)] = price;  // 唯一 键
            wj[movie].push(tuple(price, shop));
        }
    }
    
    vector<int> search(int movie) {
        vector<int> ans;

        vector<tuple<int,int>> tmp;
        map<tuple<int,int>, bool> book;

        int cnt = 0;
        while(cnt < 5 && !wj[movie].empty()) {
            auto [price, shop] = wj[movie].top(); wj[movie].pop();
            if(vis[tuple(shop, movie)] < 0 || book[tuple(shop, movie)] == true) {
                continue;
            }
            ++cnt;
            book[tuple(shop, movie)] = true;
            tmp.push_back(tuple(price, shop));
            ans.push_back(shop);
        }
        
        for(auto t: tmp) {
            wj[movie].push(t);
        }
        
        return ans;
    }
    
    void rent(int shop, int movie) {
        int price = vis[tuple(shop, movie)];  //这时候一定是 正数
        vis[tuple(shop, movie)] = -price;

        jc.push(tuple(price, shop, movie));
    }
    
    void drop(int shop, int movie) {
        int price = vis[tuple(shop, movie)];  //这时候一定是 负数
        vis[tuple(shop, movie)] = -price;
        
        wj[movie].push(tuple(-price, shop));
    }
    
    vector<vector<int>> report() {
        vector<vector<int>> ans;
        vector<int> t(2);
        map<tuple<int,int>, bool> book;
        
        vector<tuple<int,int,int> > tmp;
        int cnt = 0;
        while(cnt < 5 && !jc.empty()) {
            auto [price, shop, movie] = jc.top(); jc.pop();
            if(vis[tuple(shop, movie)] > 0 || book[tuple(shop, movie)] == true) {
                continue;
            }
            ++cnt;
            tmp.push_back(tuple(price, shop, movie));
            book[tuple(shop, movie)] = true;
            t[0] = shop, t[1] = movie;
            ans.push_back(t);
        }
        for(auto tt: tmp) {
            jc.push(tt);
        }
        
        return ans;
    }
};

/**
 * Your MovieRentingSystem object will be instantiated and called as such:
 * MovieRentingSystem* obj = new MovieRentingSystem(n, entries);
 * vector<int> param_1 = obj->search(movie);
 * obj->rent(shop,movie);
 * obj->drop(shop,movie);
 * vector<vector<int>> param_4 = obj->report();
 */
```

