---
title: "LeetCode 第371场周赛"
date: 2023-11-24 19:01:08
updated: 2023-11-24 19:01:08
description: ["LeetCode 第371场周赛。复健中。。 | LeetCode Weekly Contest 371. "]
toc: true
author: tabris
img: https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/202311241902364.png
katex: true
summary:
categories:
    - leetcode
tags:
    - 周赛
    - 双周赛

---

> Because work requested, I have to study English now. So I will write the problem solutions with English.

![image-202311241902364](https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/202311241902364.png)



# [2932. Maximum Strong Pair XOR I](https://leetcode.com/problems/maximum-strong-pair-xor-i/) | [2932. 找出强数对的最大异或值 I](https://leetcode.cn/problems/maximum-strong-pair-xor-i/)

Same with 2935

# [2933. High-Access Employees](https://leetcode.com/problems/high-access-employees/) | [2933. 高访问员工](https://leetcode.cn/problems/high-access-employees/)

Firstly, we can create a `map<{employee}, {access times array}>` to distinguish access times of each employee.

For each employee, we sort their access times array from little to big. And then iterate over this array, when there is $ access\_times_{i} - access\_times_{i-1} < 60 $, this employee is a High-Access Employee.


```cpp
class Solution {
public:
    vector<string> findHighAccessEmployees(vector<vector<string>>& _a) {
        map<string, vector<int>> m;

        for(auto a: _a){
            string key = a[0], time = a[1];

            int t = ((time[0]-'0')*10+(time[1]-'0'))*60+((time[2]-'0')*10+(time[3]-'0'));

            m[key].push_back(t);
        }

        vector<string> ans;

        for(auto [k, ta]: m) {
            sort(ta.begin(), ta.end());

            bool flag = false;
            for(int i=2; i<ta.size(); i++) {
                if (ta[i] - ta[i-2] < 60) {
                    flag = true;
                    break;
                }
            }

            if (flag) ans.push_back(k);
        }

        sort(ans.begin(), ans.end());

        return ans;
    }
};
```

# [2934. Minimum Operations to Maximize Last Elements in Arrays](https://leetcode.com/problems/minimum-operations-to-maximize-last-elements-in-arrays/) | [2934. 最大化数组末位元素的最少操作次数](https://leetcode.cn/problems/minimum-operations-to-maximize-last-elements-in-arrays/)

Follow the mean of this problem, care detail is ok.

```cpp
class Solution {
public:
    int solve(vector<int> a, vector<int> b) {
        int n = a.size();
        int cnt = 0;
        for(int i=0; i<n-1; i++) {
            if (a[i] > a[n-1]) {
                ++cnt;
                swap(a[i], b[i]);
            }
        }
        for(int i=0; i<n-1; i++) {
            if (b[i] > b[n-1]) {
                ++cnt;
                swap(a[i], b[i]);
            }
        }
        for(int i=0; i<n-1; i++) {
            if (a[i] > a[n-1]) return -1;
            if (b[i] > b[n-1]) return -1;
        }
        return cnt;
    }

    int minOperations(vector<int>& a, vector<int>& b) {
        int n = a.size();
        int ans1 = solve(a, b);

        swap(a[n-1], b[n-1]);
        int ans2 = solve(a, b);
        if (ans2 != -1) ++ans2;

        if (ans1 == -1 && ans2 == -1) return -1;
        if (ans1 == -1) ans1 = n+1;
        if (ans2 == -1) ans2 = n+1;

        return min(ans1, ans2);
    }
};
```

# [2935. Maximum Strong Pair XOR II](https://leetcode.com/problems/maximum-strong-pair-xor-ii/) | [2935. 找出强数对的最大异或值 II](https://leetcode.cn/problems/maximum-strong-pair-xor-ii/)

> 01 trie + two points

Firstly, we need to find all Strong Pairs.

For a Strong Pairs: (x, y). if $ x < y $, we can transfer $ |x-y| <= min(x, y) $ to $ x \times 2 >= y $.

When we fix the little number is $ nums_{i} $, the other number must in $ [ nums_{i}, nums_{i} \times 2] $, When we sort `nums` from little to big, the other numbers must consequent and start $ nums_{i} $.

Consider $ nums,length() <= 5 \times 10^4 $, two layer for must TLE.

We can use `two points` , fix a number to find the period of the other number. 

But answer is maximum `XOR` of all Strong Pairs. We can use `01-trie`. When we fix a number, add all other numbers to `01-trie`, so that we can find maximum `XOR` about the fix number quickly. We loop `nums` array, fix each $ nums_{i} $. We can get the final answer. 

Note: When we visit $ nums_{i} $, we need to delete $ nums_{0}, nums_{1}, ... , nums{i-1} $ from `01-trie`.

----

首先，我们要找到所有的 Strong Pair

对于一个 Strong Pair: (x, y)。当 x < y 时，我们可以把 $ |x - y| <= min(x, y) $ 转换成 $ x*2 >= y$。

当固定小的那个数字为 $ nums_{i} $ 时，另一个数字的大小一定在 $ [ nums_{i}, nums_{i} \times 2] $。对 `nums` 从小到大排序后，另一个数字一定是从 $ nums_{i} $ 开始连续的数。

考虑到 $ nums.length() <= 5*10^4 $, 两层 for 一定会超时。

我们可以使用`双指针`的技巧在固定一个数字时找到另一个数字的区间。而我们的结果是求 Strong Pair 的 `XOR`，我们可以使用 `01-trie` ，固定一个数字 x 时，把另外的数字都加到 `01-trie` 中，来快速地找到与 x `XOR` 最大的结果是多少。这样遍历一遍 nums 既可，取最大的既可。

注意当遍历到 $ nums_{i} $ 时，需要吧 $ nums_{0}, nums{1}, ... , nums_{i-1} $ 从 `01-trie` 中删掉。

```cpp
const int N = 5e4+7;

class Solution {
    int trie[N*20][2], val[N*20], cnt;
    void add(int x, int v) {
        int now = 0, bt;
        for(int i=19; i>=0; i--) {
            bt = (x&(1<<i))?1:0;
            if(!trie[now][bt]) trie[now][bt] = ++cnt;
            now=trie[now][bt];
            val[now] += v;
        }
    }
    int ask(int x) {
        int now = 0, bt, ret = 0;
        for(int i=19; i>=0; i--) {
            bt = (x&(1<<i))?1:0;
            if(trie[now][1-bt] && val[trie[now][1-bt]] > 0) {
                bt = 1-bt;
                ret |= 1<<i;
            }
            now=trie[now][bt];
        }
        return ret;
    }
    
public:
    int maximumStrongPairXor(vector<int>& a) {
        cnt = 0;
        memset(trie,0,sizeof(trie));
        memset(val,0,sizeof(val));

        int ans = 0;
        
        sort(a.begin(), a.end());
        
        int n = a.size();

        for(int l=0, r=0; l<n; l++) {
            for(; r<n && a[r]<=a[l]+a[l]; r++) add(a[r], 1);
            ans = max(ans, ask(a[l]));
            add(a[l], -1);
        }
        
        return ans;
    }
};
```

