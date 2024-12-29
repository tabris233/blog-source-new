---
title: "LeetCode 第430场周赛"
date: 2024-12-29 11:45:16
updated: 2024-12-29 11:45:16
description: ["LeetCode 第430场周赛。复健中。。 | LeetCode Weekly Contest 430. "]
toc: true
author: tabris
img: https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/202412291141015.png
katex: true
summary:
categories:
    - leetcode
tags:
    - 周赛
    - 双周赛

---

> As my job requires it, I am studing English now. Therefore, I will write the solutions to these problems in English.

[第 430 场周赛](https://leetcode.cn/contest/weekly-contest-430)

![](https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/202412291141015.png)

![image-20241229121302469](/Users/dongquan/Library/Application Support/typora-user-images/image-20241229121302469.png)

虽然排名最后跌到 86 了，但还是复健比较成功的一场。

# `3 分` - [使每一列严格递增的最少操作次数](https://leetcode.cn/problems/minimum-operations-to-make-columns-strictly-increasing/)

如果 `grid[i][j]` 比 `grid[i-1][j]` 小，就要变成 `grid[i-1][j]+1` 才能满足严格递增。

从上到下遍历，同时记录变更的 diff 就行。

```c++
class Solution {
public:
    int minimumOperations(vector<vector<int>>& grid) {
        int n = grid.size();
        int m = grid[0].size();


        int ans = 0;
        for(int i=1; i<n; i++) {
            for(int j=0; j<m; j++) {
                if(grid[i][j] <= grid[i-1][j]) {
                    ans += grid[i-1][j]+1-grid[i][j];
                    grid[i][j] = grid[i-1][j]+1;
                }
            }
        }

        return ans;
    }
};
```

# `4 分` - [从盒子中找出字典序最大的字符串](https://leetcode.cn/problems/find-the-lexicographically-largest-string-from-the-box-i/)

将字符串分成 `numFriends` ，子串最长为 `word.length() - numFriends +1`、最短为 1。

如果前缀相同，长的字符串字典序大，所以尽可能让子串最长。

考虑到 `word` 长度只有 1000，不用上 Suffix Array 啥的，直接暴力 $O(n^2)$ 处理既可。

遍历 `word` ，截取每个位置开头长度最长为 `word.length() - numFriends +1` 的子串，排序后取最大的。

注意 `numFriends == ` 的情况，直接返回 `word` 既可。

```c++
class Solution {
public:
    string answerString(string word, int numFriends) {
        if(1 == numFriends) return word;
        int len = word.length();
        int mxLen = len - numFriends+1;

        vector<string> vec;
        for(int i=0; i<len; i++) {
            vec.push_back(word.substr(i, mxLen));
        }

        sort(vec.begin(), vec.end());

        return vec.back();
    }
};
```

# `5 分` - [统计特殊子序列的数目](https://leetcode.cn/problems/count-special-subsequences/)

`nums[p] * nums[r] == nums[q] * nums[s]` 可以变形为 `nums[p] / nums[q] == nums[s] / nums[r]`

这样可以把等号两边的数字分在左右两边。枚举 `q` 时，`p` 在 `q` 的前面， `s` 和`r` 在 `q` 的后面。

我们预处理下所有 `nums[s] / nums[r]` 的个数用个 map 存下来既可，枚举 `q` 时把落在 `q+1` 范围内的值删掉，然后计算贡献既可。

⚠️： `nums[s] / nums[r]` 可能无法整除，我们把这两个数字约分后错位加在一起就行： `nums[s]/x*1000 + nums[r]/x` 




```c++
using LL = long long;

class Solution {
public:
    long long numberOfSubsequences(vector<int>& a) {
        int n = a.size();
        int g;
        
        unordered_map<int, int> mr;
        for(int i=n-1; i>0; i--) { // 不算 0 
            for(int j=i+2; j<n; j++) {
                g = __gcd(a[i], a[j]);
                // printf("%d %d => %d %d | %d %d\n", a[i], a[j], a[i]/g, a[j]/g, i, j);
                mr[a[j]/g*1000 + a[i]/g] ++;
            }
        }

        // for(auto [k, v]: mr) printf("-> %d %d\n", k, v); puts("");

        LL ans = 0;
        for(int i=0; i<n; i++) {
            for(int j=i+3; j<n; j++) {
                g = __gcd(a[i+1], a[j]);
                // printf("> %d %d %d %d/%d\n", i, i+1, j);
                mr[a[j]/g*1000 + a[i+1]/g]--;
            }
            // for(auto [k, v]: mr) printf("%d %d | ", k, v); puts("");
            for(int j=0; j<i-1; j++) {
                g = __gcd(a[i], a[j]);
                // printf("+ %d %d %2d %d\n", i, j, a[i]*a[j], mr[a[j]/g*1000 + a[i]/g]);
                ans += mr[a[j]/g*1000 + a[i]/g];
            }
        }

        return ans;
    }
};
```





# `6 分` - [统计恰好有 K 个相等相邻元素的数组数目](https://leetcode.cn/problems/count-the-number-of-arrays-with-k-matching-adjacent-elements/)

考虑 `k==0`，则长为 `n` 的数组，值域为 `[1, m]` 的方案数是 $ m * (m-1) ^ {n-1}$.

考虑 `k`：

- `k=1`, 如果 `i` 这个位置是  `arr[i - 1] == arr[i]`  的
  - 那么  `arr[i - 2] !=  arr[i - 1]` ， `arr[i]  != arr[i+1]`。
- `k=2`, 如果 `i`和 `i+1` 这两个位置是  `arr[i - 1] == arr[i]`  的
  - 那么  `arr[i - 2] !=  arr[i - 1]`  ， `arr[i]  == arr[i+1]`，`arr[i+1]  != arr[i+2]`。

容易看出`k=x`时，相当于是在构造长度为 `n-x` 的数组，`k=0` 时值域为 `[1, m]` 的数组，方案数是 $ m * (m-1) ^ {n-x-1}$.

 `k` 只能放到 `[2, n]` 上，所以 `k` 的位置组合就是 `C(n-1, k)`.

最终答案就是 $ C(n-1, k) \times m \times m^{n-k-1}$



```c++
using LL = long long;
const LL MOD = 1e9 + 7;
const int N = 2e5+7;

LL Fac[N], Inv[N];

LL qmod(LL a, LL b) {
    LL res = 1ll;
    for (; b; b >>= 1, a = a * a % MOD)
        if (b & 1) res = res * a % MOD;
    return res;
}
void init(int n) {
    // 方法一 费马小定理
    Fac[0] = 1;
    for (LL i = 1; i < n; i++) Fac[i] = Fac[i - 1] * i % MOD;
    for (LL i = 1; i < n; i++) Inv[i] = qmod(Fac[i], MOD - 2);
}

LL C(int n, int m) {
    if (m >= n) return 1;
    return Fac[n] * Inv[m] % MOD * Inv[n - m] % MOD;
}


class Solution {
public:
    int countGoodArrays(int n, int m, int k) {
        init(n+1);
        // printf("-- %d\n",C(1, 1));
        LL ans = (0==k)?1:C(n-1, k);
        // printf("%d\n", ans);
        ans = (ans * m) %MOD;
        // printf("%d\n", ans);
        ans = (ans * qmod(m-1, n-k-1)) % MOD;
        // printf("%d\n", ans);

        return ans;
    }
};
```

