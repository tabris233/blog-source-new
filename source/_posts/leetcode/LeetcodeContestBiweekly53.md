---
title: "LeetCode 第53场双周赛"
date: 2021-05-30 01:02:27
Updated: 2021-05-30 01:02:27
description: ["LeetCode 第53场双周赛。。 菜死了啊"]
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

![image-20210530010253910](https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/05/30/20210530010253.png)

GG

# 5754. 长度为三且各字符不同的子字符串

简单模拟。

```cpp
class Solution {
public:
    int countGoodSubstrings(string s) {
        int ans =0 ;
        for(int i=2;i<s.size(); i++) {
            if(s[i] != s[i-1] && s[i] != s[i-2] && s[i-1] != s[i-2]){
                ans ++;
            }
        }
        return ans;
    }
};
```



# 5755. 数组中最大数对和的最小值

排序之后最大的和最小的配对，次大的和次小的配对，以此类推。

```cpp
class Solution {
public:
    int minPairSum(vector<int>& nums) {
        sort(nums.begin(), nums.end());
        
        int ans = 0;
        for(int i=0;i<nums.size()/2;i++) {
            ans = max(ans, nums[i] + nums[nums.size()-1-i]);
        }
        return ans;
    }
};
```



# 5757. 矩阵中最大的三个菱形和

模拟题

单开个数组记录下`grid[i][j]` 开始*右*下长度为$k$的和 与 `grid[i][j]` 开始*左*下长度为$k$的和就好了。 

结果就枚举在每个`[i][j]`位置美剧边长为$k$的菱形的和。

最后去重后取最大的$3$个就好了。复杂度$O(N^3)$

```cpp
class Solution {
public:
    
    int sum[2][101][101][101];     // sum[0][i][j][k] grid[i][j] 开始*右*下长度为k的和
                                   // sum[1][i][j][k] grid[i][j] 开始*左*下长度为k的和 
    
    vector<int> getBiggestThree(vector<vector<int>>& grid) {
        int n = grid.size(), m = grid[0].size();
        
        for(int i=0;i<n;i++) {
            for(int j=0;j<m;j++) {
                sum[0][i][j][0] = grid[i][j];
                for(int k=1; i+k<n && j+k<m; k++) {
                    sum[0][i][j][k] = sum[0][i][j][k-1] + grid[i+k][j+k];
                }
                sum[1][i][j][0] = grid[i][j];
                for(int k=1; i+k<n && j-k>=0; k++) {
                    sum[1][i][j][k] = sum[1][i][j][k-1] + grid[i+k][j-k];
                }
                
            }
        }

        priority_queue <int,vector<int>,greater<int> > q;
        set<int> s;
        for(int i=0;i<n;i++) {
            for(int j=0;j<m;j++) {
                if(!s.count(grid[i][j])) 
                    q.push(grid[i][j]);
                s.insert(grid[i][j]);
                for(int k=1;i+k+k<n && j-k>=0 && j+k<m; k++) {
                    int tmp = sum[0][i][j][k] + sum[1][i][j][k] + sum[0][i+k][j-k][k] + sum[1][i+k][j+k][k];
                    tmp -= grid[i][j] + grid[i+k][j-k] + grid[i+k][j+k] + grid[i+k+k][j];
                    if(!s.count(tmp)) q.push(tmp);
                    s.insert(tmp);
                }
                while(q.size() > 3) q.pop();
            }
        }
        
        vector<int> ans;
        while(!q.empty()) {ans.push_back(q.top()); q.pop();}
        for(int i=0;i<ans.size()/2;i++) swap(ans[0], ans[ans.size()-1-i]);
        return ans;
    }
};
```



# 5756. 两个数组最小的异或值之和

这题看了数据范围，$n=14$就开始想有啥暴力的解没有。。。

最后是转化了下求了个二分图最大权匹配。。。

a+b = a ^ b + ((a&b)<<1) => a^b = a+b -((a&b)<<1)

a+b 的值 恒定，所以只要找边权为与值的 nums1 和 nums2 二分图最大权匹配就好。 搞个板子就怼上去了。



不过这题还是可以把$n=14$利用上的。赛后看了大佬的做法，可以dp

i表示 nums2数组被使用的情况，

dp[i] 表示 nums2中对应的那些元素排序后与nums1中前几个元素异或值的和的最小值

转移为

dp[i | (1<<j)] = min(dp[i|(1<<j)], dp[i] + (nums1[i的二进制中1的个数] ^ nums2[j]))；  

---

二分图做法


