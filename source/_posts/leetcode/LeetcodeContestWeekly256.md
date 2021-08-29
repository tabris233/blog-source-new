---
title: "LeetCode 第256场周赛"
date: 2021-08-29 11:41:28
updated: 2021-08-29 11:41:28
description: ["LeetCode 第256场周赛。。 "]
toc: true
author: tabris
img: https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/08/29/20210829114205.png
katex: true
summary:
categories:
    - leetcode
tags:
    - 周赛
    - 双周赛

---

![image-20210829114205849](https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/08/29/20210829114205.png)



# [5854. 学生分数的最小差值](https://leetcode-cn.com/contest/weekly-contest-256/problems/minimum-difference-between-highest-and-lowest-of-k-scores/)

排序之后 遍历一遍每个大小为k的区间， 然后计算下就好

```cpp
class Solution {
public:
    int minimumDifference(vector<int>& nums, int k) {
        sort(nums.begin(), nums.end());
        
        int ans = nums[nums.size()-1] - nums[0];
        for(int i=k-1; i<nums.size(); i++) {
            ans = min(ans, nums[i]-nums[i-k+1]);
        }
        
        return ans;
    }
};
```



# [5855. 找出数组中的第 K 大整数](https://leetcode-cn.com/contest/weekly-contest-256/problems/find-the-kth-largest-integer-in-the-array/)

还是排序， 注意字符串比较和整数不同，需要补上前导`0`. 最后别忘了把前导`0`去掉。

```cpp
class Solution {
public:
    string kthLargestNumber(vector<string>& nums, int k) {
        
        int mx = 1;
        for(auto a: nums) {
            mx = max(mx, int(a.size()));
        }
        
        for(int i=0; i<nums.size(); i++) {
            string &a = nums[i];
            while(a.size() < mx) {
                a = "0" + a;
            }
        }
        
        sort(nums.begin(), nums.end());
        
        string ans = nums[nums.size()-k];
        
        int p = 0;
        for(p=0; p<ans.size() && ans[p]=='0'; p++) {
            
        }
        
        ans = ans.substr(p, ans.size()-p);
        
        return (ans.empty()?"0":ans);
    }
};
```



# [5856. 完成任务的最少工作时间段](https://leetcode-cn.com/contest/weekly-contest-256/problems/minimum-number-of-work-sessions-to-finish-the-tasks/)

经典状压dp

dp[i] 表示 在i的二进制下 干了对应的任务所花的最少工作时间段。

```cpp
class Solution {
public:
    
    int minSessions(vector<int>& tasks, int sessionTime) {
        int n = tasks.size();
        
        vector<int> dp(1<<n), cost(1<<n);
        
        int t = 1<<n;
        for(int i=1, sum; i < t; i++) {
            dp[i] = 100000000;
            
            sum = 0;
            for(int j=0; j<n; j++) {
                if(i>>j&1) {
                    sum += tasks[j];
                }
            }
            cost[i] = sum;
        }
        

        dp[0] = cost[0] = 0;
        for(int i=1; i < t; i++) {
            for(int j=i; j; j = (j-1)&i) {
                if(cost[j] <= sessionTime) {
                    dp[i] = min(dp[i], 
                                dp[i^j] + 1);
                }
            }
        }
        
        return dp[t-1];
        
    }
};
```



# [5857. 不同的好子序列数目](https://leetcode-cn.com/contest/weekly-contest-256/problems/number-of-unique-good-subsequences/)

类似[940. 不同的子序列 II](https://leetcode-cn.com/problems/distinct-subsequences-ii/)， 针对0特殊处理下就好了。

```cpp
class Solution {
public:
    static const int MOD = 1e9+7;
    
    int numberOfUniqueGoodSubsequences(string s) {
        int n = s.length();
        
        int ans = 0;
        for(auto c: s) {
            if (c == '0') {
                ans = 1;
            }
        }
        
        vector<int> dp(n+1);
        vector<int> last(2, -1);
        
        dp[0] = 1;
        for(int i=0; i<n; i++) {
            int x = s[i] - '0';
            if(x == 1) {
                dp[i+1] = dp[i] * 2 % MOD;
                if (last[x] >= 0) {
                    dp[i+1] = (dp[i+1] - dp[last[x]]+MOD) % MOD;
                }
            } else { // if(x == 0)
                dp[i+1] = (dp[i]*2-1) %MOD;
                if (last[x] >= 0) {
                    dp[i+1] = (dp[i+1] - dp[last[x]] + 1+MOD) % MOD;
                }
            }
            last[x] = i;
        }
        
        return (dp[n]+ans-1)%MOD;
    }
};
```

