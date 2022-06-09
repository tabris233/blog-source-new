---
title: "LeetCode 第244场周赛"
date: 2021-06-06 12:47:24
description: ["LeetCode 第244场周赛。。 再再再再再再再再次翻车"]
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

>    再再再再再再再再次翻车了
>
>    第三个没仔细想+超时挂了3次
>
>    第四个最大值没搞好+瞎提交WA了3次。。。
>
>    思维又慢，码得又慢，没救了。

![image-20210606122647168](https://fastly.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/06/06/20210606122647.png)

# [5776. 判断矩阵经轮转后是否一致](https://leetcode-cn.com/contest/weekly-contest-244/problems/determine-whether-matrix-can-be-obtained-by-rotation/)

模拟一下， 旋转可以多次，但状态最多就4种：

​	 不旋转，旋转90度，旋转180度，旋转270度。

这里循环一下， 每次旋转90度，在比较是否相等就好了。

```cpp
class Solution {
public:
    bool findRotation(vector<vector<int>>& mat, vector<vector<int>>& target) {
        int n = mat.size();
        vector<vector<int>> temp(n, vector<int>(n));
        
        // puts("+++++++++++++++++=");
        
        for(int k=0;k<4;k++) {
            if(mat == target) return true;
            for(int i=0;i<n/2;i++){
                for(int j=i;j<n-1-i;j++) {
                    temp[i][j] = mat[n-1-j][i];
                    temp[j][n-1-i] = mat[i][j];
                    temp[n-1-i][n-1-j] = mat[j][n-1-i];
                    temp[n-1-j][i] = mat[n-1-i][n-1-j];
                }
            }
            if(n&1) temp[n/2][n/2] = mat[n/2][n/2];

            mat = temp;
            // for(int i=0;i<n;i++) {
            //     for(int j=0;j<n;j++) printf("%d ",mat[i][j]);
            //     puts("");
            // }
            // puts("===");
        }
        return false;
    }
};
```



# [5777. 使数组元素相等的减少操作次数](https://leetcode-cn.com/contest/weekly-contest-244/problems/reduction-operations-to-make-the-array-elements-equal/)

如果存在两个数，a<b ， 则b一定会变到a的转态。所以统计会变成当前数字的数字个数就好。

处理的话，因为有重复值，所以放到一个桶里面。从小到大遍历上去统计就好。

```cpp
class Solution {
public:
    int sum[50005];
    int reductionOperations(vector<int>& nums) {
        int tot = nums.size();
        int mx = nums[0];

        memset(sum, 0, sizeof(sum));
        
        for(auto a: nums) sum[a]++, mx = max(mx, a);
        
        int ans = 0;
        
        for(int i=1;i<=mx;i++) {
            if(sum[i] != 0) {
                ans += tot - sum[i] - sum[i-1];
            }
            sum[i]+=sum[i-1];
        }
        
        return ans;
    }
};
```



# [5778. 使二进制字符串字符交替的最少反转次数](https://leetcode-cn.com/contest/weekly-contest-244/problems/minimum-number-of-flips-to-make-the-binary-string-alternating/)

>   这里写的有点丑，代码没法看，，

分情况讨论：

1.  **偶数长度**的字符串：它要么变成`0101010`这种，要么变成`101010`这种，第一个操作对他无影响。
2.  **奇数长度**的字符串：第一个操作对他有影响，`10101`是可以变成`10110`的。所以这种情况下，可以分成两段，前一段1开头，后一段0开头。或者前一段0开头，后一段1开头。

注意奇数的情况下处理时可以线性。

```cpp
class Solution {
public:
    int f(string s, bool flag, bool flag2 = false) {
        int l = s.size();
        
        string s1, s2;
        for(int i=0;i<l;i++) {
            s1 += (i&1^flag)?"0":"1";
            s2 += (i&1^flag)?"1":"0";
        }
        // printf("%s %s\n", s1.c_str(), s2.c_str());
        int sum = 0;
        for(int i=0;i<l;i++) {
            if(s[i] != s1[i]) ++sum;
        }
        if(flag2) return sum;
        
        int ans = sum;
        int cnt1 = 0, cnt2 = 0;
        for(int i=0;i<l;i++){
            if(s[i] != s1[i]) ++cnt1;
            if(s[i] != s2[i]) ++cnt2;
            ans = min(ans, sum - cnt1 + cnt2);
        }
        return ans;
    }    

    int minFlips(string s) {
        int l = s.size();
        if(l&1) return min(f(s, true), f(s, false));
        else {
            return min(f(s, true, true), f(s, false, true));
        }
    }
};
```



# [5779. 装包裹的最小浪费空间](https://leetcode-cn.com/contest/weekly-contest-244/problems/minimum-space-wasted-from-packaging/)

考虑数据范围，只可以遍历所有商家的所有盒子。

针对不同商家统计每个盒子的空余空间就好。一个盒子，利用两个前缀和统计需要使用它的货物个数，以及需要使用它货物占用大小总数，最后减一下就好了。



```cpp
class Solution {
public:
    static const int MOD = 1e9+7;
    static const int N = 1e5+7;
    
    typedef long long int LL;

    LL sum[N];
    LL tot[N];
    int minWastedSpace(vector<int>& a, vector<vector<int>>& b) {

        memset(sum, 0, sizeof(sum));
        memset(tot, 0, sizeof(tot));
        int mx = 0;
        for(auto x: a) sum[x]++, tot[x]+=x, mx = max(mx, x);
        for(int i=1;i<=100000;i++) {
            sum[i] += sum[i-1];
            tot[i] += tot[i-1];
        }

        // puts("------------------");
        // for(int i=0;i<=mx;i++) printf("%lld ", sum[i]); puts("");
        // for(int i=0;i<=mx;i++) printf("%lld ", tot[i]); puts("");
        
        LL ans = 1000000000000LL;
        bool flag = false;
        for(auto t: b) {
            // puts(">>>");

            sort(t.begin(), t.end());
            if(t[t.size()-1] >= mx) flag = true;
            else continue;
            
            int p = 0;
            LL tmp = 0;
            for(auto bb: t){
                tmp += (sum[bb]-sum[p]) * bb - (tot[bb] - tot[p]);
                // printf("%lld (%d, %d): %lld(%lld-%lld) * %lld - %lld\n", tmp, bb, p, (sum[bb]-sum[p]), sum[bb], sum[p], bb, (tot[bb] - tot[p]));
                p = bb;
            }
            ans = min(ans, tmp);
        }
        
        if(!flag) return -1;

        return ans%MOD;
    }
};
```
