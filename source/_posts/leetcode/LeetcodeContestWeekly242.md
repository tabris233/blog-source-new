---
title: "LeetCode 第242场周赛"
date: 2021-05-23 12:54:04
description: ["LeetCode 第242场周赛。。 彻底翻车"]
toc: true
author: tabris
img: https://fastly.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/05/16/20210516113536.png
katex: true
summary:
categories:
    - leetcode
tags:
    - 周赛
    - 双周赛
---

>    彻底翻车了
>
>   最后一个都没写出来。。
>
>   找个理由强行解释下，可能是没睡好hhh。

![image-20210523122642357](https://fastly.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/05/23/20210523122642.png)

# [5763 哪种连续子字符串更长](https://leetcode-cn.com/problems/longer-contiguous-segments-of-ones-than-zeros/)

简单模拟， 不过我这里写的太麻烦了。

```cpp
class Solution {
public:
    bool checkZeroOnes(string s) {
        char p = '2';
        int mx1 = 0, mx0 = 0;
        int sum1 = 0, sum0 = 0;
        for(auto c: s) {
            if(c == p){
                if(c == '0'){
                    sum0++;
                    mx0 = max(mx0, sum0);
                } else if(c == '1') {
                    sum1++;
                    mx1 = max(mx1, sum1);
                }
            } else {
                if(c == '0'){
                    mx0 = max(mx0, sum0);
                    sum0=1;
                    mx0 = max(mx0, sum0);
                } else if(c == '1') {
                    mx1 = max(mx1, sum1);
                    sum1=1;
                    mx1 = max(mx1, sum1);
                }
            }
            p = c;
        }
        
        return mx1 > mx0;
    }
};
```



# [5764 准时到达的列车最小时速](https://leetcode-cn.com/problems/minimum-speed-to-arrive-on-time/)

二分答案。 注意浮点数二分用次数就好了。



```cpp
class Solution {
public:
    bool check(vector<int>& dist, double hour, double x) {
        double time = 0;
        for(auto a: dist) {
            time = ceil(time);
            time += a/x;
        }
        
        return time <= hour;
    }
    
    int minSpeedOnTime(vector<int>& dist, double hour) {
        double l = 0, r = 1e9, mid;
        bool flag = false;
        for(int i=0;i<100;i++) {
            mid = (l+r) / 2.0;
            if(check(dist, hour, mid)) r = mid, flag = true;
            else l=mid;
        }
        
        if(flag) return ceil(r);
        else return -1;
    }
};
```



# [5765 跳跃游戏 VII](https://leetcode-cn.com/problems/jump-game-vii/)

这里因为`minJump`和`maxJump`的取值范围很有可能使$maxJump - minJump \in [0, 1e5]$ , 所以不建议直接bfs。

这里可以参考扫描线维护区间覆盖的方式逐步遍历过去。同时判断每个位置是否被覆盖了。 

```cpp
class Solution {
public:
    bool canReach(string s, int minJump, int maxJump) {
        int n = s.size();
        
        vector<int> vis(n+1);
        bool flag = false;
        for(int i=0;i<n;i++) {
            if(i) vis[i] += vis[i-1];
            if(s[i] == '0' && (vis[i]>0 || i==0)) {
                vis[min(n, i+minJump)] ++;
                vis[min(n-1,i+maxJump)+1] --;
            }
        }
        
        // for(int i=0;i<=n;i++) printf("%d ", vis[i]); puts("");
        
        return s[n-1] == '0' && vis[n-1]>0;
    }
};
```



# [5766 石子游戏 VIII](https://leetcode-cn.com/problems/stone-game-viii/)

这里最开始脑抽了，以为可以讨论出来，后面才想起来得DP，（DP水平实在太差了）

这里设$dp(i)$ 表示 第$i$个位置已经是新的合并后的石子时先手的最大差值。

>   这里注意下，我一直觉得这里很绕。
>
>   当轮到后手时因为他的值在差值里时被减的，后手又想要差值尽可能小。
>
>   这个时候如果不考虑之前的状态，那么可以说明这个后手就是现在的`先手`, 所以后手的选择策略和先手时一样的。

转移方程为：

$dp(i) = \{MAX(-dp(j) + (\sum_{idx=i}^{j}stone[idx] + \sum_{idx=0}^{i-1}stone[idx]  )\ ) , j \in [i+1, n)\}$

这一手货的的积分减去下一手的最大差值

这里因为题目条件有个这个

>   3.  将一个 **新的石子** 放在最左边，且新石子的值为被移除石子值之和。

所以第$i$个位置其实已经是 $\sum_{idx=0}^{i} stone[idx]$ 了。

化简一下转移方程就是：

$dp(i) = \{MAX(-dp(j) + \sum_{idx=0}^{j}stone[idx]   ) , j \in [i+1, n)\}$

连续和维护一个前缀和$sum(i)$就好了。

最终转移就是

$dp(i) = max(-dp(j) + sum[j]), j \in [i+1, n)$ 

最大值可以从后遍历维护，总的复杂度是$O(N)$的

```cpp
class Solution {
public:
    int stoneGameVIII(vector<int>& a) {
        int n = a.size();
        vector<int> sum(n+1);
        for(int i=0;i<n;i++) sum[i+1] = sum[i] + a[i];

        vector<int> dp(n+1); 

        dp[n] = 0;
        int mx = sum[n];
        for(int i=n-1;i>0;i--) {
            dp[i] = mx;
            mx = max(mx, -dp[i]+sum[i]);
        }
        
        return dp[1];
    }
};
```

