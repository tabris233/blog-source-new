---
title: "LeetCode 第426场周赛"
date: 2024-12-01 11:24:16
updated: 2024-12-01 11:24:16
description: ["LeetCode 第426场周赛。复健中。。 | LeetCode Weekly Contest 426. "]
toc: true
author: tabris
img: https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/202412011126208.png
katex: true
summary:
categories:
    - leetcode
tags:
    - 周赛
    - 双周赛

---

> As my job requires it, I am studing English now. Therefore, I will write the solutions to these problems in English.

[第 426 场周赛](https://leetcode.cn/contest/weekly-contest-426)

![image-20241201112558769](https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/202412011126208.png)



# [3370. Smallest Number With All Set BitsEasy](https://leetcode.com/problems/smallest-number-with-all-set-bits) | [100501. 仅含置位位的最小整数](https://leetcode.cn/contest/weekly-contest-426/problems/smallest-number-with-all-set-bits/)

enumerate $ 2^i-1 $

```c++
class Solution {
public:
    int smallestNumber(int n) {
        for(int i=0; i<20; i++) {
            if((1<<i)-1 >= n) return (1<<i)-1;
        }
        return -1;
    }
};
```

# [3371. Identify the Largest Outlier in an ArrayMed.](https://leetcode.com/problems/identify-the-largest-outlier-in-an-array) | [100444. 识别数组中的最大异常值](https://leetcode.cn/contest/weekly-contest-426/problems/identify-the-largest-outlier-in-an-array/)

Traverse the array `nums`  in descending order and evaluta each element as a potenial answers. if $ \frac{sum(nums) - nums[i]}{2} $ exist in the array `nums`(excluding nums[i]), then nums[i] is the answer.

We need a map to count the number of nums[i]. when we visit nums[i], we should delete nums[i] in the map.

note: if $ sum(nums) - nums[i] $ is odd, we should skip this case.

```c++
class Solution {
public:
    int getLargestOutlier(vector<int>& nums) {
        unordered_map<int, int> m;
        sort(nums.begin(), nums.end(), greater<int>());

        int sum = 0;
        for(auto &a: nums) {
            m[a]++;
            sum += a;
        }

        //printf("sum=%d\n", sum);
        for(auto &a: nums) {
            int t = sum-a;
            //printf("%d %d %d\n", a, t, (t%2+2)%2);
            if((t%2+2)%2 == 1) continue;

            m[a]--;

            if(m[t/2] > 0) return a;

            m[a]++;
        }

        return -1;
    }
};
```



# [3372. Maximize the Number of Target Nodes After Connecting Trees IMed.](https://leetcode.com/problems/maximize-the-number-of-target-nodes-after-connecting-trees-i) | [100475. 连接两棵树后最大目标节点数目 I](https://leetcode.cn/contest/weekly-contest-426/problems/maximize-the-number-of-target-nodes-after-connecting-trees-i/)

Given $ n,m \leq 1000 $, we can use DFS to compute the number of nodes whose distance  from node `i`is less than or equal to $ k $.

For any node in Tree1, we can connect it to a node in Tree2. For nodes in Tree2, we can find the one node with the maximum count of nodes `is less than or equal to $ k-1 $.

Reason for $ k-1 $, the node in Tree1 and the node in Tree2 requrie one additional edge to be connected.

```c++
class Solution {
    int dfs(int u, int fa, int limit, vector<vector<int>> &g) {
        if(limit < 0) return 0;
        int sum = 1;
        for(auto to: g[u]) {
            if(to == fa) continue;
            sum += dfs(to, u, limit-1, g);
        }
        return sum;
    };

public:
    vector<int> maxTargetNodes(vector<vector<int>>& edges1, vector<vector<int>>& edges2, int k) {
        int n = edges1.size()+1;
        int m = edges2.size()+1;

        vector<vector<int>> g1(n), g2(m);
        for(auto &e: edges1) {
            int u = e[0], v = e[1];
            g1[u].push_back(v);
            g1[v].push_back(u);
        }

        for(auto &e: edges2) {
            int u = e[0], v = e[1];
            g2[u].push_back(v);
            g2[v].push_back(u);
        }

        int mx = 0;
        for(int i=0; i<m; i++) {
            mx = max(mx, dfs(i, -1, k-1, g2));
            // printf("i(%d), mx = %d\n", i, mx);
        }

        vector<int> ans(n, 0);
        for(int i=0; i<n; i++) {
            ans[i] = mx + dfs(i, -1, k, g1);
        }

        return ans;
    }
};
```



# [3373. Maximize the Number of Target Nodes After Connecting Trees IIHard](https://leetcode.com/problems/maximize-the-number-of-target-nodes-after-connecting-trees-ii) | [100485. 连接两棵树后最大目标节点数目 II](https://leetcode.cn/contest/weekly-contest-426/problems/maximize-the-number-of-target-nodes-after-connecting-trees-ii/)

In a list, if the distance between two index is even, both indexs either be odd or even.

In a tree, if the distance between two nodes is even, both nodes must beling to the same layer(odd or even).

For Tree1, we count the number of nodes in odd and even layers using DFS.

When connecting Tree1 to Tree2:

- If we connect to a node in an **odd** layer, the result will include nodes in the **even** layer.
- If we connect to a node in an **even** layer, the result will include nodes in the **odd** layer.

```c++
class Solution {
    void dfs(int u, int fa, int flag, vector<vector<int>> &g, vector<vector<int>> &t) {
        t[flag].push_back(u);
        for(auto to: g[u]) {
            if(to == fa) continue;
            dfs(to, u, 1-flag, g, t);
        }
        return ;
    };

public:
    vector<int> maxTargetNodes(vector<vector<int>>& edges1, vector<vector<int>>& edges2) {
        int n = edges1.size()+1;
        int m = edges2.size()+1;

        vector<vector<int>> g1(n), g2(m);
        for(auto &e: edges1) {
            int u = e[0], v = e[1];
            g1[u].push_back(v);
            g1[v].push_back(u);
        }

        for(auto &e: edges2) {
            int u = e[0], v = e[1];
            g2[u].push_back(v);
            g2[v].push_back(u);
        }

        vector<vector<int>> t1(2), t2(2);

        dfs(0, -1, 0, g1, t1);
        dfs(0, -1, 0, g2, t2);

        int mx = max(t2[0].size(), t2[1].size());

        vector<int> ans(n, mx);
        for(auto arr: t1) {
            for(auto a: arr) {
                ans[a] += arr.size();
            }
        }

        return ans;
    }
};
```



