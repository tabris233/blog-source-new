---
title: "LeetCode 第252场周赛"
date: 2021-08-01 11:57:58
updated: 2021-08-01 11:57:58
description: ["LeetCode 第252场周赛。。 做的有点慢"]
toc: true
author: tabris
img: https://fastly.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/08/01/20210801111711.png
katex: true
summary:
categories:
    - leetcode
tags:
    - 周赛
    - 双周赛

---

![image-20210801111711882](https://fastly.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/08/01/20210801111711.png)

# [5830. 三除数](https://leetcode-cn.com/contest/weekly-contest-252/problems/three-divisors/)

遍历下1->n， 统计下约数个数，最后看是否等于3就好了

```cpp
class Solution {
public:
    bool isThree(int n) {
        int ans = 0;

        for(int i=1; i<=n; i++) {
            if (n%i == 0) ans ++;
        }

        return ans == 3;
    }
};
```

# [5831. 你可以工作的最大周数](https://leetcode-cn.com/contest/weekly-contest-252/problems/maximum-number-of-weeks-for-which-you-can-work/)

取最大值， 看最大值能否被其他穿插开就好。 穿插不开就要舍弃不能连续的部分（值变成sum-mx+1）。

```cpp
class Solution {
public:
    typedef long long int LL;

    long long numberOfWeeks(vector<int>& milestones) {
        LL sum = 0;
        int mx = 0;
        for(auto a: milestones) {
            sum += a;
            mx = max(mx, a);
        }

        sum -= mx;

        if(mx > sum + 1) {
            mx = sum + 1;
        }

        return mx + sum;
    }
};
```

# [5832. 收集足够苹果的最小花园周长](https://leetcode-cn.com/contest/weekly-contest-252/problems/minimum-garden-perimeter-to-collect-enough-apples/)

经典的二分， 对花园的半径（<0,0> 到一边的长度）二分。 最后计算答案就好

```cpp
class Solution {
public:
    long long sum(long long x) {
        long long ans = (0+x)*(x+1)/2;

        return ans * 4 * (x*2+1);
    }

    long long minimumPerimeter(long long neededApples) {
        long long l = 1,r = 1e5,mid = -1, ans = 1 ;
        while(l<=r) {
            mid = (r+l)/2;
            if(sum(mid) >= neededApples) {
                ans = mid;
                r = mid-1;
            } else {
                l = mid +1;
            }
        }

        return ans * 8;
    }
};
```

# [5833. 统计特殊子序列的数目](https://leetcode-cn.com/contest/weekly-contest-252/problems/count-number-of-special-subsequences/)

dp问题；$dp[1e5][3]$

设$dp[i][j]$ 表示到第$i$个位置的以结尾的序列个数。



对于非当前位置a

1.  $dp[i][j] = dp[i-1][j]$ $\ \ \ j \in \{0, 1,2\}, j != a$

对于当前位置为a

2.  $dp[i][a] = dp[i-1][a]*2 + dp[i-1][a-1]$



```cpp
class Solution {
public:
    static const int N = 1e5+7;
    static const int MOD = 1e9+7;

    long long dp[N][3];

    int countSpecialSubsequences(vector<int>& nums) {

        int i = 0;
        for(auto a: nums) {
            ++i;

            for(int j=0; j<3; j++) dp[i][j] = dp[i-1][j];

            dp[i][a] = dp[i-1][a]*2 + ((a==0)?1:dp[i-1][a-1]);
            dp[i][a] %= MOD;
        }

        return dp[nums.size()][2];
    }
};
```
