---
title: "LeetCode 第243场周赛"
date: 2021-05-30 12:33:00
description: ["LeetCode 第243场周赛。。 再次翻车"]
toc: true
author: tabris
img: https://fastly.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/05/30/20210530121215.png
katex: true
summary:
categories:
    - leetcode
tags:
    - 周赛
    - 双周赛

---

>    再次翻车了
>
>    第四个卡精度搞了好久好久，，还是看了讨论区才处理精度问题的。。
>
>    第三个纯属自己代码写的太丑了，脑子不够清晰吧。赛后1分钟ac。。。

![image-20210530121215404](https://fastly.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/05/30/20210530121215.png)

# [5772. 检查某单词是否等于两单词之和](https://leetcode-cn.com/contest/weekly-contest-243/problems/check-if-word-equals-summation-of-two-words/)

简单模拟， 不过我这里写的太麻烦了。

```cpp
class Solution {
public:
    bool isSumEqual(string firstWord, string secondWord, string targetWord) {
        int a = 0, b = 0, t = 0;
        for(auto c: firstWord) {
            a = a*10 + (c-'a');
        }
        for(auto c: secondWord) {
            b = b*10 + (c-'a');
        }
        for(auto c: targetWord) {
            t = t*10 + (c-'a');
        }
        return a+b == t;
    }
};
```



# [5773. 插入后的最大值](https://leetcode-cn.com/contest/weekly-contest-243/problems/maximum-value-after-insertion/)

正负情况讨论一下就好了，

正数：

想要值最大就把这个数字放到第一个比他**小**的数字前面

负数

想要值最大就把这个数字放到第一个比他**大**的数字前面



```cpp
class Solution {
public:
    string maxValue(string n, int x) {
        string ans;
        if(n[0] == '-') {
            for(auto c: n) {
                if(x!=0 && c!='-' && c-'0' > x) {ans += '0'+x;x=0;}
                ans += c;
            }
        } else {
            for(auto c: n) {
                if(x!=0 && c-'0' < x) {ans += '0'+x;x=0;}
                ans += c;
            }
        }
        
        if(x != 0) ans += '0'+x;

        return ans;
    }
};
```



# [5774. 使用服务器处理任务](https://leetcode-cn.com/contest/weekly-contest-243/problems/process-tasks-using-servers/)

这里维护两个优先队列就好了，

一个维护`tuple<int, int>` 表示 权重，下标。这个就是空闲的服务器队列

另一个维护`tuple<int, int, int>` 表示 任务处理的时间，权重，下标。这个就是正在处理任务的服务器队列



过程中维护一个时间就行了， 如果空闲队列为空，就从非空闲的队列中吧做完任务的拿出来，如果没有做完的就调大下时间。

>   这里好像有卡长，我最开始写的比较丑，是`++time`这样玩儿的，因为值域是$2e5$ 所以这里`++time`的次数应该也是线性的才对，理论上不会增大复杂度的。



```cpp
class Solution {
public:
    vector<int> assignTasks(vector<int>& servers, vector<int>& tasks) {
        priority_queue<tuple<int, int>, vector<tuple<int, int>>, greater<tuple<int, int>>> q1;                   // weight, index
        priority_queue<tuple<int, int, int>, vector<tuple<int, int, int>>, greater<tuple<int, int, int> > > q2;  // time, weight, index.
        
        int idx = 0;
        for(auto server: servers) {
            q1.push({server, idx++});
        }

        vector<int> ans;
        
        int time = 0;
        int p = 0;
        for(; p <= time && p<tasks.size(); p++) {
            int &task = tasks[p];
            
            if(q1.empty()) time = max(time, std::get<0>(q2.top()));
            while(!q2.empty() && std::get<0>(q2.top()) <= time) {
                auto [t, w, i] = q2.top(); q2.pop();
                q1.push({w, i});
            }

            auto [weight, index] = q1.top(); q1.pop();
            q2.push({time+task, weight, index});
            ans.push_back(index);

            if(p >= time) time = max(time+1, p);
        }
        
        return ans;
    }
};
```



# [5775. 准时抵达会议现场的最小跳过休息次数](https://leetcode-cn.com/contest/weekly-contest-243/problems/minimum-skips-to-arrive-at-meeting-on-time/)

又是一个DP，，

不过这个DP很简单

设`dp[i][j]`表示前i个道路 跳过j个所花的时间

转移就是
$$
cur = \frac{dist[i]}{speed};
\\
dp[i][j] = max(dp[i-1][j-1] + cur, \lceil dp[i-1][j] \rceil + cur)
$$

---

因为浮点数精度问题， 所以我这里改了下

设`dp[i][j]`表示前i个道路 跳过j个 “行进” 距离，   

转移就是
$$
cur = dist[i];
\\
dp[i][j] = max(dp[i-1][j-1] + cur, \lceil \frac{dp[i-1][j]}{speed} \rceil \times speed + cur)
$$


```cpp
class Solution {
public:
    typedef long long int LL;
    LL dp[1001][1001];  // 前i个道路 跳过j个的 “行进”距离， 
    int minSkips(vector<int>& dist, int speed, int hoursBefore) {
        int n = dist.size();
        
        for(int i=0;i<=n;i++) for(int j=0;j<=n;j++) dp[i][j] = 1e14;

        dp[0][0] = 0;
        for(int i=1; i<=n; i++) {
            LL cur = dist[i-1];

            dp[i][0] = (dp[i-1][0]+speed-1)/speed*speed + cur;

            for(int j = 1; j<i; j++) {
                dp[i][j] = min(dp[i][j],
                               min(dp[i-1][j-1] + cur,
                                   (dp[i-1][j]+speed-1)/speed*speed + cur));
            }
        }
        
        int ans = -1;
        for(int i=0;i<=n;i++) {
            if(dp[n][i] <= 1LL * hoursBefore * speed) {
                ans = i;
                break;
            }
        }
        return ans;
    }
};
```