```cpp
template <typename T>
struct hungarian {  // km
  int n;
  vector<int> matchx;
  vector<int> matchy;
  vector<int> pre;
  vector<bool> visx;
  vector<bool> visy;
  vector<T> lx;
  vector<T> ly;
  vector<vector<T> > g;
  vector<T> slack;
  T inf;
  T res;
  queue<int> q;
  int org_n;
  int org_m;

  hungarian(int _n, int _m) {
    org_n = _n;
    org_m = _m;
    n = max(_n, _m);
    inf = numeric_limits<T>::max();
    res = 0;
    g = vector<vector<T> >(n, vector<T>(n));
    matchx = vector<int>(n, -1);
    matchy = vector<int>(n, -1);
    pre = vector<int>(n);
    visx = vector<bool>(n);
    visy = vector<bool>(n);
    lx = vector<T>(n, -inf);
    ly = vector<T>(n);
    slack = vector<T>(n);
  }

  void addEdge(int u, int v, int w) {
    g[u][v] = max(w, 0);  // 负值还不如不匹配 因此设为0不影响
  }

  bool check(int v) {
    visy[v] = true;
    if (matchy[v] != -1) {
      q.push(matchy[v]);
      visx[matchy[v]] = true;
      return false;
    }
    while (v != -1) {
      matchy[v] = pre[v];
      swap(v, matchx[pre[v]]);
    }
    return true;
  }

  void bfs(int i) {
    while (!q.empty()) {
      q.pop();
    }
    q.push(i);
    visx[i] = true;
    while (true) {
      while (!q.empty()) {
        int u = q.front();
        q.pop();
        for (int v = 0; v < n; v++) {
          if (!visy[v]) {
            T delta = lx[u] + ly[v] - g[u][v];
            if (slack[v] >= delta) {
              pre[v] = u;
              if (delta) {
                slack[v] = delta;
              } else if (check(v)) {
                return;
              }
            }
          }
        }
      }
      // 没有增广路 修改顶标
      T a = inf;
      for (int j = 0; j < n; j++) {
        if (!visy[j]) {
          a = min(a, slack[j]);
        }
      }
      for (int j = 0; j < n; j++) {
        if (visx[j]) {  // S
          lx[j] -= a;
        }
        if (visy[j]) {  // T
          ly[j] += a;
        } else {  // T'
          slack[j] -= a;
        }
      }
      for (int j = 0; j < n; j++) {
        if (!visy[j] && slack[j] == 0 && check(j)) {
          return;
        }
      }
    }
  }

  void solve() {
    // 初始顶标
    for (int i = 0; i < n; i++) {
      for (int j = 0; j < n; j++) {
        lx[i] = max(lx[i], g[i][j]);
      }
    }

    for (int i = 0; i < n; i++) {
      fill(slack.begin(), slack.end(), inf);
      fill(visx.begin(), visx.end(), false);
      fill(visy.begin(), visy.end(), false);
      bfs(i);
    }

    // custom
    for (int i = 0; i < n; i++) {
      if (g[i][matchx[i]] > 0) {
        res += g[i][matchx[i]];
      } else {
        matchx[i] = -1;
      }
    }
    cout << res << "\n";
    for (int i = 0; i < org_n; i++) {
      cout << matchx[i] + 1 << " ";
    }
    cout << "\n";
  }
};

class Solution {
public:
    int x[15][15];
    int vis[2][15];
    
    int minimumXORSum(vector<int>& nums1, vector<int>& nums2) {
        int n = nums1.size();
        hungarian<int> solver(n, n);
        
        for(int i=0;i<n;i++) for(int j=0;j<n;j++) {
            solver.addEdge(i, j, nums1[i] & nums2[j]);
        }
        solver.solve();
        int ans = 0;
        for(int i=0;i<n;i++) ans+=nums1[i] + nums2[i];
        ans -= solver.res*2;
        return ans;
    }
};
```

dp做法

```cpp
class Solution {
public:
    #define lowbit(x) ((x)&(-x))
    int minimumXORSum(vector<int>& nums1, vector<int>& nums2) {
        int n = nums1.size();

        vector<int> dp(1<<n, 1e9);

        dp[0] = 0;
        for(int i=0, tmp, idx; i<(1<<n); i++) {
            tmp = i, idx = 0;
            for(;tmp;tmp-=lowbit(tmp)) ++idx;
            
            for(int j=0;j<n;j++) {
                if(i&(1<<j)) continue;
                dp[i|(1<<j)] = min(dp[i|(1<<j)], dp[i] + (nums1[idx] ^ nums2[j]));
            }
        }
        
        return dp[(1<<n) - 1];
    }
};
```